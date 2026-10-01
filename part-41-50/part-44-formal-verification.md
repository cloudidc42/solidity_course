# Part 44: Formal Verification & Static Analysis

## สารบัญ
1. Formal Verification Overview
2. Certora Prover
3. Halmos (Symbolic Testing)
4. Slither Static Analysis
5. Workshop: Verified ERC-20

---

## 1. Formal Verification Overview

```
Formal Verification คืออะไร:
- การพิสูจน์ทางคณิตศาสตร์ว่า code ถูกต้อง
- ต่างจาก testing ที่ test specific inputs
- Formal verification พิสูจน์ ALL possible inputs

Tools:
1. Certora Prover: rule-based prover (most used)
2. Halmos: symbolic execution + SMT solver
3. hevm: Ethereum symbolic VM
4. K Framework: formal semantics

เมื่อไหร่ควรใช้:
- Core DeFi protocols (Aave, Compound, MakerDAO ใช้ทั้งหมด)
- Bridges (high value, complex invariants)
- Token standards implementation
- ถ้า TVL > $100M → formal verification คุ้ม

Limitations:
- ช้า (hours to verify)
- ต้องเขียน spec เอง
- ไม่ครอบคลุม off-chain logic
- Specification bugs ก็ bug
```

---

## 2. Certora Prover

```javascript
// certora/specs/ERC20.spec

// Specification ของ ERC-20 invariants

methods {
    function totalSupply() external returns (uint256) envfree;
    function balanceOf(address) external returns (uint256) envfree;
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
    function allowance(address, address) external returns (uint256) envfree;
}

// Invariant: sum of all balances = totalSupply
invariant totalSupplyEqualsSum(address a, address b)
    (a != b) => (balanceOf(a) + balanceOf(b) <= totalSupply())
    {
        preserved with (env e) {
            require a != 0 && b != 0;
        }
    }

// Rule: transfer decreases sender balance by exactly amount
rule transferDecreasesBalance(address to, uint256 amount) {
    env e;
    address sender = e.msg.sender;
    
    uint256 senderBefore = balanceOf(sender);
    uint256 receiverBefore = balanceOf(to);
    
    require sender != to;
    require amount <= senderBefore;
    
    bool success = transfer(e, to, amount);
    
    uint256 senderAfter = balanceOf(sender);
    uint256 receiverAfter = balanceOf(to);
    
    assert success => senderAfter == senderBefore - amount;
    assert success => receiverAfter == receiverBefore + amount;
}

// Rule: no transfer can increase totalSupply
rule noInflation(method f) {
    env e;
    calldataarg args;
    
    uint256 before = totalSupply();
    f(e, args);
    uint256 after = totalSupply();
    
    // Only mint can increase supply
    assert after > before => f.selector == sig:mint(address,uint256).selector;
}

// Rule: allowance is only decreased by approved spender
rule allowanceOnlyDecreasedBySpender(address owner, address spender, uint256 amount) {
    env e;
    
    uint256 allowanceBefore = allowance(owner, spender);
    
    transferFrom(e, owner, spender, amount);
    
    uint256 allowanceAfter = allowance(owner, spender);
    
    // Only the approved spender can use allowance
    assert e.msg.sender == spender => allowanceAfter == allowanceBefore - amount;
    assert e.msg.sender != spender => allowanceAfter == allowanceBefore;
}
```

```bash
# Run Certora verification
certoraRun src/MyToken.sol:MyToken \
  --verify MyToken:certora/specs/ERC20.spec \
  --solc solc-0.8.24 \
  --msg "ERC-20 formal verification"
```

---

## 3. Halmos (Symbolic Testing)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "halmos-cheatcodes/src/SVM.sol";
import "../src/MyVault.sol";

/**
 * Halmos Symbolic Tests
 * ต่างจาก fuzz: ลองทุก possible value ด้วย SMT
 * 
 * ติดตั้ง: pip install halmos
 * รัน: halmos --contract HalmosTest
 */
