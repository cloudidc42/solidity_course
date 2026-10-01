# Part 24: Cross-chain Bridges

## สารบัญ
1. Bridge Architecture
2. Lock-and-Mint Pattern
3. Burn-and-Mint Pattern
4. Message Passing (LayerZero / CCIP)
5. Workshop: Simple Bridge

---

## 1. Bridge Architecture

```
Cross-chain Bridge:
ย้าย assets จาก chain A → chain B

Types:
1. Trusted/Centralized: มี relayer/validator กลาง (Binance Bridge)
2. Optimistic: ส่งข้อมูล + รอ challenge (Across Protocol)
3. ZK: ส่ง ZK proof พิสูจน์ validity
4. Liquidity Network: LP ทั้งสอง chain (Hop, Stargate)

Patterns:
A. Lock-and-Mint:
   - Lock ETH ใน L1 contract
   - Mint wETH ใน L2
   - Burn wETH ใน L2 → Unlock ETH ใน L1
   
B. Burn-and-Mint:
   - Burn token ใน chain A
   - Mint token ใน chain B
   - ต้องมี token standard ที่รองรับทั้งสอง chain

C. Liquidity Pools:
   - LP ใน chain A + chain B
   - ส่ง message → LP ใน destination chain จ่าย user ทันที
   - ไม่ต้องรอ finality

Risks:
- Smart contract bugs (Ronin Bridge: $625M)
- Validator compromise
- Message replay attacks
- Price oracle manipulation
```

---

## 2. Lock-and-Mint Bridge

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * L1Bridge: Lock ETH/ERC20 tokens
 * L2Bridge: Mint wrapped tokens
 * 
 * Flow:
 * 1. User locks ETH ใน L1Bridge
 * 2. Oracle/Relayer ตรวจจับ event
 * 3. Oracle เรียก L2Bridge.mint()
 * 4. User ได้ wETH ใน L2
 */
contract L1Bridge {
    
    IERC20 public immutable token;
    
    // Nonce ป้องกัน replay
    mapping(bytes32 => bool) public processedDeposits;
    
    address public operator; // trusted relayer
    uint256 public depositNonce;
    
    event Deposit(
        address indexed from,
        address indexed to,
        uint256 amount,
        uint256 nonce,
        uint256 indexed l2ChainId
    );
    
    event Withdrawal(
        address indexed to,
        uint256 amount,
        bytes32 indexed txHash
    );
    
    error AlreadyProcessed(bytes32 depositId);
    error NotOperator();
    error InvalidSignature();
    
    modifier onlyOperator() {
        if (msg.sender != operator) revert NotOperator();
        _;
    }
    
    constructor(address _token, address _operator) {
        token = IERC20(_token);
        operator = _operator;
    }
    
    // User deposit tokens → lock in bridge
    function deposit(
        address to,      // recipient on L2
        uint256 amount,
        uint256 l2ChainId
    ) external {
        token.transferFrom(msg.sender, address(this), amount);
        
        uint256 nonce = depositNonce++;
        
        emit Deposit(msg.sender, to, amount, nonce, l2ChainId);
    }
    
    // Operator releases tokens after L2 burn
    function withdraw(
        address to,
        uint256 amount,
        bytes32 l2TxHash,
        bytes calldata signature
    ) external {
        // Verify signature from oracle
        bytes32 depositId = keccak256(abi.encodePacked(to, amount, l2TxHash));
        
        if (processedDeposits[depositId]) revert AlreadyProcessed(depositId);
        
        // Verify operator signed this withdrawal
        bytes32 messageHash = keccak256(abi.encodePacked(
            "\x19Ethereum Signed Message:\n32",
            keccak256(abi.encodePacked(to, amount, l2TxHash, block.chainid))
        ));
        
        address signer = _recover(messageHash, signature);
        if (signer != operator) revert InvalidSignature();
        
        processedDeposits[depositId] = true;
        
        token.transfer(to, amount);
        
        emit Withdrawal(to, amount, l2TxHash);
    }
    
    function _recover(bytes32 hash, bytes calldata sig) internal pure returns (address) {
        require(sig.length == 65, "Invalid sig length");
        bytes32 r;
        bytes32 s;
        uint8 v;
        assembly {
            r := calldataload(sig.offset)
            s := calldataload(add(sig.offset, 32))
            v := byte(0, calldataload(add(sig.offset, 64)))
        }
        return ecrecover(hash, v, r, s);
    }
}

