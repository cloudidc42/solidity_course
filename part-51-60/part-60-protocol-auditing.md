# Part 60: Smart Contract Auditing Process

## บทนำ

Smart contract auditing เป็นขั้นตอนสำคัญก่อน deploy protocol ใดก็ตาม เพราะ:
- Smart contracts ไม่สามารถ patch ได้หลัง deploy (ส่วนใหญ่)
- Bug ใน DeFi protocol มักทำให้เสียเงินหลายล้านดอลลาร์
- User ต้องไว้ใจ protocol อย่างสมบูรณ์

**สถิติที่น่าตกใจ (ข้อมูลจาก public reports):**
- ปี 2022-2024 มีเงินถูก hack มากกว่า $3 billion จาก DeFi
- 70%+ ของ exploits มาจาก logic errors ที่ audit ควรจับได้
- Average cost of audit: $20,000-$200,000 แต่ประหยัดกว่า exploit มาก

---

## 1. Auditing Methodology

### Phase 1: Reconnaissance

```
1. Document Review
   - Read README และ specification
   - ทำความเข้าใจ business logic
   - ระบุ trust boundaries

2. Codebase Overview
   - นับ lines of code (LoC)
   - ระบุ external dependencies
   - Map contract interactions

3. Previous Audit Reports
   - อ่าน audit reports เก่า (ถ้ามี)
   - ตรวจสอบว่า fixes ถูก implement ถูกต้อง
```

### Phase 2: Threat Modeling

```
1. Asset Identification
   - Token balances ใน contracts
   - Admin privileges
   - Oracle dependencies

2. Attack Surface Analysis
   - External entry points (public/external functions)
   - Price oracle dependencies
   - Cross-contract interactions
   - Upgrade mechanisms

3. Threat Actors
   - Malicious users
   - Malicious contracts (flash loan attackers)
   - Compromised admin keys
   - MEV bots
```

### Phase 3: Code Review

```solidity
// ตัวอย่าง code review checklist ในรูปแบบ Solidity comment

// AUDIT CHECKLIST:
// [x] 1. Access control - ทุก sensitive function มี modifier?
// [x] 2. Reentrancy - ใช้ CEI pattern หรือ ReentrancyGuard?
// [ ] 3. Integer overflow - ใช้ SafeMath หรือ Solidity ^0.8?
// [ ] 4. Oracle manipulation - ใช้ TWAP หรือ single block price?
// [ ] 5. Signature replay - มี nonce/chainId ใน signed data?
```

---

## 2. Common Vulnerability Patterns

### Vulnerability 1: Reentrancy

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title ReentrancyVulnerable - VULNERABLE (อย่าใช้ใน production)
 * @notice ตัวอย่าง reentrancy vulnerability แบบคลาสสิก
 */
contract ReentrancyVulnerable {
    mapping(address => uint256) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    // BUG: State update หลัง external call
    function withdraw(uint256 amount) external {
        require(balances[msg.sender] >= amount, "Insufficient balance");

        // VULNERABLE: ส่ง ETH ก่อนอัปเดต state
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");

        // state update หลัง external call = reentrancy vulnerability
        balances[msg.sender] -= amount; // ← ยังไม่ถูกอัปเดตตอนที่ attacker เรียก withdraw ซ้ำ
    }
}

/**
 * @title ReentrancyAttacker
 * @notice Contract ที่ exploit ReentrancyVulnerable
 */
contract ReentrancyAttacker {
    ReentrancyVulnerable public target;
    uint256 public attackAmount;

    constructor(address _target) {
        target = ReentrancyVulnerable(_target);
    }

    function attack() external payable {
        require(msg.value > 0, "Need ETH");
        attackAmount = msg.value;
        target.deposit{value: msg.value}();
        target.withdraw(attackAmount);
    }

    // Fallback: เรียก withdraw ซ้ำก่อน state update
    receive() external payable {
        if (address(target).balance >= attackAmount) {
            target.withdraw(attackAmount);
        }
    }

    function getBalance() external view returns (uint256) {
        return address(this).balance;
    }
}

