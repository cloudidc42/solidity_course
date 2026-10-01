# Part 59: Cross-Chain Bridges

## บทนำ

Cross-chain bridges เป็น infrastructure สำคัญที่เชื่อม blockchains ต่างๆ เข้าด้วยกัน บทนี้ครอบคลุม:

1. **Lock-and-Mint Bridge** - architecture พื้นฐาน: lock บน source chain, mint บน destination
2. **Optimistic Bridge** - 7-day fraud window พร้อม challenge mechanism
3. **Message Passing** - ส่ง arbitrary calldata ข้าม chain ผ่าน Chainlink CCIP
4. **Bridge Security** - replay attacks, signature verification, nonce, guardian multisig

---

## ทฤษฎี Cross-Chain Bridge Patterns

### 1. Lock-and-Mint (Wrapped Token)
```
Source Chain:       Destination Chain:
[Token] → Lock     → Mint [Wrapped Token]
         Unlock   ← Burn
```

### 2. Burn-and-Mint (Native Cross-chain)
```
Source Chain:       Destination Chain:
[Token] → Burn     → Mint [Token]
```

### 3. Liquidity Pool Based
```
Source Chain:       Destination Chain:
[Token] → Pool A   ← Pool B [Token]
          (rebalancing happens off-chain)
```

---

## 1. Lock-and-Mint Bridge

### Smart Contract: LockBox (Source Chain)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/utils/Pausable.sol";

/**
 * @title LockBox
 * @notice Source chain contract ที่ lock ERC-20 tokens สำหรับ cross-chain bridge
 * @dev Deploy บน Ethereum (source chain)
 *      Validators observe events แล้ว relay ไปยัง destination chain
 */
contract LockBox is AccessControl, ReentrancyGuard, Pausable {
    using SafeERC20 for IERC20;

    // ============ Roles ============
    bytes32 public constant VALIDATOR_ROLE = keccak256("VALIDATOR_ROLE");
    bytes32 public constant GUARDIAN_ROLE = keccak256("GUARDIAN_ROLE");

    // ============ Structs ============
    struct BridgeRequest {
        address sender;         // ผู้ส่ง
        address recipient;      // ผู้รับบน destination chain
        address token;          // token ที่ bridge
        uint256 amount;         // จำนวน
        uint256 destChainId;    // destination chain ID
        uint256 nonce;          // nonce ของ request
        uint256 timestamp;      // เวลาที่สร้าง
        BridgeStatus status;    // สถานะปัจจุบัน
    }

    enum BridgeStatus {
        PENDING,    // รอ validator relay
        COMPLETED,  // สำเร็จ
        REFUNDED    // คืน token แล้ว (ล้มเหลว)
    }

    // ============ Constants ============
    uint256 public constant MIN_BRIDGE_AMOUNT = 1e6;   // minimum bridge amount
    uint256 public constant MAX_BRIDGE_AMOUNT = 1e24;  // maximum per transaction
    uint256 public constant BRIDGE_FEE_BPS = 10;       // 0.1% fee
    uint256 public constant REFUND_DELAY = 7 days;     // เวลารอก่อน refund

    // ============ State ============
    mapping(bytes32 => BridgeRequest) public bridgeRequests;
    mapping(address => uint256) public userNonces;
    mapping(address => bool) public supportedTokens;
    mapping(uint256 => bool) public supportedChains;
    mapping(address => uint256) public collectedFees;
    mapping(bytes32 => bool) public processedRelays; // prevent replay on destination

    address public treasury;
    uint256 public totalLockedValue;

    // Daily limit per user
    mapping(address => mapping(uint256 => uint256)) public dailyBridgeVolume; // user → day → amount
    uint256 public dailyLimitPerUser = 10000e18; // 10,000 tokens per day

    // ============ Events ============
    event TokensLocked(
        bytes32 indexed requestId,
        address indexed sender,
        address indexed recipient,
        address token,
        uint256 amount,
        uint256 destChainId,
        uint256 nonce
    );
    event TokensUnlocked(
        bytes32 indexed requestId,
        address indexed recipient,
        address token,
        uint256 amount
    );
    event BridgeRequestRefunded(bytes32 indexed requestId);
    event TokenSupportUpdated(address indexed token, bool supported);
    event ChainSupportUpdated(uint256 chainId, bool supported);

    // ============ Constructor ============
    constructor(address _treasury) {
        require(_treasury != address(0), "LockBox: zero treasury");
        treasury = _treasury;
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(GUARDIAN_ROLE, msg.sender);
    }

    // ============ User Functions ============

    /**
     * @notice Lock tokens เพื่อ bridge ไปยัง destination chain
     * @param token Token ที่ต้องการ bridge
     * @param amount จำนวน token
     * @param recipient Address ผู้รับบน destination chain
     * @param destChainId Chain ID ของ destination
     */
    function lockTokens(
        address token,
        uint256 amount,
        address recipient,
        uint256 destChainId
    ) external nonReentrant whenNotPaused returns (bytes32 requestId) {
        require(supportedTokens[token], "LockBox: token not supported");
        require(supportedChains[destChainId], "LockBox: chain not supported");
        require(recipient != address(0), "LockBox: zero recipient");
        require(amount >= MIN_BRIDGE_AMOUNT, "LockBox: amount too small");
        require(amount <= MAX_BRIDGE_AMOUNT, "LockBox: amount too large");

        // ตรวจสอบ daily limit
        uint256 today = block.timestamp / 1 days;
        require(
            dailyBridgeVolume[msg.sender][today] + amount <= dailyLimitPerUser,
            "LockBox: daily limit exceeded"
        );
        dailyBridgeVolume[msg.sender][today] += amount;

        // คิด fee
        uint256 fee = amount * BRIDGE_FEE_BPS / 10000;
        uint256 netAmount = amount - fee;

        // สร้าง request ID
        uint256 nonce = userNonces[msg.sender]++;
        requestId = keccak256(abi.encodePacked(
            msg.sender,
            recipient,
            token,
            amount,
            destChainId,
            nonce,
            block.chainid
        ));

        require(bridgeRequests[requestId].timestamp == 0, "LockBox: duplicate request");

        bridgeRequests[requestId] = BridgeRequest({
            sender: msg.sender,
            recipient: recipient,
            token: token,
            amount: netAmount,
            destChainId: destChainId,
            nonce: nonce,
            timestamp: block.timestamp,
            status: BridgeStatus.PENDING
        });

        collectedFees[token] += fee;
        totalLockedValue += amount;

        // Lock tokens
        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);

        emit TokensLocked(requestId, msg.sender, recipient, token, netAmount, destChainId, nonce);
    }

    /**
     * @notice Unlock tokens เมื่อ bridge กลับมาจาก destination
     * @dev เรียกโดย validator หลังจาก burn บน destination chain
     * @param requestId Request ID จาก destination chain
     * @param recipient ผู้รับ tokens
     * @param token Token address
     * @param amount จำนวน
     * @param signatures Validator signatures (multisig)
     */
    function unlockTokens(
        bytes32 requestId,
        address recipient,
        address token,
        uint256 amount,
        bytes[] calldata signatures
    ) external nonReentrant whenNotPaused {
        require(!processedRelays[requestId], "LockBox: already processed");
        require(recipient != address(0), "LockBox: zero recipient");

        // ตรวจสอบ validator signatures (multisig threshold)
        _verifyValidatorSignatures(requestId, recipient, token, amount, signatures);

        processedRelays[requestId] = true;
        totalLockedValue -= amount;

        IERC20(token).safeTransfer(recipient, amount);

        emit TokensUnlocked(requestId, recipient, token, amount);
    }

    /**
     * @notice Refund tokens ถ้า bridge ล้มเหลว (หลัง 7 วัน)
     * @param requestId Request ID ที่ต้องการ refund
     */
    function refundFailedBridge(bytes32 requestId) external nonReentrant {
        BridgeRequest storage request = bridgeRequests[requestId];

        require(request.sender == msg.sender, "LockBox: not requester");
        require(request.status == BridgeStatus.PENDING, "LockBox: not pending");
        require(
            block.timestamp >= request.timestamp + REFUND_DELAY,
            "LockBox: refund delay not passed"
        );

        request.status = BridgeStatus.REFUNDED;
        totalLockedValue -= request.amount;

        IERC20(request.token).safeTransfer(request.sender, request.amount);

        emit BridgeRequestRefunded(requestId);
    }

    // ============ Internal Validation ============

    function _verifyValidatorSignatures(
        bytes32 requestId,
        address recipient,
        address token,
        uint256 amount,
        bytes[] calldata signatures
    ) internal view {
        uint256 validatorCount = getRoleMemberCount(VALIDATOR_ROLE);
        uint256 threshold = (validatorCount * 2 / 3) + 1; // 2/3 + 1 threshold

        require(signatures.length >= threshold, "LockBox: insufficient signatures");

        bytes32 messageHash = keccak256(abi.encodePacked(
            requestId,
            recipient,
            token,
            amount,
            block.chainid
        ));
        bytes32 ethSignedHash = keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", messageHash));

        address[] memory signers = new address[](signatures.length);
        uint256 validCount = 0;

        for (uint256 i = 0; i < signatures.length; i++) {
            address signer = _recoverSigner(ethSignedHash, signatures[i]);
            if (hasRole(VALIDATOR_ROLE, signer)) {
                // ตรวจสอบไม่มี duplicate signer
                bool isDuplicate = false;
                for (uint256 j = 0; j < validCount; j++) {
                    if (signers[j] == signer) {
                        isDuplicate = true;
                        break;
                    }
                }
                if (!isDuplicate) {
                    signers[validCount++] = signer;
                }
            }
        }

        require(validCount >= threshold, "LockBox: not enough valid signatures");
    }

    function _recoverSigner(
        bytes32 ethSignedHash,
        bytes calldata signature
    ) internal pure returns (address) {
        require(signature.length == 65, "LockBox: invalid signature length");

        bytes32 r;
        bytes32 s;
        uint8 v;

        assembly {
            r := calldataload(signature.offset)
            s := calldataload(add(signature.offset, 32))
            v := byte(0, calldataload(add(signature.offset, 64)))
        }

        return ecrecover(ethSignedHash, v, r, s);
    }

    // ============ Admin Functions ============

    function addSupportedToken(address token) external onlyRole(DEFAULT_ADMIN_ROLE) {
        supportedTokens[token] = true;
        emit TokenSupportUpdated(token, true);
    }

    function addSupportedChain(uint256 chainId) external onlyRole(DEFAULT_ADMIN_ROLE) {
        supportedChains[chainId] = true;
        emit ChainSupportUpdated(chainId, true);
    }

    function withdrawFees(address token) external onlyRole(DEFAULT_ADMIN_ROLE) {
        uint256 amount = collectedFees[token];
        require(amount > 0, "LockBox: no fees");
        collectedFees[token] = 0;
        IERC20(token).safeTransfer(treasury, amount);
    }

    function pause() external onlyRole(GUARDIAN_ROLE) { _pause(); }
    function unpause() external onlyRole(GUARDIAN_ROLE) { _unpause(); }

    function getRoleMemberCount(bytes32 role) public view returns (uint256) {
        return _getRoleMemberCount(role);
    }

    function _getRoleMemberCount(bytes32 role) internal view returns (uint256 count) {
        // Simplified: production จะใช้ EnumerableSet
        // นี่เป็น placeholder
        return 3; // assume 3 validators for demo
    }
}

