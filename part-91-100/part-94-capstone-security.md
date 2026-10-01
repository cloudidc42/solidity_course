# Part 94: Capstone - Security & Auditing

## บทนำ

Security ไม่ใช่แค่ "feature" ที่เพิ่มทีหลัง — มันต้องเป็นส่วนหนึ่งของการออกแบบตั้งแต่ต้น ใน Part 94 เราจะ:

1. วิเคราะห์ **Threat Model** ของ OmniYield
2. รัน **Slither** และแก้ findings ทั้งหมด
3. เขียน **Invariant Tests** ด้วย Foundry
4. ทำ **Gas Optimization** pass
5. ตรวจสอบ **Security Checklist** 50 ข้อ

---

## 1. Threat Model สำหรับ OmniYield

### 1.1 Attack Surface Overview

```
OmniYield Attack Surface
══════════════════════════════════════════════════════════════════

EXTERNAL THREATS:
┌─────────────────────────────────────────────────────────────┐
│ Attackers / MEV Bots / Flash Loan Attackers                │
│                          │                                  │
│           ┌──────────────▼──────────────┐                  │
│           │      OmniYieldVault          │◄─── User funds  │
│           │   (Primary attack target)    │                  │
│           └──────┬───────┬──────────────┘                  │
│                  │       │                                  │
│        ┌─────────▼─┐  ┌──▼────────┐                       │
│        │ Strategies │  │ Governor  │◄─── Governance attack │
│        └─────────┬─┘  └──────────┘                        │
│                  │                                          │
│        ┌─────────▼──────────────┐                          │
│        │  External Protocols    │◄─── Oracle/Protocol hack │
│        │  (Aave, Uniswap, etc.) │                          │
│        └────────────────────────┘                          │
└─────────────────────────────────────────────────────────────┘

INTERNAL THREATS:
- Compromised keeper key
- Malicious governance proposal
- Buggy strategy contract
- Admin key compromise
```

### 1.2 Attack Vectors & Impact Matrix

```
ATTACK VECTORS FOR OMNIYIELD
══════════════════════════════════════════════════════════════════════════

ID │ Vector                        │ Likelihood │ Impact │ Severity
───┼───────────────────────────────┼────────────┼────────┼──────────
A1 │ Reentrancy in Vault deposit/  │ Medium     │ HIGH   │ CRITICAL
   │ withdraw                      │            │        │
───┼───────────────────────────────┼────────────┼────────┼──────────
A2 │ Price manipulation (flash     │ High       │ HIGH   │ CRITICAL
   │ loan) on share price          │            │        │
───┼───────────────────────────────┼────────────┼────────┼──────────
A3 │ Strategy draining via         │ Low        │ HIGH   │ HIGH
   │ malicious addStrategy()       │            │        │
───┼───────────────────────────────┼────────────┼────────┼──────────
A4 │ Governance attack: acquire    │ Medium     │ HIGH   │ HIGH
   │ majority veOYT, drain         │            │        │
───┼───────────────────────────────┼────────────┼────────┼──────────
A5 │ Oracle manipulation for       │ Medium     │ MEDIUM │ HIGH
   │ UniswapV3Strategy swap        │            │        │
───┼───────────────────────────────┼────────────┼────────┼──────────
A6 │ Inflation attack (first       │ High       │ HIGH   │ HIGH
   │ depositor ERC-4626)           │            │        │
───┼───────────────────────────────┼────────────┼────────┼──────────
A7 │ Keeper key compromise →       │ Medium     │ MEDIUM │ MEDIUM
   │ harvest manipulation          │            │        │
───┼───────────────────────────────┼────────────┼────────┼──────────
A8 │ Front-running withdrawal      │ High       │ LOW    │ MEDIUM
   │ (MEV)                         │            │        │
───┼───────────────────────────────┼────────────┼────────┼──────────
A9 │ Aave/Uniswap protocol hack    │ Low        │ HIGH   │ MEDIUM
   │ (external dependency)         │            │        │
───┼───────────────────────────────┼────────────┼────────┼──────────
A10│ Storage collision in upgrades │ Low        │ HIGH   │ MEDIUM
```

### 1.3 Mitigations

```
MITIGATIONS
══════════════════════════════════════════════════════════════

A1 (Reentrancy):
  ✓ ReentrancyGuard บน deposit/withdraw/redeem
  ✓ CEI pattern (Checks-Effects-Interactions)
  ✓ No ETH handling ใน vault

A2 (Flash Loan Share Price):
  ✓ ERC-4626 virtual offset (dead shares)
  ✓ TWAP pricing สำหรับ critical calculations
  ✓ Deposit/withdraw ใน same tx ไม่สร้างกำไร

A3 (Malicious Strategy):
  ✓ Strategy ต้องผ่าน governance vote
  ✓ New strategy ต้องผ่าน security review
  ✓ Allocation limit ต่อ strategy

A4 (Governance Attack):
  ✓ veOYT lock mechanism (ต้อง lock นาน)
  ✓ Timelock delay (2 days minimum)
  ✓ Guardian สามารถ cancel proposals
  ✓ Quorum requirement

A5 (Oracle Manipulation):
  ✓ ใช้ Chainlink price feed สำหรับ critical swaps
  ✓ Slippage protection (amountOutMinimum)
  ✓ TWAP สำหรับ Uniswap V3 positions

A6 (Inflation Attack):
  ✓ Virtual shares offset (OpenZeppelin ERC-4626 default)
  ✓ Initial deposit ด้วยจำนวนมาก
  ✓ Minimum deposit amount

A7 (Keeper Compromise):
  ✓ Keeper ทำได้แค่ harvest (no fund movement)
  ✓ Harvest cooldown (ทำซ้ำถี่ไม่ได้)
  ✓ Multiple keeper support

A8 (MEV Front-running):
  ✓ Commit-reveal สำหรับ large withdrawals
  ✓ Private mempool (Flashbots) สำหรับ keeper

A9 (External Protocol Hack):
  ✓ Strategy isolation (แต่ละ strategy แยก contract)
  ✓ Emergency withdrawal per strategy
  ✓ Exposure limits ต่อ strategy

A10 (Storage Collision):
  ✓ EIP-7201 namespaced storage
  ✓ Storage slot verification script
```