/**
 * @title ReentrancyFixed - SECURE
 * @notice ตัวอย่าง pattern ที่ถูกต้อง (CEI + ReentrancyGuard)
 */
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract ReentrancyFixed is ReentrancyGuard {
    mapping(address => uint256) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    // FIX 1: CEI Pattern (Checks-Effects-Interactions)
    function withdraw(uint256 amount) external nonReentrant {
        // CHECKS
        require(balances[msg.sender] >= amount, "Insufficient balance");

        // EFFECTS (state update ก่อน external call)
        balances[msg.sender] -= amount;

        // INTERACTIONS (external call ท้ายสุด)
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

### Vulnerability 2: Price Manipulation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface IUniswapV2Pair {
    function getReserves() external view returns (uint112, uint112, uint32);
}

/**
 * @title PriceManipulationVulnerable - VULNERABLE
 * @notice ใช้ spot price โดยตรงจาก AMM = manipulatable
 */
contract PriceManipulationVulnerable {
    IUniswapV2Pair public pair;

    // BUG: spot price จาก AMM ถูก manipulate ด้วย flash loan ได้
    function getTokenPrice() public view returns (uint256) {
        (uint112 reserve0, uint112 reserve1, ) = pair.getReserves();
        return uint256(reserve1) * 1e18 / uint256(reserve0); // ← vulnerable
    }

    function borrow(address token, uint256 collateralAmount) external {
        uint256 price = getTokenPrice(); // อาจถูก manipulate ใน same block
        uint256 borrowLimit = collateralAmount * price / 1e18;
        // ... ให้ยืม token ตาม borrowLimit
        // Attacker: flash loan inflate price → borrow มากกว่าควร → drain protocol
    }
}

/**
 * @title TWAPOracle
 * @notice TWAP (Time-Weighted Average Price) ที่ resistance ต่อ manipulation
 */
contract TWAPOracle {
    struct Observation {
        uint256 timestamp;
        uint256 price0Cumulative;
        uint256 price1Cumulative;
    }

    IUniswapV2Pair public pair;
    Observation[] public observations;
    uint256 public constant TWAP_PERIOD = 30 minutes;
    uint256 public constant MIN_OBSERVATIONS = 2;

    event ObservationRecorded(uint256 timestamp, uint256 price0Cumulative);

    /**
     * @notice บันทึก price observation (เรียกโดย keeper/bot)
     */
    function recordObservation() external {
        (uint112 r0, uint112 r1, uint32 blockTimestampLast) = pair.getReserves();
        require(
            observations.length == 0 ||
            block.timestamp - observations[observations.length - 1].timestamp >= 5 minutes,
            "TWAPOracle: too frequent"
        );

        // คำนวณ cumulative price (Uniswap V2 style)
        uint256 price0 = observations.length > 0
            ? observations[observations.length - 1].price0Cumulative
            : 0;
        uint256 price1 = observations.length > 0
            ? observations[observations.length - 1].price1Cumulative
            : 0;

        // เพิ่ม time-weighted price
        uint256 timeElapsed = block.timestamp - blockTimestampLast;
        if (timeElapsed > 0 && r0 > 0 && r1 > 0) {
            price0 += uint256(r1) * 1e18 / uint256(r0) * timeElapsed;
            price1 += uint256(r0) * 1e18 / uint256(r1) * timeElapsed;
        }

        observations.push(Observation({
            timestamp: block.timestamp,
            price0Cumulative: price0,
            price1Cumulative: price1
        }));

        emit ObservationRecorded(block.timestamp, price0);
    }

    /**
     * @notice ดึง TWAP ในช่วงเวลาที่กำหนด
     * @dev ยาก manipulate เพราะต้องรักษา price ผิดปกติหลายบล็อก
     */
    function getTWAP() external view returns (uint256 twapPrice) {
        require(observations.length >= MIN_OBSERVATIONS, "TWAPOracle: insufficient data");

        Observation memory latest = observations[observations.length - 1];
        require(
            block.timestamp - latest.timestamp <= TWAP_PERIOD,
            "TWAPOracle: stale data"
        );

        // หา observation ที่ใกล้เคียง TWAP_PERIOD ที่สุด
        Observation memory oldest = observations[0];
        for (uint256 i = 1; i < observations.length; i++) {
            if (block.timestamp - observations[i].timestamp <= TWAP_PERIOD) {
                oldest = i > 0 ? observations[i - 1] : observations[0];
                break;
            }
        }

        uint256 timeElapsed = latest.timestamp - oldest.timestamp;
        require(timeElapsed > 0, "TWAPOracle: zero elapsed");

        twapPrice = (latest.price0Cumulative - oldest.price0Cumulative) / timeElapsed;
    }
}
```

### Vulnerability 3: Access Control Bypass

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title AccessControlVulnerable - VULNERABLE patterns
 */
contract AccessControlVulnerable {
    address public admin;
    mapping(address => bool) public whitelist;

    constructor() {
        admin = msg.sender;
    }

    // BUG 1: ใช้ tx.origin แทน msg.sender
    function withdrawWithTxOrigin(uint256 amount) external {
        require(tx.origin == admin, "Not admin"); // ← vulnerable to phishing
        // Attacker สร้าง malicious contract ให้ admin เรียก
        // tx.origin ยังเป็น admin แม้จะผ่าน intermediary contract
        payable(admin).transfer(amount);
    }

    // BUG 2: Unprotected initialization
    address public owner;
    bool private initialized;

    function initialize(address _owner) external {
        // ไม่มีการตรวจสอบว่า initialized แล้วหรือยัง
        // BUG: ใครก็ได้ call initialize ได้!
        owner = _owner;
    }

    // BUG 3: Missing access control on critical function
    function addToWhitelist(address user) external {
        // ไม่มี modifier! ใครก็ได้ whitelist ตัวเอง
        whitelist[user] = true;
    }
}

/**
 * @title AccessControlFixed - SECURE
 * @notice Pattern ที่ถูกต้องสำหรับ access control
 */
import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/proxy/utils/Initializable.sol";

contract AccessControlFixed is AccessControl, Initializable {
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");
    bytes32 public constant WHITELIST_MANAGER = keccak256("WHITELIST_MANAGER");

    mapping(address => bool) public whitelist;

    // FIX: ใช้ initializer modifier ป้องกัน re-initialization
    function initialize(address _admin) external initializer {
        _grantRole(DEFAULT_ADMIN_ROLE, _admin);
        _grantRole(ADMIN_ROLE, _admin);
    }

    // FIX: ใช้ msg.sender ไม่ใช่ tx.origin
    function withdraw(uint256 amount) external onlyRole(ADMIN_ROLE) {
        // msg.sender ต้องเป็น admin โดยตรง
        payable(msg.sender).transfer(amount);
    }

    // FIX: Access control บน whitelist management
    function addToWhitelist(address user) external onlyRole(WHITELIST_MANAGER) {
        whitelist[user] = true;
    }

    receive() external payable {}
}
```

### Vulnerability 4: Integer Overflow Edge Cases

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title IntegerEdgeCases
 * @notice ตัวอย่าง integer overflow/underflow edge cases ที่ยังเกิดได้ใน 0.8+
 */
contract IntegerEdgeCases {
    // ใน Solidity 0.8+ arithmetic revert on overflow/underflow
    // แต่ยังมี edge cases ที่ต้องระวัง:

    // EDGE CASE 1: unchecked block
    function unsafeIncrement(uint256 x) external pure returns (uint256) {
        unchecked {
            return x + 1; // ถ้า x = type(uint256).max → wraps to 0 ไม่ revert!
        }
    }

    // EDGE CASE 2: downcasting
    function unsafeCast(uint256 large) external pure returns (uint128) {
        return uint128(large); // truncates silently! 2^128 + 1 → 1
    }

    // EDGE CASE 3: multiplication overflow ก่อน division
    function unsafeMulDiv(uint256 a, uint256 b, uint256 c) external pure returns (uint256) {
        return a * b / c; // a*b อาจ overflow แม้ผลลัพธ์สุดท้ายจะ fit
    }

    // ============ FIXED Versions ============

    // FIX 1: ใช้ checked arithmetic (default ใน 0.8+)
    function safeIncrement(uint256 x) external pure returns (uint256) {
        return x + 1; // revert ถ้า overflow
    }

    // FIX 2: Safe cast ด้วย OpenZeppelin SafeCast
    function safeCast(uint256 large) external pure returns (uint128) {
        require(large <= type(uint128).max, "SafeCast: overflow");
        return uint128(large);
    }

    // FIX 3: Full precision MulDiv (Uniswap FullMath algorithm)
    function safeMulDiv(
        uint256 a,
        uint256 b,
        uint256 denominator
    ) external pure returns (uint256 result) {
        require(denominator > 0, "Division by zero");

        // 512-bit multiply [prod1 prod0] = a * b
        uint256 prod0; // Least significant 256 bits
        uint256 prod1; // Most significant 256 bits

        assembly {
            let mm := mulmod(a, b, not(0))
            prod0 := mul(a, b)
            prod1 := sub(sub(mm, prod0), lt(mm, prod0))
        }

        // Short circuit ถ้าไม่ overflow
        if (prod1 == 0) {
            return prod0 / denominator;
        }

        require(denominator > prod1, "MulDiv overflow");

        // ใช้ algorithm จาก Uniswap V3 FullMath
        uint256 remainder;
        assembly {
            remainder := mulmod(a, b, denominator)
            prod1 := sub(prod1, gt(remainder, prod0))
            prod0 := sub(prod0, remainder)
        }

        uint256 twos = denominator & (~denominator + 1);
        assembly {
            denominator := div(denominator, twos)
            prod0 := div(prod0, twos)
            twos := add(div(sub(0, twos), twos), 1)
        }

        prod0 |= prod1 * twos;

        uint256 inv = (3 * denominator) ^ 2;
        inv *= 2 - denominator * inv;
        inv *= 2 - denominator * inv;
        inv *= 2 - denominator * inv;
        inv *= 2 - denominator * inv;
        inv *= 2 - denominator * inv;
        inv *= 2 - denominator * inv;

        result = prod0 * inv;
    }

    // EDGE CASE 4: signed integer operations
    function signedEdgeCase(int256 x) external pure returns (int256) {
        // type(int256).min / -1 = overflow!
        // -(-type(int256).min) = overflow!
        require(x != type(int256).min, "Cannot negate min int256");
        return -x;
    }

    // EDGE CASE 5: Phantom overflow ใน fee calculations
    function feeCalculationBug(
        uint256 amount,
        uint256 feeRate // basis points (0-10000)
    ) external pure returns (uint256 fee) {
        // BUG: ถ้า amount ใหญ่มาก: amount * feeRate อาจ overflow
        // fee = amount * feeRate / 10000;

        // FIX: ตรวจสอบก่อน หรือใช้ mulDiv
        require(feeRate <= 10000, "Invalid fee rate");
        // มั่นใจว่า amount * feeRate ไม่ overflow:
        // max amount = type(uint256).max / 10000 ≈ 1.15e73
        fee = amount * feeRate / 10000;
    }
}
```

---

## 3. Audit Report Format

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * ============================================================
 * AUDIT REPORT: ExampleDeFiProtocol
 * ============================================================
 * Date: 2024-01-15
 * Auditor: Security Research Team
 * Scope: contracts/core/Vault.sol, contracts/core/Oracle.sol
 * Commit: abc123def456...
 * ============================================================
 *
 * EXECUTIVE SUMMARY:
 * ทีม audit ได้ตรวจสอบ ExampleDeFiProtocol และพบ:
 * - 2 Critical findings
 * - 3 High findings
 * - 5 Medium findings
 * - 8 Low findings
 * - 12 Informational findings
 *
 * ============================================================
 * FINDINGS FORMAT:
 *
 * [FINDING-001]
 * Title: Reentrancy in withdraw() function
 * Severity: CRITICAL
 * File: contracts/core/Vault.sol
 * Line: 145
 *
 * Description:
 * ฟังก์ชัน withdraw() ทำการส่ง ETH ก่อนอัปเดต state variable
 * ทำให้ attacker สามารถเรียก withdraw() ซ้ำผ่าน receive() callback
 *
 * Proof of Concept:
 * ```
 * // Attacker contract
 * receive() external payable {
 *     if (vault.balance > 0) {
 *         vault.withdraw(amount);
 *     }
 * }
 * ```
 *
 * Impact:
 * Attacker สามารถ drain ทุก ETH ใน Vault contract
 *
 * Recommendation:
 * ใช้ CEI (Checks-Effects-Interactions) pattern:
 * ```solidity
 * function withdraw(uint256 amount) external nonReentrant {
 *     require(balances[msg.sender] >= amount); // CHECK
 *     balances[msg.sender] -= amount;           // EFFECT
 *     payable(msg.sender).transfer(amount);     // INTERACTION
 * }
 * ```
 *
 * Status: FIXED in commit def789...
 * ============================================================
 */

/**
 * @title VulnerableVault
 * @notice ตัวอย่าง contract ที่มี vulnerabilities สำหรับ demo audit
 * @dev DO NOT USE IN PRODUCTION
 */
contract VulnerableVault {
    mapping(address => uint256) public deposits;
    address public oracle;
    address public admin;

    constructor(address _oracle) {
        oracle = _oracle;
        admin = msg.sender;
    }

    // FINDING-001: CRITICAL - Reentrancy
    function withdraw() external {
        uint256 amount = deposits[msg.sender];
        require(amount > 0, "No deposits");

        // BUG: external call ก่อน state update
        (bool success, ) = msg.sender.call{value: amount}(""); // ← VULNERABLE
        require(success);
        deposits[msg.sender] = 0; // ← ควรอยู่ก่อน call
    }

    // FINDING-002: HIGH - Spot price oracle
    function getLiquidationThreshold() external view returns (uint256) {
        // BUG: อ่าน spot price ที่ manipulatable
        (bool success, bytes memory data) = oracle.staticcall(
            abi.encodeWithSignature("getPrice()")
        );
        require(success);
        uint256 price = abi.decode(data, (uint256));
        return price * 8 / 10; // ← ใช้ spot price ที่ manipulatable
    }

    // FINDING-003: HIGH - Centralization risk (single admin)
    function emergencyDrain(address to) external {
        require(msg.sender == admin, "Not admin"); // ← single point of failure
        payable(to).transfer(address(this).balance);
    }

    // FINDING-004: MEDIUM - No slippage protection
    function swap(address tokenIn, uint256 amountIn) external returns (uint256) {
        // BUG: ไม่มี minAmountOut parameter
        // User อาจได้ receive น้อยมากเพราะ sandwich attack
        return amountIn * 99 / 100; // simplified
    }

    // FINDING-005: MEDIUM - Unsafe ERC-20 transfer
    function transferToken(address token, address to, uint256 amount) external {
        // BUG: ไม่ check return value ของ transfer
        // USDT และ token บางตัว return false แทน revert
        IERC20(token).transfer(to, amount); // ← ควรใช้ SafeERC20
    }

    receive() external payable {
        deposits[msg.sender] += msg.value;
    }
}

interface IERC20 {
    function transfer(address to, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
    function approve(address spender, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}
```

---

## 4. Solidity Security Checklist (30+ items)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title SecurityChecklist
 * @notice Contract ที่ implement security best practices ครบ
 * @dev ใช้เป็น reference สำหรับ audit checklist
 *
 * ============================================================
 * SOLIDITY SECURITY CHECKLIST v2024
 * ============================================================
 *
 * ✅ REENTRANCY
 * [ ] 1. ใช้ ReentrancyGuard หรือ nonReentrant modifier
 * [ ] 2. ปฏิบัติตาม CEI (Checks-Effects-Interactions) pattern
 * [ ] 3. ระวัง cross-function reentrancy (read-only reentrancy)
 *
 * ✅ ACCESS CONTROL
 * [ ] 4. ทุก admin function มี access control
 * [ ] 5. ใช้ msg.sender ไม่ใช่ tx.origin
 * [ ] 6. Initializer functions ถูก protect ด้วย initializer modifier
 * [ ] 7. Role transfer มี 2-step process (propose + accept)
 * [ ] 8. Time locks บน critical governance actions
 *
 * ✅ ARITHMETIC
 * [ ] 9. ใช้ Solidity ^0.8.0 หรือ SafeMath
 * [ ] 10. ระวัง downcasting (uint256 → uint128)
 * [ ] 11. ระวัง mulDiv overflow ใน precision calculations
 * [ ] 12. Division ก่อน multiplication สูญเสีย precision
 * [ ] 13. ระวัง int256.min negation overflow
 *
 * ✅ ORACLE SECURITY
 * [ ] 14. ใช้ TWAP ไม่ใช่ spot price สำหรับ critical calculations
 * [ ] 15. ตรวจสอบ staleness ของ oracle data
 * [ ] 16. มี fallback oracle หรือ circuit breaker
 * [ ] 17. ใช้ multiple oracles สำหรับ validation
 *
 * ✅ TOKEN HANDLING
 * [ ] 18. ใช้ SafeERC20 สำหรับ token transfers
 * [ ] 19. ระวัง rebasing tokens (AMPL, stETH)
 * [ ] 20. ระวัง fee-on-transfer tokens
 * [ ] 21. ระวัง tokens ที่มี callback (ERC-777, ERC-1363)
 * [ ] 22. ตรวจสอบ return value ของ approve/transfer
 *
 * ✅ EXTERNAL CALLS
 * [ ] 23. ตรวจสอบ return values ของ low-level calls
 * [ ] 24. ระวัง delegatecall กับ untrusted contracts
 * [ ] 25. ตรวจสอบ code existence ก่อน call
 * [ ] 26. Gas stipend ที่กำหนดให้ call อาจไม่พอ
 *
 * ✅ RANDOMNESS
 * [ ] 27. ไม่ใช้ block.timestamp หรือ blockhash สำหรับ randomness
 * [ ] 28. ใช้ Chainlink VRF สำหรับ verifiable randomness
 *
 * ✅ SIGNATURE SECURITY
 * [ ] 29. ใช้ EIP-712 typed signatures ไม่ใช่ raw hash
 * [ ] 30. Include chainId ใน signed data
 * [ ] 31. Include nonce ใน signed data (replay protection)
 * [ ] 32. Include deadline ใน signed data
 * [ ] 33. ระวัง signature malleability (check s value)
 *
 * ✅ UPGRADEABLE CONTRACTS
 * [ ] 34. Storage layout compatibility ระหว่าง versions
 * [ ] 35. Constructor ใน implementation contracts ไม่ควรมี state
 * [ ] 36. Initializer ถูก call ครั้งเดียว
 * [ ] 37. Time lock บน upgrade functions
 *
 * ✅ FLASH LOAN VECTORS
 * [ ] 38. ไม่ใช้ token balance โดยตรงเป็น price reference
 * [ ] 39. ตรวจสอบ invariants หลัง external calls
 * [ ] 40. ระวัง single-block price manipulation
 */

import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/utils/cryptography/EIP712.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

/**
 * @title SecureVault
 * @notice Vault ที่ implement best practices ทั้งหมด
 */
contract SecureVault is ReentrancyGuard, AccessControl, EIP712 {
    using SafeERC20 for IERC20;
    using ECDSA for bytes32;

    // ============ Roles ============
    bytes32 public constant OPERATOR_ROLE = keccak256("OPERATOR_ROLE");
    bytes32 public constant GUARDIAN_ROLE = keccak256("GUARDIAN_ROLE");

    // ============ EIP-712 ============
    bytes32 public constant WITHDRAW_TYPEHASH = keccak256(
        "Withdraw(address owner,address token,uint256 amount,uint256 nonce,uint256 deadline)"
    );

    // ============ State ============
    mapping(address => mapping(address => uint256)) public deposits; // user → token → amount
    mapping(address => uint256) public nonces;

    // Pending admin transfer (2-step)
    address public pendingAdmin;
    uint256 public adminTransferInitiatedAt;
    uint256 public constant ADMIN_TRANSFER_DELAY = 2 days;

    // Circuit breaker
    bool public emergencyPaused;
    uint256 public constant MAX_SINGLE_WITHDRAWAL = 1_000_000e18;

    // ============ Events ============
    event Deposited(address indexed user, address indexed token, uint256 amount);
    event Withdrawn(address indexed user, address indexed token, uint256 amount);
    event AdminTransferInitiated(address indexed newAdmin);
    event AdminTransferCompleted(address indexed newAdmin);
    event EmergencyPause(address indexed guardian);

    // ============ Constructor ============
    constructor() EIP712("SecureVault", "1") {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(GUARDIAN_ROLE, msg.sender);
        _grantRole(OPERATOR_ROLE, msg.sender);
    }

    // ============ Modifiers ============
    modifier whenNotPaused() {
        require(!emergencyPaused, "SecureVault: paused");
        _;
    }

    // ============ User Functions ============

    /**
     * @notice Deposit ERC-20 tokens
     * @dev ใช้ SafeERC20 สำหรับ fee-on-transfer compat
     */
    function deposit(
        address token,
        uint256 amount
    ) external nonReentrant whenNotPaused {
        require(token != address(0), "SecureVault: zero token");
        require(amount > 0, "SecureVault: zero amount");

        // ใช้ balance before/after pattern สำหรับ fee-on-transfer tokens
        uint256 balanceBefore = IERC20(token).balanceOf(address(this));
        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
        uint256 balanceAfter = IERC20(token).balanceOf(address(this));

        uint256 actualAmount = balanceAfter - balanceBefore;
        deposits[msg.sender][token] += actualAmount;

        emit Deposited(msg.sender, token, actualAmount);
    }

    /**
     * @notice Withdraw tokens (CEI pattern)
     */
    function withdraw(
        address token,
        uint256 amount
    ) external nonReentrant whenNotPaused {
        // CHECKS
        require(amount > 0, "SecureVault: zero amount");
        require(amount <= MAX_SINGLE_WITHDRAWAL, "SecureVault: exceeds limit");
        require(deposits[msg.sender][token] >= amount, "SecureVault: insufficient");

        // EFFECTS (update state BEFORE external call)
        deposits[msg.sender][token] -= amount;

        // INTERACTIONS (external call LAST)
        IERC20(token).safeTransfer(msg.sender, amount);

        emit Withdrawn(msg.sender, token, amount);
    }

    /**
     * @notice Withdraw ผ่าน EIP-712 signature (gasless for user)
     */
    function withdrawWithPermit(
        address owner,
        address token,
        uint256 amount,
        uint256 deadline,
        bytes calldata signature
    ) external nonReentrant whenNotPaused {
        // ตรวจสอบ deadline
        require(block.timestamp <= deadline, "SecureVault: expired");

        // ตรวจสอบ nonce
        uint256 currentNonce = nonces[owner];

        // ตรวจสอบ EIP-712 signature
        bytes32 structHash = keccak256(abi.encode(
            WITHDRAW_TYPEHASH,
            owner,
            token,
            amount,
            currentNonce,
            deadline
        ));
        bytes32 digest = _hashTypedDataV4(structHash);
        address signer = digest.recover(signature);

        require(signer == owner, "SecureVault: invalid signature");
        require(deposits[owner][token] >= amount, "SecureVault: insufficient");
        require(amount <= MAX_SINGLE_WITHDRAWAL, "SecureVault: exceeds limit");

        // Increment nonce (replay protection)
        nonces[owner]++;

        // CEI
        deposits[owner][token] -= amount;
        IERC20(token).safeTransfer(owner, amount);

        emit Withdrawn(owner, token, amount);
    }

    // ============ Admin (2-step transfer) ============

    /**
     * @notice ขั้นตอนที่ 1: Initiate admin transfer
     */
    function initiateAdminTransfer(
        address newAdmin
    ) external onlyRole(DEFAULT_ADMIN_ROLE) {
        require(newAdmin != address(0), "SecureVault: zero address");
        pendingAdmin = newAdmin;
        adminTransferInitiatedAt = block.timestamp;
        emit AdminTransferInitiated(newAdmin);
    }

    /**
     * @notice ขั้นตอนที่ 2: Accept admin role (หลัง delay)
     */
    function acceptAdminRole() external {
        require(msg.sender == pendingAdmin, "SecureVault: not pending admin");
        require(
            block.timestamp >= adminTransferInitiatedAt + ADMIN_TRANSFER_DELAY,
            "SecureVault: delay not passed"
        );

        _revokeRole(DEFAULT_ADMIN_ROLE, getRoleMember(DEFAULT_ADMIN_ROLE, 0));
        _grantRole(DEFAULT_ADMIN_ROLE, pendingAdmin);
        emit AdminTransferCompleted(pendingAdmin);
        pendingAdmin = address(0);
    }

    // ============ Guardian ============

    /**
     * @notice Emergency pause (guardian only)
     */
    function emergencyPause() external onlyRole(GUARDIAN_ROLE) {
        emergencyPaused = true;
        emit EmergencyPause(msg.sender);
    }

    function unpause() external onlyRole(DEFAULT_ADMIN_ROLE) {
        emergencyPaused = false;
    }

    // ============ View Functions ============

    function getRoleMember(bytes32 role, uint256 index) public view returns (address) {
        // Simplified - production จะใช้ EnumerableSet
        return address(0); // placeholder
    }
}
```

---

## 5. Preparing Code for Audit

### NatSpec Documentation Example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title LendingPool
 * @author Protocol Team
 * @notice ระบบ lending pool สำหรับ deposit และ borrow tokens
 * @dev ใช้ interest rate model แบบ linear utilization-based
 *
 * Architecture:
 * - Users deposit collateral tokens เพื่อรับ interest
 * - Users borrow tokens โดยใช้ collateral
 * - Liquidation เมื่อ health factor < 1.0
 *
 * Security Assumptions:
 * - Oracle ที่ใช้ไม่ถูก manipulate (ใช้ TWAP 30 นาที)
 * - Admin key เป็น multisig 3/5
 * - Maximum single transaction = 1M tokens
 *
 * Known Limitations:
 * - ไม่รองรับ rebasing tokens
 * - ไม่รองรับ tokens ที่มี blacklist (USDC, USDT)
 */
contract LendingPoolDocumented {
    // ============ State Variables ============

    /// @notice Collateral factor ต่อ token (in basis points)
    /// @dev 8000 = 80% LTV (Loan-to-Value)
    mapping(address => uint256) public collateralFactor;

    /// @notice Interest rate model parameters
    /// @dev base + multiplier * utilization
    uint256 public baseInterestRate;    // APY ขั้นต่ำ (basis points)
    uint256 public multiplier;          // เพิ่มตาม utilization

    // ============ Functions ============

    /**
     * @notice Deposit collateral เข้า pool
     * @dev ใช้ SafeERC20 รองรับ fee-on-transfer tokens
     *      เก็บ actual amount ที่รับได้ (ไม่ใช่ amount parameter)
     *
     * Requirements:
     * - `token` ต้องเป็น whitelisted collateral
     * - `amount` ต้องมากกว่า minimum deposit (1e6)
     * - User ต้องมี balance และ allowance เพียงพอ
     *
     * @param token Address ของ token ที่ deposit
     * @param amount จำนวน token ที่ต้องการ deposit
     * @return actualAmount จำนวน token จริงที่ pool ได้รับ (หลัง fee)
     *
     * Emits {Deposited} event
     */
    function depositCollateral(
        address token,
        uint256 amount
    ) external returns (uint256 actualAmount) {
        // implementation
    }

    /**
     * @notice คำนวณ health factor ของ position
     * @dev Health factor = (collateral value * CF) / debt value
     *      ถ้า < 1e18 (1.0) = liquidatable
     *
     * @param user Address ของ user
     * @return healthFactor ค่า health factor (1e18 = 1.0)
     *
     * @custom:note ใช้ TWAP price ไม่ใช่ spot price
     */
    function getHealthFactor(address user) external view returns (uint256 healthFactor) {
        // implementation
    }
}
```

### Test Coverage Preparation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "../src/SecureVault.sol";

/**
 * @title SecureVaultTest
 * @notice Comprehensive test suite สำหรับ audit preparation
 * @dev Target coverage: 100% line + 100% branch
 */
contract SecureVaultTest is Test {
    SecureVault public vault;
    address public admin = address(1);
    address public user1 = address(2);
    address public user2 = address(3);
    address public attacker = address(4);

    MockERC20 public token;

    function setUp() public {
        vm.startPrank(admin);
        vault = new SecureVault();
        token = new MockERC20("Test Token", "TEST");
        vm.stopPrank();

        // Setup balances
        token.mint(user1, 1000e18);
        token.mint(user2, 1000e18);
        token.mint(attacker, 100e18);
    }

    // ============ Happy Path Tests ============

    function test_DepositSucceeds() public {
        vm.startPrank(user1);
        token.approve(address(vault), 100e18);
        vault.deposit(address(token), 100e18);
        vm.stopPrank();

        assertEq(vault.deposits(user1, address(token)), 100e18);
    }

    function test_WithdrawSucceeds() public {
        // Setup
        vm.startPrank(user1);
        token.approve(address(vault), 100e18);
        vault.deposit(address(token), 100e18);

        // Execute
        vault.withdraw(address(token), 50e18);
        vm.stopPrank();

        assertEq(vault.deposits(user1, address(token)), 50e18);
        assertEq(token.balanceOf(user1), 950e18);
    }

    // ============ Security Tests ============

    /// @notice ทดสอบ reentrancy attack
    function test_ReentrancyAttackFails() public {
        // Deploy attacker
        ReentrancyAttacker reAttacker = new ReentrancyAttacker(address(vault));

        vm.deal(address(reAttacker), 1 ether);

        // Mock ETH deposit
        // ใน production จะ test กับ wrapped ETH

        // ไม่ควร drain vault
        // assertEq(address(vault).balance, initialBalance);
    }

    /// @notice ทดสอบว่า replay attack ใช้ไม่ได้
    function test_SignatureReplayFails() public {
        uint256 deadline = block.timestamp + 1 hours;
        uint256 amount = 100e18;

        // Setup deposit
        vm.startPrank(user1);
        token.approve(address(vault), amount);
        vault.deposit(address(token), amount);
        vm.stopPrank();

        // สร้าง signature
        (address owner_, uint256 privKey) = makeAddrAndKey("user1");
        bytes32 structHash = keccak256(abi.encode(
            vault.WITHDRAW_TYPEHASH(),
            owner_,
            address(token),
            amount,
            vault.nonces(owner_),
            deadline
        ));

        // Replay ครั้งที่สอง ต้อง fail
        // (ครั้งแรก nonce = 0, ครั้งที่สอง nonce ไม่ตรง)
        vm.expectRevert("SecureVault: invalid signature");
        vault.withdrawWithPermit(owner_, address(token), amount, deadline, "");
    }

    /// @notice Fuzz test สำหรับ deposit/withdraw
    function testFuzz_DepositWithdraw(uint256 amount) public {
        amount = bound(amount, 1e6, 1_000_000e18);

        token.mint(user1, amount);

        vm.startPrank(user1);
        token.approve(address(vault), amount);
        vault.deposit(address(token), amount);

        vault.withdraw(address(token), amount);
        vm.stopPrank();

        assertEq(vault.deposits(user1, address(token)), 0);
        assertEq(token.balanceOf(user1), amount);
    }

    /// @notice Invariant test: total deposits = contract balance
    function invariant_TotalDepositsMatchBalance() public {
        // Balance ของ contract ต้องเท่ากับ sum ของ all deposits
        // Foundry invariant testing จะเรียก function นี้ซ้ำหลายครั้ง
    }
}

// Mock token สำหรับ testing
contract MockERC20 {
    string public name;
    string public symbol;
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    constructor(string memory _name, string memory _symbol) {
        name = _name;
        symbol = _symbol;
    }

    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount;
        totalSupply += amount;
        emit Transfer(address(0), to, amount);
    }

    function transfer(address to, uint256 amount) external returns (bool) {
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
        allowance[from][msg.sender] -= amount;
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        emit Transfer(from, to, amount);
        return true;
    }
}