/**
 * @title BridgeMint
 * @notice Destination chain contract ที่ mint wrapped tokens
 * @dev Deploy บน Polygon, BSC, Arbitrum, etc.
 */
contract BridgeMint is AccessControl, ReentrancyGuard, Pausable {
    bytes32 public constant VALIDATOR_ROLE = keccak256("VALIDATOR_ROLE");
    bytes32 public constant GUARDIAN_ROLE = keccak256("GUARDIAN_ROLE");

    // ============ State ============
    mapping(address => address) public wrappedTokens;     // original → wrapped
    mapping(address => address) public originalTokens;     // wrapped → original
    mapping(bytes32 => bool) public processedDeposits;    // prevent replay

    uint256 public constant VALIDATOR_THRESHOLD_BPS = 6667; // 66.67%

    // ============ Events ============
    event TokensMinted(
        bytes32 indexed depositId,
        address indexed recipient,
        address wrappedToken,
        uint256 amount,
        uint256 sourceChainId
    );
    event TokensBurned(
        bytes32 indexed withdrawId,
        address indexed sender,
        address wrappedToken,
        uint256 amount,
        uint256 destChainId,
        address destRecipient
    );

    constructor() {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(GUARDIAN_ROLE, msg.sender);
    }

    /**
     * @notice Mint wrapped tokens เมื่อ deposit ถูก lock บน source chain
     * @param depositId Request ID จาก source chain
     * @param recipient ผู้รับ wrapped tokens
     * @param originalToken Original token address (source chain)
     * @param wrappedToken Wrapped token address (this chain)
     * @param amount จำนวน
     * @param sourceChainId Chain ID ของ source
     * @param signatures Validator signatures
     */
    function mintWrappedTokens(
        bytes32 depositId,
        address recipient,
        address originalToken,
        address wrappedToken,
        uint256 amount,
        uint256 sourceChainId,
        bytes[] calldata signatures
    ) external nonReentrant whenNotPaused {
        require(!processedDeposits[depositId], "BridgeMint: already processed");
        require(wrappedTokens[originalToken] == wrappedToken, "BridgeMint: invalid wrapped token");

        _verifySignatures(depositId, recipient, wrappedToken, amount, sourceChainId, signatures);

        processedDeposits[depositId] = true;
        IBridgedToken(wrappedToken).bridgeMint(recipient, amount);

        emit TokensMinted(depositId, recipient, wrappedToken, amount, sourceChainId);
    }

    /**
     * @notice Burn wrapped tokens เพื่อ bridge กลับไป source chain
     * @param wrappedToken Wrapped token address
     * @param amount จำนวน
     * @param destChainId Destination chain ID (source chain)
     * @param destRecipient ผู้รับบน destination
     */
    function burnWrappedTokens(
        address wrappedToken,
        uint256 amount,
        uint256 destChainId,
        address destRecipient
    ) external nonReentrant whenNotPaused returns (bytes32 withdrawId) {
        require(originalTokens[wrappedToken] != address(0), "BridgeMint: not wrapped token");
        require(destRecipient != address(0), "BridgeMint: zero recipient");
        require(amount > 0, "BridgeMint: zero amount");

        withdrawId = keccak256(abi.encodePacked(
            msg.sender,
            wrappedToken,
            amount,
            destChainId,
            block.timestamp,
            block.chainid
        ));

        IBridgedToken(wrappedToken).bridgeBurn(msg.sender, amount);

        emit TokensBurned(withdrawId, msg.sender, wrappedToken, amount, destChainId, destRecipient);
    }

    function _verifySignatures(
        bytes32 depositId,
        address recipient,
        address wrappedToken,
        uint256 amount,
        uint256 sourceChainId,
        bytes[] calldata signatures
    ) internal view {
        bytes32 messageHash = keccak256(abi.encodePacked(
            depositId, recipient, wrappedToken, amount, sourceChainId, block.chainid
        ));
        bytes32 ethHash = keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", messageHash));

        // Simplified threshold check: require at least 2 unique validator sigs
        require(signatures.length >= 2, "BridgeMint: insufficient sigs");

        address prev = address(0);
        for (uint256 i = 0; i < signatures.length; i++) {
            address signer = _recoverSigner(ethHash, signatures[i]);
            require(hasRole(VALIDATOR_ROLE, signer), "BridgeMint: invalid signer");
            require(uint160(signer) > uint160(prev), "BridgeMint: duplicate/unordered signers");
            prev = signer;
        }
    }

    function _recoverSigner(bytes32 hash, bytes calldata sig) internal pure returns (address) {
        require(sig.length == 65, "BridgeMint: bad sig length");
        bytes32 r; bytes32 s; uint8 v;
        assembly {
            r := calldataload(sig.offset)
            s := calldataload(add(sig.offset, 32))
            v := byte(0, calldataload(add(sig.offset, 64)))
        }
        return ecrecover(hash, v, r, s);
    }

    function registerWrappedToken(
        address originalToken,
        address wrappedToken
    ) external onlyRole(DEFAULT_ADMIN_ROLE) {
        wrappedTokens[originalToken] = wrappedToken;
        originalTokens[wrappedToken] = originalToken;
    }

    function pause() external onlyRole(GUARDIAN_ROLE) { _pause(); }
    function unpause() external onlyRole(GUARDIAN_ROLE) { _unpause(); }
}

