# Part 27: Smart Contract Auditing

## สารบัญ
1. Audit Process
2. Common Vulnerability Patterns
3. Static Analysis Tools
4. Formal Verification
5. Workshop: Audit Practice

---

## 1. Audit Process

```
Audit Workflow:
1. Scope Definition: กำหนด contracts ที่จะ audit
2. Code Review: อ่านและทำความเข้าใจ codebase
3. Threat Modeling: ระบุ attack vectors
4. Vulnerability Testing: ทดสอบ exploits
5. Report Writing: สรุป findings พร้อม severity
6. Remediation: dev แก้ไข
7. Re-audit: ตรวจสอบ fixes

Severity Levels:
- Critical: สามารถ drain funds ทั้งหมด (CVSS 9-10)
- High: loss of funds หรือ major malfunction (7-8)
- Medium: loss of funds ในบางกรณี หรือ significant bug (4-6)
- Low: minor issues, best practices (1-3)
- Informational: code quality, gas optimization

Top Audit Firms:
- Trail of Bits
- OpenZeppelin
- Certik
- Code4rena (contest)
- Sherlock (contest + insurance)
- Spearbit
```

---

## 2. Common Vulnerability Patterns

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// ❌ VULNERABLE: Reentrancy
contract VulnerableBank {
    mapping(address => uint256) public balances;
    
    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }
    
    function withdraw(uint256 amount) external {
        require(balances[msg.sender] >= amount);
        
        // ❌ External call BEFORE state update
        (bool success,) = msg.sender.call{value: amount}("");
        require(success);
        
        balances[msg.sender] -= amount; // Too late!
    }
}

// ✅ FIXED: Reentrancy
contract SafeBank {
    mapping(address => uint256) public balances;
    bool private _locked;
    
    modifier nonReentrant() {
        require(!_locked, "Reentrant");
        _locked = true;
        _;
        _locked = false;
    }
    
    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }
    
    function withdraw(uint256 amount) external nonReentrant {
        require(balances[msg.sender] >= amount);
        
        // ✅ CEI: Check → Effect → Interaction
        balances[msg.sender] -= amount; // Effect FIRST
        
        (bool success,) = msg.sender.call{value: amount}(""); // Then interact
        require(success);
    }
}

// ❌ VULNERABLE: Integer Overflow (pre-0.8)
// In Solidity 0.8+, overflow reverts by default
// แต่ unchecked block ยังต้องระวัง!
contract UncheckedArithmetic {
    function unsafeAdd(uint256 a, uint256 b) external pure returns (uint256) {
        unchecked {
            return a + b; // ❌ สามารถ overflow ได้
        }
    }
    
    function safeAdd(uint256 a, uint256 b) external pure returns (uint256) {
        return a + b; // ✅ reverts on overflow (0.8+)
    }
    
    // ใช้ unchecked เฉพาะเมื่อ proven ว่า ไม่ overflow
    function efficientLoop(uint256[] calldata arr) external pure returns (uint256 sum) {
        for (uint256 i = 0; i < arr.length;) {
            sum += arr[i];
            unchecked { ++i; } // ✅ safe: i < arr.length.max
        }
    }
}

// ❌ VULNERABLE: Access Control Missing
contract NoAccessControl {
    uint256 public fee;
    
    function setFee(uint256 newFee) external {
        fee = newFee; // ❌ Anyone can set fee!
    }
}

// ✅ FIXED:
contract WithAccessControl {
    address public owner;
    uint256 public fee;
    
    constructor() { owner = msg.sender; }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    function setFee(uint256 newFee) external onlyOwner {
        fee = newFee; // ✅
    }
}

// ❌ VULNERABLE: Tx.origin auth
contract TxOriginAuth {
    address public owner;
    
    modifier onlyOwner() {
        require(tx.origin == owner, "Not owner"); // ❌ tx.origin attack!
        _;
    }
    
    // Attack: Phishing contract calls onlyOwner function
    // tx.origin = victim, msg.sender = attacker contract
}

// ✅ FIXED: use msg.sender
contract MsgSenderAuth {
    address public owner;
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner"); // ✅
        _;
    }
}

// ❌ VULNERABLE: Uninitialized Storage Pointer
// (less common in 0.8+ but still possible)
contract StorageBug {
    struct Data {
        uint256 value;
        address owner;
    }
    
    Data[] public items;
    
    function buggyFunc() external {
        Data storage item; // ❌ Uninitialized! Points to slot 0
        item.value = 100;  // Modifies items.length or first element!
    }
}

// ❌ VULNERABLE: ERC20 Approval Race Condition
// approve(100) → attacker frontruns → spends 100 → approve takes effect → spends another 100
// Total: attacker spent 200 instead of 100
// ✅ Fix: use increaseAllowance/decreaseAllowance or EIP-2612 Permit

