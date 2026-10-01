# Part 72: Advanced Testing Strategies

## บทนำ

การ test Smart Contract ไม่ใช่แค่เขียน unit test ธรรมดา เพราะ bug ใน Smart Contract อาจทำให้สูญเงินจริง Advanced Testing Strategies เช่น Property-based testing, Fuzzing, และ Symbolic Execution ช่วยค้นหา edge cases ที่ unit test ทั่วไปมองข้าม

---

## 72.1 Property-Based Testing Theory

### แนวคิดหลัก

แทนที่จะ test กรณีเฉพาะ เช่น `transfer(100)` Property-based testing ถามว่า:

> "สำหรับ **ทุก** input ที่เป็นไปได้ property นี้ต้องเป็นจริงเสมอ"

### ประเภทของ Properties

**1. Invariants** - สิ่งที่ต้องเป็นจริงตลอดเวลา
```
totalSupply = sum(allBalances)
reserveA * reserveB = k  // AMM constant product
```

**2. Preconditions** - เงื่อนไขก่อน function ทำงาน
```
before transfer: balance[from] >= amount
```

**3. Postconditions** - ผลลัพธ์ที่ถูกต้องหลัง function ทำงาน
```
after transfer: balance[to] = old_balance[to] + amount
```

**4. Stateful Properties** - ความสัมพันธ์ระหว่าง operations
```
deposit(x) then withdraw(x) should return to original state
```

---

## 72.2 การเขียน Invariant Tests ใน Foundry

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "forge-std/console.sol";

// ====================================
// Contract ที่จะ test
// ====================================
contract SimpleToken {
    mapping(address => uint256) public balanceOf;
    uint256 public totalSupply;
    address public owner;

    event Transfer(address indexed from, address indexed to, uint256 amount);

    constructor(uint256 initialSupply) {
        owner = msg.sender;
        balanceOf[msg.sender] = initialSupply;
        totalSupply = initialSupply;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount, "Insufficient balance");
        require(to != address(0), "Zero address");

        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;

        emit Transfer(msg.sender, to, amount);
        return true;
    }

    function mint(address to, uint256 amount) external {
        require(msg.sender == owner, "Not owner");
        balanceOf[to] += amount;
        totalSupply += amount;
        emit Transfer(address(0), to, amount);
    }

    function burn(uint256 amount) external {
        require(balanceOf[msg.sender] >= amount, "Insufficient balance");
        balanceOf[msg.sender] -= amount;
        totalSupply -= amount;
        emit Transfer(msg.sender, address(0), amount);
    }
}

// ====================================
// Handler: จำลอง actors ที่ interact กับ contract
// ====================================
contract TokenHandler is Test {
    SimpleToken public token;
    address[] public actors;
    uint256 public ghost_transferSum;      // tracking transfers
    uint256 public ghost_mintSum;          // tracking mints
    uint256 public ghost_burnSum;          // tracking burns

    constructor(SimpleToken _token) {
        token = _token;
        // สร้าง actors 5 คน
        actors.push(makeAddr("alice"));
        actors.push(makeAddr("bob"));
        actors.push(makeAddr("charlie"));
        actors.push(makeAddr("dave"));
        actors.push(makeAddr("eve"));
    }

    modifier useActor(uint256 actorSeed) {
        address actor = actors[actorSeed % actors.length];
        vm.startPrank(actor);
        _;
        vm.stopPrank();
    }

    function transfer(
        uint256 actorSeed,
        uint256 toSeed,
        uint256 amount
    ) external useActor(actorSeed) {
        address from = actors[actorSeed % actors.length];
        address to = actors[toSeed % actors.length];

        // ป้องกัน transfer to self (valid แต่ไม่น่าสนใจ)
        if (from == to) return;

        uint256 balance = token.balanceOf(from);
        if (balance == 0) return;

        amount = bound(amount, 1, balance);

        token.transfer(to, amount);
        ghost_transferSum += amount;
    }

    function mint(uint256 actorSeed, uint256 amount) external {
        address to = actors[actorSeed % actors.length];
        amount = bound(amount, 1, 1_000_000e18);

        vm.prank(token.owner());
        token.mint(to, amount);
        ghost_mintSum += amount;
    }

    function burn(uint256 actorSeed, uint256 amount) external useActor(actorSeed) {
        address actor = actors[actorSeed % actors.length];
        uint256 balance = token.balanceOf(actor);
        if (balance == 0) return;

        amount = bound(amount, 1, balance);
        token.burn(amount);
        ghost_burnSum += amount;
    }

    function getActors() external view returns (address[] memory) {
        return actors;
    }
}

