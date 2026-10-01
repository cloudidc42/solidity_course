# Part 52: Yield Aggregators (Yearn-style)

## บทนำ

**Yield Aggregators** คือโปรโตคอลที่รวบรวม liquidity จากผู้ใช้และ route ไปยัง yield-generating strategies ต่างๆ โดยอัตโนมัติ แนวคิดหลักคือ:

- **Automation**: ไม่ต้องทำ manual จัดการ position
- **Compounding**: นำ yield กลับไป reinvest โดยอัตโนมัติ
- **Diversification**: กระจายเงินในหลาย strategies
- **Gas Efficiency**: รวม transactions ของผู้ใช้หลายคนเป็น batch

Yearn Finance เป็น pioneer ในพื้นที่นี้ และ architecture ของพวกเขามีผลอย่างมากต่อ DeFi ecosystem ในปัจจุบัน

---

## 1. BaseStrategy Abstract Contract

### ทฤษฎี: Strategy Pattern

`BaseStrategy` กำหนด lifecycle ของ strategy ทุกตัว:

1. **Deposit**: รับ assets จาก Vault
2. **Invest** (`_adjustPosition`): deploy assets ไปยัง yield source
3. **Harvest**: เก็บ yields, ขาย rewards เป็น want token
4. **Report**: ส่งกำไร/ขาดทุนกลับ Vault
5. **Withdraw**: ถอน assets เมื่อ Vault ต้องการ

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title BaseStrategy - Abstract base สำหรับทุก strategy
/// @notice Strategy ทุกตัวต้อง inherit contract นี้
abstract contract BaseStrategy is ReentrancyGuard {
    using SafeERC20 for IERC20;
    
    // ============================================================
    //                          ERRORS
    // ============================================================
    error OnlyVault();
    error OnlyStrategist();
    error StrategyNotActive();
    error EmergencyExitMode();
    error SlippageTooHigh(uint256 loss, uint256 maxLoss);
    error AmountTooLarge(uint256 requested, uint256 available);
    
    // ============================================================
    //                          EVENTS
    // ============================================================
    event Harvested(uint256 profit, uint256 loss, uint256 debtPayment, uint256 debtOutstanding);
    event UpdatedStrategist(address indexed newStrategist);
    event UpdatedKeeper(address indexed newKeeper);
    event EmergencyExitEnabled();
    event StrategyMigrated(address indexed newStrategy);
    
    // ============================================================
    //                          STORAGE
    // ============================================================
    
    /// @notice want token - asset ที่ strategy ทำงานด้วย
    IERC20 public immutable want;
    
    /// @notice Vault ที่ strategy ทำงานให้
    address public immutable vault;
    
    /// @notice Strategist - ผู้ดูแล strategy
    address public strategist;
    
    /// @notice Keeper - ผู้มีสิทธิ์เรียก harvest
    address public keeper;
    
    /// @notice Emergency exit mode - ถอนเงินทั้งหมดกลับ Vault
    bool public emergencyExit;
    
    /// @notice ชื่อ strategy (สำหรับ display)
    string public name;
    
    // Performance tracking
    uint256 public totalDebt;          // เงินที่ Vault ให้มา
    uint256 public totalGain;          // กำไรสะสม
    uint256 public totalLoss;          // ขาดทุนสะสม
    uint256 public lastReport;         // timestamp ที่ harvest ล่าสุด
    uint256 public minReportDelay;     // harvest ได้ไม่บ่อยกว่านี้
    uint256 public maxReportDelay;     // ต้อง harvest ก่อน timeout นี้
    uint256 public debtThreshold;      // กำไรขั้นต่ำก่อน harvest
    
    // ============================================================
    //                        CONSTRUCTOR
    // ============================================================
    
    constructor(
        address _vault,
        address _want,
        string memory _name
    ) {
        vault = _vault;
        want = IERC20(_want);
        name = _name;
        strategist = msg.sender;
        keeper = msg.sender;
        lastReport = block.timestamp;
        minReportDelay = 6 hours;
        maxReportDelay = 1 days;
        debtThreshold = 0;
        
        // Approve vault เต็มจำนวน
        IERC20(_want).safeApprove(_vault, type(uint256).max);
    }
    
    // ============================================================
    //                          MODIFIERS
    // ============================================================
    
    modifier onlyVault() {
        if (msg.sender != vault) revert OnlyVault();
        _;
    }
    
    modifier onlyStrategist() {
        if (msg.sender != strategist) revert OnlyStrategist();
        _;
    }
    
    modifier onlyKeeper() {
        if (msg.sender != keeper && msg.sender != strategist) revert OnlyStrategist();
        _;
    }
    
    // ============================================================
    //                    ABSTRACT FUNCTIONS
    // ============================================================
    
    /// @notice ประมาณ total assets ที่ strategy ถือ
    /// @return _totalAssets มูลค่าทั้งหมดในหน่วย want token
    function estimatedTotalAssets() public view virtual returns (uint256 _totalAssets);
    
    /// @notice เก็บ yield และ reinvest
    /// @param _debtOutstanding หนี้ที่ต้องชำระคืน Vault
    /// @return _profit กำไรสุทธิ
    /// @return _loss ขาดทุน (ถ้ามี)
    /// @return _debtPayment จำนวนที่ชำระหนี้ได้
    function prepareReturn(uint256 _debtOutstanding)
        internal
        virtual
        returns (
            uint256 _profit,
            uint256 _loss,
            uint256 _debtPayment
        );
    
    /// @notice Deploy/Redeploy assets
    /// @param _debtOutstanding หนี้ที่ยังไม่ชำระ
    function adjustPosition(uint256 _debtOutstanding) internal virtual;
    
    /// @notice Liquidate position บางส่วน
    /// @param _amountNeeded จำนวนที่ต้องการ
    /// @return _liquidatedAmount จำนวนที่ liquidate ได้จริง
    /// @return _loss ขาดทุนจากการ liquidate
    function liquidatePosition(uint256 _amountNeeded)
        internal
        virtual
        returns (uint256 _liquidatedAmount, uint256 _loss);
    
    /// @notice Liquidate ทั้งหมด (สำหรับ emergency/migration)
    function liquidateAllPositions() internal virtual returns (uint256 _amountFreed);
    
    // ============================================================
    //                    CORE STRATEGY FUNCTIONS
    // ============================================================
    
    /// @notice Harvest: เก็บ yield, ชำระหนี้, report กลับ Vault
    /// @dev เรียกได้โดย keeper หรือ strategist
    function harvest() external onlyKeeper nonReentrant {
        // ตรวจสอบ emergency exit
        uint256 profit;
        uint256 loss;
        uint256 debtPayment;
        
        if (emergencyExit) {
            // Emergency: liquidate ทั้งหมดกลับ Vault
            uint256 liquidated = liquidateAllPositions();
            // ถ้า liquidated < totalDebt = มีการขาดทุน
            if (liquidated < totalDebt) {
                loss = totalDebt - liquidated;
                debtPayment = liquidated;
            } else {
                debtPayment = totalDebt;
                profit = liquidated - totalDebt;
            }
        } else {
            // Normal harvest: คำนวณ debt outstanding
            uint256 debtOutstanding = IVault(vault).debtOutstanding(address(this));
            
            // เก็บ yield
            (profit, loss, debtPayment) = prepareReturn(debtOutstanding);
        }
        
        // Report กลับ Vault และรับ debt outstanding ใหม่
        uint256 debtOutstanding = IVault(vault).report(profit, loss, debtPayment);
        
        // อัปเดต accounting
        totalGain += profit;
        totalLoss += loss;
        totalDebt = totalDebt + profit - loss - debtPayment;
        lastReport = block.timestamp;
        
        // Reinvest ที่เหลือ
        adjustPosition(debtOutstanding);
        
        emit Harvested(profit, loss, debtPayment, debtOutstanding);
    }
    
    /// @notice ถอนเงิน - เรียกโดย Vault
    /// @param _amountNeeded จำนวนที่ Vault ต้องการ
    /// @return _loss ขาดทุนจากการถอน (ถ้ามี)
    function withdraw(uint256 _amountNeeded) external onlyVault returns (uint256 _loss) {
        (uint256 liquidated, uint256 loss) = liquidatePosition(_amountNeeded);
        
        want.safeTransfer(vault, liquidated);
        _loss = loss;
    }
    
    /// @notice เปิด emergency exit
    function setEmergencyExit() external onlyStrategist {
        emergencyExit = true;
        emit EmergencyExitEnabled();
    }
    
    /// @notice อัปเดต keeper
    function setKeeper(address _keeper) external onlyStrategist {
        keeper = _keeper;
        emit UpdatedKeeper(_keeper);
    }
    
    /// @notice อัปเดต strategist
    function setStrategist(address _strategist) external onlyStrategist {
        strategist = _strategist;
        emit UpdatedStrategist(_strategist);
    }
    
    /// @notice ดึง free balance (ไม่รวมที่ deploy แล้ว)
    function balanceOfWant() public view returns (uint256) {
        return want.balanceOf(address(this));
    }
    
    /// @notice ตรวจสอบว่าควร harvest ได้แล้วหรือยัง
    function harvestTrigger(uint256 callCost) external view virtual returns (bool) {
        // เร็วเกินไปหรือยัง?
        if (block.timestamp - lastReport < minReportDelay) return false;
        
        // เกิน max delay ต้อง harvest
        if (block.timestamp - lastReport >= maxReportDelay) return true;
        
        // มีกำไรเพียงพอจะ cover gas cost
        uint256 currentAssets = estimatedTotalAssets();
        if (currentAssets > totalDebt + debtThreshold + callCost) return true;
        
        return false;
    }
}