/**
 * @title IBridgedToken
 * @notice Interface สำหรับ wrapped token ที่ bridge mint/burn ได้
 */
interface IBridgedToken {
    function bridgeMint(address to, uint256 amount) external;
    function bridgeBurn(address from, uint256 amount) external;
}

/**
 * @title BridgedToken
 * @notice ERC-20 token ที่ mint/burn โดย bridge
 */
contract BridgedToken is AccessControl, IBridgedToken {
    bytes32 public constant BRIDGE_ROLE = keccak256("BRIDGE_ROLE");

    string public name;
    string public symbol;
    uint8 public decimals;
    uint256 public totalSupply;

    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    constructor(
        string memory _name,
        string memory _symbol,
        uint8 _decimals,
        address _bridge
    ) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(BRIDGE_ROLE, _bridge);
    }

    function bridgeMint(address to, uint256 amount) external override onlyRole(BRIDGE_ROLE) {
        totalSupply += amount;
        balanceOf[to] += amount;
        emit Transfer(address(0), to, amount);
    }

    function bridgeBurn(address from, uint256 amount) external override onlyRole(BRIDGE_ROLE) {
        require(balanceOf[from] >= amount, "BridgedToken: insufficient balance");
        totalSupply -= amount;
        balanceOf[from] -= amount;
        emit Transfer(from, address(0), amount);
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount, "BridgedToken: insufficient balance");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(allowance[from][msg.sender] >= amount, "BridgedToken: insufficient allowance");
        require(balanceOf[from] >= amount, "BridgedToken: insufficient balance");
        allowance[from][msg.sender] -= amount;
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        emit Transfer(from, to, amount);
        return true;
    }
}
```

---

## 2. Optimistic Bridge

### ทฤษฎี Optimistic Bridge

Optimistic bridges ทำงานคล้าย Optimistic Rollup:
1. Relayer submit message + state root
2. รอ 7 วัน fraud window
3. ถ้าไม่มีใช้ challenge → finalize
4. ถ้ามี challenge → prove fraud → relayer ถูก slash

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title OptimisticBridge
 * @notice Bridge ที่ใช้ optimistic fraud proof window
 * @dev 7-day challenge period ก่อน finalization
 */
contract OptimisticBridge is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ============ Enums ============
    enum MessageStatus {
        PENDING,     // ยังอยู่ใน challenge period
        FINALIZED,   // ผ่าน challenge period แล้ว
        DISPUTED,    // อยู่ระหว่าง challenge
        REJECTED     // ถูก challenge สำเร็จ
    }

    // ============ Structs ============
    struct BridgeMessage {
        bytes32 messageId;
        address sender;
        address recipient;
        address token;
        uint256 amount;
        uint256 sourceChainId;
        uint256 submittedAt;
        MessageStatus status;
        address relayer;        // ใครส่ง message นี้
        bytes32 stateRoot;      // state root ที่อ้างอิง
    }

    struct RelayerInfo {
        uint256 bond;           // ETH bond ที่วาง
        uint256 submittedCount;
        uint256 challengedCount;
        uint256 slashedAmount;
        bool isActive;
    }

    // ============ Constants ============
    uint256 public constant CHALLENGE_PERIOD = 7 days;
    uint256 public constant MIN_RELAYER_BOND = 1 ether;
    uint256 public constant CHALLENGER_REWARD_BPS = 500; // 5% of bond to challenger
    uint256 public constant FRAUD_SLASH_BPS = 10000;    // 100% slash on fraud

    // ============ State ============
    mapping(bytes32 => BridgeMessage) public messages;
    mapping(address => RelayerInfo) public relayers;
    mapping(bytes32 => address) public challenges;    // messageId → challenger
    mapping(bytes32 => uint256) public challengeTime; // messageId → challenge start

    bytes32[] public pendingMessages;
    uint256 public totalRelayerBonds;

    // ============ Events ============
    event MessageSubmitted(
        bytes32 indexed messageId,
        address indexed relayer,
        address recipient,
        address token,
        uint256 amount,
        uint256 sourceChainId
    );
    event MessageFinalized(bytes32 indexed messageId, address indexed recipient);
    event MessageChallenged(
        bytes32 indexed messageId,
        address indexed challenger,
        uint256 challengeTime_
    );
    event ChallengeResolved(
        bytes32 indexed messageId,
        bool fraudProven,
        address indexed winner
    );
    event RelayerBonded(address indexed relayer, uint256 amount);
    event RelayerSlashed(address indexed relayer, uint256 amount, address challenger);

    // ============ Constructor ============
    constructor() Ownable(msg.sender) {}

    // ============ Relayer Registration ============

    /**
     * @notice Relayer วาง bond เพื่อเป็น authorized relayer
     */
    function bondRelayer() external payable {
        require(msg.value >= MIN_RELAYER_BOND, "OptimisticBridge: insufficient bond");

        RelayerInfo storage info = relayers[msg.sender];
        info.bond += msg.value;
        info.isActive = true;
        totalRelayerBonds += msg.value;

        emit RelayerBonded(msg.sender, msg.value);
    }

    /**
     * @notice Relayer ถอน bond (เฉพาะเมื่อไม่มี pending messages)
     */
    function unbondRelayer() external {
        RelayerInfo storage info = relayers[msg.sender];
        require(info.isActive, "OptimisticBridge: not active");
        require(info.bond > 0, "OptimisticBridge: no bond");

        uint256 amount = info.bond;
        info.bond = 0;
        info.isActive = false;
        totalRelayerBonds -= amount;

        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "OptimisticBridge: transfer failed");
    }

    // ============ Message Relay ============

    /**
     * @notice Relayer submit bridge message
     * @param messageId Unique ID ของ message (จาก source chain)
     * @param recipient ผู้รับ
     * @param token Token address
     * @param amount จำนวน
     * @param sourceChainId Source chain ID
     * @param stateRoot State root ที่ include message นี้
     */
    function submitMessage(
        bytes32 messageId,
        address recipient,
        address token,
        uint256 amount,
        uint256 sourceChainId,
        bytes32 stateRoot
    ) external {
        require(relayers[msg.sender].isActive, "OptimisticBridge: not active relayer");
        require(relayers[msg.sender].bond >= MIN_RELAYER_BOND, "OptimisticBridge: insufficient bond");
        require(messages[messageId].submittedAt == 0, "OptimisticBridge: already submitted");
        require(recipient != address(0), "OptimisticBridge: zero recipient");
        require(amount > 0, "OptimisticBridge: zero amount");

        messages[messageId] = BridgeMessage({
            messageId: messageId,
            sender: address(0), // known on source chain
            recipient: recipient,
            token: token,
            amount: amount,
            sourceChainId: sourceChainId,
            submittedAt: block.timestamp,
            status: MessageStatus.PENDING,
            relayer: msg.sender,
            stateRoot: stateRoot
        });

        pendingMessages.push(messageId);
        relayers[msg.sender].submittedCount++;

        emit MessageSubmitted(messageId, msg.sender, recipient, token, amount, sourceChainId);
    }

    /**
     * @notice Finalize message หลังผ่าน challenge period
     * @param messageId Message ที่ต้องการ finalize
     */
    function finalizeMessage(bytes32 messageId) external nonReentrant {
        BridgeMessage storage message = messages[messageId];

        require(message.status == MessageStatus.PENDING, "OptimisticBridge: not pending");
        require(
            block.timestamp >= message.submittedAt + CHALLENGE_PERIOD,
            "OptimisticBridge: challenge period not ended"
        );

        message.status = MessageStatus.FINALIZED;

        // Transfer tokens ให้ recipient
        IERC20(message.token).safeTransfer(message.recipient, message.amount);

        emit MessageFinalized(messageId, message.recipient);
    }

    // ============ Challenge Mechanism ============

    /**
     * @notice Challenge message ที่สงสัยว่า fraudulent
     * @dev ต้องวาง bond เพื่อ challenge (ป้องกัน spam)
     * @param messageId Message ที่ต้องการ challenge
     */
    function challengeMessage(bytes32 messageId) external payable {
        BridgeMessage storage message = messages[messageId];

        require(message.status == MessageStatus.PENDING, "OptimisticBridge: not pending");
        require(
            block.timestamp < message.submittedAt + CHALLENGE_PERIOD,
            "OptimisticBridge: too late to challenge"
        );
        require(challenges[messageId] == address(0), "OptimisticBridge: already challenged");
        require(msg.value >= 0.1 ether, "OptimisticBridge: insufficient challenge bond");

        message.status = MessageStatus.DISPUTED;
        challenges[messageId] = msg.sender;
        challengeTime[messageId] = block.timestamp;
        relayers[message.relayer].challengedCount++;

        emit MessageChallenged(messageId, msg.sender, block.timestamp);
    }

    /**
     * @notice Resolve challenge ด้วย state proof
     * @param messageId Message ที่ถูก challenge
     * @param proof Merkle proof ว่า message ถูกต้อง
     * @param fraudProven true ถ้าพิสูจน์ได้ว่า fraud
     */
    function resolveChallenge(
        bytes32 messageId,
        bytes calldata proof,
        bool fraudProven
    ) external onlyOwner {
        BridgeMessage storage message = messages[messageId];
        require(message.status == MessageStatus.DISPUTED, "OptimisticBridge: not disputed");

        address challenger = challenges[messageId];
        address relayer = message.relayer;

        if (fraudProven) {
            // Fraud confirmed: slash relayer, reward challenger
            message.status = MessageStatus.REJECTED;

            uint256 relayerBond = relayers[relayer].bond;
            uint256 challengerReward = relayerBond * CHALLENGER_REWARD_BPS / 10000;

            relayers[relayer].bond = 0;
            relayers[relayer].isActive = false;
            relayers[relayer].slashedAmount += relayerBond;
            totalRelayerBonds -= relayerBond;

            // คืน challenge bond + reward ให้ challenger
            (bool success, ) = challenger.call{value: 0.1 ether + challengerReward}("");
            require(success, "OptimisticBridge: challenger reward failed");

            // ส่ง bond ที่เหลือไป treasury
            (success, ) = owner().call{value: relayerBond - challengerReward}("");
            require(success, "OptimisticBridge: treasury transfer failed");

            emit RelayerSlashed(relayer, relayerBond, challenger);
        } else {
            // Challenge ล้มเหลว: message ถูกต้อง → finalize
            message.status = MessageStatus.PENDING;
            delete challenges[messageId];

            // Challenger เสีย bond
            (bool success, ) = relayer.call{value: 0.1 ether}("");
            require(success, "OptimisticBridge: relayer reward failed");

            emit ChallengeResolved(messageId, false, relayer);
        }
    }

    // ============ View Functions ============

    function getMessageStatus(bytes32 messageId) external view returns (
        MessageStatus status,
        uint256 timeToFinality,
        bool isChallenged
    ) {
        BridgeMessage storage message = messages[messageId];
        status = message.status;
        isChallenged = challenges[messageId] != address(0);

        if (status == MessageStatus.PENDING) {
            uint256 finalityTime = message.submittedAt + CHALLENGE_PERIOD;
            timeToFinality = block.timestamp >= finalityTime
                ? 0
                : finalityTime - block.timestamp;
        }
    }

    receive() external payable {}
}
```