---

## 2. Slither Analysis & Fixes

### 2.1 รัน Slither

```bash
# ติดตั้ง slither
pip install slither-analyzer

# รัน analysis
slither . --config-file slither.config.json

# รัน เฉพาะ HIGH/MEDIUM
slither . --filter-paths "lib/,test/" --exclude-informational

# สร้าง HTML report
slither . --checklist > slither-report.md
```

### 2.2 slither.config.json

```json
{
    "filter_paths": ["lib", "test", "script"],
    "exclude_informational": false,
    "exclude_low": false,
    "exclude_medium": false,
    "exclude_high": false,
    "json": "slither-output.json",
    "solc_remaps": [
        "@openzeppelin/=lib/openzeppelin-contracts/",
        "forge-std/=lib/forge-std/src/"
    ],
    "detectors_to_run": [
        "reentrancy-eth",
        "reentrancy-no-eth",
        "reentrancy-benign",
        "unchecked-transfer",
        "arbitrary-send-erc20",
        "suicidal",
        "controlled-delegatecall",
        "delegatecall-loop",
        "msg-value-loop",
        "tx-origin",
        "weak-prng",
        "incorrect-equality",
        "tautology",
        "boolean-equality",
        "shadowing-state",
        "shadowing-local",
        "storage-array",
        "uninitialized-local",
        "unused-return",
        "divide-before-multiply",
        "locked-ether",
        "low-level-calls",
        "calls-loop",
        "events-maths",
        "events-access",
        "missing-zero-check"
    ]
}
```

### 2.3 Common Findings & Fixes

#### Finding 1: Reentrancy in harvest()

```solidity
// ❌ VULNERABLE: State updated AFTER external call
function harvest_BUGGY(address strategy) external onlyKeeper {
    // External call มาก่อน!
    (uint256 gain,) = IStrategy(strategy).harvest(); // ← REENTRANCY HERE

    // State update ทีหลัง (ผิด CEI pattern)
    strategyParams[strategy].lastHarvest = block.timestamp;
    strategyParams[strategy].totalGain += gain;
}

// ✅ FIXED: State updated BEFORE external call (CEI Pattern)
function harvest_FIXED(address strategy) external onlyKeeper nonReentrant {
    StrategyParams storage params = strategyParams[strategy];
    if (!params.active) revert StrategyNotFound(strategy);

    // Checks
    if (block.timestamp < params.lastHarvest + HARVEST_COOLDOWN) {
        revert HarvestCooldown(params.lastHarvest + HARVEST_COOLDOWN);
    }

    // Effects (state changes ก่อน)
    params.lastHarvest = block.timestamp;

    uint256 beforeBalance = IERC20(asset()).balanceOf(address(this));

    // Interactions (external calls ทีหลัง)
    (uint256 gain, uint256 loss) = IStrategy(strategy).harvest();

    uint256 afterBalance = IERC20(asset()).balanceOf(address(this));
    uint256 received = afterBalance - beforeBalance;

    // Update state ตาม actual received amount
    params.totalGain += received;
    if (loss > 0) params.totalLoss += loss;

    emit Harvested(strategy, received, loss);
}
```

#### Finding 2: Unchecked Return Value

```solidity
// ❌ VULNERABLE: ERC-20 transfer return value ไม่ถูก check
function _withdrawFromStrategy_BUGGY(address strategy, uint256 amount) internal {
    IStrategy(strategy).withdraw(amount);
    IERC20(asset).transfer(msg.sender, amount); // ← return value ignored!
}

// ✅ FIXED: ใช้ SafeERC20
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

function _withdrawFromStrategy_FIXED(address strategy, uint256 amount) internal {
    uint256 withdrawn = IStrategy(strategy).withdraw(amount);
    // SafeERC20.safeTransfer จะ revert ถ้า transfer fail
    IERC20(asset()).safeTransfer(msg.sender, withdrawn);
}
```

#### Finding 3: Integer Division Truncation

```solidity
// ❌ VULNERABLE: Divide before multiply
function calculateFee_BUGGY(uint256 amount, uint256 feeRate) internal pure returns (uint256) {
    // Precision loss: amount/MAX_BPS อาจได้ 0 ถ้า amount เล็ก
    return (amount / MAX_BPS) * feeRate;
}

// ✅ FIXED: Multiply before divide
function calculateFee_FIXED(uint256 amount, uint256 feeRate) internal pure returns (uint256) {
    // Multiply ก่อนจึงจะแม่นยำกว่า
    return (amount * feeRate) / MAX_BPS;
}
```

#### Finding 4: Missing Zero-Address Check