contract L2BridgeToken {
    
    string public name;
    string public symbol;
    uint8 public constant decimals = 18;
    
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    uint256 public totalSupply;
    
    address public bridge; // only bridge can mint/burn
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    
    modifier onlyBridge() {
        require(msg.sender == bridge, "Not bridge");
        _;
    }
    
    constructor(string memory _name, string memory _symbol, address _bridge) {
        name = _name;
        symbol = _symbol;
        bridge = _bridge;
    }
    
    function mint(address to, uint256 amount) external onlyBridge {
        totalSupply += amount;
        balanceOf[to] += amount;
        emit Transfer(address(0), to, amount);
    }
    
    function burn(address from, uint256 amount) external onlyBridge {
        balanceOf[from] -= amount;
        totalSupply -= amount;
        emit Transfer(from, address(0), amount);
    }
    
    function transfer(address to, uint256 amount) external returns (bool) {
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }
    
    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        allowance[from][msg.sender] -= amount;
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        emit Transfer(from, to, amount);
        return true;
    }
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
    function balanceOf(address) external view returns (uint256);
}
```

---

## 3. LayerZero (Omnichain Messaging)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * LayerZero: generic omnichain messaging
 * 
 * Endpoint ทุก chain → ส่ง/รับ message
 * 
 * Flow:
 * 1. User calls lzSend() ใน source chain
 * 2. LayerZero Relayer อ่าน transaction
 * 3. Oracle (Chainlink/others) ยืนยัน block hash
 * 4. Relayer ส่ง message ไปยัง destination Endpoint
 * 5. Endpoint เรียก lzReceive() ของ destination contract
 */
interface ILayerZeroEndpoint {
    function send(
        uint16 _dstChainId,
        bytes calldata _destination,
        bytes calldata _payload,
        address payable _refundAddress,
        address _zroPaymentAddress,
        bytes calldata _adapterParams
    ) external payable;
    
    function estimateFees(
        uint16 _dstChainId,
        address _userApplication,
        bytes calldata _payload,
        bool _payInZRO,
        bytes calldata _adapterParam
    ) external view returns (uint256 nativeFee, uint256 zroFee);
    
    function getInboundNonce(uint16 _srcChainId, bytes calldata _srcAddress) external view returns (uint64);
}

interface ILayerZeroReceiver {
    function lzReceive(
        uint16 _srcChainId,
        bytes calldata _srcAddress,
        uint64 _nonce,
        bytes calldata _payload
    ) external;
}

contract OmniToken is ILayerZeroReceiver {
    
    ILayerZeroEndpoint public immutable lzEndpoint;
    
    string public name;
    string public symbol;
    mapping(address => uint256) public balanceOf;
    uint256 public totalSupply;
    
    // Trusted remote: chain ID → contract address
    mapping(uint16 => bytes) public trustedRemote;
    
    event SendToChain(uint16 indexed dstChainId, address indexed from, bytes toAddress, uint256 amount);
    event ReceiveFromChain(uint16 indexed srcChainId, address indexed to, uint256 amount);
    
    error NotTrustedRemote();
    error NotEndpoint();
    
    constructor(
        string memory _name,
        string memory _symbol,
        address _lzEndpoint,
        uint256 initialSupply
    ) {
        name = _name;
        symbol = _symbol;
        lzEndpoint = ILayerZeroEndpoint(_lzEndpoint);
        balanceOf[msg.sender] = initialSupply;
        totalSupply = initialSupply;
    }
    
    // Owner sets trusted remote on each chain
    function setTrustedRemote(uint16 chainId, bytes calldata path) external {
        trustedRemote[chainId] = path;
    }
    
    // ส่ง tokens ข้าม chain
    function sendFrom(
        address from,
        uint16 dstChainId,
        bytes calldata toAddress,  // ABI-encoded address ใน destination
        uint256 amount
    ) external payable {
        // Burn locally
        balanceOf[from] -= amount;
        totalSupply -= amount;
        
        // Encode payload
        bytes memory payload = abi.encode(toAddress, amount);
        
        // Send via LayerZero
        lzEndpoint.send{value: msg.value}(
            dstChainId,
            trustedRemote[dstChainId],
            payload,
            payable(from),
            address(0), // no ZRO payment
            bytes("")   // default adapter params
        );
        
        emit SendToChain(dstChainId, from, toAddress, amount);
    }
    
    // LayerZero calls this on destination chain
    function lzReceive(
        uint16 srcChainId,
        bytes calldata srcAddress,
        uint64, // nonce
        bytes calldata payload
    ) external override {
        if (msg.sender != address(lzEndpoint)) revert NotEndpoint();
        
        // Verify trusted remote
        bytes memory trusted = trustedRemote[srcChainId];
        require(
            trusted.length == srcAddress.length && 
            keccak256(trusted) == keccak256(srcAddress),
            "Not trusted source"
        );
        
        // Decode payload
        (bytes memory toAddressBytes, uint256 amount) = abi.decode(payload, (bytes, uint256));
        address toAddress = abi.decode(toAddressBytes, (address));
        
        // Mint on destination
        balanceOf[toAddress] += amount;
        totalSupply += amount;
        
        emit ReceiveFromChain(srcChainId, toAddress, amount);
    }
    
    // Estimate fee for cross-chain transfer
    function estimateSendFee(
        uint16 dstChainId,
        bytes calldata toAddress,
        uint256 amount
    ) external view returns (uint256 nativeFee) {
        bytes memory payload = abi.encode(toAddress, amount);
        (nativeFee,) = lzEndpoint.estimateFees(
            dstChainId,
            address(this),
            payload,
            false,
            bytes("")
        );
    }
}
```