// ❌ VULNERABLE: Timestamp Dependence
contract TimestampDependence {
    function isLucky() external view returns (bool) {
        return block.timestamp % 10 == 0; // ❌ Miner can manipulate ±15 seconds
    }
}

// ❌ VULNERABLE: Signature Replay
contract NoNonce {
    address public signer;
    
    function execute(uint256 amount, bytes calldata sig) external {
        bytes32 hash = keccak256(abi.encodePacked(amount));
        // ❌ Same signature can be replayed!
        require(_verify(hash, sig) == signer);
        // do something
    }
    
    function _verify(bytes32 hash, bytes calldata sig) internal pure returns (address) {
        bytes32 messageHash = keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", hash));
        (bytes32 r, bytes32 s, uint8 v) = abi.decode(sig, (bytes32, bytes32, uint8));
        return ecrecover(messageHash, v, r, s);
    }
}

// ✅ FIXED with nonce + deadline
contract WithNonce {
    address public signer;
    mapping(address => uint256) public nonces;
    
    function execute(
        uint256 amount,
        uint256 deadline,
        bytes calldata sig
    ) external {
        require(block.timestamp <= deadline, "Expired");
        
        bytes32 hash = keccak256(abi.encodePacked(
            msg.sender,
            amount,
            nonces[msg.sender]++,
            deadline,
            block.chainid // ✅ ป้องกัน cross-chain replay
        ));
        
        bytes32 messageHash = keccak256(abi.encodePacked(
            "\x19Ethereum Signed Message:\n32",
            hash
        ));
        
        (bytes32 r, bytes32 s, uint8 v) = abi.decode(sig, (bytes32, bytes32, uint8));
        require(ecrecover(messageHash, v, r, s) == signer, "Invalid sig");
    }
}
```

---

## 3. Static Analysis Tools

```
Tools:

1. Slither (Trail of Bits):
   - Static analysis ใน Python
   - ตรวจหา common bugs อัตโนมัติ
   
   Usage:
   slither contracts/MyContract.sol
   slither . --filter-paths node_modules
   slither . --detect reentrancy-eth,reentrancy-no-eth

2. Mythril (Consensus):
   - Symbolic execution
   - ค้นหา exploitable paths
   
   Usage:
   myth analyze contracts/MyContract.sol

3. Echidna (Trail of Bits):
   - Fuzzing tool
   - Property-based testing
   
   Usage: ดู Part Fuzzing Testing

4. Foundry's Forge:
   - Built-in gas analysis
   - Coverage reports
   
   forge coverage
   forge snapshot

5. Semgrep:
   - Custom pattern matching
   - Rule-based analysis

Common Slither Findings:
- reentrancy-eth: reentrancy with ETH transfer
- reentrancy-no-eth: reentrancy without ETH
- unchecked-transfer: ERC20 transfer not checked
- unused-return: return value ignored
- arbitrary-send: ETH sent to arbitrary address
- controlled-delegatecall: dangerous delegatecall
- msg-value-loop: msg.value in loop
```

---

## 4. Formal Verification

```
Formal Verification:
พิสูจน์ทางคณิตศาสตร์ว่า contract ทำงานถูกต้อง 100%
ไม่ใช่แค่ testing

Tools:
- Certora Prover: ใช้ CVL language เขียน specs
- K Framework: Ethereum Yellow Paper verification
- Halmos: Symbolic testing ด้วย Python