contract ReentrancyAttacker {
    SecureVault vault;

    constructor(address _vault) {
        vault = SecureVault(_vault);
    }

    receive() external payable {
        // Try reentrancy
        // vault.withdraw(address(0), 1); // should fail
    }
}
```

---

## Workshop และ Exercises

### Exercise 1: Audit Report Template

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * ============================================================
 * [TEMPLATE] SMART CONTRACT AUDIT REPORT
 * ============================================================
 * Project: [Protocol Name]
 * Date: [YYYY-MM-DD]
 * Audited by: [Auditor Name/Team]
 * Scope: [list of contracts]
 * Version: [commit hash]
 * ============================================================
 *
 * ## SEVERITY CLASSIFICATION
 *
 * CRITICAL: สูญเสียเงินโดยตรง, drain funds
 *   ต้องแก้ก่อน deploy
 *
 * HIGH: เสี่ยงสูญเสียเงินในสถานการณ์บางอย่าง
 *   ต้องแก้ก่อน deploy
 *
 * MEDIUM: ระบบทำงานผิดปกติหรือ loss of functionality
 *   ควรแก้ก่อน deploy
 *
 * LOW: Best practice violations, minor issues
 *   ควรแก้ใน future version
 *
 * INFORMATIONAL: Suggestions, code quality
 *   Optional improvements
 *
 * ============================================================
 * ## FINDING FORMAT
 *
 * ### [SEVERITY]-[NUMBER]: [TITLE]
 * **File:** path/to/contract.sol
 * **Lines:** 123-145
 * **Status:** OPEN / FIXED / ACKNOWLEDGED
 *
 * **Description:**
 * [อธิบาย vulnerability]
 *
 * **Impact:**
 * [ผลกระทบต่อ users และ protocol]
 *
 * **Proof of Concept:**
 * ```solidity
 * // Attack code
 * ```
 *
 * **Recommendation:**
 * [วิธีแก้ไข]
 *
 * **Developer Response:**
 * [developer comment หรือ fix commit]
 * ============================================================
 */

/**
 * @title AuditHelper
 * @notice Helper functions สำหรับ audit process
 */
contract AuditHelper {
    /**
     * @notice ตรวจสอบ function สำคัญที่ควร check ทุก contract
     */
    struct AuditChecks {
        bool hasReentrancyGuard;
        bool usesCEIPattern;
        bool hasAccessControl;
        bool usesSafeERC20;
        bool hasNatSpec;
        bool hasEventEmission;
        bool hasInputValidation;
        bool usesEIP712ForSigs;
        uint256 testCoveragePercent;
    }

    /**
     * @notice Severity score calculator
     */
    function calculateRiskScore(
        uint8 criticalCount,
        uint8 highCount,
        uint8 mediumCount,
        uint8 lowCount
    ) external pure returns (uint256 score, string memory rating) {
        score = uint256(criticalCount) * 1000 +
                uint256(highCount) * 100 +
                uint256(mediumCount) * 10 +
                uint256(lowCount);

        if (criticalCount > 0) {
            rating = "FAIL - Critical issues must be fixed";
        } else if (highCount > 2) {
            rating = "FAIL - Too many high severity issues";
        } else if (highCount > 0) {
            rating = "CONDITIONAL PASS - Fix high severity issues";
        } else if (mediumCount > 5) {
            rating = "CONDITIONAL PASS - Address medium issues";
        } else {
            rating = "PASS - Minor issues only";
        }
    }
}
```