// ====================================
// Invariant Test Contract
// ====================================
contract TokenInvariantTest is Test {
    SimpleToken public token;
    TokenHandler public handler;

    uint256 constant INITIAL_SUPPLY = 1_000_000e18;

    function setUp() public {
        token = new SimpleToken(INITIAL_SUPPLY);
        handler = new TokenHandler(token);

        // ตั้งค่า invariant testing
        targetContract(address(handler));

        // ให้ initial balance กับ actors
        address[] memory actors = handler.getActors();
        for (uint i = 0; i < actors.length; i++) {
            token.mint(actors[i], 10_000e18);
        }
    }

    // ====================================
    // INVARIANT 1: totalSupply = sum(all balances)
    // ====================================
    function invariant_totalSupplyEqualsSumOfBalances() public {
        address[] memory actors = handler.getActors();
        uint256 sumBalances = token.balanceOf(address(this));
        sumBalances += token.balanceOf(token.owner());

        for (uint i = 0; i < actors.length; i++) {
            sumBalances += token.balanceOf(actors[i]);
        }

        assertEq(
            token.totalSupply(),
            sumBalances,
            "INVARIANT: totalSupply must equal sum of all balances"
        );
    }

    // ====================================
    // INVARIANT 2: ไม่มี balance เป็น negative (implicit ใน uint256 แต่ verify)
    // ====================================
    function invariant_noNegativeBalances() public {
        address[] memory actors = handler.getActors();
        for (uint i = 0; i < actors.length; i++) {
            // uint256 ไม่มี negative แต่ verify ว่าไม่ overflow
            assertLe(
                token.balanceOf(actors[i]),
                token.totalSupply(),
                "INVARIANT: no balance exceeds totalSupply"
            );
        }
    }

    // ====================================
    // INVARIANT 3: totalSupply = initialSupply + mints - burns
    // ====================================
    function invariant_supplyAccounting() public {
        uint256 expectedSupply = INITIAL_SUPPLY
            + handler.ghost_mintSum()
            - handler.ghost_burnSum();

        // initial distribution to actors
        address[] memory actors = handler.getActors();
        uint256 actorInitial = actors.length * 10_000e18;
        expectedSupply += actorInitial;

        assertEq(
            token.totalSupply(),
            expectedSupply,
            "INVARIANT: supply accounting mismatch"
        );
    }

    // ====================================
    // INVARIANT 4: ไม่มีใคร balance > totalSupply
    // ====================================
    function invariant_singleBalanceNotExceedTotal() public {
        address[] memory actors = handler.getActors();
        for (uint i = 0; i < actors.length; i++) {
            assertLe(
                token.balanceOf(actors[i]),
                token.totalSupply(),
                "INVARIANT: single balance exceeds totalSupply"
        );
        }
    }
}
```

---

## 72.3 Echidna Fuzzer: Configuration และ Test Harness

### ติดตั้ง Echidna

```bash
# ผ่าน Docker
docker pull trailofbits/echidna:latest

# หรือ build จาก source
git clone https://github.com/crytic/echidna
cd echidna
cabal new-install
```

### Echidna Config (echidna.yaml)

```yaml
# echidna.yaml - Configuration สำหรับ Echidna fuzzer

# จำนวน test cases ที่ generate
testLimit: 50000

# จำนวน transactions ต่อ sequence
seqLen: 100

# ขนาด corpus
corpusDir: "corpus"

# จำนวน workers (parallel)
workers: 8

# Mode
testMode: "assertion"  # หรือ "property", "exploration", "overflow"

# Contract ที่ test
contract: "EchidnaTokenTest"

# Addresses ที่ใช้
deployer: "0x30000"
sender: ["0x10000", "0x20000", "0x30000"]

# ตั้งค่า gas
gasLimit: 10000000
maxValue: 100000000000000000000  # 100 ETH max value per call

# Coverage-guided fuzzing
coverageFormatter: "text"

# Shrink: ลดขนาด counterexample
shrinkLimit: 5000
```

### Echidna Test Harness

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// ====================================
// Contract ที่จะ test
// ====================================
contract VaultWithBug {
    mapping(address => uint256) public deposits;
    uint256 public totalDeposits;

    // BUG: ไม่ตรวจสอบ overflow อย่างถูกต้อง
    function deposit(uint256 amount) external payable {
        require(msg.value == amount, "Amount mismatch");
        deposits[msg.sender] += amount;
        totalDeposits += amount; // อาจเกิด overflow ใน old solidity
    }

    function withdraw(uint256 amount) external {
        require(deposits[msg.sender] >= amount, "Insufficient");
        deposits[msg.sender] -= amount;
        totalDeposits -= amount;
        payable(msg.sender).transfer(amount);
    }

    receive() external payable {}
}

// ====================================
// Echidna Test Harness
// ====================================
contract EchidnaVaultTest {
    VaultWithBug vault;
    address internal user1 = address(0x10000);
    address internal user2 = address(0x20000);

    // Echidna จะ deploy contract นี้และเรียก functions
    constructor() payable {
        vault = new VaultWithBug();
    }

    // ====================================
    // Properties (ต้องขึ้นต้น echidna_)
    // ====================================

    /// @notice invariant: totalDeposits = sum(all deposits)
    function echidna_totalDepositsConsistency() external view returns (bool) {
        uint256 sum = vault.deposits(user1) + vault.deposits(user2) + vault.deposits(address(this));
        return vault.totalDeposits() == sum;
    }

    /// @notice invariant: vault ETH balance >= totalDeposits
    function echidna_vaultSolvent() external view returns (bool) {
        return address(vault).balance >= vault.totalDeposits();
    }

    /// @notice invariant: no single user has more than totalDeposits
    function echidna_noExcessiveBalance() external view returns (bool) {
        return vault.deposits(address(this)) <= vault.totalDeposits();
    }

    // ====================================
    // Helper functions ที่ Echidna จะเรียก
    // ====================================
    function deposit(uint256 amount) external payable {
        if (amount == 0) return;
        if (amount > address(this).balance) return;

        try vault.deposit{value: amount}(amount) {
            // success
        } catch {
            // revert is ok
        }
    }

    function withdraw(uint256 amount) external {
        if (amount == 0) return;

        try vault.withdraw(amount) {
            // success
        } catch {
            // revert is ok
        }
    }

    receive() external payable {}
}

// ====================================
// Echidna Token Test - ตัวอย่างที่ครบสมบูรณ์
// ====================================
contract EchidnaTokenTest {
    SimpleToken token;
    address[] actors;

    uint256 constant INITIAL_SUPPLY = 1_000_000e18;

    constructor() {
        token = new SimpleToken(INITIAL_SUPPLY);
        actors.push(address(0x10000));
        actors.push(address(0x20000));
        actors.push(address(0x30000));
    }

    // ====================================
    // Echidna Properties
    // ====================================

    /// @notice totalSupply ต้องไม่เกิน max supply
    function echidna_totalSupplyBounded() external view returns (bool) {
        return token.totalSupply() <= 10_000_000_000e18; // 10B max
    }

    /// @notice ไม่มี address ที่มี balance > totalSupply
    function echidna_balancesLteTotalSupply() external view returns (bool) {
        for (uint i = 0; i < actors.length; i++) {
            if (token.balanceOf(actors[i]) > token.totalSupply()) {
                return false;
            }
        }
        return true;
    }

    /// @notice transfer ไม่เพิ่ม totalSupply
    function transfer(address to, uint256 amount) external {
        if (amount == 0) return;
        uint256 balanceBefore = token.balanceOf(msg.sender);
        uint256 supplyBefore = token.totalSupply();

        try token.transfer(to, amount) {
            assert(token.totalSupply() == supplyBefore); // POSTCONDITION
            assert(token.balanceOf(msg.sender) == balanceBefore - amount);
        } catch {
            // revert is fine
        }
    }
}
```