Certora Example:
```
// spec file (.spec)
methods {
    function totalSupply() external returns (uint256) envfree;
    function balanceOf(address) external returns (uint256) envfree;
    function transfer(address, uint256) external returns (bool);
}

// Invariant: totalSupply = sum of all balances
invariant totalSupplyEqualsSum()
    forall address a. balanceOf(a) <= totalSupply();

// Rule: transfer reduces sender balance
rule transferReducesSenderBalance(address to, uint256 amount) {
    env e;
    address sender = e.msg.sender;
    
    uint256 balanceBefore = balanceOf(sender);
    require balanceBefore >= amount;
    
    transfer(e, to, amount);
    
    assert balanceOf(sender) == balanceBefore - amount;
}
```
```

---

## 5. Workshop: Audit a Vulnerable Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * EXERCISE: หา vulnerabilities ทั้งหมดใน contract นี้
 * (มี 7 bugs ซ่อนอยู่)
 */
contract AuditMe {
    
    address public owner;
    mapping(address => uint256) public balances;
    mapping(address => bool) public whitelisted;
    
    uint256 public totalFees;
    uint256 public feeRate = 100; // 1% (basis points)
    
    event Deposit(address indexed user, uint256 amount);
    event Withdrawal(address indexed user, uint256 amount);
    event Transfer(address indexed from, address indexed to, uint256 amount);
    
    constructor() {
        owner = msg.sender;
    }
    
    // Bug 1: tx.origin instead of msg.sender
    modifier onlyOwner() {
        require(tx.origin == owner, "Not owner");
        _;
    }
    
    function deposit() external payable {
        require(msg.value > 0);
        balances[msg.sender] += msg.value;
        emit Deposit(msg.sender, msg.value);
    }
    
    // Bug 2: Reentrancy (no CEI, no guard)
    function withdraw(uint256 amount) external {
        require(balances[msg.sender] >= amount, "Insufficient");
        require(whitelisted[msg.sender], "Not whitelisted");
        
        (bool success,) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
        
        balances[msg.sender] -= amount; // Too late!
        emit Withdrawal(msg.sender, amount);
    }
    
    // Bug 3: Integer division truncation (fee = 0 for small amounts)
    function calculateFee(uint256 amount) public view returns (uint256) {
        return amount / 10000 * feeRate; // ❌ should be amount * feeRate / 10000
    }
    
    // Bug 4: No slippage protection, no deadline
    function transferWithFee(address to, uint256 amount) external {
        require(balances[msg.sender] >= amount, "Insufficient");
        
        uint256 fee = calculateFee(amount);
        uint256 toAmount = amount - fee;
        
        balances[msg.sender] -= amount;
        balances[to] += toAmount;
        totalFees += fee;
        
        emit Transfer(msg.sender, to, toAmount);
    }
    
    // Bug 5: Missing access control
    function setWhitelist(address user, bool status) external {
        whitelisted[user] = status; // ❌ Anyone can whitelist!
    }
    
    // Bug 6: Unchecked return value
    function emergencyWithdraw() external onlyOwner {
        payable(owner).send(address(this).balance); // ❌ send() can return false
    }
    
    // Bug 7: Block.timestamp manipulation risk
    function isActive() public view returns (bool) {
        return block.timestamp % 2 == 0; // ❌ Miner-manipulable
    }
    
    function collectFees() external onlyOwner {
        uint256 amount = totalFees;
        totalFees = 0;
        payable(owner).transfer(amount);
    }
}

/**
 * FIXED VERSION:
 */
contract AuditeFixed {
    
    address public owner;
    mapping(address => uint256) public balances;
    mapping(address => bool) public whitelisted;
    
    uint256 public totalFees;
    uint256 public feeRate = 100;
    
    bool private _locked;
    
    event Deposit(address indexed user, uint256 amount);
    event Withdrawal(address indexed user, uint256 amount);
    
    constructor() {
        owner = msg.sender;
    }
    
    // Fix 1: msg.sender
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    // Fix 2: nonReentrant
    modifier nonReentrant() {
        require(!_locked, "Reentrant");
        _locked = true;
        _;
        _locked = false;
    }
    
    function deposit() external payable {
        require(msg.value > 0);
        balances[msg.sender] += msg.value;
        emit Deposit(msg.sender, msg.value);
    }
    
    // Fix 2: CEI pattern
    function withdraw(uint256 amount) external nonReentrant {
        require(balances[msg.sender] >= amount, "Insufficient");
        require(whitelisted[msg.sender], "Not whitelisted");
        
        balances[msg.sender] -= amount; // Effect first
        
        (bool success,) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
        
        emit Withdrawal(msg.sender, amount);
    }
    
    // Fix 3: correct fee calculation
    function calculateFee(uint256 amount) public view returns (uint256) {
        return (amount * feeRate) / 10000; // ✅
    }
    
    // Fix 4: transferWithFee with proper math
    function transferWithFee(address to, uint256 amount) external {
        require(balances[msg.sender] >= amount, "Insufficient");
        require(to != address(0), "Zero address");
        
        uint256 fee = calculateFee(amount);
        uint256 toAmount = amount - fee;
        
        balances[msg.sender] -= amount;
        balances[to] += toAmount;
        totalFees += fee;
    }
    
    // Fix 5: onlyOwner
    function setWhitelist(address user, bool status) external onlyOwner {
        whitelisted[user] = status;
    }
    
    // Fix 6: check return value
    function emergencyWithdraw() external onlyOwner {
        uint256 amount = address(this).balance;
        (bool success,) = payable(owner).call{value: amount}("");
        require(success, "Transfer failed");
    }
    
    function collectFees() external onlyOwner {
        uint256 amount = totalFees;
        totalFees = 0;
        (bool success,) = payable(owner).call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

---

## สรุป Part 27

Smart Contract Auditing ที่เรียนรู้:
- ✅ Audit process และ severity levels
- ✅ Reentrancy, integer overflow, access control
- ✅ tx.origin, signature replay, timestamp
- ✅ Static analysis tools (Slither, Mythril)
- ✅ Formal verification (Certora)
- ✅ Hands-on vulnerability exercise (7 bugs)

## Quiz

1. Critical vs High severity: ต่างกันอย่างไร?
2. CEI pattern ป้องกัน reentrancy อย่างไร?
3. tx.origin attack ทำงานอย่างไร?
4. Formal verification ต่างจาก testing อย่างไร?

---

## Next: Part 28 - Advanced Testing Patterns