### Exercise 2: Vulnerability Scanner

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title VulnerabilityPatternDetector
 * @notice ตัวอย่าง patterns ที่ควร manual review อย่างละเอียด
 */
contract VulnerabilityPatternDetector {

    // PATTERN 1: Dangerous external call ก่อน state update
    // RED FLAG: external call ที่ position ไม่ใช่ท้าย function
    function dangerousPattern1(address user, uint256 amount) internal {
        // ❌ WRONG ORDER:
        // payable(user).call{value: amount}(""); // ← external call ก่อน
        // balances[user] = 0;                    // ← state update หลัง

        // ✅ CORRECT ORDER (CEI):
        // balances[user] = 0;                    // ← state update ก่อน
        // payable(user).call{value: amount}("");  // ← external call หลัง
    }

    // PATTERN 2: tx.origin ใน authorization
    function dangerousPattern2() internal view returns (bool) {
        // ❌ WRONG:
        // return tx.origin == owner;

        // ✅ CORRECT:
        return msg.sender == address(0); // placeholder
    }

    // PATTERN 3: Block timestamp dependency ที่ sensitive มาก
    function dangerousPattern3() internal view returns (bool) {
        // ❌ DANGEROUS for ±15 second precision:
        // return block.timestamp % 2 == 0; // miner can manipulate

        // ✅ ACCEPTABLE for coarse timing (>15 min):
        return block.timestamp > 1700000000; // far-future check
    }

    // PATTERN 4: Unsafe token balance ใช้เป็น oracle
    function dangerousPattern4(address pool) internal view returns (uint256) {
        // ❌ MANIPULATABLE:
        // return IERC20(token0).balanceOf(pool); // flash loan เปลี่ยนได้

        // ✅ USE RESERVES:
        // (uint112 r0, , ) = IUniswapV2Pair(pool).getReserves();
        // return r0;
        return 0;
    }
}
```

### Exercise 3: Pre-Audit Code Preparation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * PRE-AUDIT CHECKLIST FOR DEVELOPERS
 *
 * ก่อนส่ง code ให้ auditor ตรวจ ควรทำสิ่งเหล่านี้:
 *
 * 1. DOCUMENTATION
 *    [ ] NatSpec ครบทุก public/external function
 *    [ ] README อธิบาย architecture ชัดเจน
 *    [ ] Sequence diagrams สำหรับ complex flows
 *    [ ] Known assumptions และ limitations
 *    [ ] Trust model (ใคร trust ใคร)
 *
 * 2. TESTING
 *    [ ] Unit tests ครอบคลุม happy paths
 *    [ ] Unit tests ครอบคลุม edge cases
 *    [ ] Integration tests
 *    [ ] Fuzz tests สำหรับ arithmetic functions
 *    [ ] Invariant tests
 *    [ ] Coverage report (target: >95% line + branch)
 *
 * 3. CODE QUALITY
 *    [ ] ไม่มี TODO หรือ FIXME
 *    [ ] ไม่มี dead code
 *    [ ] ตัวแปร naming ชัดเจน
 *    [ ] Constants ใช้แทน magic numbers
 *    [ ] ลบ debug code และ console.log ออก
 *
 * 4. SECURITY BASICS (ทำก่อนส่ง)
 *    [ ] Slither static analysis ผ่าน
 *    [ ] Mythril scan ผ่าน
 *    [ ] Echidna fuzz testing
 *    [ ] Manual review ตาม checklist ด้านบน
 *
 * 5. DEPLOYMENT PLAN
 *    [ ] Deployment script พร้อม
 *    [ ] Testnet deployment สำเร็จ
 *    [ ] Mainnet configuration review
 *    [ ] Emergency pause plan
 */

/**
 * @title WellDocumentedContract
 * @notice ตัวอย่าง contract ที่มี documentation และ structure ที่ดี
 */
contract WellDocumentedContract {
    // ============ Type Declarations ============
    /// @notice สถานะของ position
    enum PositionStatus { ACTIVE, CLOSED, LIQUIDATED }

    /// @notice ข้อมูล position ของ user
    struct Position {
        uint256 collateral;     /// @dev Wei amount ของ collateral
        uint256 debt;           /// @dev Wei amount ของ debt
        uint256 timestamp;      /// @dev Unix timestamp ที่เปิด
        PositionStatus status;
    }

    // ============ State Variables ============

    /// @notice Map ของ user address ไปยัง positions ของพวกเขา
    mapping(address => Position[]) public positions;

    /// @notice Minimum collateral ratio (basis points, 15000 = 150%)
    uint256 public constant MIN_COLLATERAL_RATIO = 15000;

    /// @notice Maximum positions ต่อ user (ป้องกัน gas limit issues)
    uint256 public constant MAX_POSITIONS_PER_USER = 20;

    // ============ Events ============

    /// @notice Emitted เมื่อ user เปิด position ใหม่
    /// @param user Address ของ user
    /// @param positionId Index ใน array
    /// @param collateral จำนวน collateral (wei)
    /// @param debt จำนวน debt (wei)
    event PositionOpened(
        address indexed user,
        uint256 indexed positionId,
        uint256 collateral,
        uint256 debt
    );

    // ============ Errors ============

    /// @notice ใช้ custom errors เพื่อประหยัด gas
    error InsufficientCollateral(uint256 provided, uint256 required);
    error MaxPositionsReached(address user);
    error PositionNotFound(address user, uint256 positionId);

    // ============ Functions ============

    /**
     * @notice เปิด leveraged position ใหม่
     *
     * @dev คำนวณ health factor ก่อน approve position:
     *      healthFactor = (collateral * price) / debt
     *      ต้องมากกว่า MIN_COLLATERAL_RATIO
     *
     * Requirements:
     * - Collateral ratio ต้องสูงกว่า MIN_COLLATERAL_RATIO
     * - User ต้องมี positions น้อยกว่า MAX_POSITIONS_PER_USER
     * - `debtAmount` ต้องมากกว่า 0 และน้อยกว่า MAX_DEBT
     *
     * @param collateralAmount จำนวน collateral (in wei)
     * @param debtAmount จำนวน debt ที่ต้องการ (in wei)
     * @return positionId Index ของ position ที่สร้าง
     *
     * Emits {PositionOpened}
     * Reverts with {InsufficientCollateral} if ratio too low
     * Reverts with {MaxPositionsReached} if limit reached
     */
    function openPosition(
        uint256 collateralAmount,
        uint256 debtAmount
    ) external returns (uint256 positionId) {
        if (positions[msg.sender].length >= MAX_POSITIONS_PER_USER) {
            revert MaxPositionsReached(msg.sender);
        }

        uint256 collateralRatio = collateralAmount * 10000 / debtAmount;
        if (collateralRatio < MIN_COLLATERAL_RATIO) {
            revert InsufficientCollateral(collateralRatio, MIN_COLLATERAL_RATIO);
        }

        positionId = positions[msg.sender].length;
        positions[msg.sender].push(Position({
            collateral: collateralAmount,
            debt: debtAmount,
            timestamp: block.timestamp,
            status: PositionStatus.ACTIVE
        }));

        emit PositionOpened(msg.sender, positionId, collateralAmount, debtAmount);
    }
}
```