---

## 3. Message Passing ผ่าน Chainlink CCIP

### ทฤษฎี Cross-Chain Message Passing

นอกจาก token bridge แล้ว เราสามารถส่ง arbitrary calldata ข้าม chain ได้ ซึ่งเปิดให้ทำ:
- Cross-chain governance (vote บน Ethereum, execute บน Polygon)
- Cross-chain DeFi (borrow บน chain A, collateral บน chain B)
- Cross-chain NFT operations

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// Chainlink CCIP interfaces (simplified)
interface IRouterClient {
    struct EVM2AnyMessage {
        bytes receiver;         // abi.encode(address)
        bytes data;             // calldata
        TokenAmount[] tokenAmounts;
        address feeToken;       // address(0) = native ETH
        bytes extraArgs;        // additional options
    }

    struct TokenAmount {
        address token;
        uint256 amount;
    }

    function getFee(
        uint64 destinationChainSelector,
        EVM2AnyMessage memory message
    ) external view returns (uint256 fee);

    function ccipSend(
        uint64 destinationChainSelector,
        EVM2AnyMessage memory message
    ) external payable returns (bytes32 messageId);
}

interface CCIPReceiver {
    struct Any2EVMMessage {
        bytes32 messageId;
        uint64 sourceChainSelector;
        bytes sender;           // abi.encode(address)
        bytes data;
        IRouterClient.TokenAmount[] destTokenAmounts;
    }