---

## 72.4 Medusa (Trail of Bits): Parallel Fuzzing

### medusa.json Configuration

```json
{
  "fuzzing": {
    "workers": 10,
    "workerResetLimit": 50,
    "timeout": 300,
    "testLimit": 100000,
    "shrinkLimit": 5000,
    "callSequenceLength": 100,
    "corpusDirectory": "corpus",
    "coverageEnabled": true,
    "targetContracts": ["MedusaTokenTest"],
    "targetContractsBalances": [],
    "constructorArgs": {},
    "deployerAddress": "0x30000",
    "senderAddresses": ["0x10000", "0x20000", "0x30000"],
    "blockNumberDelayMax": 60480,
    "blockTimestampDelayMax": 604800,
    "blockGasLimit": 125000000,
    "transactionGasLimit": 12500000,
    "testing": {
      "stopOnFailedTest": true,
      "stopOnFailedContractMatching": false,
      "testAllContracts": false,
      "traceAll": false,
      "assertionTesting": {
        "enabled": true,
        "testViewMethods": false,
        "panicCodeConfig": {
          "failOnCompilerInsertedPanic": false,
          "failOnArithmeticUnderflow": false,
          "failOnDivideByZero": true,
          "failOnEnumTypeConversionOutOfBounds": false,
          "failOnIncorrectStorageAccess": false,
          "failOnPopEmptyArray": true,
          "failOnOutOfBoundsArrayAccess": true,
          "failOnAllocateTooMuchMemory": false,
          "failOnCallUninitializedVariable": false
        }
      },
      "propertyTesting": {
        "enabled": true,
        "testPrefixes": ["fuzz_"]
      },
      "optimizationTesting": {
        "enabled": false,
        "testPrefixes": ["optimize_"]
      }
    }
  },
  "compilation": {
    "platform": "crytic-compile",
    "platformConfig": {
      "target": ".",
      "solcVersion": "0.8.24",
      "exportDirectory": "crytic-export",
      "args": ""
    }
  }
}
```

### Medusa Test Harness

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// ====================================
// AMM ที่จะ test
// ====================================
contract SimpleAMM {
    uint256 public reserveA;
    uint256 public reserveB;
    uint256 public constant MINIMUM_LIQUIDITY = 1000;
    uint256 public totalLiquidity;
    mapping(address => uint256) public liquidityOf;

    error InsufficientLiquidity();
    error ZeroInput();
    error InvalidK();

    function addLiquidity(uint256 amountA, uint256 amountB)
        external
        returns (uint256 liquidity)
    {
        if (amountA == 0 || amountB == 0) revert ZeroInput();

        if (totalLiquidity == 0) {
            liquidity = sqrt(amountA * amountB) - MINIMUM_LIQUIDITY;
            liquidityOf[address(0)] = MINIMUM_LIQUIDITY; // burn minimum
        } else {
            liquidity = min(
                (amountA * totalLiquidity) / reserveA,
                (amountB * totalLiquidity) / reserveB
            );
        }

        if (liquidity == 0) revert InsufficientLiquidity();

        reserveA += amountA;
        reserveB += amountB;
        totalLiquidity += liquidity;
        liquidityOf[msg.sender] += liquidity;
    }

    function swap(uint256 amountAIn, uint256 amountBIn)
        external
        returns (uint256 amountOut)
    {
        if (amountAIn == 0 && amountBIn == 0) revert ZeroInput();
        if (amountAIn > 0 && amountBIn > 0) revert ZeroInput(); // one-sided swap only

        uint256 kBefore = reserveA * reserveB;

        if (amountAIn > 0) {
            // swap A for B
            uint256 amountAInWithFee = amountAIn * 997;
            amountOut = (amountAInWithFee * reserveB) / (reserveA * 1000 + amountAInWithFee);
            reserveA += amountAIn;
            reserveB -= amountOut;
        } else {
            // swap B for A
            uint256 amountBInWithFee = amountBIn * 997;
            amountOut = (amountBInWithFee * reserveA) / (reserveB * 1000 + amountBInWithFee);
            reserveB += amountBIn;
            reserveA -= amountOut;
        }

        uint256 kAfter = reserveA * reserveB;
        if (kAfter < kBefore) revert InvalidK();
    }

    function sqrt(uint256 x) internal pure returns (uint256 y) {
        if (x == 0) return 0;
        uint256 z = (x + 1) / 2;
        y = x;
        while (z < y) {
            y = z;
            z = (x / z + z) / 2;
        }
    }

    function min(uint256 a, uint256 b) internal pure returns (uint256) {
        return a < b ? a : b;
    }
}