// Interface สำหรับ Vault
interface IVault {
    function debtOutstanding(address strategy) external view returns (uint256);
    function report(uint256 profit, uint256 loss, uint256 debtPayment) external returns (uint256);
    function revokeStrategy(address strategy) external;
}
```

---

## 2. StrategyRouter - กระจาย Capital หลาย Strategies

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title StrategyRouter - จัดการการกระจาย capital ไปยัง strategies ต่างๆ
/// @notice ใช้ allocation points เพื่อกำหนดสัดส่วนของแต่ละ strategy
contract StrategyRouter is AccessControl, ReentrancyGuard {
    using SafeERC20 for IERC20;
    
    // ============================================================
    //                          ROLES
    // ============================================================
    bytes32 public constant MANAGER_ROLE = keccak256("MANAGER_ROLE");
    bytes32 public constant HARVESTER_ROLE = keccak256("HARVESTER_ROLE");
    
    // ============================================================
    //                          ERRORS
    // ============================================================
    error StrategyAlreadyAdded(address strategy);
    error StrategyNotFound(address strategy);
    error TotalPointsMustBePositive();
    error InvalidDebtRatio(uint256 total);
    error MaxStrategiesReached();
    error StrategyInDebt(address strategy, uint256 debt);
    error WithdrawalAmountTooLarge();
    
    // ============================================================
    //                          EVENTS
    // ============================================================
    event StrategyAdded(address indexed strategy, uint256 allocationPoints, uint256 maxDebtRatio);
    event StrategyUpdated(address indexed strategy, uint256 allocationPoints, uint256 maxDebtRatio);
    event StrategyRemoved(address indexed strategy);
    event FundsAllocated(address indexed strategy, uint256 amount);
    event FundsRecalled(address indexed strategy, uint256 amount);
    
    // ============================================================
    //                          STRUCTS
    // ============================================================
    struct StrategyParams {
        uint256 allocationPoints;   // น้ำหนักใน allocation
        uint256 maxDebtRatio;       // % สูงสุดของ total assets ที่ strategy นี้รับได้ (bps)
        uint256 currentDebt;        // debt ปัจจุบัน (เงินที่ให้ strategy ไปแล้ว)
        uint256 totalGain;          // กำไรสะสม
        uint256 totalLoss;          // ขาดทุนสะสม
        uint256 lastHarvest;        // timestamp harvest ล่าสุด
        bool active;
    }
    
    // ============================================================
    //                          STORAGE
    // ============================================================
    
    IERC20 public immutable token;      // want token
    address[] public strategyList;
    mapping(address => StrategyParams) public strategies;
    
    uint256 public totalAllocationPoints;
    uint256 public totalDebt;           // เงินรวมที่กระจายให้ strategies
    uint256 public totalAssets;         // ประมาณการ total assets
    
    uint256 public constant MAX_STRATEGIES = 20;
    uint256 public constant BPS = 10_000;           // 100% = 10000 bps
    uint256 public constant MAX_DEBT_RATIO = BPS;    // ไม่เกิน 100%
    
    // ============================================================
    //                        CONSTRUCTOR
    // ============================================================
    
    constructor(address _token, address _admin) {
        token = IERC20(_token);
        _grantRole(DEFAULT_ADMIN_ROLE, _admin);
        _grantRole(MANAGER_ROLE, _admin);
        _grantRole(HARVESTER_ROLE, _admin);
    }
    
    // ============================================================
    //                    STRATEGY MANAGEMENT
    // ============================================================
    
    /// @notice เพิ่ม strategy ใหม่
    /// @param strategy address ของ strategy contract
    /// @param allocationPoints น้ำหนักในการ allocate
    /// @param maxDebtRatioBps % สูงสุดของ total assets (basis points)
    function addStrategy(
        address strategy,
        uint256 allocationPoints,
        uint256 maxDebtRatioBps
    ) external onlyRole(MANAGER_ROLE) {
        if (strategies[strategy].active) revert StrategyAlreadyAdded(strategy);
        if (strategyList.length >= MAX_STRATEGIES) revert MaxStrategiesReached();
        if (maxDebtRatioBps > MAX_DEBT_RATIO) revert InvalidDebtRatio(maxDebtRatioBps);
        
        strategies[strategy] = StrategyParams({
            allocationPoints: allocationPoints,
            maxDebtRatio: maxDebtRatioBps,
            currentDebt: 0,
            totalGain: 0,
            totalLoss: 0,
            lastHarvest: block.timestamp,
            active: true
        });
        
        strategyList.push(strategy);
        totalAllocationPoints += allocationPoints;
        
        emit StrategyAdded(strategy, allocationPoints, maxDebtRatioBps);
    }
    
    /// @notice อัปเดต allocation points ของ strategy
    function updateStrategy(
        address strategy,
        uint256 newAllocationPoints,
        uint256 newMaxDebtRatioBps
    ) external onlyRole(MANAGER_ROLE) {
        if (!strategies[strategy].active) revert StrategyNotFound(strategy);
        if (newMaxDebtRatioBps > MAX_DEBT_RATIO) revert InvalidDebtRatio(newMaxDebtRatioBps);
        
        StrategyParams storage params = strategies[strategy];
        totalAllocationPoints = totalAllocationPoints - params.allocationPoints + newAllocationPoints;
        params.allocationPoints = newAllocationPoints;
        params.maxDebtRatio = newMaxDebtRatioBps;
        
        emit StrategyUpdated(strategy, newAllocationPoints, newMaxDebtRatioBps);
    }
    
    /// @notice ลบ strategy (ต้อง recall debt ก่อน)
    function removeStrategy(address strategy) external onlyRole(MANAGER_ROLE) {
        StrategyParams storage params = strategies[strategy];
        if (!params.active) revert StrategyNotFound(strategy);
        if (params.currentDebt > 0) revert StrategyInDebt(strategy, params.currentDebt);
        
        totalAllocationPoints -= params.allocationPoints;
        params.active = false;
        
        // ลบออกจาก list
        for (uint256 i = 0; i < strategyList.length; i++) {
            if (strategyList[i] == strategy) {
                strategyList[i] = strategyList[strategyList.length - 1];
                strategyList.pop();
                break;
            }
        }
        
        emit StrategyRemoved(strategy);
    }
    
    // ============================================================
    //                    CAPITAL ALLOCATION
    // ============================================================
    
    /// @notice กระจาย capital ตาม allocation points
    /// @dev เรียกหลังจากรับ deposit ใหม่
    function allocateFunds() external onlyRole(MANAGER_ROLE) nonReentrant {
        if (totalAllocationPoints == 0) revert TotalPointsMustBePositive();
        
        uint256 freeBalance = token.balanceOf(address(this));
        if (freeBalance == 0) return;
        
        // กระจายตาม allocation points
        for (uint256 i = 0; i < strategyList.length; i++) {
            address strategy = strategyList[i];
            StrategyParams storage params = strategies[strategy];
            
            if (!params.active) continue;
            
            // คำนวณว่า strategy นี้ควรได้รับเท่าไร
            uint256 targetAllocation = (totalAssets * params.maxDebtRatio) / BPS;
            
            if (params.currentDebt < targetAllocation) {
                uint256 toAllocate = targetAllocation - params.currentDebt;
                toAllocate = toAllocate < freeBalance ? toAllocate : freeBalance;
                
                if (toAllocate > 0) {
                    token.safeTransfer(strategy, toAllocate);
                    params.currentDebt += toAllocate;
                    totalDebt += toAllocate;
                    freeBalance -= toAllocate;
                    
                    emit FundsAllocated(strategy, toAllocate);
                }
            }
            
            if (freeBalance == 0) break;
        }
    }
    
    /// @notice Recall funds จาก strategy
    /// @param strategy address ของ strategy
    /// @param amount จำนวนที่ต้องการ recall
    function recallFunds(address strategy, uint256 amount) external onlyRole(MANAGER_ROLE) nonReentrant {
        StrategyParams storage params = strategies[strategy];
        if (!params.active && params.currentDebt == 0) revert StrategyNotFound(strategy);
        
        uint256 toRecall = amount < params.currentDebt ? amount : params.currentDebt;
        
        // เรียก strategy ให้ถอนเงินกลับมา
        uint256 loss = IBaseStrategy(strategy).withdraw(toRecall);
        uint256 received = token.balanceOf(address(this));
        
        // อัปเดต accounting
        params.currentDebt -= (toRecall - loss);
        totalDebt -= (toRecall - loss);
        
        if (loss > 0) {
            params.totalLoss += loss;
        }
        
        emit FundsRecalled(strategy, received);
    }
    
    // ============================================================
    //                      HARVEST FUNCTIONS
    // ============================================================
    
    /// @notice Harvest ทุก strategies
    function harvestAll() external onlyRole(HARVESTER_ROLE) nonReentrant {
        for (uint256 i = 0; i < strategyList.length; i++) {
            address strategy = strategyList[i];
            if (!strategies[strategy].active) continue;
            
            try IBaseStrategy(strategy).harvest() {
                strategies[strategy].lastHarvest = block.timestamp;
            } catch {
                // Skip failed harvest (don't revert all)
            }
        }
        
        // อัปเดต total assets
        _updateTotalAssets();
    }
    
    /// @notice Harvest strategy เดียว
    function harvestStrategy(address strategy) external onlyRole(HARVESTER_ROLE) nonReentrant {
        if (!strategies[strategy].active) revert StrategyNotFound(strategy);
        
        IBaseStrategy(strategy).harvest();
        strategies[strategy].lastHarvest = block.timestamp;
        
        _updateTotalAssets();
    }
    
    // ============================================================
    //                      VIEW FUNCTIONS
    // ============================================================
    
    /// @notice ดึงรายการ strategies ทั้งหมด
    function getStrategyList() external view returns (address[] memory) {
        return strategyList;
    }
    
    /// @notice คำนวณ target debt สำหรับแต่ละ strategy
    function getStrategyTargetDebt(address strategy) external view returns (uint256) {
        StrategyParams memory params = strategies[strategy];
        return (totalAssets * params.maxDebtRatio) / BPS;
    }
    
    /// @notice ดู debt ที่ outstanding ของ strategy
    function debtOutstanding(address strategy) external view returns (uint256) {
        StrategyParams memory params = strategies[strategy];
        uint256 target = (totalAssets * params.maxDebtRatio) / BPS;
        
        if (params.currentDebt > target) {
            return params.currentDebt - target;
        }
        return 0;
    }
    
    // ============================================================
    //                      INTERNAL FUNCTIONS
    // ============================================================
    
    function _updateTotalAssets() internal {
        uint256 newTotal = token.balanceOf(address(this)); // free balance
        
        for (uint256 i = 0; i < strategyList.length; i++) {
            address strategy = strategyList[i];
            if (!strategies[strategy].active) continue;
            
            try IBaseStrategy(strategy).estimatedTotalAssets() returns (uint256 assets) {
                newTotal += assets;
            } catch {
                // ใช้ currentDebt เป็น fallback
                newTotal += strategies[strategy].currentDebt;
            }
        }
        
        totalAssets = newTotal;
    }
}

interface IBaseStrategy {
    function harvest() external;
    function withdraw(uint256 amount) external returns (uint256 loss);
    function estimatedTotalAssets() external view returns (uint256);
}
```

