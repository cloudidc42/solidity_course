# Part 33: Account Abstraction (EIP-4337)

## สารบัญ
1. AA Overview
2. UserOperation Flow
3. Smart Wallet Implementation
4. Paymaster (Gasless Transactions)
5. Workshop: Batched Transactions

---

## 1. Account Abstraction Overview

```
Traditional Ethereum Accounts:
- EOA (Externally Owned Account): private key controlled
- Contract Account: code controlled

Problems with EOA:
1. ต้องมี ETH เพื่อจ่าย gas ทุกครั้ง
2. Single private key = single point of failure
3. ไม่สามารถ batch transactions
4. ไม่รองรับ multi-sig อย่าง native

EIP-4337 Account Abstraction:
- Smart contract wallet ที่มี logic ได้
- Bundler: รวบ UserOperations และส่งไป EntryPoint
- Paymaster: ผู้อื่นจ่าย gas แทนได้
- Session Keys: ให้ permission แก่ชั่วคราว

Use Cases:
1. Social Recovery: เพื่อนช่วย recover wallet
2. Multi-sig: ต้องมีหลายคน approve
3. Gasless: dApp จ่าย gas แทน user
4. Batch: หลาย actions ในครั้งเดียว
5. Spending limits: จำกัดการใช้จ่าย
6. Session keys: game, trading bot
```

---

## 2. EntryPoint และ UserOperation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * EIP-4337 Core Interfaces
 * 
 * UserOperation: คำสั่งที่ user ต้องการทำ
 * EntryPoint: contract กลางที่รับ UserOps และ dispatch
 * Account: smart wallet ที่ validate และ execute
 * Paymaster: ผู้จ่าย gas แทน
 */
struct UserOperation {
    address sender;          // smart wallet address
    uint256 nonce;
    bytes initCode;          // create wallet if not exists
    bytes callData;          // what to execute
    uint256 callGasLimit;
    uint256 verificationGasLimit;
    uint256 preVerificationGas;
    uint256 maxFeePerGas;
    uint256 maxPriorityFeePerGas;
    bytes paymasterAndData;  // paymaster address + data
    bytes signature;         // user's signature
}

interface IAccount {
    function validateUserOp(
        UserOperation calldata userOp,
        bytes32 userOpHash,
        uint256 missingAccountFunds
    ) external returns (uint256 validationData);
}

interface IPaymaster {
    function validatePaymasterUserOp(
        UserOperation calldata userOp,
        bytes32 userOpHash,
        uint256 maxCost
    ) external returns (bytes memory context, uint256 validationData);
    
    function postOp(
        PostOpMode mode,
        bytes calldata context,
        uint256 actualGasCost
    ) external;
}

enum PostOpMode { opSucceeded, opReverted, postOpReverted }
```

---

## 3. Smart Wallet (SimpleAccount)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Simple Smart Wallet (EIP-4337 compatible)
 * 
 * Features:
 * - Single owner with ECDSA signature
 * - Batch execute
 * - Upgrade owner key
 */
contract SimpleAccount is IAccount {
    
    address public owner;
    IEntryPoint public immutable entryPoint;
    
    uint256 public nonce;
    
    event WalletCreated(address indexed owner);
    event Execute(address indexed target, uint256 value, bytes data);
    
    modifier onlyEntryPointOrOwner() {
        require(
            msg.sender == address(entryPoint) || msg.sender == owner,
            "Not authorized"
        );
        _;
    }
    
    constructor(address _entryPoint, address _owner) {
        entryPoint = IEntryPoint(_entryPoint);
        owner = _owner;
        emit WalletCreated(_owner);
    }
    
    receive() external payable {}
    
    // EntryPoint calls this to validate the UserOperation
    function validateUserOp(
        UserOperation calldata userOp,
        bytes32 userOpHash,
        uint256 missingAccountFunds
    ) external override returns (uint256 validationData) {
        require(msg.sender == address(entryPoint), "Not EntryPoint");
        
        // Validate signature
        bytes32 hash = keccak256(abi.encodePacked(
            "\x19Ethereum Signed Message:\n32",
            userOpHash
        ));
        
        address signer = _recoverSigner(hash, userOp.signature);
        
        if (signer != owner) {
            return 1; // SIG_VALIDATION_FAILED
        }
        
        // Validate nonce
        if (userOp.nonce != nonce) {
            return 1;
        }
        
        nonce++;
        
        // Pay prefund to EntryPoint if needed
        if (missingAccountFunds > 0) {
            (bool success,) = payable(msg.sender).call{value: missingAccountFunds}("");
            (success); // ignore failure
        }
        
        return 0; // success
    }
    
    // Execute single call
    function execute(
        address target,
        uint256 value,
        bytes calldata data
    ) external onlyEntryPointOrOwner {
        (bool success, bytes memory result) = target.call{value: value}(data);
        if (!success) {
            assembly {
                revert(add(result, 32), mload(result))
            }
        }
        emit Execute(target, value, data);
    }
    
    // Execute batch of calls
    function executeBatch(
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata datas
    ) external onlyEntryPointOrOwner {
        require(targets.length == values.length && values.length == datas.length, "Length mismatch");
        
        for (uint256 i = 0; i < targets.length; i++) {
            (bool success, bytes memory result) = targets[i].call{value: values[i]}(datas[i]);
            if (!success) {
                assembly {
                    revert(add(result, 32), mload(result))
                }
            }
            emit Execute(targets[i], values[i], datas[i]);
        }
    }
    
    function _recoverSigner(bytes32 hash, bytes calldata sig) internal pure returns (address) {
        require(sig.length == 65, "Invalid sig");
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
    
    // Change owner (social recovery could call this)
    function changeOwner(address newOwner) external onlyEntryPointOrOwner {
        owner = newOwner;
    }
}

interface IEntryPoint {
    function getUserOpHash(UserOperation calldata userOp) external view returns (bytes32);
    function depositTo(address account) external payable;
    function balanceOf(address account) external view returns (uint256);
}
```