    function ccipReceive(Any2EVMMessage calldata message) external;
}

/**
 * @title CrossChainCallProxy
 * @notice ส่ง arbitrary function calls ข้าม chain ผ่าน Chainlink CCIP
 * @dev สามารถ trigger function calls บน remote chain
 */
contract CrossChainCallProxy {
    // ============ Errors ============
    error NotEnoughBalance(uint256 balance, uint256 fees);
    error NothingToWithdraw();
    error CallerNotAllowed(address caller);
    error SourceChainNotAllowed(uint64 sourceChain);
    error SenderNotAllowed(address sender);
    error InvalidMessageId();

    // ============ Structs ============
    struct CrossChainCall {
        address target;         // contract ที่ต้องการ call
        bytes callData;         // function selector + args
        uint256 value;          // ETH ที่ส่งพร้อม call
        bool allowFailure;      // อนุญาตให้ fail ได้หรือไม่
    }

    struct MessageRecord {
        bytes32 messageId;
        uint64 destChain;
        address sender;
        uint256 timestamp;
        bool executed;
    }

    // ============ State ============
    IRouterClient public immutable ccipRouter;
    address public immutable linkToken;
    address public owner;

    mapping(uint64 => address) public trustedRemotes;     // chainSelector → remote contract
    mapping(bytes32 => MessageRecord) public sentMessages;
    mapping(bytes32 => bool) public receivedMessages;

    // ============ Events ============
    event MessageSent(
        bytes32 indexed messageId,
        uint64 indexed destinationChainSelector,
        address indexed receiver,
        CrossChainCall[] calls,
        address feeToken,
        uint256 fees
    );
    event MessageReceived(
        bytes32 indexed messageId,
        uint64 indexed sourceChainSelector,
        address indexed sender,
        CrossChainCall[] calls
    );
    event CallExecuted(
        bytes32 indexed messageId,
        uint256 callIndex,
        bool success,
        bytes returnData
    );

    // ============ Constructor ============
    constructor(address _router, address _link) {
        ccipRouter = IRouterClient(_router);
        linkToken = _link;
        owner = msg.sender;
    }

    // ============ Modifiers ============
    modifier onlyOwner() {
        require(msg.sender == owner, "CrossChainCallProxy: not owner");
        _;
    }

    modifier onlyRouter() {
        require(msg.sender == address(ccipRouter), "CrossChainCallProxy: not router");
        _;
    }

    // ============ Sending ============

    /**
     * @notice ส่ง function calls ไปยัง chain อื่น (จ่าย LINK)
     * @param destinationChainSelector Chainlink chain selector
     * @param calls Array ของ function calls ที่ต้องการ execute
     * @param useNative ใช้ native ETH จ่าย fee (แทน LINK)
     */
    function sendCrossChainCalls(
        uint64 destinationChainSelector,
        CrossChainCall[] calldata calls,
        bool useNative
    ) external payable returns (bytes32 messageId) {
        address remoteContract = trustedRemotes[destinationChainSelector];
        require(remoteContract != address(0), "CrossChainCallProxy: chain not configured");
        require(calls.length > 0, "CrossChainCallProxy: no calls");

        // สร้าง CCIP message
        IRouterClient.EVM2AnyMessage memory ccipMessage = IRouterClient.EVM2AnyMessage({
            receiver: abi.encode(remoteContract),
            data: abi.encode(calls),
            tokenAmounts: new IRouterClient.TokenAmount[](0),
            feeToken: useNative ? address(0) : linkToken,
            extraArgs: ""
        });

        // คำนวณ fee
        uint256 fees = ccipRouter.getFee(destinationChainSelector, ccipMessage);

        if (useNative) {
            require(msg.value >= fees, "CrossChainCallProxy: insufficient ETH for fees");
            messageId = ccipRouter.ccipSend{value: fees}(destinationChainSelector, ccipMessage);

            // คืน ETH ส่วนเกิน
            if (msg.value > fees) {
                (bool success, ) = msg.sender.call{value: msg.value - fees}("");
                require(success, "CrossChainCallProxy: ETH refund failed");
            }
        } else {
            // Transfer LINK สำหรับ fee
            IERC20(linkToken).transferFrom(msg.sender, address(this), fees);
            IERC20(linkToken).approve(address(ccipRouter), fees);
            messageId = ccipRouter.ccipSend(destinationChainSelector, ccipMessage);
        }

        sentMessages[messageId] = MessageRecord({
            messageId: messageId,
            destChain: destinationChainSelector,
            sender: msg.sender,
            timestamp: block.timestamp,
            executed: false
        });

        emit MessageSent(messageId, destinationChainSelector, remoteContract, calls, useNative ? address(0) : linkToken, fees);
    }

    /**
     * @notice ส่ง token พร้อม cross-chain call
     */
    function sendTokensWithCall(
        uint64 destinationChainSelector,
        address token,
        uint256 amount,
        CrossChainCall[] calldata calls
    ) external payable returns (bytes32 messageId) {
        address remoteContract = trustedRemotes[destinationChainSelector];
        require(remoteContract != address(0), "CrossChainCallProxy: chain not configured");

        IERC20(token).transferFrom(msg.sender, address(this), amount);
        IERC20(token).approve(address(ccipRouter), amount);

        IRouterClient.TokenAmount[] memory tokenAmounts = new IRouterClient.TokenAmount[](1);
        tokenAmounts[0] = IRouterClient.TokenAmount({token: token, amount: amount});

        IRouterClient.EVM2AnyMessage memory ccipMessage = IRouterClient.EVM2AnyMessage({
            receiver: abi.encode(remoteContract),
            data: abi.encode(calls),
            tokenAmounts: tokenAmounts,
            feeToken: address(0), // pay with native
            extraArgs: ""
        });

        uint256 fees = ccipRouter.getFee(destinationChainSelector, ccipMessage);
        require(msg.value >= fees, "CrossChainCallProxy: insufficient fees");

        messageId = ccipRouter.ccipSend{value: fees}(destinationChainSelector, ccipMessage);

        emit MessageSent(messageId, destinationChainSelector, remoteContract, calls, address(0), fees);
    }

    // ============ Receiving ============

    /**
     * @notice รับและ execute cross-chain message
     * @dev เรียกโดย CCIP Router เท่านั้น
     */
    function ccipReceive(
        CCIPReceiver.Any2EVMMessage calldata message
    ) external onlyRouter {
        require(!receivedMessages[message.messageId], "CrossChainCallProxy: duplicate");

        uint64 sourceChain = message.sourceChainSelector;
        address sender = abi.decode(message.sender, (address));

        require(trustedRemotes[sourceChain] == sender, "CrossChainCallProxy: untrusted source");

        receivedMessages[message.messageId] = true;

        CrossChainCall[] memory calls = abi.decode(message.data, (CrossChainCall[]));

        emit MessageReceived(message.messageId, sourceChain, sender, calls);

        // Execute each call
        for (uint256 i = 0; i < calls.length; i++) {
            _executeCall(message.messageId, i, calls[i]);
        }
    }

    function _executeCall(
        bytes32 messageId,
        uint256 index,
        CrossChainCall memory call_
    ) internal {
        (bool success, bytes memory returnData) = call_.target.call{value: call_.value}(call_.callData);

        if (!call_.allowFailure && !success) {
            // bubble up revert
            assembly {
                revert(add(returnData, 32), mload(returnData))
            }
        }

        emit CallExecuted(messageId, index, success, returnData);
    }

    // ============ Admin ============

    function setTrustedRemote(
        uint64 chainSelector,
        address remoteContract
    ) external onlyOwner {
        trustedRemotes[chainSelector] = remoteContract;
    }

    function withdrawLink() external onlyOwner {
        uint256 balance = IERC20(linkToken).balanceOf(address(this));
        require(balance > 0, "CrossChainCallProxy: no LINK");
        IERC20(linkToken).transfer(owner, balance);
    }

    receive() external payable {}
}
```

---

## 4. Bridge Security Patterns

### Nonce Management & Replay Protection

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import "@openzeppelin/contracts/utils/cryptography/EIP712.sol";

/**
 * @title BridgeSecurityModule
 * @notice ระบบ security สำหรับ bridge: nonce, replay protection, guardian multisig
 */
contract BridgeSecurityModule is AccessControl, EIP712 {
    using ECDSA for bytes32;

    // ============ Roles ============
    bytes32 public constant GUARDIAN_ROLE = keccak256("GUARDIAN_ROLE");
    bytes32 public constant VALIDATOR_ROLE = keccak256("VALIDATOR_ROLE");

    // ============ EIP-712 Type Hashes ============
    bytes32 public constant BRIDGE_MESSAGE_TYPEHASH = keccak256(
        "BridgeMessage(bytes32 messageId,address recipient,address token,uint256 amount,uint256 sourceChainId,uint256 destChainId,uint256 nonce,uint256 deadline)"
    );

    // ============ State ============

    // Global nonce ต่อ (sourceChain, sender) pair
    mapping(uint256 => mapping(address => uint256)) public chainNonces;

    // Processed message IDs (replay protection)
    mapping(bytes32 => bool) public processedMessages;

    // Emergency pause per chain
    mapping(uint256 => bool) public chainPaused;

    // Rate limiting: max bridge amount per block
    mapping(address => uint256) public lastBridgeBlock;
    mapping(address => uint256) public bridgeCountThisBlock;
    uint256 public constant MAX_BRIDGES_PER_BLOCK = 5;

    // Guardian multisig threshold
    uint256 public guardianThreshold;
    mapping(bytes32 => mapping(address => bool)) public guardianVotes;
    mapping(bytes32 => uint256) public guardianVoteCount;
    mapping(bytes32 => bool) public guardianExecuted;

    // ============ Events ============
    event MessageVerified(bytes32 indexed messageId, address indexed verifier);
    event GuardianVoted(bytes32 indexed proposalId, address indexed guardian);
    event GuardianActionExecuted(bytes32 indexed proposalId);
    event ChainPaused(uint256 indexed chainId, address guardian);
    event ChainUnpaused(uint256 indexed chainId, address guardian);
    event ReplayAttackPrevented(bytes32 indexed messageId, address indexed attacker);

    // ============ Constructor ============
    constructor(
        uint256 _guardianThreshold
    ) EIP712("BridgeSecurityModule", "1") {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(GUARDIAN_ROLE, msg.sender);
        guardianThreshold = _guardianThreshold;
    }

    // ============ Message Verification ============

    /**
     * @notice ตรวจสอบ bridge message ด้วย EIP-712 typed signature
     * @dev ป้องกัน replay attacks ด้วย nonce และ deadline
     */
    function verifyBridgeMessage(
        bytes32 messageId,
        address recipient,
        address token,
        uint256 amount,
        uint256 sourceChainId,
        uint256 destChainId,
        uint256 nonce,
        uint256 deadline,
        bytes[] calldata validatorSignatures
    ) external returns (bool) {
        // 1. ตรวจสอบว่าไม่เคย process แล้ว (replay protection)
        require(!processedMessages[messageId], "BridgeSecurity: message already processed");

        // 2. ตรวจสอบ deadline
        require(block.timestamp <= deadline, "BridgeSecurity: message expired");

        // 3. ตรวจสอบ nonce
        require(
            nonce == chainNonces[sourceChainId][recipient],
            "BridgeSecurity: invalid nonce"
        );

        // 4. ตรวจสอบว่า chain ไม่ได้ pause
        require(!chainPaused[sourceChainId], "BridgeSecurity: source chain paused");
        require(!chainPaused[destChainId], "BridgeSecurity: dest chain paused");

        // 5. Rate limiting
        if (lastBridgeBlock[recipient] == block.number) {
            require(
                bridgeCountThisBlock[recipient] < MAX_BRIDGES_PER_BLOCK,
                "BridgeSecurity: rate limit exceeded"
            );
            bridgeCountThisBlock[recipient]++;
        } else {
            lastBridgeBlock[recipient] = block.number;
            bridgeCountThisBlock[recipient] = 1;
        }

        // 6. ตรวจสอบ EIP-712 signatures
        bytes32 structHash = keccak256(abi.encode(
            BRIDGE_MESSAGE_TYPEHASH,
            messageId,
            recipient,
            token,
            amount,
            sourceChainId,
            destChainId,
            nonce,
            deadline
        ));
        bytes32 digest = _hashTypedDataV4(structHash);

        uint256 validSigs = _countValidSignatures(digest, validatorSignatures);
        require(
            validSigs >= _getRequiredSignatures(),
            "BridgeSecurity: insufficient valid signatures"
        );

        // 7. Mark as processed
        processedMessages[messageId] = true;
        chainNonces[sourceChainId][recipient]++;

        emit MessageVerified(messageId, msg.sender);
        return true;
    }

    /**
     * @notice ตรวจสอบและนับ valid signatures
     */
    function _countValidSignatures(
        bytes32 digest,
        bytes[] calldata signatures
    ) internal view returns (uint256 count) {
        address prev = address(0);

        for (uint256 i = 0; i < signatures.length; i++) {
            address signer = digest.recover(signatures[i]);

            // ต้อง ordered เพื่อป้องกัน duplicate
            require(uint160(signer) > uint160(prev), "BridgeSecurity: unsorted signers");
            prev = signer;

            if (hasRole(VALIDATOR_ROLE, signer)) {
                count++;
            }
        }
    }

    function _getRequiredSignatures() internal view returns (uint256) {
        // Simplified: hard-coded to 2/3 threshold
        return 2;
    }

    // ============ Guardian Multisig ============

    /**
     * @notice Guardian vote สำหรับ emergency action
     * @param proposalId ID ของ proposal (เช่น hash ของ action)
     */
    function guardianVote(bytes32 proposalId) external onlyRole(GUARDIAN_ROLE) {
        require(!guardianExecuted[proposalId], "BridgeSecurity: already executed");
        require(!guardianVotes[proposalId][msg.sender], "BridgeSecurity: already voted");

        guardianVotes[proposalId][msg.sender] = true;
        guardianVoteCount[proposalId]++;

        emit GuardianVoted(proposalId, msg.sender);

        if (guardianVoteCount[proposalId] >= guardianThreshold) {
            guardianExecuted[proposalId] = true;
            emit GuardianActionExecuted(proposalId);
        }
    }

    /**
     * @notice Guardian emergency pause ของ chain
     */
    function emergencyPause(uint256 chainId) external onlyRole(GUARDIAN_ROLE) {
        bytes32 proposalId = keccak256(abi.encodePacked("PAUSE", chainId, block.timestamp / 1 hours));

        // Self-vote
        if (!guardianVotes[proposalId][msg.sender]) {
            guardianVote(proposalId);
        }

        // Execute ถ้ามีเสียงพอ
        if (guardianExecuted[proposalId] && !chainPaused[chainId]) {
            chainPaused[chainId] = true;
            emit ChainPaused(chainId, msg.sender);
        }
    }

    /**
     * @notice Guardian unpause chain
     */
    function unpauseChain(uint256 chainId) external onlyRole(DEFAULT_ADMIN_ROLE) {
        chainPaused[chainId] = false;
        emit ChainUnpaused(chainId, msg.sender);
    }

    // ============ Nonce Management ============

    /**
     * @notice ดู nonce ปัจจุบันของ address
     */
    function getNonce(uint256 chainId, address user) external view returns (uint256) {
        return chainNonces[chainId][user];
    }

    /**
     * @notice สร้าง message hash สำหรับ signing (ใช้ off-chain)
     */
    function getMessageHash(
        bytes32 messageId,
        address recipient,
        address token,
        uint256 amount,
        uint256 sourceChainId,
        uint256 destChainId,
        uint256 nonce,
        uint256 deadline
    ) external view returns (bytes32) {
        bytes32 structHash = keccak256(abi.encode(
            BRIDGE_MESSAGE_TYPEHASH,
            messageId,
            recipient,
            token,
            amount,
            sourceChainId,
            destChainId,
            nonce,
            deadline
        ));
        return _hashTypedDataV4(structHash);
    }

    /**
     * @notice ตรวจสอบ replay attack attempt
     * @dev ใช้ใน monitoring system
     */
    function detectReplayAttempt(bytes32 messageId) external {
        if (processedMessages[messageId]) {
            emit ReplayAttackPrevented(messageId, msg.sender);
        }
    }
}
```