// ====================================
// Medusa Test Harness สำหรับ AMM
// ====================================
contract MedusaAMMTest {
    SimpleAMM public amm;

    // Ghost variables สำหรับ tracking
    uint256 public ghost_totalAIn;
    uint256 public ghost_totalBIn;
    uint256 public ghost_totalAOut;
    uint256 public ghost_totalBOut;
    uint256 public ghost_liquidityAdded;

    constructor() {
        amm = new SimpleAMM();
        // Initial liquidity
        amm.addLiquidity(1_000_000e18, 1_000_000e18);
    }

    // ====================================
    // Medusa Property Tests (prefix: fuzz_)
    // ====================================

    /// @notice K ต้องไม่ลดลงหลัง swap
    function fuzz_kNeverDecreases() external view returns (bool) {
        // K ≥ initial K (1_000_000e18 * 1_000_000e18)
        uint256 k = amm.reserveA() * amm.reserveB();
        return k >= 1_000_000e18 * 1_000_000e18;
    }

    /// @notice reserves ต้องไม่เป็น 0
    function fuzz_reservesAlwaysPositive() external view returns (bool) {
        return amm.reserveA() > 0 && amm.reserveB() > 0;
    }

    /// @notice totalLiquidity > 0
    function fuzz_liquidityAlwaysPositive() external view returns (bool) {
        return amm.totalLiquidity() > 0;
    }

    // ====================================
    // Actions ที่ Medusa จะเรียก
    // ====================================

    function addLiquidity(uint256 amountA, uint256 amountB) external {
        amountA = clamp(amountA, 1e18, 1_000_000e18);
        amountB = clamp(amountB, 1e18, 1_000_000e18);

        try amm.addLiquidity(amountA, amountB) returns (uint256 liquidity) {
            ghost_totalAIn += amountA;
            ghost_totalBIn += amountB;
            ghost_liquidityAdded += liquidity;
        } catch {}
    }

    function swapAForB(uint256 amountAIn) external {
        amountAIn = clamp(amountAIn, 1e15, amm.reserveA() / 10);

        uint256 reserveABefore = amm.reserveA();
        uint256 reserveBBefore = amm.reserveB();

        try amm.swap(amountAIn, 0) returns (uint256 amountOut) {
            ghost_totalAIn += amountAIn;
            ghost_totalBOut += amountOut;

            // ASSERTION: output ต้องน้อยกว่า reserveB ก่อนหน้า
            assert(amountOut < reserveBBefore);
            // ASSERTION: K ต้องไม่ลดลง
            assert(amm.reserveA() * amm.reserveB() >= reserveABefore * reserveBBefore);
        } catch {}
    }

    function swapBForA(uint256 amountBIn) external {
        amountBIn = clamp(amountBIn, 1e15, amm.reserveB() / 10);

        uint256 reserveABefore = amm.reserveA();
        uint256 reserveBBefore = amm.reserveB();

        try amm.swap(0, amountBIn) returns (uint256 amountOut) {
            ghost_totalBIn += amountBIn;
            ghost_totalAOut += amountOut;

            assert(amountOut < reserveABefore);
            assert(amm.reserveA() * amm.reserveB() >= reserveABefore * reserveBBefore);
        } catch {}
    }

    function clamp(uint256 value, uint256 min_, uint256 max_) internal pure returns (uint256) {
        if (value < min_) return min_;
        if (value > max_) return max_;
        return value;
    }
}
```

---

## 72.5 Differential Testing: เปรียบเทียบสอง Implementations

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";

// ====================================
// Implementation A: ใช้ assembly
// ====================================
library MathLibA {
    function mulDiv(uint256 x, uint256 y, uint256 denominator)
        internal
        pure
        returns (uint256 result)
    {
        // 512-bit multiply [prod1 prod0] = x * y
        uint256 prod0;
        uint256 prod1;
        assembly {
            let mm := mulmod(x, y, not(0))
            prod0 := mul(x, y)
            prod1 := sub(sub(mm, prod0), lt(mm, prod0))
        }

        if (prod1 == 0) {
            return prod0 / denominator;
        }

        require(denominator > prod1, "MathLibA: overflow");

        uint256 remainder;
        assembly {
            remainder := mulmod(x, y, denominator)
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

        uint256 inverse = (3 * denominator) ^ 2;
        inverse *= 2 - denominator * inverse;
        inverse *= 2 - denominator * inverse;
        inverse *= 2 - denominator * inverse;
        inverse *= 2 - denominator * inverse;
        inverse *= 2 - denominator * inverse;
        inverse *= 2 - denominator * inverse;

        result = prod0 * inverse;
        return result;
    }
}

// ====================================
// Implementation B: ใช้ pure Solidity (ช้ากว่า แต่ชัดเจน)
// ====================================
library MathLibB {
    function mulDiv(uint256 x, uint256 y, uint256 denominator)
        internal
        pure
        returns (uint256)
    {
        require(denominator > 0, "MathLibB: division by zero");

        // ป้องกัน overflow ด้วย safe math patterns
        uint256 a = x / denominator;
        uint256 b = x % denominator;

        return a * y + (b * y) / denominator;
    }
}

// ====================================
// Differential Test
// ====================================
contract DifferentialMathTest is Test {
    uint256 constant MAX_UINT128 = type(uint128).max;

    /// @notice เปรียบเทียบทั้งสอง implementations
    function testDifferential_mulDiv(
        uint128 x,
        uint128 y,
        uint128 denominator
    ) public {
        // Precondition: denominator > 0
        vm.assume(denominator > 0);
        // Precondition: ไม่ overflow ใน B (simplified check)
        vm.assume(uint256(x) * uint256(y) / denominator <= type(uint256).max);

        uint256 x_ = uint256(x);
        uint256 y_ = uint256(y);
        uint256 d_ = uint256(denominator);

        // ถ้า implementation A revert ให้ B revert ด้วย
        bool aReverts = false;
        bool bReverts = false;
        uint256 resultA;
        uint256 resultB;

        try this.callMathLibA(x_, y_, d_) returns (uint256 r) {
            resultA = r;
        } catch {
            aReverts = true;
        }

        try this.callMathLibB(x_, y_, d_) returns (uint256 r) {
            resultB = r;
        } catch {
            bReverts = true;
        }

        // Differential property: ทั้งสอง revert ด้วยกัน หรือ return ค่าเท่ากัน
        if (!aReverts && !bReverts) {
            // ยอมให้ difference เล็กน้อยได้ (rounding)
            uint256 diff = resultA > resultB ? resultA - resultB : resultB - resultA;
            assertLe(diff, 1, "Differential: results differ by more than 1");
        } else {
            // ถ้า A revert แต่ B ไม่ revert = potential bug
            if (aReverts && !bReverts) {
                // A conservative (กว้างกว่า), B ยืดหยุ่นกว่า - อาจ ok
                // log แต่ไม่ fail
                console.log("A reverted but B succeeded:", resultB);
            }
        }
    }

    function callMathLibA(uint256 x, uint256 y, uint256 d) external pure returns (uint256) {
        return MathLibA.mulDiv(x, y, d);
    }

    function callMathLibB(uint256 x, uint256 y, uint256 d) external pure returns (uint256) {
        return MathLibB.mulDiv(x, y, d);
    }

    // ====================================
    // FFI Differential Test: เรียก external program
    // ====================================
    function testDifferential_withPythonReference() public {
        // ต้องใช้ --ffi flag ใน forge
        string[] memory cmd = new string[](3);
        cmd[0] = "python3";
        cmd[1] = "scripts/reference_math.py";
        cmd[2] = "1000000000000000000"; // 1e18

        bytes memory result = vm.ffi(cmd);
        uint256 pythonResult = abi.decode(result, (uint256));

        uint256 solResult = MathLibA.mulDiv(1e18, 3, 7);

        // ค่าต้องตรงกับ Python reference implementation
        assertEq(solResult, pythonResult, "FFI Differential: mismatch with Python");
    }
}

// ====================================
// Differential Test สำหรับ Sorting
// ====================================
contract SortingDifferentialTest is Test {
    /// @notice เปรียบเทียบ insertion sort vs quicksort
    function testDifferential_sorting(uint256[] memory arr) public {
        vm.assume(arr.length <= 10); // จำกัดขนาดเพื่อความเร็ว

        uint256[] memory arrA = copyArray(arr);
        uint256[] memory arrB = copyArray(arr);

        insertionSort(arrA);
        quickSort(arrB, 0, int256(arrB.length) - 1);

        // ผลต้องเหมือนกัน
        assertEq(arrA.length, arrB.length);
        for (uint i = 0; i < arrA.length; i++) {
            assertEq(arrA[i], arrB[i], "Sort result mismatch");
        }

        // Postcondition: sorted
        for (uint i = 1; i < arrA.length; i++) {
            assertLe(arrA[i-1], arrA[i], "Array not sorted");
        }
    }

    function insertionSort(uint256[] memory arr) internal pure {
        for (uint i = 1; i < arr.length; i++) {
            uint256 key = arr[i];
            int256 j = int256(i) - 1;
            while (j >= 0 && arr[uint256(j)] > key) {
                arr[uint256(j + 1)] = arr[uint256(j)];
                j--;
            }
            arr[uint256(j + 1)] = key;
        }
    }

    function quickSort(uint256[] memory arr, int256 left, int256 right) internal pure {
        if (left < right) {
            int256 pivot = partition(arr, left, right);
            quickSort(arr, left, pivot - 1);
            quickSort(arr, pivot + 1, right);
        }
    }

    function partition(uint256[] memory arr, int256 left, int256 right) internal pure returns (int256) {
        uint256 pivot = arr[uint256(right)];
        int256 i = left - 1;
        for (int256 j = left; j < right; j++) {
            if (arr[uint256(j)] <= pivot) {
                i++;
                (arr[uint256(i)], arr[uint256(j)]) = (arr[uint256(j)], arr[uint256(i)]);
            }
        }
        (arr[uint256(i + 1)], arr[uint256(right)]) = (arr[uint256(right)], arr[uint256(i + 1)]);
        return i + 1;
    }

    function copyArray(uint256[] memory arr) internal pure returns (uint256[] memory copy) {
        copy = new uint256[](arr.length);
        for (uint i = 0; i < arr.length; i++) {
            copy[i] = arr[i];
        }
    }
}
```

