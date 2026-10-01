# Part 86: Account Abstraction Advanced (EIP-4337 Deep Dive)

## บทนำ

Account Abstraction (AA) คือหนึ่งในนวัตกรรมที่สำคัญที่สุดใน Ethereum ecosystem ที่ช่วยให้ smart contract สามารถทำหน้าที่เป็น "wallet" ได้อย่างสมบูรณ์ EIP-4337 นำเสนอแนวทางที่ไม่ต้องแก้ไข consensus layer โดยใช้ระบบ UserOperation แทน transaction ธรรมดา

ในบทนี้เราจะศึกษา:
- EntryPoint contract mechanics อย่างละเอียด
- Paymaster patterns หลากหลายรูปแบบ
- Social recovery wallet
- Session keys สำหรับ limited permissions
- Bundler economics

---

## 1. EntryPoint Contract Mechanics

### ความเข้าใจ UserOperation

`UserOperation` คือ pseudo-transaction ที่มีโครงสร้างพิเศษ:

```solidity
struct UserOperation {
    address sender;           // Smart contract wallet address
    uint256 nonce;            // Sequential nonce (ป้องกัน replay attacks)
    bytes initCode;           // Factory calldata สำหรับ deploy wallet ใหม่
    bytes callData;           // Action ที่ wallet จะ execute
    uint256 callGasLimit;     // Gas limit สำหรับ execution phase
    uint256 verificationGasLimit; // Gas limit สำหรับ validation phase
    uint256 preVerificationGas;   // Gas เพิ่มเติมสำหรับ bundler overhead
    uint256 maxFeePerGas;     // EIP-1559 max fee
    uint256 maxPriorityFeePerGas; // EIP-1559 priority fee
    bytes paymasterAndData;   // Paymaster address + data (ถ้ามี)
    bytes signature;          // Wallet owner's signature
}
```

### EntryPoint Flow

```
User → UserOperation → Bundler → EntryPoint.handleOps()
                                       ↓
                              validateUserOp() ← Wallet
                                       ↓
                              validatePaymasterOp() ← Paymaster
                                       ↓
                              executeUserOp() → Wallet
                                       ↓
                              postOp() ← Paymaster
```

### EntryPoint Contract หลัก

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title EntryPoint
 * @notice Core EIP-4337 EntryPoint contract
 * @dev Handles validation and execution of UserOperations
 */
