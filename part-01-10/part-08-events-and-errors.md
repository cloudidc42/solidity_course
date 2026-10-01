# Part 08: Events และ Error Handling

## สารบัญ
1. Events เชิงลึก
2. Indexed Parameters
3. Event Filtering และ Logs
4. Error Handling Complete Guide
5. Custom Errors เชิงลึก
6. Panic Errors
7. Error Propagation
8. Events สำหรับ Off-chain Integration
9. Workshop: Audit Trail System

---

## 1. Events เชิงลึก

Events คือ **mechanism** สำหรับบันทึกข้อมูลลงใน transaction logs ซึ่ง:
- ถูกกว่าการเก็บใน storage มาก
- อ่านได้จาก off-chain (frontend)
- ไม่สามารถอ่านจาก contract อื่นได้

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract EventsDemo {
    
    // === Event Declarations ===
    
    // Basic event
    event Transfer(address from, address to, uint256 amount);
    
    // Event กับ indexed parameters (สำหรับ filtering)
    event TokenTransfer(
        address indexed from,    // indexed: searchable
        address indexed to,      // indexed: searchable
        uint256 amount,          // not indexed: only in data
        uint256 timestamp
    );
    
    // Anonymous event (ไม่มี topic[0] = event signature)
    event Checkpoint(uint256 blockNumber) anonymous;
    
    // Complex event
    event OrderPlaced(
        uint256 indexed orderId,
        address indexed buyer,
        address indexed seller,
        uint256 amount,
        string item,
        uint256 timestamp
    );
    
    // === Emitting Events ===
    
    mapping(address => uint256) public balances;
    uint256 public orderCounter;
    
    function transfer(address to, uint256 amount) public {
        require(balances[msg.sender] >= amount, "Insufficient");
        
        balances[msg.sender] -= amount;
        balances[to] += amount;
        
        // Emit event
        emit Transfer(msg.sender, to, amount);
        emit TokenTransfer(msg.sender, to, amount, block.timestamp);
    }
    
    function placeOrder(address seller, string calldata item) 
        public payable returns (uint256 orderId) 
    {
        orderId = ++orderCounter;
        
        emit OrderPlaced(
            orderId,
            msg.sender,
            seller,
            msg.value,
            item,
            block.timestamp
        );
    }
    
    // === Event ใน Constructor ===
    
    event ContractDeployed(address indexed deployer, uint256 timestamp);
    
    constructor() {
        emit ContractDeployed(msg.sender, block.timestamp);
    }
    
    // === Event พร้อมกับ State Change ===
    
    event BalanceUpdated(
        address indexed account,
        uint256 oldBalance,
        uint256 newBalance,
        int256 delta
    );
    
    function updateBalance(address account, uint256 newAmount) public {
        uint256 oldBalance = balances[account];
        balances[account] = newAmount;
        
        int256 delta = int256(newAmount) - int256(oldBalance);
        emit BalanceUpdated(account, oldBalance, newAmount, delta);
    }
}
```

---

## 2. Indexed Parameters

```solidity
contract IndexedEvents {
    
    // Indexed parameters เก็บเป็น topics (สำหรับ filtering)
    // Non-indexed เก็บเป็น data (ต้อง decode)
    
    // สูงสุด 3 indexed parameters (+ topic[0] = event signature)
    // Anonymous events: สูงสุด 4 indexed (ไม่มี signature topic)
    
    event WithdrawalEvent(
        address indexed account,      // topic[1]: searchable
        uint256 indexed tokenId,      // topic[2]: searchable
        address indexed recipient,    // topic[3]: searchable
        uint256 amount,               // data: not searchable directly
        string reason,                // data: not searchable directly
        uint256 timestamp             // data: not searchable directly
    );
    
    // การ filter events จาก Frontend:
    // contract.queryFilter(
    //     contract.filters.WithdrawalEvent(
    //         "0xUserAddress",  // filter by account
    //         null,             // any tokenId
    //         null              // any recipient
    //     )
    // )
    
    // Indexed กับ Dynamic Types (string, bytes, arrays)
    // ⚠️ ถ้า string หรือ bytes ถูก index → จะถูก hash ด้วย keccak256
    // ไม่สามารถ decode กลับมาเป็น string ได้!
    
    event DocumentStored(
        bytes32 indexed documentHash,  // ✅ hash ที่ค้นหาได้
        string indexed title,          // ⚠️ เก็บเป็น keccak256(title) ค้นหาได้แต่ decode ไม่ได้
        address indexed uploader,
        string content,                // ✅ ไม่ indexed = เก็บใน data ครบถ้วน
        uint256 timestamp
    );
    
    function storeDocument(
        string calldata title,
        string calldata content
    ) public {
        bytes32 hash = keccak256(bytes(content));
        
        emit DocumentStored(hash, title, msg.sender, content, block.timestamp);
    }
    
    // === Reading Events Off-chain (ethers.js) ===
    
    /*
    // JavaScript
    
    // Get all Transfer events
    const filter = contract.filters.Transfer();
    const events = await contract.queryFilter(filter, fromBlock, toBlock);
    
    // Filter by indexed parameter
    const userFilter = contract.filters.Transfer(userAddress, null);
    const userEvents = await contract.queryFilter(userFilter);
    
    // Parse event
    events.forEach(event => {
        console.log({
            from: event.args.from,
            to: event.args.to,
            amount: ethers.formatEther(event.args.amount),
            blockNumber: event.blockNumber,
            transactionHash: event.transactionHash
        });
    });
    
    // Listen to real-time events
    contract.on("Transfer", (from, to, amount, event) => {
        console.log(`Transfer: ${from} → ${to}: ${amount}`);
    });
    */
}
```

---

## 3. Events สำหรับ Off-chain

```solidity
contract EventPatterns {
    
    // === Pattern 1: State Change Event ===
    
    enum Status { Active, Paused, Cancelled }
    Status public contractStatus;
    
    event StatusChanged(
        Status indexed oldStatus,
        Status indexed newStatus,
        address indexed changedBy,
        uint256 timestamp,
        string reason
    );
    
    function setStatus(Status newStatus, string calldata reason) public {
        Status oldStatus = contractStatus;
        contractStatus = newStatus;
        
        emit StatusChanged(oldStatus, newStatus, msg.sender, block.timestamp, reason);
    }
    
    // === Pattern 2: Batch Event ===
    
    event BatchTransfer(
        address indexed initiator,
        address[] recipients,
        uint256[] amounts,
        uint256 total,
        uint256 timestamp
    );
    
    mapping(address => uint256) public balances;
    
    function batchTransfer(
        address[] calldata recipients,
        uint256[] calldata amounts
    ) public {
        require(recipients.length == amounts.length, "Length mismatch");
        
        uint256 total = 0;
        for (uint256 i = 0; i < amounts.length; i++) {
            total += amounts[i];
        }
        
        require(balances[msg.sender] >= total, "Insufficient");
        balances[msg.sender] -= total;
        
        for (uint256 i = 0; i < recipients.length; i++) {
            balances[recipients[i]] += amounts[i];
        }
        
        emit BatchTransfer(msg.sender, recipients, amounts, total, block.timestamp);
    }
    
    // === Pattern 3: Upgrade Event ===
    
    event ContractUpgraded(
        address indexed oldImplementation,
        address indexed newImplementation,
        address indexed admin,
        uint256 timestamp
    );
    
    address public implementation;
    
    function upgrade(address newImpl) public {
        address old = implementation;
        implementation = newImpl;
        emit ContractUpgraded(old, newImpl, msg.sender, block.timestamp);
    }
    
    // === Pattern 4: Price Oracle Event ===
    
    event PriceUpdated(
        bytes32 indexed assetId,
        uint256 indexed oldPrice,
        uint256 indexed newPrice,
        uint256 timestamp,
        address source
    );
    
    mapping(bytes32 => uint256) public prices;
    
    function updatePrice(bytes32 assetId, uint256 newPrice) public {
        uint256 oldPrice = prices[assetId];
        prices[assetId] = newPrice;
        emit PriceUpdated(assetId, oldPrice, newPrice, block.timestamp, msg.sender);
    }
}
```

---

## 4. Error Handling Complete Guide

```solidity
contract ErrorHandlingGuide {
    
    // === 1. require() ===
    // ใช้สำหรับ: Input validation, preconditions
    // Gas: คืน gas ที่เหลือ + refund (ยกเว้น gas ที่ใช้ไปแล้ว)
    
    function requireDemo(uint256 amount) public pure {
        // Basic require
        require(amount > 0, "Amount must be positive");
        
        // require กับ condition ซับซ้อน
        require(
            amount >= 100 && amount <= 10000,
            "Amount must be between 100 and 10000"
        );
        
        // require ไม่มี message (ประหยัด gas แต่ debug ยาก)
        require(amount % 2 == 0);
    }
    
    // === 2. revert() ===
    // ใช้สำหรับ: Complex conditions, custom messages
    
    enum Role { None, User, Admin, Owner }
    
    function revertDemo(Role role, uint256 amount) public pure {
        // revert กับ string
        if (role == Role.None) {
            revert("No role assigned");
        }
        
        // revert กับ condition ซับซ้อน
        if (role != Role.Admin && role != Role.Owner) {
            if (amount > 1000) {
                revert(string.concat(
                    "Insufficient permissions: amount ",
                    _toString(amount),
                    " exceeds limit for role"
                ));
            }
        }
    }
    
    // === 3. assert() ===
    // ใช้สำหรับ: Invariants ที่ไม่ควรเป็น false
    // Gas: ใช้ gas ทั้งหมด (ต่างจาก require!)
    
    uint256 public constant MAX_SUPPLY = 1_000_000 * 1e18;
    uint256 public totalSupply;
    mapping(address => uint256) public balances;
    
    function mint(address to, uint256 amount) public {
        // require: ตรวจสอบ conditions ที่ผู้ใช้อาจ trigger
        require(amount > 0, "Zero amount");
        require(totalSupply + amount <= MAX_SUPPLY, "Exceeds max supply");
        
        totalSupply += amount;
        balances[to] += amount;
        
        // assert: invariant ที่ไม่ควรผิด (bug ใน code)
        assert(totalSupply <= MAX_SUPPLY); // redundant แต่เป็น safety net
    }
    
    function _toString(uint256 n) internal pure returns (string memory) {
        if (n == 0) return "0";
        uint256 j = n; uint256 len;
        while (j != 0) { len++; j /= 10; }
        bytes memory bstr = new bytes(len);
        uint256 k = len;
        while (n != 0) { k--; bstr[k] = bytes1(uint8(48 + n % 10)); n /= 10; }
        return string(bstr);
    }
}
```

---

## 5. Custom Errors เชิงลึก

```solidity
contract CustomErrorsAdvanced {
    
    // === Error Hierarchy ===
    
    // Base errors (สำหรับ inheritance)
    error BaseError();
    error ParameterError(string parameter, string reason);
    
    // Specific errors
    error ZeroAddress();
    error ZeroAmount();
    error InsufficientBalance(address account, uint256 available, uint256 required);
    error Unauthorized(address caller, bytes32 requiredRole);
    error Paused();
    error AlreadyInitialized();
    error NotInitialized();
    error Deadline(uint256 deadline, uint256 currentTime);
    error OutOfRange(uint256 value, uint256 min, uint256 max);
    error NotFound(bytes32 id);
    error AlreadyExists(bytes32 id);
    error InvalidSignature(address signer, address expected);
    error SlippageExceeded(uint256 expected, uint256 actual, uint256 maxSlippage);
    
    // === Gas Comparison ===
    
    // ❌ String error: ~50 gas per character + overhead
    function requireWithString(uint256 amount) public pure {
        if (amount == 0) revert("Amount cannot be zero"); // ~660 gas overhead
    }
    
    // ✅ Custom error: ~35 gas per parameter
    function requireCustomError(uint256 amount) public pure {
        if (amount == 0) revert ZeroAmount(); // ~22 gas overhead
    }
    
    // === Error กับ Context ===
    
    mapping(address => uint256) public balances;
    bool public isPaused;
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    function withdraw(uint256 amount) public {
        if (isPaused) revert Paused();
        if (amount == 0) revert ZeroAmount();
        
        uint256 balance = balances[msg.sender];
        if (balance < amount) {
            revert InsufficientBalance(msg.sender, balance, amount);
        }
        
        balances[msg.sender] -= amount;
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed"); // low-level failures ยังใช้ require
    }
    
    // === Catching Custom Errors ===
    
    interface IVault {
        error InsufficientFunds(uint256 available, uint256 required);
    }
    
    function safeWithdraw(address vault, uint256 amount) public returns (bool success) {
        try IVault(vault).InsufficientFunds.selector == bytes4(keccak256("InsufficientFunds(uint256,uint256)"))
            ? this._attemptWithdraw(vault, amount)
            : this._attemptWithdraw(vault, amount)
        {
            return true;
        } catch {
            return false;
        }
    }
    
    function _attemptWithdraw(address, uint256) external pure returns (bool) {
        return true;
    }
    
    // === Error ใน Interface ===
    
    interface IERC20Extended {
        error TransferToZeroAddress();
        error TransferAmountExceedsBalance(uint256 balance, uint256 amount);
        error ApprovalToZeroAddress();
        
        function transfer(address to, uint256 amount) external returns (bool);
    }
    
    // === Error Encoding/Decoding ===
    
    function encodeError() public pure returns (bytes memory) {
        return abi.encodeWithSelector(
            InsufficientBalance.selector,
            address(0x1234),
            100,
            200
        );
    }
    
    function decodeError(bytes memory errorData) public pure returns (
        address account,
        uint256 available,
        uint256 required
    ) {
        (account, available, required) = abi.decode(
            errorData[4:], // skip 4-byte selector
            (address, uint256, uint256)
        );
    }
    
    // Error selector
    function getErrorSelector() public pure returns (bytes4) {
        return InsufficientBalance.selector;
        // = bytes4(keccak256("InsufficientBalance(address,uint256,uint256)"))
    }
}
```

---

## 6. Panic Errors

```solidity
contract PanicErrors {
    
    // Panic codes:
    // 0x00: generic
    // 0x01: assert failure
    // 0x11: arithmetic overflow/underflow
    // 0x12: divide by zero
    // 0x21: invalid enum conversion
    // 0x22: access to invalid storage byte array
    // 0x31: pop on empty array
    // 0x32: out of bounds array access
    // 0x41: too much memory allocation
    // 0x51: call to uninitialized function
    
    // === Examples ===
    
    uint256[] public arr;
    
    function overflowExample() public pure returns (uint256) {
        uint256 max = type(uint256).max;
        return max + 1; // Panic: 0x11 (overflow in checked arithmetic)
    }
    
    function divByZero() public pure returns (uint256) {
        uint256 a = 10;
        uint256 b = 0;
        return a / b; // Panic: 0x12
    }
    
    function outOfBounds() public view returns (uint256) {
        return arr[100]; // Panic: 0x32 (if arr.length < 101)
    }
    
    function popEmpty() public {
        arr.pop(); // Panic: 0x31 (if arr is empty)
    }
    
    // === Catching Panic ===
    
    address public externalContract;
    
    function safeDivide(uint256 a, uint256 b) public pure returns (uint256 result) {
        if (b == 0) return 0; // Handle before panic
        return a / b;
    }
    
    function safeArrayAccess(uint256[] memory data, uint256 index) 
        public pure returns (uint256) 
    {
        if (index >= data.length) return 0;
        return data[index];
    }
    
    // Unchecked arithmetic (เพื่อ avoid overflow panic)
    function uncheckedAdd(uint256 a, uint256 b) public pure returns (uint256) {
        unchecked {
            return a + b; // ไม่ throw panic แต่ overflow ได้
        }
    }
}
```

---

## 7. Workshop: Audit Trail System

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title AuditTrail
 * @dev ระบบบันทึก Audit Trail ที่ไม่สามารถแก้ไขได้
 * ใช้ Events เป็น primary storage (cheap)
 * State เก็บ minimal data
 */
contract AuditTrail {
    
    // === Types ===
    
    enum ActionType {
        Created,
        Updated,
        Deleted,
        Accessed,
        Transferred,
        Approved,
        Rejected,
        Escalated,
        Archived
    }
    
    enum Severity {
        Info,
        Warning,
        Critical
    }
    
    // === Custom Errors ===
    
    error InvalidActor(address actor);
    error InvalidSubject(bytes32 subject);
    error EmptyDescription();
    error UnauthorizedAuditor(address caller);
    error RecordSealed(bytes32 recordId);
    error InvalidNonce(uint256 expected, uint256 provided);
    
    // === Events (Primary Storage) ===
    
    event AuditLog(
        bytes32 indexed recordId,       // ID ของ record
        address indexed actor,          // ผู้กระทำ
        bytes32 indexed subject,        // สิ่งที่ถูกกระทำ
        ActionType action,
        Severity severity,
        string description,
        bytes32 previousHash,           // hash ของ log ก่อนหน้า (chain)
        uint256 timestamp,
        uint256 blockNumber
    );
    
    event AuditorAdded(address indexed auditor, address indexed addedBy, uint256 timestamp);
    event AuditorRemoved(address indexed auditor, address indexed removedBy, uint256 timestamp);
    event SystemEvent(
        string indexed eventType,
        string description,
        Severity severity,
        uint256 timestamp
    );
    
    // === State (Minimal) ===
    
    address public immutable ADMIN;
    mapping(address => bool) public auditors;
    
    // Nonces ป้องกัน replay attacks
    mapping(address => uint256) public nonces;
    
    // Hash chain (เพื่อ integrity)
    bytes32 public lastHash;
    uint256 public logCount;
    
    // Sealed records (ไม่สามารถบันทึกซ้ำได้)
    mapping(bytes32 => bool) public sealedRecords;
    
    // === Constructor ===
    
    constructor() {
        ADMIN = msg.sender;
        auditors[msg.sender] = true;
        
        emit SystemEvent("SYSTEM_INIT", "Audit Trail initialized", Severity.Info, block.timestamp);
    }
    
    // === Modifiers ===
    
    modifier onlyAuditor() {
        if (!auditors[msg.sender]) revert UnauthorizedAuditor(msg.sender);
        _;
    }
    
    modifier onlyAdmin() {
        require(msg.sender == ADMIN, "Not admin");
        _;
    }
    
    // === Core Logging ===
    
    function log(
        address actor,
        bytes32 subject,
        ActionType action,
        Severity severity,
        string calldata description
    ) external onlyAuditor returns (bytes32 recordId) {
        if (actor == address(0)) revert InvalidActor(actor);
        if (subject == bytes32(0)) revert InvalidSubject(subject);
        if (bytes(description).length == 0) revert EmptyDescription();
        
        // สร้าง unique record ID
        recordId = keccak256(abi.encodePacked(
            actor,
            subject,
            action,
            block.timestamp,
            logCount
        ));
        
        // Hash chain
        bytes32 prevHash = lastHash;
        lastHash = keccak256(abi.encodePacked(prevHash, recordId));
        logCount++;
        
        emit AuditLog(
            recordId,
            actor,
            subject,
            action,
            severity,
            description,
            prevHash,
            block.timestamp,
            block.number
        );
    }
    
    function logWithNonce(
        address actor,
        bytes32 subject,
        ActionType action,
        Severity severity,
        string calldata description,
        uint256 nonce
    ) external onlyAuditor returns (bytes32 recordId) {
        if (nonces[actor] != nonce) revert InvalidNonce(nonces[actor], nonce);
        nonces[actor]++;
        
        return this.log(actor, subject, action, severity, description);
    }
    
    // Batch logging
    function batchLog(
        address[] calldata actors,
        bytes32[] calldata subjects,
        ActionType[] calldata actions,
        Severity[] calldata severities,
        string[] calldata descriptions
    ) external onlyAuditor {
        require(
            actors.length == subjects.length &&
            subjects.length == actions.length &&
            actions.length == severities.length &&
            severities.length == descriptions.length,
            "Length mismatch"
        );
        require(actors.length <= 50, "Too many logs");
        
        for (uint256 i = 0; i < actors.length; i++) {
            if (actors[i] == address(0)) continue;
            if (subjects[i] == bytes32(0)) continue;
            
            bytes32 recordId = keccak256(abi.encodePacked(
                actors[i], subjects[i], actions[i], block.timestamp, logCount + i
            ));
            
            bytes32 prevHash = lastHash;
            lastHash = keccak256(abi.encodePacked(prevHash, recordId));
            
            emit AuditLog(
                recordId,
                actors[i],
                subjects[i],
                actions[i],
                severities[i],
                descriptions[i],
                prevHash,
                block.timestamp,
                block.number
            );
        }
        
        logCount += actors.length;
    }
    
    // Seal a record (cannot be modified)
    function sealRecord(bytes32 recordId) external onlyAdmin {
        sealedRecords[recordId] = true;
    }
    
    // === Auditor Management ===
    
    function addAuditor(address auditor) external onlyAdmin {
        require(auditor != address(0), "Zero address");
        require(!auditors[auditor], "Already auditor");
        
        auditors[auditor] = true;
        emit AuditorAdded(auditor, msg.sender, block.timestamp);
    }
    
    function removeAuditor(address auditor) external onlyAdmin {
        require(auditors[auditor], "Not auditor");
        require(auditor != ADMIN, "Cannot remove admin");
        
        auditors[auditor] = false;
        emit AuditorRemoved(auditor, msg.sender, block.timestamp);
    }
    
    // === Integrity Verification ===
    
    function verifyChain(
        bytes32[] calldata recordIds
    ) external view returns (bool valid, uint256 validCount) {
        bytes32 computedHash = bytes32(0);
        
        for (uint256 i = 0; i < recordIds.length; i++) {
            computedHash = keccak256(abi.encodePacked(computedHash, recordIds[i]));
            validCount++;
        }
        
        valid = computedHash == lastHash;
    }
    
    // === View Helpers ===
    
    function getCurrentNonce(address actor) external view returns (uint256) {
        return nonces[actor];
    }
    
    function getLastHash() external view returns (bytes32) {
        return lastHash;
    }
    
    // Helper สำหรับ off-chain
    function computeRecordId(
        address actor,
        bytes32 subject,
        ActionType action,
        uint256 timestamp,
        uint256 count
    ) external pure returns (bytes32) {
        return keccak256(abi.encodePacked(actor, subject, action, timestamp, count));
    }
}

/**
 * @title AuditableContract
 * @dev ตัวอย่าง Contract ที่ใช้ AuditTrail
 */
contract AuditableContract {
    
    AuditTrail public auditTrail;
    
    struct Document {
        bytes32 id;
        string name;
        string contentHash; // IPFS hash
        address owner;
        bool exists;
    }
    
    mapping(bytes32 => Document) public documents;
    
    constructor(address _auditTrail) {
        auditTrail = AuditTrail(_auditTrail);
    }
    
    function createDocument(
        string calldata name,
        string calldata contentHash
    ) external returns (bytes32 docId) {
        docId = keccak256(abi.encodePacked(msg.sender, name, block.timestamp));
        
        documents[docId] = Document({
            id: docId,
            name: name,
            contentHash: contentHash,
            owner: msg.sender,
            exists: true
        });
        
        // Log ลงใน AuditTrail
        auditTrail.log(
            msg.sender,
            docId,
            AuditTrail.ActionType.Created,
            AuditTrail.Severity.Info,
            string.concat("Document created: ", name)
        );
    }
    
    function updateDocument(
        bytes32 docId,
        string calldata newContentHash
    ) external {
        Document storage doc = documents[docId];
        require(doc.exists, "Not found");
        require(doc.owner == msg.sender, "Not owner");
        
        doc.contentHash = newContentHash;
        
        auditTrail.log(
            msg.sender,
            docId,
            AuditTrail.ActionType.Updated,
            AuditTrail.Severity.Info,
            "Document content updated"
        );
    }
    
    function deleteDocument(bytes32 docId) external {
        Document storage doc = documents[docId];
        require(doc.exists, "Not found");
        require(doc.owner == msg.sender, "Not owner");
        
        delete documents[docId];
        
        auditTrail.log(
            msg.sender,
            docId,
            AuditTrail.ActionType.Deleted,
            AuditTrail.Severity.Warning,
            "Document deleted"
        );
    }
    
    function transferOwnership(bytes32 docId, address newOwner) external {
        Document storage doc = documents[docId];
        require(doc.exists, "Not found");
        require(doc.owner == msg.sender, "Not owner");
        require(newOwner != address(0), "Zero address");
        
        address oldOwner = doc.owner;
        doc.owner = newOwner;
        
        auditTrail.log(
            msg.sender,
            docId,
            AuditTrail.ActionType.Transferred,
            AuditTrail.Severity.Info,
            string.concat("Ownership transferred to ", _toHexString(newOwner))
        );
    }
    
    function _toHexString(address addr) internal pure returns (string memory) {
        bytes memory buffer = new bytes(42);
        buffer[0] = "0";
        buffer[1] = "x";
        bytes16 symbols = "0123456789abcdef";
        uint160 value = uint160(addr);
        for (uint256 i = 41; i > 1; --i) {
            buffer[i] = symbols[value & 0xf];
            value >>= 4;
        }
        return string(buffer);
    }
}
```