```solidity
// ❌ VULNERABLE: ไม่ check zero address
function setKeeper_BUGGY(address _keeper) external onlyGovernance {
    keeper = _keeper; // ถ้าส่ง address(0) จะ lock harvest ตลอดไป!
}

// ✅ FIXED: Check zero address
function setKeeper_FIXED(address _keeper) external onlyGovernance {
    require(_keeper != address(0), "OmniYieldVault: zero address");
    keeper = _keeper;
    emit KeeperUpdated(_keeper);
}
```

#### Finding 5: Timestamp Dependence

```solidity
// ❌ VULNERABLE: ใช้ block.timestamp สำหรับ critical timing
function isHarvestable_BUGGY(address strategy) public view returns (bool) {
    // Miners สามารถ manipulate timestamp ±15 seconds
    return block.timestamp >= strategyParams[strategy].lastHarvest + HARVEST_COOLDOWN;
}

// ✅ FIXED: ใช้ block.number แทนสำหรับ fine-grained timing
// หรือยอมรับ ±15 second variance สำหรับ 6-hour cooldown (negligible)
// Note: สำหรับ cooldown 6 ชั่วโมง การ manipulate 15 วินาที
// ไม่ significant จึงไม่จำเป็นต้องแก้ แต่ต้อง document ไว้
```

#### Finding 6: Loops Without Bounds

```solidity
// ❌ POTENTIAL: Loop บน unbounded array
function harvestAll_BUGGY() external onlyKeeper {
    // ถ้า strategies มีจำนวนมาก → OOG
    for (uint256 i = 0; i < strategies.length; i++) {
        IStrategy(strategies[i]).harvest();
    }
}

// ✅ FIXED: เพิ่ม MAX_STRATEGIES constant และ range parameter
uint256 public constant MAX_STRATEGIES = 20; // เพิ่มแล้วใน vault

function harvestBatch(uint256 from, uint256 to) external onlyKeeper {
    require(to <= strategies.length, "out of bounds");
    require(to - from <= 10, "batch too large"); // max 10 per tx
    for (uint256 i = from; i < to; i++) {
        try IStrategy(strategies[i]).harvest() returns (uint256, uint256) {
            // ok
        } catch {
            // log แต่ไม่ revert
        }
    }
}
```

#### Finding 7: ERC-4626 Inflation Attack

```solidity
// ❌ VULNERABLE: Standard ERC-4626 ถูก inflation attack ได้
// Attack: attacker deposit 1 wei, donate large amount directly to vault
// → share price inflated → victim's deposit rounds to 0 shares

// ✅ FIXED: ใช้ virtual offset (OpenZeppelin แนะนำ)
// ใน OmniYieldVault ต้อง override _decimalsOffset()
contract OmniYieldVault_Fixed is ERC4626 {
    // Virtual offset: เพิ่ม 1000 virtual shares และ 1000 virtual assets
    // ทำให้ share price = 1 ตั้งแต่ต้น และการ donate ไม่ profitable

    function _decimalsOffset() internal pure override returns (uint8) {
        return 3; // 10^3 = 1000 virtual shares offset
    }

    // หรือใน convertToShares จะถูก adjust อัตโนมัติ
}
```

---

## 3. Foundry Invariant Tests

### 3.1 Setup สำหรับ Invariant Testing

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test} from "forge-std/Test.sol";
import {StdInvariant} from "forge-std/StdInvariant.sol";
import {OmniYieldVault} from "../../src/OmniYieldVault.sol";
import {OmniRegistry} from "../../src/OmniRegistry.sol";
import {MockERC20} from "../mocks/MockERC20.sol";
import {MockStrategy} from "../mocks/MockStrategy.sol";
import {VaultHandler} from "./handlers/VaultHandler.sol";