contract HalmosVaultTest {
    
    MyVault vault;
    
    function setUp() public {
        vault = new MyVault();
    }
    
    // Symbolic: prove for ALL possible inputs
    function check_noOverflow(uint256 a, uint256 b) public view {
        vm.assume(a < type(uint128).max);
        vm.assume(b < type(uint128).max);
        
        // This should never overflow given the assumptions
        uint256 result = vault.add(a, b);
        assert(result == a + b);
    }
    
    // Prove: no matter what sequence of deposits/withdrawals
    // vault cannot go negative
    function check_solvency(
        uint256 depositAmount1,
        uint256 depositAmount2,
        uint256 withdrawAmount
    ) public {
        vm.assume(depositAmount1 > 0 && depositAmount1 < 1e24);
        vm.assume(depositAmount2 > 0 && depositAmount2 < 1e24);
        
        address user1 = address(0x1);
        address user2 = address(0x2);
        
        vm.deal(user1, depositAmount1);
        vm.deal(user2, depositAmount2);
        
        vm.prank(user1);
        vault.deposit{value: depositAmount1}();
        
        vm.prank(user2);
        vault.deposit{value: depositAmount2}();
        
        // Withdraw amount bounded by user1's deposit
        uint256 maxWithdraw = vault.getBalance(user1);
        vm.assume(withdrawAmount <= maxWithdraw);
        
        vm.prank(user1);
        vault.withdraw(withdrawAmount);
        
        // INVARIANT: vault balance >= total deposits remaining
        uint256 remaining = vault.getBalance(user1) + vault.getBalance(user2);
        assert(address(vault).balance >= remaining);
    }
    
    // Prove: access control cannot be bypassed
    function check_accessControl(address caller) public {
        vm.assume(caller != vault.owner()); // Caller is NOT the owner
        
        vm.prank(caller);
        
        // Try to call admin function
        try vault.setFee(100) {
            // If it didn't revert, that's a bug!
            assert(false);
        } catch {
            // Expected: should revert
        }
    }
}
```

---

## 4. Slither Static Analysis

```python
# slither_config.json
{
  "detectors_to_exclude": [],
  "exclude_informational": false,
  "exclude_low": false,
  "exclude_medium": false,
  "exclude_high": false
}
```

```bash
# Run Slither
slither src/MyContract.sol

# Generate summary
slither src/MyContract.sol --print human-summary

# Check specific detector
slither src/MyContract.sol --detect reentrancy-eth

# All available detectors:
# - reentrancy-eth, reentrancy-no-eth, reentrancy-benign
# - tx-origin
# - suicidal
# - uninitialized-storage
# - arbitrary-send-eth
# - controlled-delegatecall
# - weak-prng
# - tautology
# - boolean-equality
# - unused-return
# - low-level-calls
# - locked-ether
# - shadowing-state, shadowing-abstract
# - events-maths, events-access-control
```

```python
# Custom Slither detector
from slither.detectors.abstract_detector import AbstractDetector, DetectorClassification
from slither.core.declarations import Function

class NoTimestampManipulation(AbstractDetector):
    """
    Detect use of block.timestamp for critical decisions
    """
    
    ARGUMENT = "timestamp-manipulation"
    HELP = "Dangerous use of block.timestamp"
    IMPACT = DetectorClassification.MEDIUM
    CONFIDENCE = DetectorClassification.MEDIUM
    
    WIKI = "https://docs.soliditylang.org"
    WIKI_TITLE = "Timestamp Manipulation"
    WIKI_DESCRIPTION = "block.timestamp can be manipulated by miners"
    WIKI_EXPLOIT_SCENARIO = "Miner sets timestamp to favorable value"
    WIKI_RECOMMENDATION = "Use block.number or TWAP for critical decisions"
    
    def _detect(self):
        results = []
        
        for contract in self.slither.contracts:
            for function in contract.functions:
                for node in function.nodes:
                    if "block.timestamp" in str(node.expression):
                        # Check if used in comparison (critical)
                        if any(op in str(node.expression) for op in ["<", ">", "==", "<=", ">="]):
                            info = [
                                function, " uses block.timestamp for comparison in ",
                                node, "\n"
                            ]
                            res = self.generate_result(info)
                            results.append(res)
        
        return results