### Tests

```typescript
// test/AuditTrail.test.ts
import { expect } from "chai";
import { ethers } from "hardhat";

describe("AuditTrail", function () {
    let auditTrail: any;
    let owner: any, auditor1: any, user1: any;
    
    beforeEach(async function () {
        [owner, auditor1, user1] = await ethers.getSigners();
        
        const AuditTrail = await ethers.getContractFactory("AuditTrail");
        auditTrail = await AuditTrail.deploy();
        
        await auditTrail.addAuditor(auditor1.address);
    });
    
    it("Should log an event", async function () {
        const subject = ethers.id("test-subject");
        
        const tx = await auditTrail.connect(auditor1).log(
            user1.address,
            subject,
            0, // ActionType.Created
            0, // Severity.Info
            "Test log entry"
        );
        
        const receipt = await tx.wait();
        const events = receipt.logs.filter((log: any) => {
            try {
                auditTrail.interface.parseLog(log);
                return true;
            } catch { return false; }
        });
        
        expect(events.length).to.be.gt(0);
        expect(await auditTrail.logCount()).to.equal(1);
    });
    
    it("Should reject unauthorized auditor", async function () {
        await expect(
            auditTrail.connect(user1).log(
                user1.address,
                ethers.id("test"),
                0, 0, "test"
            )
        ).to.be.reverted;
    });
    
    it("Should maintain hash chain", async function () {
        const subject = ethers.id("subject");
        
        const initialHash = await auditTrail.lastHash();
        expect(initialHash).to.equal(ethers.ZeroHash);
        
        await auditTrail.connect(auditor1).log(user1.address, subject, 0, 0, "log 1");
        const hash1 = await auditTrail.lastHash();
        expect(hash1).to.not.equal(ethers.ZeroHash);
        
        await auditTrail.connect(auditor1).log(user1.address, subject, 1, 0, "log 2");
        const hash2 = await auditTrail.lastHash();
        expect(hash2).to.not.equal(hash1);
    });
    
    it("Should batch log efficiently", async function () {
        const subject = ethers.id("subject");
        const count = 10;
        
        const actors = Array(count).fill(user1.address);
        const subjects = Array(count).fill(subject);
        const actions = Array(count).fill(0);
        const severities = Array(count).fill(0);
        const descriptions = Array(count).fill("Batch log").map((s, i) => s + i);
        
        await auditTrail.connect(auditor1).batchLog(
            actors, subjects, actions, severities, descriptions
        );
        
        expect(await auditTrail.logCount()).to.equal(count);
    });
});
```

---

## สรุป Part 08

Events และ Error Handling ที่เรียนรู้:
- ✅ Events (basic, indexed, anonymous)
- ✅ Indexed parameters และ filtering
- ✅ Event patterns (batch, state change, chain)
- ✅ require, revert, assert (เมื่อใช้อะไร)
- ✅ Custom errors (syntax, gas savings)
- ✅ Panic errors (types, prevention)
- ✅ Error encoding/decoding
- ✅ Audit Trail system

## Quiz

1. สูงสุดกี่ indexed parameters ใน non-anonymous event?
2. ทำไม `assert()` ใช้ gas ทั้งหมดแต่ `require()` ไม่ใช่?
3. ทำไม Custom Error ถูกกว่า string message?
4. String indexed parameter เก็บอะไรใน topic?

---

## Next: Part 09 - Inheritance และ Interfaces