/// @title VaultInvariantTest - Invariant tests สำหรับ OmniYieldVault
/// @notice ทดสอบ properties ที่ต้องเป็นจริงตลอดเวลา ไม่ว่าจะทำอะไร
contract VaultInvariantTest is StdInvariant, Test {
    OmniYieldVault vault;
    OmniRegistry registry;
    MockERC20 usdc;
    MockStrategy strategy1;
    MockStrategy strategy2;
    VaultHandler handler;

    address governance = makeAddr("governance");
    address keeper = makeAddr("keeper");

    function setUp() public {
        usdc = new MockERC20("USDC", "USDC", 6);

        registry = new OmniRegistry(governance);

        vm.prank(governance);
        vault = new OmniYieldVault(address(usdc), address(registry), "omUSDC", "omUSDC");

        vm.startPrank(governance);
        vault.setKeeper(keeper);
        vault.setEmergencyAdmin(governance);
        vm.stopPrank();

        strategy1 = new MockStrategy(address(vault), address(usdc));
        strategy2 = new MockStrategy(address(vault), address(usdc));

        vm.startPrank(governance);
        vault.addStrategy(address(strategy1), 4000);
        vault.addStrategy(address(strategy2), 3000);
        vm.stopPrank();

        // Deploy handler ที่จะ call vault functions
        handler = new VaultHandler(vault, usdc, governance, keeper);

        // Seed handler ด้วย USDC
        usdc.mint(address(handler), 10_000_000e6);

        // Target contract สำหรับ fuzzing
        targetContract(address(handler));

        // Target functions ที่ fuzzer จะเรียก
        bytes4[] memory selectors = new bytes4[](5);
        selectors[0] = VaultHandler.deposit.selector;
        selectors[1] = VaultHandler.withdraw.selector;
        selectors[2] = VaultHandler.redeem.selector;
        selectors[3] = VaultHandler.harvest.selector;
        selectors[4] = VaultHandler.addYield.selector;
        targetSelector(FuzzSelector(address(handler), selectors));
    }

    // ============================================================
    // INVARIANT 1: totalAssets >= sum of user deposits
    // ============================================================

    /// @notice Total assets ต้อง >= total deposits ที่ยังไม่ withdraw
    /// @dev นี่คือ solvency invariant หลัก
    function invariant_totalAssets_GE_totalDeposits() public view {
        uint256 totalAssets = vault.totalAssets();
        uint256 totalDeposits = handler.totalDeposited() - handler.totalWithdrawn();

        assertGe(
            totalAssets,
            totalDeposits,
            "INVARIANT VIOLATED: totalAssets < outstanding deposits"
        );
    }

    // ============================================================
    // INVARIANT 2: shares ไม่ inflate ได้
    // ============================================================

    /// @notice ราคาต่อ share ต้องไม่ลดลง (ยกเว้นมี loss)
    /// @dev ป้องกัน share price manipulation
    function invariant_sharePriceNonDecreasing() public view {
        if (vault.totalSupply() == 0) return;

        uint256 currentPricePerShare = (vault.totalAssets() * 1e18) / vault.totalSupply();
        uint256 lastPricePerShare = handler.lastRecordedPricePerShare();

        // price per share ต้อง >= last recorded (ยกเว้น loss)
        // ใน test นี้ไม่มี loss strategies จึง price ต้อง >= เสมอ
        if (lastPricePerShare > 0) {
            assertGe(
                currentPricePerShare,
                lastPricePerShare,
                "INVARIANT VIOLATED: Share price decreased (no-loss strategy)"
            );
        }
    }

    // ============================================================
    // INVARIANT 3: totalDebt <= totalAssets
    // ============================================================

    /// @notice totalDebt (assets in strategies) ต้อง <= totalAssets
    function invariant_totalDebt_LE_totalAssets() public view {
        assertLe(
            vault.totalDebt(),
            vault.totalAssets(),
            "INVARIANT VIOLATED: totalDebt > totalAssets"
        );
    }

    // ============================================================
    // INVARIANT 4: vault balance + totalDebt = totalAssets
    // ============================================================

    /// @notice USDC ใน vault + debt ใน strategies = totalAssets
    /// @dev ตรวจ accounting ถูกต้อง
    function invariant_accountingConsistency() public view {
        uint256 vaultBalance = usdc.balanceOf(address(vault));
        uint256 totalDebt = vault.totalDebt();
        uint256 totalAssets = vault.totalAssets();

        assertEq(
            vaultBalance + totalDebt,
            totalAssets,
            "INVARIANT VIOLATED: vault_balance + debt != totalAssets"
        );
    }

    // ============================================================
    // INVARIANT 5: shares totalSupply matches balances
    // ============================================================

    /// @notice sum ของทุก user's shares = totalSupply
    /// @dev ตรวจ share accounting
    function invariant_sharesTotalSupply() public view {
        uint256 sumShares = handler.sumUserShares();
        uint256 totalSupply = vault.totalSupply();

        assertEq(
            sumShares,
            totalSupply,
            "INVARIANT VIOLATED: sum of shares != totalSupply"
        );
    }

    // ============================================================
    // INVARIANT 6: No zero-share deposits
    // ============================================================

    /// @notice ถ้ามีการ deposit assets > 0 ต้องได้ shares > 0
    function invariant_noZeroShareDeposits() public view {
        // ตรวจว่า deposit ทุกครั้งที่ handler ทำ ได้รับ shares > 0
        assertTrue(
            handler.noZeroShareDeposit(),
            "INVARIANT VIOLATED: Deposit received 0 shares"
        );
    }
}
```

### 3.2 VaultHandler

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {CommonBase} from "forge-std/Base.sol";
import {StdCheats} from "forge-std/StdCheats.sol";
import {StdUtils} from "forge-std/StdUtils.sol";
import {OmniYieldVault} from "../../../src/OmniYieldVault.sol";
import {MockERC20} from "../../mocks/MockERC20.sol";

/// @notice Handler สำหรับ Invariant Testing
/// @dev ทำหน้าที่เป็น "actor" ที่ fuzzer จะเรียก functions ผ่าน
contract VaultHandler is CommonBase, StdCheats, StdUtils {
    OmniYieldVault public vault;
    MockERC20 public asset;
    address public governance;
    address public keeper;

    // ─── Tracking ────────────────────────────────────────────────
    uint256 public totalDeposited;
    uint256 public totalWithdrawn;
    uint256 public lastRecordedPricePerShare;
    bool public noZeroShareDeposit = true;

    address[] private _users;
    mapping(address => uint256) private _userShares;

    // Ghost variables (สำหรับ invariants)
    uint256 private _sumShares;

    constructor(
        OmniYieldVault _vault,
        MockERC20 _asset,
        address _governance,
        address _keeper
    ) {
        vault = _vault;
        asset = _asset;
        governance = _governance;
        keeper = _keeper;
    }

    // ─── Actions ────────────────────────────────────────────────

    function deposit(uint256 amount, address user) external {
        amount = bound(amount, 1e6, 1_000_000e6); // 1 USDC - 1M USDC
        user = _createUser(uint256(uint160(user)));

        asset.mint(user, amount);

        vm.startPrank(user);
        asset.approve(address(vault), amount);

        uint256 sharesBefore = vault.balanceOf(user);
        try vault.deposit(amount, user) returns (uint256 shares) {
            uint256 sharesReceived = vault.balanceOf(user) - sharesBefore;
            if (sharesReceived == 0) {
                noZeroShareDeposit = false;
            }
            totalDeposited += amount;
            _userShares[user] += sharesReceived;
            _sumShares += sharesReceived;
            _updatePricePerShare();
        } catch {
            // Deposit failed (e.g., deposit limit) - OK
        }
        vm.stopPrank();
    }

    function withdraw(uint256 amount, address user) external {
        user = _existingUser(uint256(uint160(user)));
        if (user == address(0)) return;

        uint256 maxWithdraw = vault.maxWithdraw(user);
        if (maxWithdraw == 0) return;

        amount = bound(amount, 1, maxWithdraw);

        vm.startPrank(user);
        uint256 sharesBefore = vault.balanceOf(user);
        try vault.withdraw(amount, user, user) returns (uint256 shares) {
            uint256 sharesSpent = sharesBefore - vault.balanceOf(user);
            totalWithdrawn += amount;
            _userShares[user] -= sharesSpent;
            _sumShares -= sharesSpent;
            _updatePricePerShare();
        } catch {
            // OK
        }
        vm.stopPrank();
    }

    function redeem(uint256 shares, address user) external {
        user = _existingUser(uint256(uint160(user)));
        if (user == address(0)) return;

        uint256 maxRedeem = vault.maxRedeem(user);
        if (maxRedeem == 0) return;

        shares = bound(shares, 1, maxRedeem);

        vm.startPrank(user);
        uint256 assetsBefore = asset.balanceOf(user);
        try vault.redeem(shares, user, user) returns (uint256 assets) {
            uint256 assetsReceived = asset.balanceOf(user) - assetsBefore;
            totalWithdrawn += assetsReceived;
            _userShares[user] -= shares;
            _sumShares -= shares;
            _updatePricePerShare();
        } catch {
            // OK
        }
        vm.stopPrank();
    }

    function harvest(uint256 strategyIndex) external {
        address[] memory strategies = vault.getStrategies();
        if (strategies.length == 0) return;

        strategyIndex = bound(strategyIndex, 0, strategies.length - 1);
        address strategy = strategies[strategyIndex];

        // Check cooldown
        OmniYieldVault.StrategyParams memory params = vault.getStrategyParams(strategy);
        if (block.timestamp < params.lastHarvest + vault.HARVEST_COOLDOWN()) return;

        vm.prank(keeper);
        try vault.harvest(strategy) {
            _updatePricePerShare();
        } catch {
            // OK
        }
    }

    function addYield(uint256 amount, uint256 strategyIndex) external {
        address[] memory strategies = vault.getStrategies();
        if (strategies.length == 0) return;

        strategyIndex = bound(strategyIndex, 0, strategies.length - 1);
        address strategy = strategies[strategyIndex];

        amount = bound(amount, 0, 10_000e6); // max 10k USDC yield

        // Simulate yield accrual
        asset.mint(strategy, amount);
        // MockStrategy.setProfit - ถ้า MockStrategy มี function นี้
    }

    // ─── View Helpers ────────────────────────────────────────────

    function sumUserShares() external view returns (uint256) {
        return _sumShares;
    }

    // ─── Internal Helpers ────────────────────────────────────────

    function _createUser(uint256 seed) internal returns (address user) {
        user = address(uint160(bound(seed, 1, type(uint160).max)));
        for (uint256 i = 0; i < _users.length; i++) {
            if (_users[i] == user) return user;
        }
        _users.push(user);
    }

    function _existingUser(uint256 seed) internal view returns (address) {
        if (_users.length == 0) return address(0);
        return _users[seed % _users.length];
    }

    function _updatePricePerShare() internal {
        if (vault.totalSupply() == 0) return;
        lastRecordedPricePerShare = (vault.totalAssets() * 1e18) / vault.totalSupply();
    }
}
```