---

## 4. Chainlink CCIP (Cross-Chain Interoperability Protocol)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Chainlink CCIP - Enterprise-grade cross-chain
 * 
 * ปลอดภัยกว่า LayerZero เพราะใช้ Chainlink decentralized oracle network
 * 
 * Features:
 * - Token transfers
 * - Arbitrary message passing
 * - Programmable token transfers (message + tokens ในครั้งเดียว)
 */
interface IRouterClient {
    struct EVM2AnyMessage {
        bytes receiver;
        bytes data;
        TokenAmount[] tokenAmounts;
        address feeToken;
        bytes extraArgs;
    }
    
    struct TokenAmount {
        address token;
        uint256 amount;
    }
    
    function getFee(uint64 destinationChainSelector, EVM2AnyMessage memory message)
        external view returns (uint256 fee);
    
    function ccipSend(uint64 destinationChainSelector, EVM2AnyMessage memory message)
        external payable returns (bytes32 messageId);
}

interface IAny2EVMMessageReceiver {
    struct Any2EVMMessage {
        bytes32 messageId;
        uint64 sourceChainSelector;
        bytes sender;
        bytes data;
        IRouterClient.TokenAmount[] tokenAmounts;
    }
    
    function ccipReceive(Any2EVMMessage calldata message) external;
}