contract EntryPoint is ReentrancyGuard {
    
    // ========== Data Structures ==========
    
    struct UserOperation {
        address sender;
        uint256 nonce;
        bytes initCode;
        bytes callData;
        uint256 callGasLimit;
        uint256 verificationGasLimit;
        uint256 preVerificationGas;
        uint256 maxFeePerGas;
        uint256 maxPriorityFeePerGas;
        bytes paymasterAndData;
        bytes signature;
    }
    
    struct MemoryUserOp {
        address sender;
        uint256 nonce;
        uint256 callGasLimit;
        uint256 verificationGasLimit;
        uint256 preVerificationGas;
        address paymaster;
        uint256 maxFeePerGas;
        uint256 maxPriorityFeePerGas;
    }
    
    struct UserOpInfo {
        MemoryUserOp mUserOp;
        bytes32 userOpHash;
        uint256 prefund;
        uint256 contextOffset;
        uint256 preOpGas;
    }
    
    // ========== State Variables ==========
    
    // depositInfo[account] = DepositInfo
    mapping(address => DepositInfo) public deposits;
    
    // nonceSequenceNumber[account][key] = nonce
    mapping(address => mapping(uint192 => uint256)) public nonceSequenceNumber;
    
    struct DepositInfo {
        uint112 deposit;        // ETH deposited ใน EntryPoint
        bool staked;            // Whether account has staked
        uint112 stake;          // Staked ETH amount
        uint32 unstakeDelaySec; // Delay before unstake allowed
        uint48 withdrawTime;    // Timestamp สำหรับ unstake
    }
    
    uint256 public constant SIG_VALIDATION_FAILED = 1;
    uint256 public constant SIG_VALIDATION_SUCCESS = 0;
    
    // ========== Events ==========
    
    event UserOperationEvent(
        bytes32 indexed userOpHash,
        address indexed sender,
        address indexed paymaster,
        uint256 nonce,
        bool success,
        uint256 actualGasCost,
        uint256 actualGasUsed
    );
    
    event AccountDeployed(
        bytes32 indexed userOpHash,
        address indexed sender,
        address factory,
        address paymaster
    );
    
    event Deposited(address indexed account, uint256 totalDeposit);
    event Withdrawn(address indexed account, address withdrawAddress, uint256 amount);
    
    // ========== Core Functions ==========
    
    /**
     * @notice Main entry point - process batch of UserOperations
     * @param ops Array of UserOperations to process
     * @param beneficiary Address ที่รับ gas refund
     */
    function handleOps(
        UserOperation[] calldata ops,
        address payable beneficiary
    ) external nonReentrant {
        uint256 opslen = ops.length;
        UserOpInfo[] memory opInfos = new UserOpInfo[](opslen);
        
        // Phase 1: Validate all ops
        for (uint256 i = 0; i < opslen; i++) {
            UserOpInfo memory opInfo = opInfos[i];
            (uint256 validationData, uint256 pmValidationData) = _validatePrepayment(i, ops[i], opInfo);
            _validateAccountAndPaymasterValidationData(i, validationData, pmValidationData, address(0));
        }
        
        uint256 collected = 0;
        
        // Phase 2: Execute all ops
        for (uint256 i = 0; i < opslen; i++) {
            collected += _executeUserOp(i, ops[i], opInfos[i]);
        }
        
        // ส่ง collected gas fees ไปให้ beneficiary
        _compensate(beneficiary, collected);
    }
    
    /**
     * @notice Validate UserOperation ก่อน execute
     */
    function _validatePrepayment(
        uint256 opIndex,
        UserOperation calldata userOp,
        UserOpInfo memory outOpInfo
    ) private returns (uint256 validationData, uint256 paymasterValidationData) {
        uint256 preGas = gasleft();
        MemoryUserOp memory mUserOp = outOpInfo.mUserOp;
        _copyUserOpToMemory(userOp, mUserOp);
        outOpInfo.userOpHash = getUserOpHash(userOp);
        
        // Validate nonce
        uint256 maxGasValues = mUserOp.preVerificationGas | mUserOp.verificationGasLimit |
            mUserOp.callGasLimit | userOp.maxFeePerGas | userOp.maxPriorityFeePerGas;
        require(maxGasValues <= type(uint120).max, "AA94 gas values overflow");
        
        uint256 gasUsedByValidateAccountPrepayment;
        (uint256 requiredPreFund) = _getRequiredPrefund(mUserOp);
        
        (gasUsedByValidateAccountPrepayment, validationData) = _validateAccountPrepayment(
            opIndex, userOp, outOpInfo, requiredPreFund
        );
        
        if (!_validateAndUpdateNonce(mUserOp.sender, userOp.nonce)) {
            revert FailedOp(opIndex, "AA25 invalid account nonce");
        }
        
        // ถ้ามี paymaster, validate
        bytes memory context;
        if (mUserOp.paymaster != address(0)) {
            (context, paymasterValidationData) = _validatePaymasterPrepayment(
                opIndex, userOp, outOpInfo, requiredPreFund, gasUsedByValidateAccountPrepayment
            );
        }
        
        unchecked {
            uint256 gasUsed = preGas - gasleft();
            if (userOp.verificationGasLimit < gasUsed) {
                revert FailedOp(opIndex, "AA40 over verificationGasLimit");
            }
            outOpInfo.prefund = requiredPreFund;
            outOpInfo.contextOffset = getOffsetOfMemoryBytes(context);
            outOpInfo.preOpGas = preGas - gasleft() + userOp.preVerificationGas;
        }
    }
    
    /**
     * @notice คำนวณ prefund ที่ต้องการ
     */
    function _getRequiredPrefund(MemoryUserOp memory mUserOp) internal pure returns (uint256 requiredPrefund) {
        unchecked {
            uint256 maxGas = mUserOp.callGasLimit + mUserOp.verificationGasLimit + mUserOp.preVerificationGas;
            if (mUserOp.paymaster != address(0)) {
                maxGas += mUserOp.verificationGasLimit; // paymaster validation gas
            }
            requiredPrefund = maxGas * mUserOp.maxFeePerGas;
        }
    }
    
    /**
     * @notice ดึง nonce ปัจจุบันของ account
     * @param sender Account address
     * @param key Nonce key (upper 192 bits)
     */
    function getNonce(address sender, uint192 key) public view returns (uint256 nonce) {
        return nonceSequenceNumber[sender][key] | (uint256(key) << 64);
    }
    
    /**
     * @notice Validate และ update nonce
     */
    function _validateAndUpdateNonce(address sender, uint256 nonce) internal returns (bool) {
        uint192 key = uint192(nonce >> 64);
        uint64 seq = uint64(nonce);
        return nonceSequenceNumber[sender][key]++ == seq;
    }
    
    /**
     * @notice Execute UserOperation หลัง validation ผ่าน
     */
    function _executeUserOp(
        uint256 opIndex,
        UserOperation calldata userOp,
        UserOpInfo memory opInfo
    ) private returns (uint256 collected) {
        uint256 preGas = gasleft();
        bytes memory context = getMemoryBytesFromOffset(opInfo.contextOffset);
        bool success;
        
        {
            uint256 saveFreePtr;
            assembly { saveFreePtr := mload(0x40) }
            
            bytes calldata callData = userOp.callData;
            bytes memory contextCopy = context;
            
            (bool innerSuccess, ) = address(this).call{
                gas: opInfo.mUserOp.callGasLimit
            }(abi.encodeCall(this.innerHandleOp, (callData, opInfo, contextCopy)));
            
            success = innerSuccess;
            assembly { mstore(0x40, saveFreePtr) }
        }
        
        unchecked {
            uint256 actualGas = preGas - gasleft() + opInfo.preOpGas;
            collected = _postExecution(opInfo, context, actualGas, success);
        }
    }
    
    /**
     * @notice คำนวณ UserOperation hash
     */
    function getUserOpHash(UserOperation calldata userOp) public view returns (bytes32) {
        return keccak256(abi.encode(
            keccak256(abi.encode(
                userOp.sender,
                userOp.nonce,
                keccak256(userOp.initCode),
                keccak256(userOp.callData),
                userOp.callGasLimit,
                userOp.verificationGasLimit,
                userOp.preVerificationGas,
                userOp.maxFeePerGas,
                userOp.maxPriorityFeePerGas,
                keccak256(userOp.paymasterAndData)
            )),
            address(this),
            block.chainid
        ));
    }
    
    /**
     * @notice Deposit ETH เข้า EntryPoint
     */
    function depositTo(address account) public payable {
        _incrementDeposit(account, msg.value);
        DepositInfo storage info = deposits[account];
        emit Deposited(account, info.deposit);
    }
    
    function _incrementDeposit(address account, uint256 amount) internal {
        DepositInfo storage info = deposits[account];
        uint256 newAmount = info.deposit + amount;
        require(newAmount <= type(uint112).max, "deposit overflow");
        info.deposit = uint112(newAmount);
    }
    
    /**
     * @notice Withdraw deposited ETH
     */
    function withdrawTo(address payable withdrawAddress, uint256 withdrawAmount) external {
        DepositInfo storage info = deposits[msg.sender];
        require(withdrawAmount <= info.deposit, "Withdraw amount too large");
        info.deposit = uint112(info.deposit - withdrawAmount);
        emit Withdrawn(msg.sender, withdrawAddress, withdrawAmount);
        (bool success,) = withdrawAddress.call{value: withdrawAmount}("");
        require(success, "failed to withdraw");
    }
    
    // ========== Helper Functions ==========
    
    function _copyUserOpToMemory(UserOperation calldata userOp, MemoryUserOp memory mUserOp) internal pure {
        mUserOp.sender = userOp.sender;
        mUserOp.nonce = userOp.nonce;
        mUserOp.callGasLimit = userOp.callGasLimit;
        mUserOp.verificationGasLimit = userOp.verificationGasLimit;
        mUserOp.preVerificationGas = userOp.preVerificationGas;
        mUserOp.maxFeePerGas = userOp.maxFeePerGas;
        mUserOp.maxPriorityFeePerGas = userOp.maxPriorityFeePerGas;
        bytes calldata paymasterAndData = userOp.paymasterAndData;
        if (paymasterAndData.length > 0) {
            require(paymasterAndData.length >= 20, "AA93 invalid paymasterAndData");
            mUserOp.paymaster = address(bytes20(paymasterAndData[:20]));
        } else {
            mUserOp.paymaster = address(0);
        }
    }
    
    function getOffsetOfMemoryBytes(bytes memory data) internal pure returns (uint256 offset) {
        assembly { offset := data }
    }
    
    function getMemoryBytesFromOffset(uint256 offset) internal pure returns (bytes memory data) {
        assembly { data := offset }
    }
    
    error FailedOp(uint256 opIndex, string reason);
    
    // Stub functions (implement in full version)
    function _validateAccountPrepayment(uint256, UserOperation calldata, UserOpInfo memory, uint256) 
        internal returns (uint256, uint256) { return (0, 0); }
    function _validatePaymasterPrepayment(uint256, UserOperation calldata, UserOpInfo memory, uint256, uint256)
        internal returns (bytes memory, uint256) { return ("", 0); }
    function _validateAccountAndPaymasterValidationData(uint256, uint256, uint256, address) internal {}
    function _postExecution(UserOpInfo memory, bytes memory, uint256, bool) internal returns (uint256) { return 0; }
    function _compensate(address payable, uint256) internal {}
    function innerHandleOp(bytes calldata, UserOpInfo calldata, bytes calldata) external returns (uint256) { return 0; }
    
    receive() external payable {
        depositTo(msg.sender);
    }
}
```

---

## 2. Smart Contract Wallet

### BaseAccount Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title IAccount
 * @notice Interface ที่ wallet contract ต้อง implement
 */
interface IAccount {
    /**
     * @notice Validate UserOperation
     * @param userOp UserOperation ที่ต้อง validate
     * @param userOpHash Hash ของ UserOperation
     * @param missingAccountFunds ETH ที่ต้อง deposit เพิ่ม
     * @return validationData Packed validation result
     */
    function validateUserOp(
        UserOperation calldata userOp,
        bytes32 userOpHash,
        uint256 missingAccountFunds
    ) external returns (uint256 validationData);
}

/**
 * @title SimpleSmartWallet
 * @notice Basic EIP-4337 compatible smart wallet
 */
contract SimpleSmartWallet {
    
    address public immutable entryPoint;
    address public owner;
    uint256 private _nonce;
    
    event WalletInitialized(address indexed entryPoint, address indexed owner);
    event Execute(address indexed target, uint256 value, bytes data);
    
    modifier onlyEntryPoint() {
        require(msg.sender == entryPoint, "not from EntryPoint");
        _;
    }
    
    modifier onlyOwnerOrEntryPoint() {
        require(
            msg.sender == owner || msg.sender == entryPoint,
            "not owner or entrypoint"
        );
        _;
    }
    
    constructor(address _entryPoint, address _owner) {
        entryPoint = _entryPoint;
        owner = _owner;
        emit WalletInitialized(_entryPoint, _owner);
    }
    
    /**
     * @notice Validate UserOperation
     * @dev Validate signature และ pay prefund ถ้าต้องการ
     */
    function validateUserOp(
        EntryPoint.UserOperation calldata userOp,
        bytes32 userOpHash,
        uint256 missingAccountFunds
    ) external onlyEntryPoint returns (uint256 validationData) {
        validationData = _validateSignature(userOp, userOpHash);
        
        // Pay prefund ถ้าต้องการ
        if (missingAccountFunds != 0) {
            (bool success,) = payable(entryPoint).call{value: missingAccountFunds}("");
            (success); // Ignore failure (deposit เพิ่มได้ภายหลัง)
        }
    }
    
    /**
     * @notice Validate ECDSA signature
     */
    function _validateSignature(
        EntryPoint.UserOperation calldata userOp,
        bytes32 userOpHash
    ) internal view returns (uint256 validationData) {
        bytes32 hash = keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", userOpHash));
        address signer = _recover(hash, userOp.signature);
        
        if (signer != owner) {
            return 1; // SIG_VALIDATION_FAILED
        }
        return 0; // SIG_VALIDATION_SUCCESS
    }
    
    /**
     * @notice Execute single call
     */
    function execute(
        address dest,
        uint256 value,
        bytes calldata func
    ) external onlyEntryPoint {
        _call(dest, value, func);
        emit Execute(dest, value, func);
    }
    
    /**
     * @notice Execute batch of calls
     */
    function executeBatch(
        address[] calldata dests,
        uint256[] calldata values,
        bytes[] calldata funcs
    ) external onlyEntryPoint {
        require(dests.length == funcs.length, "wrong array lengths");
        require(values.length == 0 || values.length == dests.length, "wrong values array length");
        
        for (uint256 i = 0; i < dests.length; i++) {
            uint256 value = values.length == 0 ? 0 : values[i];
            _call(dests[i], value, funcs[i]);
        }
    }
    
    function _call(address target, uint256 value, bytes memory data) internal {
        (bool success, bytes memory result) = target.call{value: value}(data);
        if (!success) {
            assembly {
                revert(add(result, 32), mload(result))
            }
        }
    }
    
    function _recover(bytes32 hash, bytes memory sig) internal pure returns (address) {
        require(sig.length == 65, "invalid signature length");
        bytes32 r;
        bytes32 s;
        uint8 v;
        assembly {
            r := mload(add(sig, 32))
            s := mload(add(sig, 64))
            v := byte(0, mload(add(sig, 96)))
        }
        return ecrecover(hash, v, r, s);
    }
    
    receive() external payable {}
}
```