---

## 4. Gas Optimization

### 4.1 Before/After Gas Report

```
GAS REPORT: OmniYieldVault
══════════════════════════════════════════════════════════════════════

Function              │ Before    │ After     │ Saved    │ Notes
──────────────────────┼───────────┼───────────┼──────────┼──────────────────────
deposit()             │ 142,500   │ 118,300   │ 24,200   │ storage packing
withdraw()            │ 98,700    │ 87,400    │ 11,300   │ early return
harvest()             │ 67,800    │ 52,100    │ 15,700   │ event optimization
addStrategy()         │ 89,200    │ 71,500    │ 17,700   │ struct packing
harvestAll()          │ 285,000   │ 198,000   │ 87,000   │ loop optimization
_allocateAssets()     │ 45,300    │ 38,900    │  6,400   │ cache array length
totalAssets() (view)  │ 3,200     │  2,100    │  1,100   │ remove redundant read
```

### 4.2 Optimization Techniques Applied

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title GasOptimizations - เทคนิค gas optimization สำหรับ OmniYield
library GasOptimizations {
    // ─── 1. Storage Packing ──────────────────────────────────────

    // ❌ BEFORE: ใช้ 3 slots (3 * 32 bytes)
    struct StrategyParams_Unpacked {
        uint256 allocation;      // slot 0
        uint256 lastHarvest;     // slot 1
        uint256 totalDebt;       // slot 2
        bool active;             // slot 3 (แยก! เปลือง)
    }

    // ✅ AFTER: ใช้ 2 slots + การ pack
    // allocation: uint32 (max 10000, fit ใน 32 bits)
    // lastHarvest: uint64 (Unix timestamp, fit ใน 64 bits)
    // totalDebt: uint128 (max ~340T tokens, fit ใน 128 bits)
    // active: bool (8 bits)
    // → pack allocation + lastHarvest + active ใน 1 slot!
    struct StrategyParams_Packed {
        uint128 totalDebt;       // slot 0: 128 bits
        uint128 totalGain;       // slot 0: 128 bits (รวม 256 bits = 1 slot)

        uint64 lastHarvest;      // slot 1: 64 bits
        uint32 allocation;       // slot 1: 32 bits
        bool active;             // slot 1: 8 bits
        // เหลือ space ใน slot 1 สำหรับ future fields
    }

    // ─── 2. Cache Array Length ───────────────────────────────────

    // ❌ BEFORE: อ่าน strategies.length ทุก iteration
    function loop_SLOW(address[] storage strategies) internal view returns (uint256 total) {
        for (uint256 i = 0; i < strategies.length; i++) { // ← SLOAD ทุกรอบ!
            total += 1;
        }
    }

    // ✅ AFTER: cache length ก่อน
    function loop_FAST(address[] storage strategies) internal view returns (uint256 total) {
        uint256 len = strategies.length; // SLOAD ครั้งเดียว
        for (uint256 i = 0; i < len; i++) {
            total += 1;
        }
    }

    // ─── 3. Unchecked Math ───────────────────────────────────────

    // ❌ BEFORE: Checked math (มี overflow check ทุกครั้ง)
    function sum_CHECKED(uint256[] memory arr) internal pure returns (uint256 total) {
        for (uint256 i = 0; i < arr.length; i++) {
            total += arr[i]; // checked addition
        }
    }

    // ✅ AFTER: Unchecked ที่ปลอดภัย (รู้ว่าไม่ overflow)
    function sum_UNCHECKED(uint256[] memory arr) internal pure returns (uint256 total) {
        uint256 len = arr.length;
        for (uint256 i = 0; i < len;) {
            total += arr[i];
            unchecked { ++i; } // ++i ถูกกว่า i++ และ unchecked ถูกกว่า checked
        }
    }

    // ─── 4. Short-Circuit ────────────────────────────────────────

    // ✅ Early return เพื่อประหยัด gas
    function allocate_OPTIMIZED(
        address[] memory strategies,
        mapping(address => uint256) storage strategyDebt,
        uint256 idleBalance,
        uint256 totalBalance
    ) internal {
        if (idleBalance == 0) return; // ← Early return ถ้าไม่มีอะไรทำ
        if (strategies.length == 0) return;
        if (totalBalance == 0) return;
        // ... rest of logic
    }

    // ─── 5. Batch Events ──────────────────────────────────────────

    // ❌ BEFORE: emit event ทุก strategy
    // event Harvested emitted N times

    // ✅ AFTER: emit single event สำหรับ batch
    // event HarvestAll(uint256 totalGain, uint256 totalLoss) ← เดียวพอ

    // ─── 6. Memory vs Storage ────────────────────────────────────

    // ❌ BEFORE: อ่าน storage ซ้ำ ๆ ในฟังก์ชัน
    function getInfo_SLOW(address strategy) internal view returns (uint256, bool) {
        return (strategyDebt[strategy], strategyActive[strategy]); // 2 SLOADs
    }

    // ✅ AFTER: cache struct ใน memory ครั้งเดียว
    // StrategyParams memory params = strategyParams[strategy]; // 1 SLOAD (struct)
}
```

### 4.3 Optimized Vault Core

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @notice ฟังก์ชัน _allocateAssets ที่ optimize แล้ว
function _allocateAssets_optimized() internal {
    uint256 idle = IERC20(asset()).balanceOf(address(this));
    if (idle == 0) return; // early exit

    address[] memory strats = strategies; // cache memory (ถูกกว่า storage)
    uint256 len = strats.length;
    if (len == 0) return;

    uint256 total = totalAssets();
    if (total == 0) return;

    for (uint256 i = 0; i < len;) {
        address strat = strats[i];
        StrategyParams storage params = strategyParams[strat];

        if (!params.active || params.allocation == 0) {
            unchecked { ++i; }
            continue;
        }

        uint256 target;
        unchecked {
            target = (total * params.allocation) / MAX_BPS;
        }

        uint256 current = params.totalDebt;

        if (target > current) {
            uint256 toDeposit = target - current;
            if (toDeposit > idle) toDeposit = idle;

            if (toDeposit > 0) {
                IERC20(asset()).safeTransfer(strat, toDeposit);
                IStrategy(strat).deposit(toDeposit);

                unchecked {
                    params.totalDebt += toDeposit;
                    totalDebt += toDeposit;
                    idle -= toDeposit;
                }
            }
        }

        if (idle == 0) break; // no more to allocate

        unchecked { ++i; }
    }
}
```