contract CCIPTokenSender {
    
    IRouterClient public immutable router;
    IERC20 public immutable linkToken; // for fees
    
    event MessageSent(
        bytes32 indexed messageId,
        uint64 indexed destinationChainSelector,
        address receiver,
        uint256 amount,
        uint256 fees
    );
    
    constructor(address _router, address _link) {
        router = IRouterClient(_router);
        linkToken = IERC20(_link);
    }
    
    // Send tokens via CCIP (pay fees in LINK)
    function sendToken(
        uint64 destinationChainSelector,
        address receiver,
        address token,
        uint256 amount
    ) external returns (bytes32 messageId) {
        IRouterClient.TokenAmount[] memory tokenAmounts = new IRouterClient.TokenAmount[](1);
        tokenAmounts[0] = IRouterClient.TokenAmount({token: token, amount: amount});
        
        IRouterClient.EVM2AnyMessage memory message = IRouterClient.EVM2AnyMessage({
            receiver: abi.encode(receiver),
            data: "",
            tokenAmounts: tokenAmounts,
            feeToken: address(linkToken), // pay in LINK
            extraArgs: ""
        });
        
        uint256 fees = router.getFee(destinationChainSelector, message);
        
        // Approve LINK for fees
        linkToken.approve(address(router), fees);
        
        // Approve token for transfer
        IERC20(token).transferFrom(msg.sender, address(this), amount);
        IERC20(token).approve(address(router), amount);
        
        messageId = router.ccipSend(destinationChainSelector, message);
        
        emit MessageSent(messageId, destinationChainSelector, receiver, amount, fees);
    }
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
    function approve(address, uint256) external returns (bool);
    function balanceOf(address) external view returns (uint256);
}
```

---

## 5. Workshop: Simple Optimistic Bridge

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Optimistic Bridge (Across Protocol style)
 * 
 * LP ใน destination chain จ่ายก่อน (instant)
 * Relayer ส่ง proof → claim refund หลัง challenge period
 */
contract OptimisticBridge {
    
    IERC20 public immutable token;
    
    uint256 public constant CHALLENGE_PERIOD = 1 hours;
    uint256 public constant LP_FEE = 5; // 0.05% fee to LP
    uint256 public constant FEE_DENOMINATOR = 10000;
    
    struct Deposit {
        address depositor;
        address recipient;
        uint256 amount;
        uint256 timestamp;
        bool filled;
    }
    
    mapping(bytes32 => Deposit) public deposits;
    mapping(bytes32 => bool) public filledDeposits;
    
    // LP liquidity
    mapping(address => uint256) public lpBalance;
    uint256 public totalLiquidity;
    
    event DepositMade(bytes32 indexed depositId, address indexed depositor, address recipient, uint256 amount);
    event Filled(bytes32 indexed depositId, address indexed relayer, uint256 repayAmount);
    event LiquidityAdded(address indexed lp, uint256 amount);
    event LiquidityRemoved(address indexed lp, uint256 amount);
    
    constructor(address _token) {
        token = IERC20(_token);
    }
    
    // User deposits tokens (this chain)
    function deposit(
        address recipient,
        uint256 amount,
        bytes32 salt
    ) external returns (bytes32 depositId) {
        token.transferFrom(msg.sender, address(this), amount);
        
        depositId = keccak256(abi.encodePacked(
            msg.sender,
            recipient,
            amount,
            salt,
            block.chainid
        ));
        
        deposits[depositId] = Deposit({
            depositor: msg.sender,
            recipient: recipient,
            amount: amount,
            timestamp: block.timestamp,
            filled: false
        });
        
        emit DepositMade(depositId, msg.sender, recipient, amount);
    }
    
    // LP adds liquidity for instant fills
    function addLiquidity(uint256 amount) external {
        token.transferFrom(msg.sender, address(this), amount);
        lpBalance[msg.sender] += amount;
        totalLiquidity += amount;
        emit LiquidityAdded(msg.sender, amount);
    }
    
    function removeLiquidity(uint256 amount) external {
        require(lpBalance[msg.sender] >= amount, "Insufficient LP");
        lpBalance[msg.sender] -= amount;
        totalLiquidity -= amount;
        token.transfer(msg.sender, amount);
        emit LiquidityRemoved(msg.sender, amount);
    }
    
    // Relayer fills deposit on destination chain (instant)
    function fillDeposit(
        bytes32 depositId,
        address recipient,
        uint256 originalAmount
    ) external {
        require(!filledDeposits[depositId], "Already filled");
        
        filledDeposits[depositId] = true;
        
        // LP fills minus fee
        uint256 fee = (originalAmount * LP_FEE) / FEE_DENOMINATOR;
        uint256 fillAmount = originalAmount - fee;
        
        // Relayer uses their own funds or from LP pool
        token.transferFrom(msg.sender, recipient, fillAmount);
        
        emit Filled(depositId, msg.sender, fillAmount);
    }
    
    // After challenge period: relayer claims from locked funds
    function claimDeposit(bytes32 depositId) external {
        Deposit storage dep = deposits[depositId];
        
        require(dep.depositor != address(0), "Unknown deposit");
        require(!dep.filled, "Already claimed");
        require(
            block.timestamp >= dep.timestamp + CHALLENGE_PERIOD,
            "Challenge period active"
        );
        require(filledDeposits[depositId], "Not filled on destination");
        
        dep.filled = true;
        
        uint256 fee = (dep.amount * LP_FEE) / FEE_DENOMINATOR;
        
        // Relayer gets back original + fee
        token.transfer(msg.sender, dep.amount + fee);
    }
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
    function balanceOf(address) external view returns (uint256);
}
```

---

## สรุป Part 24

Cross-chain Bridges ที่เรียนรู้:
- ✅ Lock-and-Mint pattern
- ✅ Burn-and-Mint pattern
- ✅ LayerZero omnichain messaging
- ✅ Chainlink CCIP
- ✅ Optimistic bridge (instant fills)

## Quiz

1. Lock-and-Mint vs Burn-and-Mint: ข้อดีข้อเสียแต่ละแบบ?
2. ทำไม Bridge ถึงเป็น attack target ยอดนิยม?
3. Challenge period ใน Optimistic Bridge เพื่ออะไร?
4. CCIP ปลอดภัยกว่า LayerZero ในแง่ไหน?

---

## Next: Part 25 - MEV และ Front-Running Protection