```

---

## 5. Workshop: Verified ERC-20

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Formally Verified ERC-20
 * Properties verified:
 * 1. totalSupply = sum of all balances (by construction)
 * 2. No overflow in transfer
 * 3. Allowance correctly enforced
 * 4. Only minter can mint
 */
contract VerifiedERC20 {
    
    string public name;
    string public symbol;
    uint8 public constant decimals = 18;
    
    // totalSupply is DERIVED from balances, not stored separately
    // This makes totalSupply = sum(balances) by construction
    uint256 private _totalSupply;
    
    mapping(address => uint256) private _balances;
    mapping(address => mapping(address => uint256)) private _allowances;
    
    address public minter;
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    error InsufficientBalance(address account, uint256 required, uint256 available);
    error InsufficientAllowance(address spender, uint256 required, uint256 allowed);
    error ZeroAddress();
    error NotMinter();
    
    constructor(string memory _name, string memory _symbol, address _minter) {
        name = _name;
        symbol = _symbol;
        minter = _minter;
    }
    
    function totalSupply() external view returns (uint256) {
        return _totalSupply;
    }
    
    function balanceOf(address account) external view returns (uint256) {
        return _balances[account];
    }
    
    function allowance(address owner, address spender) external view returns (uint256) {
        return _allowances[owner][spender];
    }
    
    function transfer(address to, uint256 amount) external returns (bool) {
        if (to == address(0)) revert ZeroAddress();
        
        uint256 fromBalance = _balances[msg.sender];
        if (fromBalance < amount) revert InsufficientBalance(msg.sender, amount, fromBalance);
        
        // INVARIANT: fromBalance - amount + toBalance + amount = sum(balances)
        // (no tokens created or destroyed)
        unchecked {
            _balances[msg.sender] = fromBalance - amount;
        }
        _balances[to] += amount;
        
        emit Transfer(msg.sender, to, amount);
        return true;
    }
    
    function approve(address spender, uint256 amount) external returns (bool) {
        _allowances[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        if (to == address(0)) revert ZeroAddress();
        
        uint256 currentAllowance = _allowances[from][msg.sender];
        if (currentAllowance != type(uint256).max) {
            if (currentAllowance < amount) {
                revert InsufficientAllowance(msg.sender, amount, currentAllowance);
            }
            unchecked {
                _allowances[from][msg.sender] = currentAllowance - amount;
            }
        }
        
        uint256 fromBalance = _balances[from];
        if (fromBalance < amount) revert InsufficientBalance(from, amount, fromBalance);
        
        unchecked {
            _balances[from] = fromBalance - amount;
        }
        _balances[to] += amount;
        
        emit Transfer(from, to, amount);
        return true;
    }
    
    function mint(address to, uint256 amount) external {
        if (msg.sender != minter) revert NotMinter();
        if (to == address(0)) revert ZeroAddress();
        
        _totalSupply += amount;
        _balances[to] += amount;
        
        emit Transfer(address(0), to, amount);
    }
    
    function burn(uint256 amount) external {
        uint256 fromBalance = _balances[msg.sender];
        if (fromBalance < amount) revert InsufficientBalance(msg.sender, amount, fromBalance);
        
        unchecked {
            _balances[msg.sender] = fromBalance - amount;
        }
        _totalSupply -= amount;
        
        emit Transfer(msg.sender, address(0), amount);
    }
}
```

---

## สรุป Part 44

Formal Verification ที่เรียนรู้:
- ✅ Certora Prover (rules + invariants)
- ✅ Halmos symbolic testing
- ✅ Slither static analysis
- ✅ Custom Slither detector
- ✅ Verified ERC-20 design

## Next: Part 45 - Protocol Economics Design