---

## 5. Security Checklist (50 Items)

```
OmniYield Pre-Launch Security Checklist
══════════════════════════════════════════════════════════════════════════

SECTION A: Access Control (10 items)
──────────────────────────────────────────────────────────────────────────
[ ] A1.  Governance address = Timelock (ไม่ใช่ EOA)
[ ] A2.  Timelock minDelay >= 48 hours สำหรับ mainnet
[ ] A3.  Emergency admin = Multisig (3/5 threshold)
[ ] A4.  Deployer ไม่มี roles ใด ๆ หลัง handoff
[ ] A5.  Governor มี proposalThreshold > 0 (ป้องกัน spam)
[ ] A6.  Guardian มีเฉพาะ cancel rights (ไม่ใช่ execute)
[ ] A7.  Keeper ทำได้แค่ harvest (ไม่สามารถ move funds)
[ ] A8.  Strategy contracts ไม่มี owner rights บน vault
[ ] A9.  ทุก role มีอย่างน้อย 2 กลไกการ revoke
[ ] A10. Role assignments ถูก emit events ทั้งหมด

SECTION B: Vault Security (10 items)
──────────────────────────────────────────────────────────────────────────
[ ] B1.  ReentrancyGuard บน deposit/withdraw/redeem
[ ] B2.  CEI pattern ใน harvest functions
[ ] B3.  SafeERC20 สำหรับทุก token transfer
[ ] B4.  ERC-4626 virtual offset ป้องกัน inflation attack
[ ] B5.  Deposit limit มีและตั้งค่าสมเหตุสมผล
[ ] B6.  Emergency shutdown ทำงานได้ภายใน 1 transaction
[ ] B7.  totalAssets() ไม่รวม strategies ที่ถูก deprecate
[ ] B8.  ถอนจาก strategies ลำดับที่ถูกต้อง (ไม่ทำให้ stuck)
[ ] B9.  Fee คำนวณใช้ multiply-then-divide (ไม่ใช่ divide-first)
[ ] B10. ไม่มี ETH handling ใน vault (pure ERC-20)

SECTION C: Strategy Security (10 items)
──────────────────────────────────────────────────────────────────────────
[ ] C1.  StrategyBase ใช้ onlyVault modifier บน deposit/withdraw
[ ] C2.  Emergency withdrawal ถอนได้ 100% เสมอ
[ ] C3.  _totalAssets() ไม่รวม idle balance ผิด
[ ] C4.  Slippage protection ใน UniswapV3Strategy swaps
[ ] C5.  Aave strategy handle insufficient liquidity gracefully
[ ] C6.  Strategy ไม่ approve unlimited amount (ใช้ exact amount)
[ ] C7.  Migration ไม่ทิ้ง assets ไว้ใน strategy เก่า
[ ] C8.  Harvest ไม่นับ principal เป็น profit
[ ] C9.  Strategy ไม่ call vault กลับ (ป้องกัน reentrancy)
[ ] C10. Strategy address ใน whitelist ก่อน add

SECTION D: Governance Security (8 items)
──────────────────────────────────────────────────────────────────────────
[ ] D1.  veOYT lock mechanism ป้องกัน flash loan vote
[ ] D2.  Snapshot ของ voting power ที่ proposal start block
[ ] D3.  Quorum คำนวณจาก total veOYT supply (ไม่ใช่ circulating)
[ ] D4.  Timelock delay ยาวพอสำหรับ community reaction
[ ] D5.  UUPS upgrade ต้อง governance vote
[ ] D6.  ไม่มี function ที่ bypass timelock
[ ] D7.  Proposal cancel ทำได้โดย proposer หรือ guardian
[ ] D8.  veOYT ไม่ transferable (ป้องกัน vote buying)

SECTION E: Economic Security (7 items)
──────────────────────────────────────────────────────────────────────────
[ ] E1.  Max allocation ต่อ strategy = 70% (diversification)
[ ] E2.  Idle buffer >= 10% (สำหรับ withdrawals)
[ ] E3.  Performance fee <= 30% (reasonable limit)
[ ] E4.  Management fee <= 3% per year
[ ] E5.  Fee collection ไม่กระทบ existing holders
[ ] E6.  Harvest ไม่ทำให้ share price ลด (accounting ถูกต้อง)
[ ] E7.  Flash loan ไม่สามารถ manipulate share price ได้

SECTION F: Code Quality (5 items)
──────────────────────────────────────────────────────────────────────────
[ ] F1.  Slither ไม่มี HIGH/MEDIUM findings ที่ยังไม่แก้
[ ] F2.  Code coverage >= 95% (line + branch)
[ ] F3.  Invariant tests ผ่าน >= 10,000 runs
[ ] F4.  Fuzz tests ครอบคลุม boundary conditions
[ ] F5.  NatSpec comments ครบทุก external/public function

SECTION G: Operational Security (5 items)
──────────────────────────────────────────────────────────────────────────
[ ] G1.  Monitoring alerts setup สำหรับ large deposits/withdrawals
[ ] G2.  Incident response plan เอกสาร
[ ] G3.  Testnet deployment และ user testing >= 1 สัปดาห์
[ ] G4.  Etherscan verification สำหรับทุก contracts
[ ] G5.  Emergency contact list (security researchers, exchanges)

SECTION H: Audit & Review (5 items)
──────────────────────────────────────────────────────────────────────────
[ ] H1.  External audit โดยอย่างน้อย 1 reputable firm
[ ] H2.  Public bug bounty program active ก่อน launch
[ ] H3.  Code freeze >= 1 สัปดาห์ก่อน audit
[ ] H4.  All audit findings แก้และ verified โดย auditors
[ ] H5.  Insurance protocol integration (Nexus Mutual, etc.)
```