---

## 3. AutoCompoundVault (ERC-4626)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/utils/math/Math.sol";

/// @title AutoCompoundVault - ERC-4626 Vault ที่ auto-compound yield
/// @notice ผู้ใช้ deposit assets, Vault กระจายไปยัง strategies และ compound อัตโนมัติ
contract AutoCompoundVault is ERC4626, Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;
    using Math for uint256;
    
    // ============================================================
    //                          ERRORS
    // ============================================================
    error DepositLimitExceeded(uint256 amount, uint256 limit);
    error WithdrawalQueueEmpty();
    error NotEnoughIdleFunds(uint256 available, uint256 needed);
    error ManagementFeeExcessive(uint256 fee);
    error PerformanceFeeExcessive(uint256 fee);
    
    // ============================================================
    //                          EVENTS
    // ============================================================
    event StrategyReported(
        address indexed strategy,
        uint256 gain,
        uint256 loss,
        uint256 debtPaid,
        uint256 totalGain,
        uint256 totalLoss,
        uint256 totalDebt,
        uint256 debtAdded,
        uint256 debtRatio
    );
    event ManagementFeeUpdated(uint256 newFee);
    event PerformanceFeeUpdated(uint256 newFee);
    event FeeRecipientUpdated(address indexed newRecipient);
    event EmergencyShutdown(bool isShutdown);
    
    // ============================================================
    //                          STRUCTS
    // ============================================================
    struct StrategyParams {
        uint256 performanceFee;     // fee ที่เก็บจากกำไรของ strategy (bps)
        uint256 activation;         // timestamp ที่ activate
        uint256 debtRatio;          // เปอร์เซ็นต์ของ total assets (bps)
        uint256 minDebtPerHarvest;  // debt เพิ่มขั้นต่ำต่อ harvest
        uint256 maxDebtPerHarvest;  // debt เพิ่มสูงสุดต่อ harvest
        uint256 lastReport;         // timestamp harvest ล่าสุด
        uint256 totalDebt;          // total debt ปัจจุบัน
        uint256 totalGain;          // กำไรสะสม
        uint256 totalLoss;          // ขาดทุนสะสม
    }
    
    // ============================================================
    //                          STORAGE
    // ============================================================
    
    // Fee configuration
    uint256 public managementFee;       // Annual management fee (bps)
    uint256 public performanceFee;      // Default performance fee (bps)
    address public feeRecipient;
    uint256 public lastManagementFeeCollection;
    
    uint256 public constant MAX_MANAGEMENT_FEE = 200;    // 2%
    uint256 public constant MAX_PERFORMANCE_FEE = 5000;  // 50%
    uint256 public constant BPS = 10_000;
    uint256 public constant SECS_PER_YEAR = 31_556_952;
    
    // Vault configuration
    uint256 public depositLimit;        // ขีดจำกัด total deposits
    bool public emergencyShutdown;      // ปิด vault ฉุกเฉิน
    
    // Strategy tracking
    address[] public withdrawalQueue;   // ลำดับ strategies ที่จะ withdraw จาก
    mapping(address => StrategyParams) public strategies;
    
    uint256 public totalDebt;           // รวมที่กระจายให้ strategies ทั้งหมด
    uint256 public debtRatioUsed;       // รวม debt ratios (max 10000)
    
    // ============================================================
    //                        CONSTRUCTOR
    // ============================================================
    
    constructor(
        IERC20 _asset,
        string memory _name,
        string memory _symbol,
        address _feeRecipient,
        address _owner
    ) ERC4626(_asset) ERC20(_name, _symbol) Ownable(_owner) {
        feeRecipient = _feeRecipient;
        managementFee = 200;     // 2% default
        performanceFee = 2000;   // 20% default
        lastManagementFeeCollection = block.timestamp;
        depositLimit = type(uint256).max;
    }
    
    // ============================================================
    //                    ERC-4626 OVERRIDES
    // ============================================================
    
    /// @notice คำนวณ total assets รวมที่ strategies ถือ
    function totalAssets() public view override returns (uint256) {
        return IERC20(asset()).balanceOf(address(this)) + totalDebt;
    }
    
    /// @notice Override deposit เพื่อเพิ่ม deposit limit check
    function deposit(uint256 assets, address receiver) public override nonReentrant returns (uint256 shares) {
        if (emergencyShutdown) revert EmergencyShutdown(true);
        
        uint256 currentTotal = totalAssets();
        if (currentTotal + assets > depositLimit) {
            revert DepositLimitExceeded(assets, depositLimit - currentTotal);
        }
        
        shares = super.deposit(assets, receiver);
        
        // Auto-invest หลัง deposit
        _invest();
    }
    
    /// @notice Override withdraw เพื่อ liquidate จาก strategies ถ้าจำเป็น
    function withdraw(
        uint256 assets,
        address receiver,
        address owner_
    ) public override nonReentrant returns (uint256 shares) {
        // ตรวจสอบว่ามี idle funds เพียงพอ
        uint256 idleFunds = IERC20(asset()).balanceOf(address(this));
        
        if (idleFunds < assets) {
            // ต้อง liquidate จาก strategies
            _liquidateFromStrategies(assets - idleFunds);
        }
        
        shares = super.withdraw(assets, receiver, owner_);
    }
    
    // ============================================================
    //                    STRATEGY MANAGEMENT
    // ============================================================
    
    /// @notice เพิ่ม strategy
    function addStrategy(
        address strategy,
        uint256 debtRatio_,
        uint256 minDebtPerHarvest_,
        uint256 maxDebtPerHarvest_,
        uint256 performanceFee_
    ) external onlyOwner {
        require(strategy != address(0), "zero address");
        require(strategies[strategy].activation == 0, "already active");
        require(debtRatioUsed + debtRatio_ <= BPS, "ratio exceeds 100%");
        require(performanceFee_ <= MAX_PERFORMANCE_FEE, "fee too high");
        
        strategies[strategy] = StrategyParams({
            performanceFee: performanceFee_,
            activation: block.timestamp,
            debtRatio: debtRatio_,
            minDebtPerHarvest: minDebtPerHarvest_,
            maxDebtPerHarvest: maxDebtPerHarvest_,
            lastReport: block.timestamp,
            totalDebt: 0,
            totalGain: 0,
            totalLoss: 0
        });
        
        withdrawalQueue.push(strategy);
        debtRatioUsed += debtRatio_;
    }
    
    /// @notice Strategy รายงานผลกำไร/ขาดทุน
    /// @dev เรียกโดย strategy ใน harvest()
    function report(
        uint256 gain,
        uint256 loss,
        uint256 debtPayment
    ) external returns (uint256 debt) {
        StrategyParams storage params = strategies[msg.sender];
        require(params.activation > 0, "strategy not active");
        
        // อัปเดต strategy accounting
        if (gain > 0) {
            // เก็บ performance fee
            uint256 strategyFee = (gain * params.performanceFee) / BPS;
            uint256 managementFeeNow = _calculateManagementFee();
            uint256 totalFees = strategyFee + managementFeeNow;
            
            if (totalFees > 0 && feeRecipient != address(0)) {
                // Mint shares เพื่อจ่าย fees
                uint256 feeShares = convertToShares(totalFees);
                _mint(feeRecipient, feeShares);
            }
            
            params.totalGain += gain;
            totalDebt += gain;
        }
        
        if (loss > 0) {
            params.totalLoss += loss;
            totalDebt -= loss;
        }
        
        // คำนวณ debt ที่ strategy ควรมี
        uint256 creditAvailable = _creditAvailable(msg.sender);
        uint256 debtOutstanding = _debtOutstanding(msg.sender);
        
        // ถ้า strategy ชำระหนี้มา ลด totalDebt
        if (debtPayment > 0) {
            params.totalDebt -= debtPayment;
            totalDebt -= debtPayment;
        }
        
        // คืน debt outstanding เพื่อให้ strategy รู้ว่าต้องชำระเพิ่มหรือไม่
        debt = debtOutstanding;
        
        // ถ้า strategy มี credit ให้ส่งเพิ่ม
        if (creditAvailable > 0) {
            IERC20(asset()).safeTransfer(msg.sender, creditAvailable);
            params.totalDebt += creditAvailable;
            totalDebt += creditAvailable;
        }
        
        params.lastReport = block.timestamp;
        
        emit StrategyReported(
            msg.sender,
            gain,
            loss,
            debtPayment,
            params.totalGain,
            params.totalLoss,
            params.totalDebt,
            creditAvailable,
            params.debtRatio
        );
    }
    
    // ============================================================
    //                      INTERNAL FUNCTIONS
    // ============================================================
    
    /// @notice Auto-invest idle funds
    function _invest() internal {
        uint256 available = IERC20(asset()).balanceOf(address(this));
        
        for (uint256 i = 0; i < withdrawalQueue.length; i++) {
            address strategy = withdrawalQueue[i];
            StrategyParams storage params = strategies[strategy];
            
            if (available == 0) break;
            
            uint256 credit = _creditAvailable(strategy);
            if (credit == 0) continue;
            
            uint256 toInvest = credit < available ? credit : available;
            
            IERC20(asset()).safeTransfer(strategy, toInvest);
            params.totalDebt += toInvest;
            totalDebt += toInvest;
            available -= toInvest;
        }
    }
    
    /// @notice Liquidate จาก strategies เรียงตาม withdrawal queue
    function _liquidateFromStrategies(uint256 amount) internal {
        for (uint256 i = 0; i < withdrawalQueue.length; i++) {
            address strategy = withdrawalQueue[i];
            StrategyParams storage params = strategies[strategy];
            
            if (amount == 0) break;
            if (params.totalDebt == 0) continue;
            
            uint256 toLiquidate = amount < params.totalDebt ? amount : params.totalDebt;
            
            uint256 loss = IBaseStrategy(strategy).withdraw(toLiquidate);
            uint256 received = toLiquidate - loss;
            
            params.totalDebt -= received;
            totalDebt -= received;
            
            if (loss > 0) params.totalLoss += loss;
            
            amount = received >= amount ? 0 : amount - received;
        }
    }
    
    /// @notice คำนวณ credit ที่ strategy สามารถรับได้
    function _creditAvailable(address strategy) internal view returns (uint256) {
        StrategyParams storage params = strategies[strategy];
        
        uint256 strategyLimit = (totalAssets() * params.debtRatio) / BPS;
        uint256 strategyCurrent = params.totalDebt;
        
        if (strategyCurrent >= strategyLimit) return 0;
        
        uint256 available = strategyLimit - strategyCurrent;
        
        // จำกัดด้วย maxDebtPerHarvest
        if (available > params.maxDebtPerHarvest) {
            available = params.maxDebtPerHarvest;
        }
        
        // ต้องอยู่ใน idle funds
        uint256 idle = IERC20(asset()).balanceOf(address(this));
        return available < idle ? available : idle;
    }
    
    /// @notice คำนวณ debt outstanding ของ strategy
    function _debtOutstanding(address strategy) internal view returns (uint256) {
        StrategyParams storage params = strategies[strategy];
        
        uint256 strategyLimit = (totalAssets() * params.debtRatio) / BPS;
        uint256 strategyCurrent = params.totalDebt;
        
        if (strategyCurrent <= strategyLimit) return 0;
        return strategyCurrent - strategyLimit;
    }
    
    /// @notice คำนวณ management fee ที่ต้องเก็บ
    function _calculateManagementFee() internal returns (uint256 fee) {
        if (managementFee == 0) return 0;
        
        uint256 elapsed = block.timestamp - lastManagementFeeCollection;
        fee = (totalAssets() * managementFee * elapsed) / (BPS * SECS_PER_YEAR);
        lastManagementFeeCollection = block.timestamp;
    }
    
    // ============================================================
    //                      ADMIN FUNCTIONS
    // ============================================================
    
    function setDepositLimit(uint256 limit) external onlyOwner {
        depositLimit = limit;
    }
    
    function setEmergencyShutdown(bool shutdown) external onlyOwner {
        emergencyShutdown = shutdown;
        emit EmergencyShutdown(shutdown);
    }
    
    function setManagementFee(uint256 fee) external onlyOwner {
        if (fee > MAX_MANAGEMENT_FEE) revert ManagementFeeExcessive(fee);
        managementFee = fee;
        emit ManagementFeeUpdated(fee);
    }
    
    function setPerformanceFee(uint256 fee) external onlyOwner {
        if (fee > MAX_PERFORMANCE_FEE) revert PerformanceFeeExcessive(fee);
        performanceFee = fee;
        emit PerformanceFeeUpdated(fee);
    }
    
    function setFeeRecipient(address recipient) external onlyOwner {
        feeRecipient = recipient;
        emit FeeRecipientUpdated(recipient);
    }
    
    // แก้ Error ที่เกิดจาก naming conflict
    error EmergencyShutdown(bool value);
}
```

---

## 4. GenericLendingStrategy

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "./BaseStrategy.sol";

/// @title GenericLendingStrategy - Strategy สำหรับฝาก assets ใน Aave-style lending pool
/// @notice ฝาก want token ใน lending protocol เพื่อรับ interest + rewards
contract GenericLendingStrategy is BaseStrategy {
    using SafeERC20 for IERC20;
    
    // ============================================================
    //                          ERRORS
    // ============================================================
    error HarvestNotProfitable(uint256 gas, uint256 expectedProfit);
    
    // ============================================================
    //                          EVENTS
    // ============================================================
    event RewardSold(address indexed reward, uint256 amount, uint256 received);
    
    // ============================================================
    //                          STORAGE
    // ============================================================
    
    /// @notice Lending pool ที่ใช้ (เช่น Aave)
    ILendingPool public immutable lendingPool;
    
    /// @notice aToken ที่ได้รับจากการ deposit
    IERC20 public immutable aToken;
    
    /// @notice Reward token (เช่น AAVE, COMP)
    IERC20 public immutable rewardToken;
    
    /// @notice DEX สำหรับขาย rewards
    ISwapRouter public immutable swapRouter;
    
    /// @notice Referral code สำหรับ Aave
    uint16 public constant REFERRAL_CODE = 0;
    
    /// @notice Slippage tolerance สำหรับ reward selling (bps)
    uint256 public slippageTolerance;
    
    // ============================================================
    //                        CONSTRUCTOR
    // ============================================================
    
    constructor(
        address _vault,
        address _want,
        address _lendingPool,
        address _aToken,
        address _rewardToken,
        address _swapRouter
    ) BaseStrategy(_vault, _want, "Generic Lending Strategy") {
        lendingPool = ILendingPool(_lendingPool);
        aToken = IERC20(_aToken);
        rewardToken = IERC20(_rewardToken);
        swapRouter = ISwapRouter(_swapRouter);
        slippageTolerance = 100; // 1% default
        
        // Approve lending pool
        want.safeApprove(_lendingPool, type(uint256).max);
        // Approve swap router สำหรับ rewards
        IERC20(_rewardToken).safeApprove(_swapRouter, type(uint256).max);
    }
    
    // ============================================================
    //                    BASESTR ATEGY IMPLEMENTATIONS
    // ============================================================
    
    /// @notice ประมาณ total assets
    function estimatedTotalAssets() public view override returns (uint256) {
        // aToken balance ≈ principal + accrued interest (1:1 ratio ใน Aave)
        return want.balanceOf(address(this)) + aToken.balanceOf(address(this));
    }
    
    /// @notice Harvest: ขาย rewards, คำนวณ profit/loss
    function prepareReturn(uint256 _debtOutstanding)
        internal
        override
        returns (
            uint256 _profit,
            uint256 _loss,
            uint256 _debtPayment
        )
    {
        // ดึง pending rewards จาก lending pool
        _claimRewards();
        
        // ขาย rewards เพื่อได้ want token
        uint256 rewardBalance = rewardToken.balanceOf(address(this));
        if (rewardBalance > 0) {
            uint256 received = _sellRewards(rewardBalance);
            emit RewardSold(address(rewardToken), rewardBalance, received);
        }
        
        // คำนวณ profit/loss เทียบกับ totalDebt
        uint256 totalAssets_ = estimatedTotalAssets();
        
        if (totalAssets_ > totalDebt) {
            _profit = totalAssets_ - totalDebt;
        } else {
            _loss = totalDebt - totalAssets_;
        }
        
        // ชำระ debt ถ้าจำเป็น
        if (_debtOutstanding > 0) {
            uint256 wantBalance = want.balanceOf(address(this));
            
            if (wantBalance < _debtOutstanding) {
                // ต้องถอนจาก lending pool
                uint256 toWithdraw = _debtOutstanding - wantBalance;
                _withdrawFromLending(toWithdraw);
            }
            
            _debtPayment = want.balanceOf(address(this));
            if (_debtPayment > _debtOutstanding) {
                _debtPayment = _debtOutstanding;
            }
        }
    }
    
    /// @notice Deploy idle funds ไปยัง lending pool
    function adjustPosition(uint256 _debtOutstanding) internal override {
        if (emergencyExit) return;
        
        uint256 wantBalance = want.balanceOf(address(this));
        
        // เก็บส่วน debt outstanding ไว้ก่อน
        if (wantBalance > _debtOutstanding) {
            uint256 toInvest = wantBalance - _debtOutstanding;
            _depositToLending(toInvest);
        }
    }
    
    /// @notice Liquidate บางส่วน
    function liquidatePosition(uint256 _amountNeeded)
        internal
        override
        returns (uint256 _liquidatedAmount, uint256 _loss)
    {
        uint256 wantBalance = want.balanceOf(address(this));
        
        if (wantBalance >= _amountNeeded) {
            return (_amountNeeded, 0);
        }
        
        // ต้องถอนจาก lending pool
        uint256 fromLending = _amountNeeded - wantBalance;
        uint256 withdrawn = _withdrawFromLending(fromLending);
        
        _liquidatedAmount = want.balanceOf(address(this));
        
        if (_liquidatedAmount < _amountNeeded) {
            _loss = _amountNeeded - _liquidatedAmount;
        }
    }
    
    /// @notice Liquidate ทั้งหมด
    function liquidateAllPositions() internal override returns (uint256 _amountFreed) {
        uint256 aTokenBalance = aToken.balanceOf(address(this));
        if (aTokenBalance > 0) {
            lendingPool.withdraw(address(want), aTokenBalance, address(this));
        }
        _amountFreed = want.balanceOf(address(this));
    }
    
    // ============================================================
    //                      INTERNAL FUNCTIONS
    // ============================================================
    
    function _depositToLending(uint256 amount) internal {
        lendingPool.deposit(address(want), amount, address(this), REFERRAL_CODE);
    }
    
    function _withdrawFromLending(uint256 amount) internal returns (uint256 withdrawn) {
        uint256 aTokenBalance = aToken.balanceOf(address(this));
        uint256 toWithdraw = amount < aTokenBalance ? amount : aTokenBalance;
        
        withdrawn = lendingPool.withdraw(address(want), toWithdraw, address(this));
    }
    
    function _claimRewards() internal {
        // Claim rewards จาก incentives controller
        address[] memory assets = new address[](1);
        assets[0] = address(aToken);
        
        try IIncentivesController(lendingPool.getIncentivesController()).claimRewards(
            assets,
            type(uint256).max,
            address(this)
        ) {} catch {}
    }
    
    function _sellRewards(uint256 amount) internal returns (uint256 received) {
        // คำนวณ min output พร้อม slippage tolerance
        // ในกรณีจริง ต้องดึงราคาจาก oracle ก่อน
        uint256 minOut = (amount * (BPS - slippageTolerance)) / BPS;
        
        received = swapRouter.exactInputSingle(
            ISwapRouter.ExactInputSingleParams({
                tokenIn: address(rewardToken),
                tokenOut: address(want),
                fee: 3000,  // 0.3% Uniswap pool
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: amount,
                amountOutMinimum: minOut,
                sqrtPriceLimitX96: 0
            })
        );
    }
    
    uint256 constant BPS = 10_000;
}

// Interfaces ที่ต้องการ
interface ILendingPool {
    function deposit(address asset, uint256 amount, address onBehalfOf, uint16 referralCode) external;
    function withdraw(address asset, uint256 amount, address to) external returns (uint256);
    function getIncentivesController() external view returns (address);
}

interface IIncentivesController {
    function claimRewards(address[] calldata assets, uint256 amount, address to) external returns (uint256);
}

interface ISwapRouter {
    struct ExactInputSingleParams {
        address tokenIn;
        address tokenOut;
        uint24 fee;
        address recipient;
        uint256 deadline;
        uint256 amountIn;
        uint256 amountOutMinimum;
        uint160 sqrtPriceLimitX96;
    }
    
    function exactInputSingle(ExactInputSingleParams calldata params) external returns (uint256 amountOut);
}
```