---

## 3. Paymaster Patterns

### 3.1 VerifyingPaymaster - สนับสนุน Gas สำหรับ Users เฉพาะ

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";

/**
 * @title VerifyingPaymaster
 * @notice Paymaster ที่ตรวจสอบ off-chain signature เพื่อ sponsor gas
 * @dev ใช้สำหรับ sponsoring specific users หรือ use cases
 */
contract VerifyingPaymaster is Ownable {
    using ECDSA for bytes32;
    
    address public immutable entryPoint;
    address public verifyingSigner;  // Off-chain signer ที่ authorize operations
    
    // Nonce สำหรับแต่ละ sender เพื่อป้องกัน replay
    mapping(address => uint256) public senderNonce;
    
    event SignerUpdated(address indexed oldSigner, address indexed newSigner);
    event GasSponsored(address indexed sender, bytes32 indexed userOpHash, uint256 gasCost);
    
    struct PaymasterAndData {
        address paymaster;      // Address ของ paymaster (20 bytes)
        uint48 validUntil;      // Unix timestamp หมดอายุ
        uint48 validAfter;      // Unix timestamp เริ่มต้น
        bytes signature;        // Off-chain signature
    }
    
    constructor(
        address _entryPoint,
        address _verifyingSigner
    ) Ownable(msg.sender) {
        entryPoint = _entryPoint;
        verifyingSigner = _verifyingSigner;
    }
    
    /**
     * @notice อัพเดท verifying signer
     */
    function setVerifyingSigner(address _newSigner) external onlyOwner {
        emit SignerUpdated(verifyingSigner, _newSigner);
        verifyingSigner = _newSigner;
    }
    
    /**
     * @notice Deposit ETH เข้า EntryPoint เพื่อ sponsor gas
     */
    function deposit() public payable {
        IEntryPoint(entryPoint).depositTo{value: msg.value}(address(this));
    }
    
    /**
     * @notice Withdraw ETH จาก EntryPoint
     */
    function withdrawTo(address payable withdrawAddress, uint256 amount) external onlyOwner {
        IEntryPoint(entryPoint).withdrawTo(withdrawAddress, amount);
    }
    
    /**
     * @notice Validate paymaster UserOp
     * @dev Called by EntryPoint during validation phase
     */
    function validatePaymasterUserOp(
        UserOperation calldata userOp,
        bytes32 userOpHash,
        uint256 maxCost
    ) external returns (bytes memory context, uint256 validationData) {
        require(msg.sender == entryPoint, "not from entrypoint");
        
        (uint48 validUntil, uint48 validAfter, bytes calldata signature) = parsePaymasterAndData(
            userOp.paymasterAndData
        );
        
        // Construct hash ที่ off-chain signer ต้อง sign
        bytes32 hash = getHash(userOp, validUntil, validAfter);
        
        // Verify signature
        address recovered = hash.toEthSignedMessageHash().recover(signature);
        bool sigFailed = recovered != verifyingSigner;
        
        // Pack validation data: sigFailed | validUntil | validAfter
        validationData = _packValidationData(sigFailed, validUntil, validAfter);
        
        // Context สำหรับ postOp
        context = abi.encode(userOp.sender, userOpHash, maxCost);
    }
    
    /**
     * @notice Called after UserOp execution
     */
    function postOp(
        PostOpMode mode,
        bytes calldata context,
        uint256 actualGasCost
    ) external {
        require(msg.sender == entryPoint, "not from entrypoint");
        
        (address sender, bytes32 userOpHash, ) = abi.decode(context, (address, bytes32, uint256));
        
        if (mode != PostOpMode.postOpReverted) {
            emit GasSponsored(sender, userOpHash, actualGasCost);
        }
    }
    
    /**
     * @notice คำนวณ hash สำหรับ signing
     */
    function getHash(
        UserOperation calldata userOp,
        uint48 validUntil,
        uint48 validAfter
    ) public view returns (bytes32) {
        return keccak256(abi.encode(
            userOp.sender,
            userOp.nonce,
            keccak256(userOp.initCode),
            keccak256(userOp.callData),
            userOp.callGasLimit,
            userOp.verificationGasLimit,
            userOp.preVerificationGas,
            userOp.maxFeePerGas,
            userOp.maxPriorityFeePerGas,
            block.chainid,
            address(this),
            validUntil,
            validAfter,
            senderNonce[userOp.sender]
        ));
    }
    
    function parsePaymasterAndData(bytes calldata paymasterAndData) 
        public pure returns (uint48 validUntil, uint48 validAfter, bytes calldata signature) {
        // paymasterAndData format: [paymaster(20)][validUntil(6)][validAfter(6)][signature]
        validUntil = uint48(bytes6(paymasterAndData[20:26]));
        validAfter = uint48(bytes6(paymasterAndData[26:32]));
        signature = paymasterAndData[32:];
    }
    
    function _packValidationData(bool sigFailed, uint48 validUntil, uint48 validAfter) 
        internal pure returns (uint256) {
        return (sigFailed ? 1 : 0) | (uint256(validUntil) << 160) | (uint256(validAfter) << 208);
    }
    
    enum PostOpMode { opSucceeded, opReverted, postOpReverted }
}
```

### 3.2 TokenPaymaster - จ่าย Gas ด้วย ERC-20

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title TokenPaymaster
 * @notice Paymaster ที่รับการชำระ gas เป็น ERC-20 token
 * @dev ใช้ Chainlink oracle สำหรับ ETH/Token price conversion
 */
contract TokenPaymaster is Ownable {
    using SafeERC20 for IERC20;
    
    address public immutable entryPoint;
    IERC20 public immutable token;
    
    // Price oracle (Chainlink compatible)
    address public priceOracle;
    
    // Markup สำหรับ gas price (ใน basis points, 10000 = 100%)
    uint256 public priceMarkup = 11000; // 110% markup (10% overhead)
    
    // Minimum token balance ที่ user ต้องมี
    uint256 public minTokenBalance;
    
    event TokensCharged(address indexed sender, uint256 tokenAmount, uint256 gasUsed);
    event PriceMarkupUpdated(uint256 oldMarkup, uint256 newMarkup);
    
    constructor(
        address _entryPoint,
        address _token,
        address _priceOracle
    ) Ownable(msg.sender) {
        entryPoint = _entryPoint;
        token = IERC20(_token);
        priceOracle = _priceOracle;
    }
    
    /**
     * @notice Validate paymaster - ตรวจสอบว่า user มี token เพียงพอ
     */
    function validatePaymasterUserOp(
        UserOperation calldata userOp,
        bytes32, // userOpHash - unused
        uint256 maxCost
    ) external returns (bytes memory context, uint256 validationData) {
        require(msg.sender == entryPoint, "not from entrypoint");
        
        // คำนวณ token cost จาก ETH cost
        uint256 tokenAmount = getTokenValueOfEth(maxCost * priceMarkup / 10000);
        
        // ตรวจสอบ allowance
        require(
            token.allowance(userOp.sender, address(this)) >= tokenAmount,
            "TokenPaymaster: insufficient allowance"
        );
        
        // ตรวจสอบ balance
        require(
            token.balanceOf(userOp.sender) >= tokenAmount,
            "TokenPaymaster: insufficient balance"
        );
        
        // Context สำหรับ postOp: ส่ง token price ณ เวลา validation
        uint256 tokenPrice = getTokenPrice();
        context = abi.encode(userOp.sender, tokenPrice, maxCost);
        validationData = 0; // Success
    }
    
    /**
     * @notice Post-op: เก็บ token จาก user
     */
    function postOp(
        PostOpMode mode,
        bytes calldata context,
        uint256 actualGasCost
    ) external {
        require(msg.sender == entryPoint, "not from entrypoint");
        
        if (mode == PostOpMode.postOpReverted) {
            return; // ไม่เก็บ token ถ้า postOp reverted
        }
        
        (address sender, uint256 tokenPrice, ) = abi.decode(context, (address, uint256, uint256));
        
        // คำนวณ token amount จาก actual gas cost
        uint256 tokenAmount = (actualGasCost * priceMarkup / 10000) * 1e18 / tokenPrice;
        
        // เก็บ token จาก user
        token.safeTransferFrom(sender, address(this), tokenAmount);
        
        emit TokensCharged(sender, tokenAmount, actualGasCost);
    }
    
    /**
     * @notice คำนวณ token value จาก ETH amount
     */
    function getTokenValueOfEth(uint256 ethAmount) public view returns (uint256) {
        uint256 tokenPrice = getTokenPrice(); // Token price ใน ETH (18 decimals)
        return ethAmount * 1e18 / tokenPrice;
    }
    
    /**
     * @notice ดึง token price จาก oracle
     */
    function getTokenPrice() public view returns (uint256) {
        // Chainlink oracle call
        (,int256 price,,,) = AggregatorV3Interface(priceOracle).latestRoundData();
        require(price > 0, "invalid oracle price");
        return uint256(price) * 1e10; // Convert to 18 decimals
    }
    
    /**
     * @notice Owner ถอน accumulated tokens
     */
    function withdrawTokens(address to, uint256 amount) external onlyOwner {
        token.safeTransfer(to, amount);
    }
    
    function setPriceMarkup(uint256 _markup) external onlyOwner {
        require(_markup >= 10000, "markup too low");
        require(_markup <= 20000, "markup too high");
        emit PriceMarkupUpdated(priceMarkup, _markup);
        priceMarkup = _markup;
    }
    
    enum PostOpMode { opSucceeded, opReverted, postOpReverted }
}

interface AggregatorV3Interface {
    function latestRoundData() external view returns (
        uint80 roundId,
        int256 answer,
        uint256 startedAt,
        uint256 updatedAt,
        uint80 answeredInRound
    );
}
```