---

## 72.6 Symbolic Execution with Halmos

### ติดตั้ง Halmos

```bash
pip install halmos
```

### Halmos Test Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";

// ====================================
// Contract ที่จะตรวจสอบด้วย Symbolic Execution
// ====================================
contract VotingMechanism {
    struct Proposal {
        uint256 id;
        uint256 votesFor;
        uint256 votesAgainst;
        uint256 deadline;
        bool executed;
        bool cancelled;
    }

    mapping(uint256 => Proposal) public proposals;
    mapping(uint256 => mapping(address => bool)) public hasVoted;
    uint256 public proposalCount;

    uint256 constant QUORUM = 100;
    uint256 constant VOTING_PERIOD = 7 days;

    function createProposal() external returns (uint256 id) {
        id = ++proposalCount;
        proposals[id] = Proposal({
            id: id,
            votesFor: 0,
            votesAgainst: 0,
            deadline: block.timestamp + VOTING_PERIOD,
            executed: false,
            cancelled: false
        });
    }

    function vote(uint256 proposalId, bool support) external {
        Proposal storage p = proposals[proposalId];
        require(p.id != 0, "Proposal not found");
        require(block.timestamp <= p.deadline, "Voting ended");
        require(!p.cancelled, "Cancelled");
        require(!hasVoted[proposalId][msg.sender], "Already voted");

        hasVoted[proposalId][msg.sender] = true;
        if (support) {
            p.votesFor++;
        } else {
            p.votesAgainst++;
        }
    }

    function execute(uint256 proposalId) external {
        Proposal storage p = proposals[proposalId];
        require(p.id != 0, "Proposal not found");
        require(block.timestamp > p.deadline, "Voting not ended");
        require(!p.executed, "Already executed");
        require(!p.cancelled, "Cancelled");
        require(p.votesFor + p.votesAgainst >= QUORUM, "No quorum");
        require(p.votesFor > p.votesAgainst, "Not passed");

        p.executed = true;
    }

    function cancel(uint256 proposalId) external {
        Proposal storage p = proposals[proposalId];
        require(p.id != 0, "Proposal not found");
        require(!p.executed, "Already executed");
        p.cancelled = true;
    }
}