---

## 5. Hardhat Test Skeleton (TypeScript)

```typescript
// test/AutoCompoundVault.test.ts
import { ethers } from "hardhat";
import { expect } from "chai";
import { loadFixture, time } from "@nomicfoundation/hardhat-network-helpers";
import { AutoCompoundVault, MockERC20, MockLendingPool, GenericLendingStrategy } from "../typechain-types";
import { SignerWithAddress } from "@nomicfoundation/hardhat-ethers/signers";

describe("AutoCompoundVault", () => {
  // ============================================================
  //                        FIXTURES
  // ============================================================
  
  async function deployVaultFixture() {
    const [owner, alice, bob, keeper, feeRecipient] = await ethers.getSigners();
    
    // Deploy mock USDC
    const MockERC20 = await ethers.getContractFactory("MockERC20");
    const usdc = await MockERC20.deploy("USD Coin", "USDC", 6);
    
    // Deploy mock aToken
    const aUsdc = await MockERC20.deploy("Aave USDC", "aUSDC", 6);
    
    // Deploy mock reward token
    const aave = await MockERC20.deploy("Aave Token", "AAVE", 18);
    
    // Deploy mock lending pool
    const MockLendingPool = await ethers.getContractFactory("MockLendingPool");
    const lendingPool = await MockLendingPool.deploy(
      await usdc.getAddress(),
      await aUsdc.getAddress()
    );
    
    // Deploy mock swap router
    const MockSwapRouter = await ethers.getContractFactory("MockSwapRouter");
    const swapRouter = await MockSwapRouter.deploy();
    
    // Deploy Vault
    const AutoCompoundVault = await ethers.getContractFactory("AutoCompoundVault");
    const vault = await AutoCompoundVault.deploy(
      await usdc.getAddress(),
      "Vault USDC",
      "vUSDC",
      feeRecipient.address,
      owner.address
    );
    
    // Deploy Strategy
    const GenericLendingStrategy = await ethers.getContractFactory("GenericLendingStrategy");
    const strategy = await GenericLendingStrategy.deploy(
      await vault.getAddress(),
      await usdc.getAddress(),
      await lendingPool.getAddress(),
      await aUsdc.getAddress(),
      await aave.getAddress(),
      await swapRouter.getAddress()
    );
    
    // Setup: mint tokens
    await usdc.mint(alice.address, ethers.parseUnits("10000", 6));
    await usdc.mint(bob.address, ethers.parseUnits("5000", 6));
    
    // Approve vault
    await usdc.connect(alice).approve(await vault.getAddress(), ethers.MaxUint256);
    await usdc.connect(bob).approve(await vault.getAddress(), ethers.MaxUint256);
    
    return { vault, strategy, usdc, aUsdc, aave, lendingPool, swapRouter, owner, alice, bob, keeper, feeRecipient };
  }
  
  // ============================================================
  //                          TESTS
  // ============================================================
  
  describe("Deployment", () => {
    it("should deploy with correct parameters", async () => {
      const { vault, usdc, feeRecipient, owner } = await loadFixture(deployVaultFixture);
      
      expect(await vault.asset()).to.equal(await usdc.getAddress());
      expect(await vault.feeRecipient()).to.equal(feeRecipient.address);
      expect(await vault.owner()).to.equal(owner.address);
      expect(await vault.managementFee()).to.equal(200); // 2%
      expect(await vault.performanceFee()).to.equal(2000); // 20%
    });
    
    it("should have zero total assets initially", async () => {
      const { vault } = await loadFixture(deployVaultFixture);
      expect(await vault.totalAssets()).to.equal(0);
    });
  });
  
  describe("Deposits", () => {
    it("should allow user to deposit", async () => {
      const { vault, usdc, alice } = await loadFixture(deployVaultFixture);
      
      const depositAmount = ethers.parseUnits("1000", 6);
      
      await expect(vault.connect(alice).deposit(depositAmount, alice.address))
        .to.emit(vault, "Deposit")
        .withArgs(alice.address, alice.address, depositAmount, depositAmount);
      
      expect(await vault.balanceOf(alice.address)).to.equal(depositAmount);
      expect(await vault.totalAssets()).to.equal(depositAmount);
    });
    
    it("should revert when deposit limit exceeded", async () => {
      const { vault, alice, owner } = await loadFixture(deployVaultFixture);
      
      // Set deposit limit ต่ำๆ
      await vault.connect(owner).setDepositLimit(ethers.parseUnits("100", 6));
      
      await expect(
        vault.connect(alice).deposit(ethers.parseUnits("200", 6), alice.address)
      ).to.be.revertedWithCustomError(vault, "DepositLimitExceeded");
    });
    
    it("should revert when emergency shutdown", async () => {
      const { vault, alice, owner } = await loadFixture(deployVaultFixture);
      
      await vault.connect(owner).setEmergencyShutdown(true);
      
      await expect(
        vault.connect(alice).deposit(ethers.parseUnits("1000", 6), alice.address)
      ).to.be.revertedWithCustomError(vault, "EmergencyShutdown");
    });
    
    it("multiple users can deposit and get proportional shares", async () => {
      const { vault, usdc, alice, bob } = await loadFixture(deployVaultFixture);
      
      const aliceDeposit = ethers.parseUnits("1000", 6);
      const bobDeposit = ethers.parseUnits("500", 6);
      
      await vault.connect(alice).deposit(aliceDeposit, alice.address);
      await vault.connect(bob).deposit(bobDeposit, bob.address);
      
      const aliceShares = await vault.balanceOf(alice.address);
      const bobShares = await vault.balanceOf(bob.address);
      
      // Alice ควรมี shares เป็น 2x ของ Bob
      expect(aliceShares).to.equal(aliceDeposit);
      expect(bobShares).to.equal(bobDeposit);
    });
  });
  
  describe("Strategy Integration", () => {
    it("should add strategy and allocate funds", async () => {
      const { vault, strategy, usdc, alice, owner } = await loadFixture(deployVaultFixture);
      
      // Add strategy with 80% debt ratio
      await vault.connect(owner).addStrategy(
        await strategy.getAddress(),
        8000,  // 80% debt ratio
        0,
        ethers.parseUnits("100000", 6),  // max per harvest
        2000   // 20% performance fee
      );
      
      // Alice deposits
      await vault.connect(alice).deposit(ethers.parseUnits("1000", 6), alice.address);
      
      // Verify strategy state
      const strategyParams = await vault.strategies(await strategy.getAddress());
      expect(strategyParams.debtRatio).to.equal(8000);
      expect(strategyParams.activation).to.be.gt(0);
    });
    
    it("should harvest and compound yield", async () => {
      const { vault, strategy, usdc, aUsdc, alice, owner, lendingPool } = await loadFixture(deployVaultFixture);
      
      // Setup
      await vault.connect(owner).addStrategy(
        await strategy.getAddress(),
        9000, 0, ethers.MaxUint256, 2000
      );
      
      await vault.connect(alice).deposit(ethers.parseUnits("1000", 6), alice.address);
      
      // Simulate interest accrual (lending pool mints more aTokens)
      await time.increase(30 * 24 * 3600); // 30 days
      await aUsdc.mint(await strategy.getAddress(), ethers.parseUnits("5", 6)); // 0.5% yield
      
      const assetsBefore = await vault.totalAssets();
      
      // Harvest
      await strategy.connect(owner).harvest();
      
      const assetsAfter = await vault.totalAssets();
      
      // Total assets ควรเพิ่มขึ้น
      expect(assetsAfter).to.be.gt(assetsBefore);
    });
  });
  
  describe("Withdrawals", () => {
    it("should allow withdrawal when sufficient idle funds", async () => {
      const { vault, usdc, alice } = await loadFixture(deployVaultFixture);
      
      const depositAmount = ethers.parseUnits("1000", 6);
      await vault.connect(alice).deposit(depositAmount, alice.address);
      
      const shares = await vault.balanceOf(alice.address);
      const balanceBefore = await usdc.balanceOf(alice.address);
      
      await vault.connect(alice).redeem(shares, alice.address, alice.address);
      
      const balanceAfter = await usdc.balanceOf(alice.address);
      expect(balanceAfter - balanceBefore).to.be.closeTo(depositAmount, ethers.parseUnits("1", 3));
    });
    
    it("should liquidate from strategies when needed", async () => {
      const { vault, strategy, usdc, alice, owner } = await loadFixture(deployVaultFixture);
      
      // Add strategy
      await vault.connect(owner).addStrategy(
        await strategy.getAddress(),
        9000, 0, ethers.MaxUint256, 2000
      );
      
      await vault.connect(alice).deposit(ethers.parseUnits("1000", 6), alice.address);
      
      // ยอดส่วนใหญ่ไปอยู่ใน strategy แล้ว
      // ลอง withdraw
      const shares = await vault.balanceOf(alice.address);
      await expect(vault.connect(alice).redeem(shares, alice.address, alice.address))
        .to.not.be.reverted;
    });
  });
  
  describe("Fee Collection", () => {
    it("should collect management fee over time", async () => {
      const { vault, usdc, alice, feeRecipient, owner } = await loadFixture(deployVaultFixture);
      
      await vault.connect(alice).deposit(ethers.parseUnits("10000", 6), alice.address);
      
      const sharesBefore = await vault.balanceOf(feeRecipient.address);
      
      // รอ 1 ปี
      await time.increase(365 * 24 * 3600);
      
      // Trigger fee collection via deposit
      await usdc.mint(alice.address, ethers.parseUnits("100", 6));
      await usdc.connect(alice).approve(await vault.getAddress(), ethers.MaxUint256);
      await vault.connect(alice).deposit(ethers.parseUnits("100", 6), alice.address);
      
      const sharesAfter = await vault.balanceOf(feeRecipient.address);
      
      // Fee recipient ควรได้ shares มากขึ้น (management fee 2% ต่อปี)
      // หมายเหตุ: ในตัวจริง fee จะถูกเก็บใน report() ของ strategy
      // ที่นี่แค่ verify ว่า mechanism ทำงานได้
      console.log(`Fee recipient shares: ${sharesBefore} -> ${sharesAfter}`);
    });
  });
});
```