### 3.3 DepositPaymaster - Prepaid Balance System

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title DepositPaymaster
 * @notice Paymaster แบบ prepaid - user deposit ETH ก่อนใช้งาน
 * @dev คล้าย mobile top-up credit system
 */
contract DepositPaymaster is Ownable {
    
    address public immutable entryPoint;
    
    // User deposits (address => balance in wei)
    mapping(address => uint256) public balances;
    
    // Minimum deposit required
    uint256 public constant MIN_DEPOSIT = 0.001 ether;
    
    // Lock period for security (ป้องกัน frontrunning)
    uint256 public constant LOCK_PERIOD = 1 days;
    
    struct UnlockRequest {
        uint256 amount;
        uint256 unlockTime;
    }
    
    mapping(address => UnlockRequest) public unlockRequests;
    
    event Deposited(address indexed user, uint256 amount);
    event WithdrawRequested(address indexed user, uint256 amount, uint256 unlockTime);
    event Withdrawn(address indexed user, uint256 amount);
    event GasCharged(address indexed user, uint256 amount);
    
    constructor(address _entryPoint) Ownable(msg.sender) {
        entryPoint = _entryPoint;
    }
    
    /**
     * @notice Deposit ETH สำหรับ gas sponsoring
     */
    function addDepositFor(address user) external payable {
        require(msg.value >= MIN_DEPOSIT, "deposit too small");
        balances[user] += msg.value;
        IEntryPoint(entryPoint).depositTo{value: msg.value}(address(this));
        emit Deposited(user, msg.value);
    }
    
    /**
     * @notice Request withdrawal (ต้องรอ lock period)
     */
    function requestWithdraw(uint256 amount) external {
        require(balances[msg.sender] >= amount, "insufficient balance");
        require(unlockRequests[msg.sender].amount == 0, "pending unlock exists");
        
        balances[msg.sender] -= amount;
        unlockRequests[msg.sender] = UnlockRequest({
            amount: amount,
            unlockTime: block.timestamp + LOCK_PERIOD
        });
        
        emit WithdrawRequested(msg.sender, amount, block.timestamp + LOCK_PERIOD);
    }
    
    /**
     * @notice Execute withdrawal หลัง lock period หมด
     */
    function executeWithdraw() external {
        UnlockRequest memory req = unlockRequests[msg.sender];
        require(req.amount > 0, "no pending unlock");
        require(block.timestamp >= req.unlockTime, "still locked");
        
        delete unlockRequests[msg.sender];
        
        // ถอนจาก EntryPoint ก่อน
        IEntryPoint(entryPoint).withdrawTo(payable(address(this)), req.amount);
        
        (bool success,) = msg.sender.call{value: req.amount}("");
        require(success, "transfer failed");
        
        emit Withdrawn(msg.sender, req.amount);
    }
    
    /**
     * @notice Validate - ตรวจสอบ balance เพียงพอ
     */
    function validatePaymasterUserOp(
        UserOperation calldata userOp,
        bytes32,
        uint256 maxCost
    ) external returns (bytes memory context, uint256 validationData) {
        require(msg.sender == entryPoint, "not from entrypoint");
        require(balances[userOp.sender] >= maxCost, "DepositPaymaster: insufficient balance");
        
        // Lock funds during execution
        balances[userOp.sender] -= maxCost;
        
        context = abi.encode(userOp.sender, maxCost);
        validationData = 0;
    }
    
    /**
     * @notice Post-op: คืนเงินส่วนเกิน
     */
    function postOp(
        PostOpMode mode,
        bytes calldata context,
        uint256 actualGasCost
    ) external {
        require(msg.sender == entryPoint, "not from entrypoint");
        
        (address sender, uint256 maxCost) = abi.decode(context, (address, uint256));
        
        // คืน unused gas
        uint256 refund = maxCost - actualGasCost;
        if (refund > 0) {
            balances[sender] += refund;
        }
        
        emit GasCharged(sender, actualGasCost);
    }
    
    enum PostOpMode { opSucceeded, opReverted, postOpReverted }
    
    receive() external payable {}
}
```

---

## 4. Social Recovery Wallet

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/utils/structs/EnumerableSet.sol";

/**
 * @title SocialRecoveryWallet
 * @notice Smart wallet ที่รองรับ social recovery ผ่าน guardians
 * @dev Guardians สามารถช่วย recover wallet ในกรณีที่ owner เสีย key
 */
contract SocialRecoveryWallet {
    using EnumerableSet for EnumerableSet.AddressSet;
    
    // ========== State Variables ==========
    
    address public owner;
    address public immutable entryPoint;
    
    EnumerableSet.AddressSet private _guardians;
    
    uint256 public recoveryThreshold;  // จำนวน guardian ที่ต้อง approve
    uint256 public recoveryDelay;      // Time lock delay (seconds)
    
    struct RecoveryRequest {
        address newOwner;           // Owner ใหม่ที่เสนอ
        uint256 executeAfter;       // Timestamp ที่ execute ได้
        uint256 approvalCount;      // จำนวน guardian ที่ approve แล้ว
        mapping(address => bool) approvals; // Guardian approvals
        bool executed;              // ถูก execute แล้วหรือไม่
    }
    
    uint256 public currentRecoveryRound;
    mapping(uint256 => RecoveryRequest) public recoveryRequests;
    
    // ========== Events ==========
    
    event GuardianAdded(address indexed guardian);
    event GuardianRemoved(address indexed guardian);
    event RecoveryInitiated(uint256 indexed round, address indexed newOwner, uint256 executeAfter);
    event RecoveryApproved(uint256 indexed round, address indexed guardian);
    event RecoveryExecuted(uint256 indexed round, address indexed newOwner);
    event RecoveryCancelled(uint256 indexed round);
    event ThresholdUpdated(uint256 oldThreshold, uint256 newThreshold);
    
    // ========== Modifiers ==========
    
    modifier onlyOwner() {
        require(msg.sender == owner, "not owner");
        _;
    }
    
    modifier onlyGuardian() {
        require(_guardians.contains(msg.sender), "not guardian");
        _;
    }
    
    modifier onlyEntryPointOrOwner() {
        require(
            msg.sender == entryPoint || msg.sender == owner,
            "not authorized"
        );
        _;
    }
    
    // ========== Constructor ==========
    
    constructor(
        address _entryPoint,
        address _owner,
        address[] memory _initialGuardians,
        uint256 _threshold,
        uint256 _recoveryDelay
    ) {
        require(_initialGuardians.length >= _threshold, "threshold too high");
        require(_threshold > 0, "zero threshold");
        
        entryPoint = _entryPoint;
        owner = _owner;
        recoveryThreshold = _threshold;
        recoveryDelay = _recoveryDelay;
        
        for (uint256 i = 0; i < _initialGuardians.length; i++) {
            require(_initialGuardians[i] != address(0), "zero guardian address");
            require(!_guardians.contains(_initialGuardians[i]), "duplicate guardian");
            _guardians.add(_initialGuardians[i]);
            emit GuardianAdded(_initialGuardians[i]);
        }
    }
    
    // ========== Guardian Management ==========
    
    /**
     * @notice เพิ่ม guardian ใหม่
     */
    function addGuardian(address guardian) external onlyOwner {
        require(guardian != address(0), "zero address");
        require(!_guardians.contains(guardian), "already guardian");
        _guardians.add(guardian);
        emit GuardianAdded(guardian);
    }
    
    /**
     * @notice ลบ guardian
     */
    function removeGuardian(address guardian) external onlyOwner {
        require(_guardians.contains(guardian), "not guardian");
        require(_guardians.length() > recoveryThreshold, "would break threshold");
        _guardians.remove(guardian);
        emit GuardianRemoved(guardian);
    }
    
    /**
     * @notice อัพเดท threshold
     */
    function setRecoveryThreshold(uint256 newThreshold) external onlyOwner {
        require(newThreshold > 0, "zero threshold");
        require(newThreshold <= _guardians.length(), "threshold too high");
        emit ThresholdUpdated(recoveryThreshold, newThreshold);
        recoveryThreshold = newThreshold;
    }
    
    // ========== Recovery Process ==========
    
    /**
     * @notice เริ่ม recovery process
     * @dev Guardian คนแรกที่ initiate จะ start timer
     */
    function initiateRecovery(address newOwner) external onlyGuardian {
        require(newOwner != address(0), "zero new owner");
        require(newOwner != owner, "same owner");
        
        // Cancel previous pending recovery ถ้ามี
        uint256 round = currentRecoveryRound;
        if (round > 0 && !recoveryRequests[round].executed) {
            // Allow new recovery to override old one
        }
        
        currentRecoveryRound++;
        uint256 newRound = currentRecoveryRound;
        
        RecoveryRequest storage request = recoveryRequests[newRound];
        request.newOwner = newOwner;
        request.executeAfter = block.timestamp + recoveryDelay;
        request.approvalCount = 1;
        request.approvals[msg.sender] = true;
        
        emit RecoveryInitiated(newRound, newOwner, request.executeAfter);
        emit RecoveryApproved(newRound, msg.sender);
    }
    
    /**
     * @notice Guardian approve recovery request
     */
    function approveRecovery(uint256 round) external onlyGuardian {
        RecoveryRequest storage request = recoveryRequests[round];
        require(request.newOwner != address(0), "no such recovery");
        require(!request.executed, "already executed");
        require(!request.approvals[msg.sender], "already approved");
        
        request.approvals[msg.sender] = true;
        request.approvalCount++;
        
        emit RecoveryApproved(round, msg.sender);
    }
    
    /**
     * @notice Execute recovery หลัง threshold และ time lock ผ่าน
     */
    function executeRecovery(uint256 round) external {
        RecoveryRequest storage request = recoveryRequests[round];
        require(request.newOwner != address(0), "no such recovery");
        require(!request.executed, "already executed");
        require(request.approvalCount >= recoveryThreshold, "not enough approvals");
        require(block.timestamp >= request.executeAfter, "time lock not expired");
        
        request.executed = true;
        address oldOwner = owner;
        owner = request.newOwner;
        
        emit RecoveryExecuted(round, request.newOwner);
    }
    
    /**
     * @notice Owner cancel pending recovery
     */
    function cancelRecovery(uint256 round) external onlyOwner {
        RecoveryRequest storage request = recoveryRequests[round];
        require(!request.executed, "already executed");
        
        // Mark as executed with zero address to prevent future execution
        request.executed = true;
        emit RecoveryCancelled(round);
    }
    
    // ========== View Functions ==========
    
    function getGuardians() external view returns (address[] memory) {
        return _guardians.values();
    }
    
    function isGuardian(address account) external view returns (bool) {
        return _guardians.contains(account);
    }
    
    function getRecoveryApprovalCount(uint256 round) external view returns (uint256) {
        return recoveryRequests[round].approvalCount;
    }
    
    function hasApproved(uint256 round, address guardian) external view returns (bool) {
        return recoveryRequests[round].approvals[guardian];
    }
    
    // ========== Wallet Execution ==========
    
    function execute(address dest, uint256 value, bytes calldata data) external onlyEntryPointOrOwner {
        (bool success, bytes memory result) = dest.call{value: value}(data);
        if (!success) {
            assembly { revert(add(result, 32), mload(result)) }
        }
    }
    
    receive() external payable {}
}
```