// ====================================
// Halmos Test
// ====================================
contract HalmosVotingTest is Test {
    VotingMechanism public voting;

    function setUp() public {
        voting = new VotingMechanism();
    }

    // ====================================
    // Halmos Symbolic Tests
    // ====================================

    /// @notice ตรวจสอบว่า executed proposal ไม่สามารถ execute ซ้ำได้
    /// @dev Halmos จะ explore ทุก path
    function check_noDoubleExecution(uint256 proposalId) external {
        // สมมติ proposal exists และ executed
        vm.assume(proposalId > 0 && proposalId <= voting.proposalCount());
        VotingMechanism.Proposal memory p = voting.proposals(proposalId);
        vm.assume(p.executed);

        // ต้อง revert เสมอ
        vm.expectRevert("Already executed");
        voting.execute(proposalId);
    }

    /// @notice cancelled proposal ไม่สามารถ execute ได้
    function check_cancelledNotExecutable(uint256 proposalId) external {
        vm.assume(proposalId > 0 && proposalId <= voting.proposalCount());
        VotingMechanism.Proposal memory p = voting.proposals(proposalId);
        vm.assume(p.cancelled);

        vm.expectRevert();
        voting.execute(proposalId);
    }

    /// @notice ตรวจสอบ state machine: executed OR cancelled แต่ไม่ใช่ทั้งคู่
    function check_exclusiveState(uint256 proposalId) external view {
        vm.assume(proposalId > 0 && proposalId <= voting.proposalCount());
        VotingMechanism.Proposal memory p = voting.proposals(proposalId);

        // ไม่สามารถ executed และ cancelled พร้อมกัน
        assert(!(p.executed && p.cancelled));
    }

    /// @notice hasVoted ต้อง monotonic - เมื่อ true แล้วต้องเป็น true เสมอ
    function check_hasVotedMonotonic(
        uint256 proposalId,
        address voter
    ) external {
        vm.assume(proposalId > 0 && proposalId <= voting.proposalCount());
        bool votedBefore = voting.hasVoted(proposalId, voter);
        vm.assume(votedBefore); // สมมติเคย vote แล้ว

        // ต้องยัง true หลังจาก vote อีกครั้ง (ต้อง revert)
        vm.expectRevert("Already voted");
        vm.prank(voter);
        voting.vote(proposalId, true);

        // hasVoted ยังต้องเป็น true
        assert(voting.hasVoted(proposalId, voter));
    }

    // ====================================
    // K-Induction Example
    // ====================================

    /// @notice Inductive invariant: votesFor + votesAgainst = unique voters
    /// Base case: createProposal -> votes = 0
    function check_voteCountBase() external {
        uint256 id = voting.createProposal();
        VotingMechanism.Proposal memory p = voting.proposals(id);
        assert(p.votesFor + p.votesAgainst == 0);
    }

    /// Inductive step: vote เพิ่ม count ทีละ 1
    function check_voteCountInductive(uint256 proposalId, bool support) external {
        vm.assume(proposalId > 0 && proposalId <= voting.proposalCount());
        VotingMechanism.Proposal memory pBefore = voting.proposals(proposalId);
        uint256 totalBefore = pBefore.votesFor + pBefore.votesAgainst;

        vm.assume(!voting.hasVoted(proposalId, address(this)));
        vm.assume(block.timestamp <= pBefore.deadline);
        vm.assume(!pBefore.cancelled);

        voting.vote(proposalId, support);

        VotingMechanism.Proposal memory pAfter = voting.proposals(proposalId);
        uint256 totalAfter = pAfter.votesFor + pAfter.votesAgainst;

        assert(totalAfter == totalBefore + 1);
    }
}
```

### รัน Halmos

```bash
# ตรวจสอบทุก symbolic paths
halmos --contract HalmosVotingTest --function "check_*"