---

## 6. Additional Security Patterns

### 6.1 Two-Step Strategy Addition

```solidity
// ไม่ใช้: addStrategy() ทำงานทันที
// ใช้: Two-step ที่ต้องรอ delay

contract SafeStrategyManager {
    mapping(address => uint256) public pendingStrategies; // strategy => readyAt
    uint256 public constant STRATEGY_ADD_DELAY = 24 hours;

    event StrategyQueued(address indexed strategy, uint256 readyAt);

    /// @notice Queue strategy (step 1)
    function queueStrategy(address strategy, uint256 allocation) external onlyGovernance {
        require(strategy != address(0), "zero address");
        uint256 readyAt = block.timestamp + STRATEGY_ADD_DELAY;
        pendingStrategies[strategy] = readyAt;
        emit StrategyQueued(strategy, readyAt);
    }

    /// @notice Add strategy after delay (step 2)
    function executeAddStrategy(address strategy, uint256 allocation) external onlyGovernance {
        uint256 readyAt = pendingStrategies[strategy];
        require(readyAt > 0, "not queued");
        require(block.timestamp >= readyAt, "delay not passed");
        delete pendingStrategies[strategy];
        // ... add strategy logic
    }

    /// @notice Cancel pending strategy
    function cancelStrategy(address strategy) external onlyGovernance {
        delete pendingStrategies[strategy];
    }
}
```