---

## 5. Session Keys

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title SessionKeyManager
 * @notice จัดการ session keys ที่มี limited permissions
 * @dev Session key = temporary key ที่ใช้สำหรับ specific actions เท่านั้น
 *      เหมาะสำหรับ gaming, DeFi automation, mobile apps
 */
contract SessionKeyManager {
    
    struct SessionKeyData {
        address sessionKey;          // Temporary key address
        uint48 validAfter;           // เริ่มใช้ได้เมื่อ
        uint48 validUntil;           // หมดอายุเมื่อ
        
        // Token allowances
        mapping(address => uint256) tokenAllowances; // token => max amount per tx
        
        // Method restrictions
        mapping(bytes4 => bool) allowedMethods;  // selector => allowed
        
        // Target restrictions
        mapping(address => bool) allowedTargets; // target => allowed
        
        // Spending limits
        uint256 dailyEthLimit;       // ETH ที่ใช้ได้ต่อวัน
        uint256 dailyEthUsed;        // ETH ที่ใช้ไปแล้ววันนี้
        uint256 lastResetDay;        // วันที่ reset ล่าสุด
        
        bool isRevoked;              // ถูก revoke แล้วหรือไม่
    }
    
    address public owner;
    
    // sessionKey address => data
    mapping(address => SessionKeyData) public sessionKeys;
    
    event SessionKeyAdded(
        address indexed sessionKey,
        uint48 validAfter,
        uint48 validUntil
    );
    event SessionKeyRevoked(address indexed sessionKey);
    event SessionKeyUsed(address indexed sessionKey, address target, bytes4 selector);
    
    modifier onlyOwner() {
        require(msg.sender == owner, "not owner");
        _;
    }
    
    constructor(address _owner) {
        owner = _owner;
    }
    
    /**
     * @notice สร้าง session key ใหม่
     */
    function createSessionKey(
        address sessionKey,
        uint48 validAfter,
        uint48 validUntil,
        uint256 dailyEthLimit
    ) external onlyOwner {
        require(sessionKey != address(0), "zero session key");
        require(validUntil > validAfter, "invalid time range");
        require(validUntil > block.timestamp, "already expired");
        
        SessionKeyData storage data = sessionKeys[sessionKey];
        data.sessionKey = sessionKey;
        data.validAfter = validAfter;
        data.validUntil = validUntil;
        data.dailyEthLimit = dailyEthLimit;
        data.lastResetDay = block.timestamp / 1 days;
        data.isRevoked = false;
        
        emit SessionKeyAdded(sessionKey, validAfter, validUntil);
    }
    
    /**
     * @notice เพิ่ม token allowance ให้ session key
     */
    function setTokenAllowance(
        address sessionKey,
        address token,
        uint256 maxAmountPerTx
    ) external onlyOwner {
        sessionKeys[sessionKey].tokenAllowances[token] = maxAmountPerTx;
    }
    
    /**
     * @notice อนุญาต method selector สำหรับ session key
     */
    function addAllowedMethod(address sessionKey, bytes4 selector) external onlyOwner {
        sessionKeys[sessionKey].allowedMethods[selector] = true;
    }
    
    /**
     * @notice อนุญาต target contract สำหรับ session key
     */
    function addAllowedTarget(address sessionKey, address target) external onlyOwner {
        sessionKeys[sessionKey].allowedTargets[target] = true;
    }
    
    /**
     * @notice Revoke session key
     */
    function revokeSessionKey(address sessionKey) external onlyOwner {
        sessionKeys[sessionKey].isRevoked = true;
        emit SessionKeyRevoked(sessionKey);
    }
    
    /**
     * @notice Validate session key สำหรับ specific operation
     */
    function validateSessionKeyOp(
        address sessionKey,
        address target,
        uint256 ethValue,
        bytes calldata callData
    ) external returns (bool) {
        SessionKeyData storage data = sessionKeys[sessionKey];
        
        // ตรวจสอบ validity
        require(!data.isRevoked, "session key revoked");
        require(data.sessionKey != address(0), "session key not found");
        require(block.timestamp >= data.validAfter, "session key not yet valid");
        require(block.timestamp <= data.validUntil, "session key expired");
        
        // ตรวจสอบ target
        if (data.allowedTargets[target] == false) {
            // ถ้าไม่มี target restrictions, skip
            // ถ้ามี, ต้อง allow ก่อน
        }
        
        // ตรวจสอบ method selector
        if (callData.length >= 4) {
            bytes4 selector = bytes4(callData[:4]);
            require(
                data.allowedMethods[selector] || _isDefaultAllowed(selector),
                "method not allowed"
            );
            emit SessionKeyUsed(sessionKey, target, selector);
        }
        
        // ตรวจสอบ ETH daily limit
        if (ethValue > 0) {
            _checkAndUpdateDailyLimit(data, ethValue);
        }
        
        return true;
    }
    
    /**
     * @notice ตรวจสอบและอัพเดท daily ETH limit
     */
    function _checkAndUpdateDailyLimit(SessionKeyData storage data, uint256 ethValue) internal {
        uint256 currentDay = block.timestamp / 1 days;
        
        // Reset ถ้าเป็นวันใหม่
        if (currentDay > data.lastResetDay) {
            data.dailyEthUsed = 0;
            data.lastResetDay = currentDay;
        }
        
        require(
            data.dailyEthUsed + ethValue <= data.dailyEthLimit,
            "daily ETH limit exceeded"
        );
        data.dailyEthUsed += ethValue;
    }
    
    /**
     * @notice Default allowed methods (basic transfers)
     */
    function _isDefaultAllowed(bytes4 selector) internal pure returns (bool) {
        return selector == bytes4(keccak256("transfer(address,uint256)"));
    }
    
    /**
     * @notice Query session key info
     */
    function getSessionKeyInfo(address sessionKey) external view returns (
        uint48 validAfter,
        uint48 validUntil,
        uint256 dailyEthLimit,
        uint256 dailyEthUsed,
        bool isRevoked
    ) {
        SessionKeyData storage data = sessionKeys[sessionKey];
        return (
            data.validAfter,
            data.validUntil,
            data.dailyEthLimit,
            data.dailyEthUsed,
            data.isRevoked
        );
    }
    
    function getTokenAllowance(address sessionKey, address token) external view returns (uint256) {
        return sessionKeys[sessionKey].tokenAllowances[token];
    }
}
```

---

## 6. Bundler Economics

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title BundlerSimulator
 * @notice Simulate bundler gas calculations และ optimization
 * @dev ใช้ offline สำหรับ estimate ก่อน submit
 */
contract BundlerSimulator {
    
    struct GasEstimation {
        uint256 preVerificationGas;   // Fixed overhead per op
        uint256 verificationGasLimit; // Gas สำหรับ validateUserOp
        uint256 callGasLimit;         // Gas สำหรับ actual execution
        uint256 totalGas;             // Total gas estimate
        uint256 maxFeePerGas;         // Suggested max fee
        uint256 maxPriorityFeePerGas; // Suggested priority fee
    }
    
    // Constants
    uint256 constant FIXED_OVERHEAD = 21000;      // Base tx overhead
    uint256 constant PER_USER_OP = 18300;         // Per UserOp overhead
    uint256 constant PER_WORD = 4;                // Per word in calldata
    uint256 constant ZERO_BYTE_GAS = 4;           // Gas per zero byte
    uint256 constant NONZERO_BYTE_GAS = 16;       // Gas per nonzero byte
    
    /**
     * @notice คำนวณ preVerificationGas
     * @dev Gas ที่ bundler ต้องจ่ายก่อน EntryPoint เรียก handleOps
     */
    function calcPreVerificationGas(
        UserOperation calldata userOp,
        bool forSimulation
    ) public pure returns (uint256) {
        bytes memory packed = packUserOp(userOp);
        uint256 lengthInWords = (packed.length + 31) / 32;
        uint256 callDataCost = 0;
        
        for (uint256 i = 0; i < packed.length; i++) {
            callDataCost += packed[i] == 0 ? ZERO_BYTE_GAS : NONZERO_BYTE_GAS;
        }
        
        uint256 ret = callDataCost + 
            (forSimulation ? 0 : PER_USER_OP) + 
            lengthInWords * PER_WORD;
        
        return ret;
    }
    
    /**
     * @notice Pack UserOperation เป็น bytes
     */
    function packUserOp(UserOperation calldata userOp) public pure returns (bytes memory) {
        return abi.encode(
            userOp.sender,
            userOp.nonce,
            keccak256(userOp.initCode),
            keccak256(userOp.callData),
            userOp.callGasLimit,
            userOp.verificationGasLimit,
            userOp.preVerificationGas,
            userOp.maxFeePerGas,
            userOp.maxPriorityFeePerGas,
            keccak256(userOp.paymasterAndData)
        );
    }
    
    /**
     * @notice คำนวณ total cost ของ UserOperation
     */
    function calcUserOpCost(
        UserOperation calldata userOp,
        uint256 gasPrice
    ) public pure returns (uint256 maxCost, uint256 estimatedCost) {
        uint256 totalGas = userOp.callGasLimit + 
            userOp.verificationGasLimit + 
            userOp.preVerificationGas;
        
        if (userOp.paymasterAndData.length > 0) {
            totalGas += userOp.verificationGasLimit; // paymaster verification
        }
        
        maxCost = totalGas * userOp.maxFeePerGas;
        estimatedCost = totalGas * gasPrice;
    }
    
    /**
     * @notice Optimize bundle: sort by priority fee
     */
    function optimizeBundle(
        UserOperation[] memory ops
    ) public pure returns (UserOperation[] memory optimized, uint256[] memory gasEstimates) {
        uint256 n = ops.length;
        gasEstimates = new uint256[](n);
        
        // คำนวณ effective priority fee สำหรับแต่ละ op
        uint256[] memory priorities = new uint256[](n);
        for (uint256 i = 0; i < n; i++) {
            priorities[i] = ops[i].maxPriorityFeePerGas;
        }
        
        // Sort by priority (bubble sort - ใช้สำหรับ demo)
        for (uint256 i = 0; i < n - 1; i++) {
            for (uint256 j = 0; j < n - i - 1; j++) {
                if (priorities[j] < priorities[j + 1]) {
                    // Swap ops
                    UserOperation memory temp = ops[j];
                    ops[j] = ops[j + 1];
                    ops[j + 1] = temp;
                    
                    uint256 tempP = priorities[j];
                    priorities[j] = priorities[j + 1];
                    priorities[j + 1] = tempP;
                }
            }
        }
        
        optimized = ops;
        return (optimized, gasEstimates);
    }
    
    // Simplified UserOperation struct for this contract
    struct UserOperation {
        address sender;
        uint256 nonce;
        bytes initCode;
        bytes callData;
        uint256 callGasLimit;
        uint256 verificationGasLimit;
        uint256 preVerificationGas;
        uint256 maxFeePerGas;
        uint256 maxPriorityFeePerGas;
        bytes paymasterAndData;
        bytes signature;
    }
}
```