---

## Workshop และ Exercises

### Exercise 1: Bridge Fee Calculator

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title BridgeFeeCalculator
 * @notice คำนวณ bridge fees แบบ dynamic
 */
contract BridgeFeeCalculator {
    struct FeeConfig {
        uint256 baseFeeWei;         // base fee ใน ETH (wei)
        uint256 percentageFee;      // percentage fee (basis points)
        uint256 minFee;             // minimum fee
        uint256 maxFee;             // maximum fee
    }

    mapping(uint64 => FeeConfig) public chainFees;

    /**
     * @notice คำนวณ total fee สำหรับ bridge
     * @param amount จำนวน token ที่ bridge
     * @param destChainSelector Chain selector ของ destination
     * @param tokenPrice ราคา token ใน wei (สำหรับ convert fee เป็น token)
     */
    function calculateFee(
        uint256 amount,
        uint64 destChainSelector,
        uint256 tokenPrice
    ) external view returns (
        uint256 baseFee,
        uint256 percentFee,
        uint256 totalFeeInToken,
        uint256 totalFeeInEth
    ) {
        FeeConfig storage config = chainFees[destChainSelector];

        baseFee = config.baseFeeWei;
        percentFee = amount * config.percentageFee / 10000;

        // Convert base fee (ETH) to token
        uint256 baseFeeInToken = tokenPrice > 0 ? baseFee * 1e18 / tokenPrice : 0;

        totalFeeInToken = baseFeeInToken + percentFee;

        // Cap at min/max
        if (totalFeeInToken < config.minFee) totalFeeInToken = config.minFee;
        if (totalFeeInToken > config.maxFee) totalFeeInToken = config.maxFee;

        totalFeeInEth = baseFee;
    }

    /**
     * @notice คำนวณ bridge ที่คุ้มค่าที่สุดสำหรับ amount ที่กำหนด
     */
    function findCheapestRoute(
        uint256 amount,
        uint64[] calldata chainSelectors,
        uint256 tokenPrice
    ) external view returns (
        uint64 bestChain,
        uint256 lowestFee
    ) {
        lowestFee = type(uint256).max;

        for (uint256 i = 0; i < chainSelectors.length; i++) {
            FeeConfig storage config = chainFees[chainSelectors[i]];
            uint256 percentFee = amount * config.percentageFee / 10000;
            uint256 baseFeeInToken = tokenPrice > 0 ? config.baseFeeWei * 1e18 / tokenPrice : 0;
            uint256 totalFee = baseFeeInToken + percentFee;

            if (totalFee < lowestFee) {
                lowestFee = totalFee;
                bestChain = chainSelectors[i];
            }
        }
    }
}
```

### Exercise 2: Bridge Monitoring Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title BridgeMonitor
 * @notice ติดตาม bridge activity และ detect anomalies
 */
contract BridgeMonitor {
    struct BridgeStats {
        uint256 totalVolume24h;
        uint256 transactionCount24h;
        uint256 largestTransaction24h;
        uint256 lastUpdateTime;
    }

    mapping(address => BridgeStats) public tokenStats;
    mapping(uint256 => uint256) public dailyVolume; // day → volume
    uint256 public constant WHALE_THRESHOLD = 100000e18; // 100k tokens

    event WhaleDetected(
        address indexed token,
        address indexed user,
        uint256 amount,
        uint256 timestamp
    );
    event UnusualVolumeDetected(uint256 indexed day, uint256 volume, uint256 normalVolume);

    /**
     * @notice บันทึกและวิเคราะห์ bridge transaction
     */
    function recordBridge(
        address token,
        address user,
        uint256 amount
    ) external returns (bool isAnomaly) {
        uint256 today = block.timestamp / 1 days;

        BridgeStats storage stats = tokenStats[token];
        if (block.timestamp / 1 days > stats.lastUpdateTime / 1 days) {
            stats.totalVolume24h = 0;
            stats.transactionCount24h = 0;
            stats.largestTransaction24h = 0;
        }

        stats.totalVolume24h += amount;
        stats.transactionCount24h++;
        stats.lastUpdateTime = block.timestamp;

        if (amount > stats.largestTransaction24h) {
            stats.largestTransaction24h = amount;
        }

        dailyVolume[today] += amount;

        // Detect whale
        if (amount >= WHALE_THRESHOLD) {
            emit WhaleDetected(token, user, amount, block.timestamp);
            isAnomaly = true;
        }

        // Detect unusual volume (>10x previous day)
        uint256 yesterday = dailyVolume[today - 1];
        if (yesterday > 0 && dailyVolume[today] > yesterday * 10) {
            emit UnusualVolumeDetected(today, dailyVolume[today], yesterday);
            isAnomaly = true;
        }
    }

    /**
     * @notice คำนวณ bridge utilization rate
     */
    function getBridgeHealth(
        uint256 tvl,         // Total Value Locked
        uint256 dailyVol,    // daily volume
        uint256 pendingTx    // pending transactions
    ) external pure returns (
        uint256 utilizationRate, // basis points
        string memory status
    ) {
        if (tvl == 0) return (0, "EMPTY");

        utilizationRate = dailyVol * 10000 / tvl;

        if (utilizationRate > 5000) {
            status = "HIGH_UTILIZATION";
        } else if (pendingTx > 1000) {
            status = "CONGESTED";
        } else if (utilizationRate > 0) {
            status = "HEALTHY";
        } else {
            status = "IDLE";
        }
    }
}
```