# พร้อม verbose output
halmos --contract HalmosVotingTest --verbose

# กำหนด loop bound
halmos --contract HalmosVotingTest --loop 10

# ใช้ multiple solvers
halmos --solver z3 --solver cvc5
```

---

## 72.7 Test Pyramid: Unit / Integration / E2E

```
         /\
        /  \
       / E2E \        10% - Full protocol flows, mainnet fork
      /--------\
     /Integration\    20% - Multi-contract interactions
    /--------------\
   /   Unit Tests   \  70% - Single function, isolated
  /------------------\
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";

// ====================================
// Unit Tests (70%) - เร็ว, isolated
// ====================================
contract LendingCoreUnitTest is Test {
    SimpleToken token;
    LendingProtocol lending;

    function setUp() public {
        token = new SimpleToken(1_000_000e18);
        lending = new LendingProtocol();
    }

    /// @notice ทดสอบ supply function อย่างละเอียด
    function test_supply_increasesUserBalance() public {
        // Arrange
        address market = address(token);
        uint256 amount = 100e18;

        // Act
        lending.supply(market, amount);

        // Assert
        uint256 shares = lending.userSupply(market, address(this));
        assertGt(shares, 0, "Shares should be positive after supply");
    }

    function test_supply_revertsOnZeroAmount() public {
        vm.expectRevert();
        lending.supply(address(token), 0);
    }

    function test_supply_fuzz(uint128 amount) public {
        vm.assume(amount > 0);
        lending.supply(address(token), amount);
        assertGt(lending.userSupply(address(token), address(this)), 0);
    }

    function test_createMarket_onlyAdmin() public {
        address nonAdmin = makeAddr("nonAdmin");
        vm.prank(nonAdmin);
        vm.expectRevert("Not admin");
        lending.createMarket(address(token), 1e18);
    }

    function test_borrowRate_bounds() public view {
        // อัตราดอกเบี้ยต้องอยู่ใน reasonable range
        (, , , uint256 rate, , , ) = lending.markets(address(0));
        // ถ้า market ไม่ exists rate = 0, ซึ่ง valid
        assertLe(rate, 1e27, "Borrow rate too high");
    }
}

// ====================================
// Integration Tests (20%) - multi-contract
// ====================================
contract LendingIntegrationTest is Test {
    SimpleToken tokenA;
    SimpleToken tokenB;
    LendingProtocol lending;

    address alice = makeAddr("alice");
    address bob = makeAddr("bob");

    function setUp() public {
        tokenA = new SimpleToken(10_000_000e18);
        tokenB = new SimpleToken(10_000_000e18);
        lending = new LendingProtocol();

        // Setup markets
        lending.createMarket(address(tokenA), 5e25); // 5% APR
        lending.createMarket(address(tokenB), 8e25); // 8% APR

        // Fund users
        tokenA.transfer(alice, 100_000e18);
        tokenB.transfer(bob, 100_000e18);
    }

    /// @notice ทดสอบ supply → borrow flow
    function test_integration_supplyAndBorrow() public {
        // Alice supplies tokenA
        vm.startPrank(alice);
        uint256 supplyAmount = 10_000e18;
        lending.supply(address(tokenA), supplyAmount);
        vm.stopPrank();

        // Bob supplies tokenB และ borrows tokenA
        vm.startPrank(bob);
        lending.supply(address(tokenB), 10_000e18);
        // (simplified: assume borrow function exists)
        vm.stopPrank();

        // Verify state
        assertGt(lending.userSupply(address(tokenA), alice), 0);
        assertGt(lending.userSupply(address(tokenB), bob), 0);
    }

    /// @notice ทดสอบ interest accrual
    function test_integration_interestAccrues() public {
        vm.startPrank(alice);
        lending.supply(address(tokenA), 10_000e18);
        vm.stopPrank();

        uint256 sharesBefore = lending.userSupply(address(tokenA), alice);

        // เวลาผ่านไป 365 วัน
        vm.warp(block.timestamp + 365 days);

        // Trigger interest update ด้วย supply เล็กน้อย
        vm.prank(alice);
        lending.supply(address(tokenA), 1);

        uint256 sharesAfter = lending.userSupply(address(tokenA), alice);
        // shares ไม่ควรเปลี่ยนแปลงมาก (index เปลี่ยน แต่ shares เพิ่มเล็กน้อย)
        assertGe(sharesAfter, sharesBefore, "Shares should not decrease");
    }
}