---

## สรุป Part 60

- **Auditing Methodology**: แบ่งเป็น 4 phases: Reconnaissance (document + codebase review), Threat Modeling (ระบุ assets, attack surfaces, threat actors), Code Review (manual + automated), Testing (PoC verification)
- **Common Vulnerabilities**: Reentrancy (แก้ด้วย CEI + nonReentrant), Price Manipulation (แก้ด้วย TWAP แทน spot price), Access Control Bypass (ไม่ใช้ tx.origin, protect initializers), Integer Edge Cases (downcasting, mulDiv overflow, unchecked arithmetic)
- **Audit Report Format**: แต่ละ finding ต้องมี Title, Severity (Critical/High/Medium/Low/Info), File+Line, Description, Impact, Proof of Concept, Recommendation, และ Developer Response
- **Security Checklist**: 40+ items ครอบคลุม reentrancy, access control, arithmetic, oracle, token handling, external calls, randomness, signatures, upgradeable contracts, flash loans
- **Pre-Audit Preparation**: Documentation (NatSpec ครบ, README, diagrams), Testing (>95% coverage, fuzz + invariant tests), Code Quality (ไม่มี TODOs, dead code), Static Analysis (Slither, Mythril), Deployment Plan

## Next: Part 61 - Protocol Documentation & NatSpec