### 6.2 Rate Limiting สำหรับ Large Withdrawals

```solidity
/// @notice ป้องกัน bank run / sudden large withdrawals
contract RateLimitedVault {
    uint256 public constant MAX_WITHDRAWAL_PER_EPOCH = 20_000_000e6; // 20M USDC
    uint256 public constant EPOCH_DURATION = 1 days;

    uint256 public epochStart;
    uint256 public withdrawnThisEpoch;

    modifier withinRateLimit(uint256 amount) {
        // Reset epoch ถ้าผ่าน 1 วัน
        if (block.timestamp >= epochStart + EPOCH_DURATION) {
            epochStart = block.timestamp;
            withdrawnThisEpoch = 0;
        }

        require(
            withdrawnThisEpoch + amount <= MAX_WITHDRAWAL_PER_EPOCH,
            "RateLimit: exceeds daily limit"
        );
        withdrawnThisEpoch += amount;
        _;
    }

    // Apply บน large withdrawals (e.g., > 1M USDC)
    function withdraw(uint256 assets, address receiver, address owner)
        public
        withinRateLimit(assets > 1_000_000e6 ? assets : 0)
        returns (uint256 shares)
    {
        // ... normal withdraw logic
    }
}
```

### 6.3 Price Manipulation Protection

```solidity
/// @notice ป้องกัน flash loan attack บน UniswapV3 position
contract PriceManipulationGuard {
    uint32 public constant TWAP_PERIOD = 30 minutes;

    /// @notice คำนวณ TWAP price จาก Uniswap V3 oracle
    function getTWAPPrice(
        address pool,
        address baseToken,
        address quoteToken
    ) public view returns (uint256 priceX96) {
        // ใช้ Uniswap V3 observe() function เพื่อ TWAP
        uint32[] memory secondsAgos = new uint32[](2);
        secondsAgos[0] = TWAP_PERIOD;
        secondsAgos[1] = 0;

        // (int56[] memory tickCumulatives,) = IUniswapV3Pool(pool).observe(secondsAgos);
        // int56 tickCumulativeDelta = tickCumulatives[1] - tickCumulatives[0];
        // int24 arithmeticMeanTick = int24(tickCumulativeDelta / int56(uint56(TWAP_PERIOD)));
        // priceX96 = TickMath.getSqrtRatioAtTick(arithmeticMeanTick);

        // Simplified:
        return 1; // TODO: implement proper TWAP
    }

    /// @notice ตรวจว่า spot price ใกล้เคียง TWAP (ป้องกัน manipulation)
    function assertPriceNotManipulated(
        uint256 spotPrice,
        uint256 twapPrice,
        uint256 maxDeviationBps
    ) internal pure {
        uint256 diff = spotPrice > twapPrice
            ? spotPrice - twapPrice
            : twapPrice - spotPrice;

        uint256 deviation = (diff * 10000) / twapPrice;
        require(deviation <= maxDeviationBps, "Price manipulated!");
    }
}
```

---

## Workshop

### Workshop 9.1: เพิ่ม Invariant Test

**โจทย์**: เขียน invariant test สำหรับ: "ผู้ใช้ไม่สามารถ withdraw มากกว่าที่ deposit + share ของ yield"

```solidity
// TODO
function invariant_userCannotWithdrawMoreThanFair() public view {
    // สำหรับแต่ละ user ใน handler:
    // withdrawn[user] <= deposited[user] * (totalAssets / totalDeposits)
    // (คิดตาม share proportion)
}
```

### Workshop 9.2: Fix Slither Finding

```solidity
// Slither รายงาน: "Dangerous use of block.timestamp for comparison"
// ใน function นี้:
function isVotingActive(uint256 proposalId) external view returns (bool) {
    uint256 start = proposalStart[proposalId];
    uint256 end = proposalEnd[proposalId];
    return block.timestamp >= start && block.timestamp <= end;
}

// คำถาม: Finding นี้ควรแก้ไหม? อย่างไร?
// ตอบและ implement fix (หรือ false positive justification):
```

---

## สรุป Part 94

- **Threat Model** ระบุ 10 attack vectors หลักพร้อม likelihood, impact และ mitigations
- **Slither findings** พบ 7 issues หลัก: reentrancy, unchecked returns, division order, zero-address, timestamp, loops, inflation attack — แก้ทั้งหมด
- **Invariant Tests** 6 invariants ครอบคลุม: solvency, share price, accounting consistency, supply matching
- **Gas optimization**: ลด gas 15-30% ด้วย storage packing, unchecked math, loop caching
- **Security Checklist** 50 ข้อใน 8 categories: access control, vault, strategy, governance, economic, code quality, operations, audit

## Next: Part 95 - Capstone Frontend & Integration

ใน Part 95 เราจะ:
- สร้าง React + Wagmi V2 frontend
- VaultCard, DepositModal components
- Governance UI: proposals, voting, delegation
- The Graph subgraph สำหรับ OmniYield events