---

## Workshop: สร้าง EIP-4337 Wallet System สมบูรณ์

### Workshop 1: Deploy และ Test Smart Wallet

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";

/**
 * @title AccountAbstractionTest
 * @notice Integration tests สำหรับ EIP-4337 system
 */
contract AccountAbstractionTest is Test {
    
    SimpleEntryPoint public entryPoint;
    SmartWalletFactory public factory;
    VerifyingPaymaster public paymaster;
    
    address public bundler = address(0xB);
    uint256 public ownerPrivateKey = 0x1234;
    address public owner;
    
    function setUp() public {
        owner = vm.addr(ownerPrivateKey);
        
        // Deploy EntryPoint
        entryPoint = new SimpleEntryPoint();
        
        // Deploy Factory
        factory = new SmartWalletFactory(address(entryPoint));
        
        // Deploy Paymaster
        paymaster = new VerifyingPaymaster(address(entryPoint), address(this));
        
        // Fund paymaster
        paymaster.deposit{value: 10 ether}();
        
        // Fund owner's future wallet
        vm.deal(address(this), 100 ether);
    }
    
    function test_deployWalletViaUserOp() public {
        // คำนวณ wallet address ล่วงหน้า
        address predictedWallet = factory.getAddress(owner, 0);
        
        // Fund wallet with ETH
        vm.deal(predictedWallet, 1 ether);
        
        // สร้าง UserOperation สำหรับ deploy + execute
        SimpleEntryPoint.UserOperation memory op = _buildDeployOp(owner, 0);
        op.callData = abi.encodeCall(SmartWallet.execute, (
            address(0xDEAD),
            0.1 ether,
            ""
        ));
        
        // Sign
        bytes32 opHash = entryPoint.getUserOpHash(op);
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(ownerPrivateKey, opHash);
        op.signature = abi.encodePacked(r, s, v);
        
        // Execute via bundler
        SimpleEntryPoint.UserOperation[] memory ops = new SimpleEntryPoint.UserOperation[](1);
        ops[0] = op;
        
        vm.prank(bundler);
        entryPoint.handleOps(ops, payable(bundler));
        
        // Verify wallet deployed
        assertTrue(predictedWallet.code.length > 0, "wallet not deployed");
        assertEq(address(0xDEAD).balance, 0.1 ether, "execution failed");
    }
    
    function test_socialRecovery() public {
        // Setup
        address guardian1 = address(0x1);
        address guardian2 = address(0x2);
        address guardian3 = address(0x3);
        address newOwner = address(0x4);
        
        address[] memory guardians = new address[](3);
        guardians[0] = guardian1;
        guardians[1] = guardian2;
        guardians[2] = guardian3;
        
        SocialRecoveryWallet wallet = new SocialRecoveryWallet(
            address(entryPoint),
            owner,
            guardians,
            2,       // threshold: 2 of 3
            1 days   // recovery delay
        );
        
        // Guardian 1 initiates recovery
        vm.prank(guardian1);
        wallet.initiateRecovery(newOwner);
        
        // Guardian 2 approves
        vm.prank(guardian2);
        wallet.approveRecovery(1);
        
        // Check approval count
        assertEq(wallet.getRecoveryApprovalCount(1), 2);
        
        // Wait for time lock
        vm.warp(block.timestamp + 1 days + 1);
        
        // Execute recovery
        wallet.executeRecovery(1);
        
        assertEq(wallet.owner(), newOwner, "recovery failed");
    }
    
    function test_sessionKey() public {
        address sessionKeyAddr = address(0xSESSION);
        
        SessionKeyManager manager = new SessionKeyManager(owner);
        
        uint48 validAfter = uint48(block.timestamp);
        uint48 validUntil = uint48(block.timestamp + 1 hours);
        
        // Owner creates session key
        vm.prank(owner);
        manager.createSessionKey(
            sessionKeyAddr,
            validAfter,
            validUntil,
            0.01 ether // daily ETH limit
        );
        
        // Allow specific method
        vm.prank(owner);
        manager.addAllowedMethod(
            sessionKeyAddr,
            bytes4(keccak256("swapExactTokensForTokens(uint256,uint256,address[],address,uint256)"))
        );
        
        // Validate session key operation
        bool valid = manager.validateSessionKeyOp(
            sessionKeyAddr,
            address(0xDEX),
            0,
            abi.encodeWithSelector(
                bytes4(keccak256("swapExactTokensForTokens(uint256,uint256,address[],address,uint256)")),
                100e18, 95e18, new address[](0), owner, block.timestamp
            )
        );
        
        assertTrue(valid, "session key validation failed");
    }
    
    function _buildDeployOp(address _owner, uint256 salt) 
        internal view returns (SimpleEntryPoint.UserOperation memory op) {
        bytes memory initCode = abi.encodePacked(
            address(factory),
            abi.encodeCall(factory.createAccount, (_owner, salt))
        );
        
        op.sender = factory.getAddress(_owner, salt);
        op.nonce = 0;
        op.initCode = initCode;
        op.callGasLimit = 200000;
        op.verificationGasLimit = 150000;
        op.preVerificationGas = 21000;
        op.maxFeePerGas = 1 gwei;
        op.maxPriorityFeePerGas = 1 gwei;
    }
}
```

### Workshop 2: Paymaster Integration Test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @notice Test Token Paymaster with ERC-20 payment
 */
contract TokenPaymasterTest is Test {
    
    MockERC20 public token;
    MockOracle public oracle;
    TokenPaymaster public paymaster;
    
    address user = address(0xUSER);
    
    function setUp() public {
        token = new MockERC20("Gas Token", "GAS", 18);
        oracle = new MockOracle(2000e8); // $2000 per ETH, 8 decimals
        paymaster = new TokenPaymaster(
            address(entryPoint),
            address(token),
            address(oracle)
        );
        
        // Mint tokens to user
        token.mint(user, 1000e18);
        
        // User approves paymaster
        vm.prank(user);
        token.approve(address(paymaster), type(uint256).max);
    }
    
    function test_payWithToken() public {
        uint256 maxCost = 0.01 ether;
        
        // คำนวณ token cost
        uint256 tokenCost = paymaster.getTokenValueOfEth(maxCost);
        
        uint256 balanceBefore = token.balanceOf(user);
        
        // Simulate validatePaymasterUserOp
        UserOperation memory op;
        op.sender = user;
        
        vm.prank(address(entryPoint));
        (bytes memory context,) = paymaster.validatePaymasterUserOp(op, bytes32(0), maxCost);
        
        // Simulate postOp
        uint256 actualCost = 0.008 ether; // Actually used less gas
        vm.prank(address(entryPoint));
        paymaster.postOp(TokenPaymaster.PostOpMode.opSucceeded, context, actualCost);
        
        uint256 balanceAfter = token.balanceOf(user);
        uint256 charged = balanceBefore - balanceAfter;
        
        // Token charge ควรสอดคล้องกับ actual cost
        uint256 expectedCharge = paymaster.getTokenValueOfEth(actualCost * paymaster.priceMarkup() / 10000);
        assertApproxEqRel(charged, expectedCharge, 0.01e18); // 1% tolerance
    }
}
```

---

## สรุป Part 86

- **EntryPoint mechanics**: handleOps flow, nonce management แบบ key-based, gas validation phases
- **Paymaster patterns**: 
  - VerifyingPaymaster: off-chain signature-based sponsoring
  - TokenPaymaster: ERC-20 payment with oracle price feed
  - DepositPaymaster: prepaid balance system with time lock
- **Social Recovery**: Guardian threshold, time lock delay, cancel mechanism
- **Session Keys**: Limited permission keys with method/target/time/value restrictions
- **Bundler Economics**: preVerificationGas calculation, bundle optimization by priority fee

## Next: Part 87 - DeFi Aggregators & Meta-Protocols