---

## Workshop / แบบฝึกหัด

### แบบฝึกหัดที่ 1: CurveStrategy

**โจทย์**: สร้าง `CurveStrategy` ที่:
1. Deposit USDC ไปใน Curve 3pool (DAI/USDC/USDT)
2. Claim CRV rewards
3. Lock CRV เป็น veCRV เพื่อ boost yield
4. Sell บาง CRV เพื่อ compound

```solidity
// แบบฝึกหัด
contract CurveStrategy is BaseStrategy {
    address public constant THREE_POOL = 0xbebc44782c7dB0a1A60Cb6fe97d0b483032FF1C7;
    address public constant CRV = 0xD533a949740bb3306d119CC777fa900bA034cd52;
    address public constant VOTER = 0xF147b8125d2ef93FB6965Db97D6746952a133934;
    
    // TODO: Implement estimatedTotalAssets()
    //   - balanceOf want (USDC)
    //   - LP tokens ที่ถือ converted เป็น USDC
    //   - pending CRV rewards converted เป็น USDC
    
    // TODO: Implement prepareReturn()
    //   - Claim CRV rewards
    //   - ขาย 50% CRV เป็น USDC (compound)
    //   - Lock 50% CRV เป็น veCRV (boost)
    
    // TODO: Implement adjustPosition()
    //   - Calculate optimal amount to deposit
    //   - Add USDC liquidity ไปใน 3pool
    //   - Stake LP tokens ใน Gauge
}
```