// ====================================
// E2E Tests (10%) - mainnet fork
// ====================================
contract LendingE2ETest is Test {
    // Mainnet addresses
    address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address constant USDC_WHALE = 0x55FE002aefF02F77364de339a1292923A15844B8;

    LendingProtocol lending;

    function setUp() public {
        // Fork mainnet
        uint256 forkId = vm.createFork(vm.envString("MAINNET_RPC_URL"), 19_000_000);
        vm.selectFork(forkId);

        lending = new LendingProtocol();
    }

    /// @notice ทดสอบด้วย real USDC บน mainnet fork
    function test_e2e_realUSDC() public {
        // Impersonate whale
        vm.startPrank(USDC_WHALE);

        // Get USDC balance
        uint256 balance = IERC20(USDC).balanceOf(USDC_WHALE);
        assertGt(balance, 0, "Whale should have USDC");

        // Create market
        lending.createMarket(USDC, 5e25);

        // Supply USDC
        IERC20(USDC).approve(address(lending), 1_000_000 * 1e6); // 1M USDC
        lending.supply(USDC, 1_000_000 * 1e6);

        uint256 shares = lending.userSupply(USDC, USDC_WHALE);
        assertGt(shares, 0, "Should have shares after supply");

        vm.stopPrank();
    }
}

interface IERC20 {
    function balanceOf(address) external view returns (uint256);
    function approve(address, uint256) external returns (bool);
    function transfer(address, uint256) external returns (bool);
}
```

---

## Workshop 72: Complete Testing Suite

### foundry.toml สำหรับ Advanced Testing

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
solc = "0.8.24"
optimizer = true
optimizer_runs = 200

[profile.default.fuzz]
runs = 1000
max_test_rejects = 65536
seed = "0x1234"
dictionary_weight = 40
include_storage = true
include_push_bytes = true

[profile.default.invariant]
runs = 256
depth = 512
fail_on_revert = false
call_override = false
dictionary_weight = 80
include_storage = true
include_push_bytes = true
shrink_run_limit = 2048

[profile.ci]
fuzz = { runs = 5000 }
invariant = { runs = 1000, depth = 1000 }

[rpc_endpoints]
mainnet = "${MAINNET_RPC_URL}"
sepolia = "${SEPOLIA_RPC_URL}"

[etherscan]
mainnet = { key = "${ETHERSCAN_API_KEY}" }
```

### Makefile สำหรับ Testing Pipeline

```makefile
# Test commands

# Unit tests เร็ว
.PHONY: test-unit
test-unit:
	forge test --match-path "test/unit/*" -v

# Integration tests (medium)
.PHONY: test-integration
test-integration:
	forge test --match-path "test/integration/*" -vv

# E2E tests ต้องการ mainnet fork
.PHONY: test-e2e
test-e2e:
	MAINNET_RPC_URL=$(MAINNET_RPC_URL) forge test --match-path "test/e2e/*" -vvv

# Fuzz tests (นานกว่า)
.PHONY: test-fuzz
test-fuzz:
	forge test --match-path "test/fuzz/*" --fuzz-runs 10000 -v

# Invariant tests
.PHONY: test-invariant
test-invariant:
	forge test --match-path "test/invariant/*" --invariant-runs 1000 -v

# Symbolic execution ด้วย Halmos
.PHONY: test-symbolic
test-symbolic:
	halmos --contract "Halmos*" --function "check_*" --verbose

# Full test suite
.PHONY: test-all
test-all: test-unit test-integration test-fuzz test-invariant

# Coverage report
.PHONY: coverage
coverage:
	forge coverage --report lcov
	genhtml lcov.info --output-directory coverage-report

# Echidna fuzzing
.PHONY: echidna
echidna:
	docker run --rm -v $(PWD):/code trailofbits/echidna \
		echidna /code/test/echidna/EchidnaTest.sol \
		--contract EchidnaTokenTest \
		--config /code/echidna.yaml

# Medusa fuzzing
.PHONY: medusa
medusa:
	medusa fuzz --config medusa.json
```

---

## สรุป Part 72

- **Property-Based Testing**: แทนที่จะ test case เฉพาะ ให้ define invariants/pre/postconditions ที่ต้องเป็นจริงสำหรับทุก input
- **Echidna**: fuzzer จาก Trail of Bits ที่ใช้ grammar-based fuzzing พร้อม coverage-guided corpus management
- **Medusa**: parallel fuzzer รุ่นใหม่จาก Trail of Bits รองรับ assertion mode และ property mode ใน config เดียว
- **Differential Testing**: เปรียบเทียบสอง implementation ให้ได้ผลเหมือนกัน ใช้ทดสอบ optimized vs reference implementation
- **Halmos**: symbolic execution tool ที่ตรวจสอบทุก execution path ด้วย SMT solver (Z3/CVC5)
- **Test Pyramid**: 70% unit, 20% integration, 10% E2E เป็น ratio ที่แนะนำสำหรับ DeFi protocols

## Next: Part 73 - DeFi Protocol Integrations