### Exercise 3: Multi-sig Guardian

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title BridgeGuardian
 * @notice Multi-sig guardian สำหรับ bridge emergency control
 */
contract BridgeGuardian {
    address[] public guardians;
    uint256 public threshold;
    mapping(address => bool) public isGuardian;

    struct Proposal {
        bytes callData;
        address target;
        uint256 approvals;
        bool executed;
        uint256 expiresAt;
        mapping(address => bool) approved;
    }

    mapping(bytes32 => Proposal) public proposals;

    event ProposalCreated(bytes32 indexed proposalId, address indexed creator);
    event ProposalApproved(bytes32 indexed proposalId, address indexed guardian);
    event ProposalExecuted(bytes32 indexed proposalId);

    constructor(address[] memory _guardians, uint256 _threshold) {
        require(_threshold <= _guardians.length, "BridgeGuardian: invalid threshold");
        require(_threshold > 0, "BridgeGuardian: zero threshold");

        for (uint256 i = 0; i < _guardians.length; i++) {
            require(!isGuardian[_guardians[i]], "BridgeGuardian: duplicate");
            isGuardian[_guardians[i]] = true;
        }

        guardians = _guardians;
        threshold = _threshold;
    }

    modifier onlyGuardian() {
        require(isGuardian[msg.sender], "BridgeGuardian: not guardian");
        _;
    }

    function createProposal(
        address target,
        bytes calldata callData
    ) external onlyGuardian returns (bytes32 proposalId) {
        proposalId = keccak256(abi.encodePacked(target, callData, block.timestamp));

        Proposal storage proposal = proposals[proposalId];
        proposal.target = target;
        proposal.callData = callData;
        proposal.expiresAt = block.timestamp + 2 days;

        emit ProposalCreated(proposalId, msg.sender);
    }

    function approveProposal(bytes32 proposalId) external onlyGuardian {
        Proposal storage proposal = proposals[proposalId];
        require(!proposal.executed, "BridgeGuardian: already executed");
        require(block.timestamp < proposal.expiresAt, "BridgeGuardian: expired");
        require(!proposal.approved[msg.sender], "BridgeGuardian: already approved");

        proposal.approved[msg.sender] = true;
        proposal.approvals++;

        emit ProposalApproved(proposalId, msg.sender);

        if (proposal.approvals >= threshold) {
            _executeProposal(proposalId);
        }
    }

    function _executeProposal(bytes32 proposalId) internal {
        Proposal storage proposal = proposals[proposalId];
        proposal.executed = true;

        (bool success, bytes memory result) = proposal.target.call(proposal.callData);
        require(success, string(result));

        emit ProposalExecuted(proposalId);
    }

    function getGuardians() external view returns (address[] memory) {
        return guardians;
    }
}
```

---

## สรุป Part 59

- **Lock-and-Mint Bridge**: LockBox บน source chain lock ERC-20 แล้ว emit event, validators relay ไปยัง BridgeMint บน destination chain ที่ mint wrapped token, multisig validation ด้วย ECDSA signatures, refund mechanism สำหรับ failed bridges หลัง 7 วัน
- **Optimistic Bridge**: Relayer submit message พร้อม bond, 7-day challenge window ที่ใครก็ได้ challenge, fraud proof ทำให้ relayer ถูก slash 100% และ challenger ได้รับ 5% reward, ถ้าไม่มี challenge → finalize อัตโนมัติ
- **Message Passing (CCIP)**: CrossChainCallProxy ส่ง arbitrary calldata ผ่าน Chainlink CCIP, รองรับ token + call ในคราวเดียว, trusted remote verification ป้องกัน unauthorized calls
- **Bridge Security**: EIP-712 typed signatures ป้องกัน replay attacks, nonce management ต่อ (chain, user) pair, deadline ป้องกัน stale messages, rate limiting ต่อ block, guardian multisig สำหรับ emergency pause

## Next: Part 60 - Protocol Auditing Process