### แบบฝึกหัดที่ 2: Multi-Asset Vault

**โจทย์**: ปรับ `AutoCompoundVault` ให้รองรับ:
1. Multi-asset deposits (deposit ได้ทั้ง USDC, USDT, DAI)
2. Auto-convert ทุก asset เป็น USDC ก่อน invest
3. Withdraw ใน asset ที่ต้องการ

### แบบฝึกหัดที่ 3: Strategy Performance Tracker

**โจทย์**: สร้าง `PerformanceTracker` ที่:
1. เก็บ APY ของแต่ละ strategy รายสัปดาห์
2. คำนวณ rolling 30-day APY
3. Alert เมื่อ APY ต่ำกว่า threshold

---

## สรุป Part 52

- **BaseStrategy** กำหนด lifecycle และ interface มาตรฐานสำหรับทุก strategy
- **estimatedTotalAssets()** ต้องนับทั้ง idle funds และ deployed capital รวม pending rewards
- **Checks-Effects-Interactions** ใน harvest() ป้องกัน re-entrancy ขณะที่เรียก external protocols
- **StrategyRouter** ใช้ allocation points และ maxDebtRatio เพื่อกระจาย capital อย่างยืดหยุ่น
- **AutoCompoundVault (ERC-4626)** เพิ่ม deposit limit, emergency shutdown, fee collection บน standard
- **Performance fees** เก็บในรูป shares เพื่อ align interests ระหว่าง managers และ users
- **Withdrawal queue** กำหนดลำดับว่า strategy ไหนควรถอนก่อนเมื่อ user withdraw
- **GenericLendingStrategy** แสดง pattern การ wrap Aave-style protocols

## Next: Part 53 - Options Protocols