---

## 4. Paymaster (Gasless Transactions)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Token Paymaster:
 * ให้ user จ่าย gas ด้วย ERC-20 token แทน ETH
 * หรือ sponsor gas ให้ user ฟรีเลย
 */
contract TokenPaymaster is IPaymaster {
    
    IEntryPoint public immutable entryPoint;
    IERC20 public immutable paymentToken;
    
    address public owner;
    
    // ETH price in payment token (e.g., 2000 USDC per ETH)
    uint256 public tokenPerEth = 2000e18;
    
    // Max gas cost to sponsor per UserOp
    uint256 public maxSponsoredGas = 0.001 ether;
    
    mapping(address => bool) public sponsored; // whitelist for free gas
    
    event GasPaidInToken(address indexed user, uint256 tokenAmount, uint256 ethEquivalent);
    
    constructor(address _entryPoint, address _paymentToken) {
        entryPoint = IEntryPoint(_entryPoint);
        paymentToken = IERC20(_paymentToken);
        owner = msg.sender;
    }
    
    // Called before UserOp execution
    function validatePaymasterUserOp(
        UserOperation calldata userOp,
        bytes32, // userOpHash
        uint256 maxCost
    ) external override returns (bytes memory context, uint256 validationData) {
        require(msg.sender == address(entryPoint), "Not EntryPoint");
        
        // Check if user is sponsored (free gas)
        if (sponsored[userOp.sender]) {
            require(maxCost <= maxSponsoredGas, "Too expensive to sponsor");
            return (abi.encode(userOp.sender, true, maxCost), 0);
        }
        
        // Otherwise: charge in payment token
        uint256 tokenCost = (maxCost * tokenPerEth) / 1e18;
        
        // Verify user has approved sufficient tokens
        require(
            paymentToken.allowance(userOp.sender, address(this)) >= tokenCost,
            "Insufficient token allowance"
        );
        
        // Collect tokens upfront (will refund excess in postOp)
        paymentToken.transferFrom(userOp.sender, address(this), tokenCost);
        
        return (abi.encode(userOp.sender, false, tokenCost), 0);
    }
    
    // Called after UserOp execution
    function postOp(
        PostOpMode mode,
        bytes calldata context,
        uint256 actualGasCost
    ) external override {
        require(msg.sender == address(entryPoint), "Not EntryPoint");
        
        (address user, bool isSponsored, uint256 preCharged) = abi.decode(
            context,
            (address, bool, uint256)
        );
        
        if (isSponsored) return; // nothing to do for sponsored
        
        uint256 actualTokenCost = (actualGasCost * tokenPerEth) / 1e18;
        
        // Refund excess tokens
        if (preCharged > actualTokenCost) {
            paymentToken.transfer(user, preCharged - actualTokenCost);
        }
        
        emit GasPaidInToken(user, actualTokenCost, actualGasCost);
    }
    
    // Admin: add to sponsor whitelist
    function setSponsor(address account, bool isSponsor) external {
        require(msg.sender == owner);
        sponsored[account] = isSponsor;
    }
    
    // Deposit ETH to EntryPoint for gas
    function depositToEntryPoint() external payable {
        entryPoint.depositTo{value: msg.value}(address(this));
    }
    
    receive() external payable {}
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
    function allowance(address, address) external view returns (uint256);
}
```

---

## 5. Social Recovery Wallet

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Social Recovery:
 * กำหนด guardians ที่สามารถ recover wallet ได้
 * ต้องการ majority vote จาก guardians
 */
contract SocialRecoveryWallet is SimpleAccount {
    
    address[] public guardians;
    uint256 public threshold; // min guardians required
    
    struct RecoveryRequest {
        address newOwner;
        uint256 approvals;
        mapping(address => bool) approved;
        uint256 createdAt;
    }
    
    mapping(bytes32 => RecoveryRequest) public recoveries;
    
    uint256 public constant RECOVERY_TIMEOUT = 3 days;
    
    event RecoveryInitiated(bytes32 indexed requestId, address newOwner);
    event RecoveryApproved(bytes32 indexed requestId, address guardian);
    event RecoveryExecuted(bytes32 indexed requestId, address newOwner);
    
    constructor(
        address _entryPoint,
        address _owner,
        address[] memory _guardians,
        uint256 _threshold
    ) SimpleAccount(_entryPoint, _owner) {
        require(_threshold > 0 && _threshold <= _guardians.length, "Invalid threshold");
        guardians = _guardians;
        threshold = _threshold;
    }
    
    function isGuardian(address addr) public view returns (bool) {
        for (uint256 i = 0; i < guardians.length; i++) {
            if (guardians[i] == addr) return true;
        }
        return false;
    }
    
    // Guardian initiates recovery
    function initiateRecovery(address newOwner) external returns (bytes32 requestId) {
        require(isGuardian(msg.sender), "Not guardian");
        
        requestId = keccak256(abi.encodePacked(newOwner, block.timestamp));
        
        RecoveryRequest storage req = recoveries[requestId];
        req.newOwner = newOwner;
        req.createdAt = block.timestamp;
        req.approvals = 1;
        req.approved[msg.sender] = true;
        
        emit RecoveryInitiated(requestId, newOwner);
        emit RecoveryApproved(requestId, msg.sender);
    }
    
    // Another guardian approves
    function approveRecovery(bytes32 requestId) external {
        require(isGuardian(msg.sender), "Not guardian");
        
        RecoveryRequest storage req = recoveries[requestId];
        require(req.newOwner != address(0), "Request not found");
        require(!req.approved[msg.sender], "Already approved");
        require(block.timestamp <= req.createdAt + RECOVERY_TIMEOUT, "Expired");
        
        req.approved[msg.sender] = true;
        req.approvals++;
        
        emit RecoveryApproved(requestId, msg.sender);
    }
    
    // Execute recovery when threshold met
    function executeRecovery(bytes32 requestId) external {
        RecoveryRequest storage req = recoveries[requestId];
        
        require(req.approvals >= threshold, "Insufficient approvals");
        require(block.timestamp <= req.createdAt + RECOVERY_TIMEOUT, "Expired");
        
        address newOwner = req.newOwner;
        delete recoveries[requestId];
        
        owner = newOwner; // Change owner
        
        emit RecoveryExecuted(requestId, newOwner);
    }
}
```

---

## สรุป Part 33

Account Abstraction ที่เรียนรู้:
- ✅ EIP-4337 architecture (EntryPoint, Bundler)
- ✅ UserOperation structure
- ✅ Smart wallet with validateUserOp
- ✅ Token Paymaster (gasless UX)
- ✅ Social Recovery wallet

## Quiz

1. ทำไม EIP-4337 ถึงไม่ต้องเปลี่ยน Ethereum protocol?
2. Paymaster ประโยชน์อะไรสำหรับ dApp?
3. Social recovery ดีกว่า seed phrase อย่างไร?
4. Bundler คืออะไร?

---

## Next: Part 34 - Concentrated Liquidity (Uniswap V3)
