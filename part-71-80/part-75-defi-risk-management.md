# Part 75: DeFi Risk Management

## บทนำ

Risk management เป็นหัวใจของ DeFi protocol ที่แข็งแกร่ง การเข้าใจและจัดการความเสี่ยงอย่างเป็นระบบ ช่วยปกป้องผู้ใช้และโปรโตคอลจากความสูญเสีย บทนี้จะครอบคลุม risk framework ครบวงจร ตั้งแต่การวิเคราะห์ความเสี่ยง, RiskOracle, Stress Testing, Circuit Breaker จนถึง Insurance Fund

---

## 1. Risk Framework

### 1.1 ประเภทความเสี่ยงใน DeFi

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title DeFiRiskFramework
 * @notice Framework สำหรับจัดประเภทและวัดความเสี่ยง DeFi
 *
 * ประเภทความเสี่ยงหลัก:
 *
 * 1. Market Risk (ความเสี่ยงตลาด)
 *    - Price Volatility: ราคา asset เปลี่ยนแปลงรุนแรง
 *    - Correlation Risk: assets หลายตัวตกพร้อมกัน
 *    - Liquidity Risk: ขาย asset ไม่ได้ในราคาที่ต้องการ
 *
 * 2. Liquidity Risk (ความเสี่ยงสภาพคล่อง)
 *    - Pool Depth: pool มีเงินน้อย → slippage สูง
 *    - Bank Run: ผู้ใช้ถอนพร้อมกัน
 *    - Impermanent Loss: LP position ขาดทุน
 *
 * 3. Smart Contract Risk
 *    - Bugs/Exploits: code มีช่องโหว่
 *    - Upgrade Risk: admin key compromise
 *    - Composability Risk: โปรโตคอลที่ต่อกันพังพร้อมกัน
 *
 * 4. Oracle Risk
 *    - Price Manipulation: oracle ถูก manipulate
 *    - Stale Data: ราคาไม่ update ทัน
 *    - Single Point of Failure: oracle เดียว
 *
 * 5. Governance Risk
 *    - Governance Attack: 51% attack บน governance
 *    - Parameter Risk: parameter ถูกเปลี่ยนเป็นอันตราย
 *    - Time Lock Bypass: ข้ามกลไก timelock
 */

/**
 * @notice Risk Score structure
 */
struct RiskScore {
    uint8 marketRisk;       // 0-100: ความเสี่ยงตลาด
    uint8 liquidityRisk;    // 0-100: ความเสี่ยงสภาพคล่อง
    uint8 contractRisk;     // 0-100: ความเสี่ยง smart contract
    uint8 oracleRisk;       // 0-100: ความเสี่ยง oracle
    uint8 governanceRisk;   // 0-100: ความเสี่ยง governance
    uint8 compositeScore;   // 0-100: คะแนนรวม weighted
    uint256 timestamp;      // เวลาที่คำนวณ
}

/**
 * @notice Position health data
 */
struct PositionHealth {
    address user;
    uint256 collateralValue;    // มูลค่า collateral ใน USD
    uint256 debtValue;          // มูลค่าหนี้ ใน USD
    uint256 healthFactor;       // collateral/debt * 100
    uint256 liquidationPrice;   // ราคาที่จะ trigger liquidation
    RiskLevel riskLevel;        // ระดับความเสี่ยง
}

enum RiskLevel {
    SAFE,        // > 150%: ปลอดภัย
    WARNING,     // 120-150%: ใกล้เส้น
    CRITICAL,    // 110-120%: วิกฤต (trigger warning)
    LIQUIDATABLE // < 110%: ถูก liquidate
}
```

### 1.2 Risk Parameters Registry

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {AccessControl} from "@openzeppelin/contracts/access/AccessControl.sol";

/**
 * @title RiskParameterRegistry
 * @notice Registry สำหรับเก็บ risk parameters ของแต่ละ asset
 * @dev Parameters ถูก govern โดย DAO พร้อม timelock
 */
contract RiskParameterRegistry is AccessControl {
    // ================== Roles ==================
    bytes32 public constant RISK_MANAGER = keccak256("RISK_MANAGER");
    bytes32 public constant GUARDIAN = keccak256("GUARDIAN");

    // ================== Types ==================
    struct AssetRiskParams {
        uint256 maxLTV;                 // Maximum Loan-to-Value (bps)
        uint256 liquidationThreshold;   // Liquidation threshold (bps)
        uint256 liquidationBonus;       // Liquidation bonus (bps)
        uint256 supplyCap;              // สูงสุด supply ใน protocol
        uint256 borrowCap;              // สูงสุด borrow ใน protocol
        uint256 reserveFactor;          // เปอร์เซ็นต์ที่เก็บเป็น reserve (bps)
        bool isActive;                  // asset active หรือไม่
        bool isBorrowable;              // กู้ได้หรือไม่
        bool isCollateral;              // ใช้เป็น collateral ได้หรือไม่
        uint256 lastUpdated;
    }

    // ================== State ==================
    mapping(address => AssetRiskParams) public assetParams;
    address[] public supportedAssets;

    // Change tracking สำหรับ audit trail
    struct ParamChange {
        address asset;
        string parameter;
        uint256 oldValue;
        uint256 newValue;
        address changedBy;
        uint256 timestamp;
    }

    ParamChange[] public paramHistory;

    // ================== Events ==================
    event AssetAdded(address indexed asset, AssetRiskParams params);
    event ParamUpdated(
        address indexed asset,
        string parameter,
        uint256 oldValue,
        uint256 newValue
    );
    event AssetDeactivated(address indexed asset);

    // ================== Constructor ==================
    constructor(address admin) {
        _grantRole(DEFAULT_ADMIN_ROLE, admin);
        _grantRole(RISK_MANAGER, admin);
    }

    // ================== Asset Management ==================

    /**
     * @notice เพิ่ม asset ใหม่เข้า registry
     */
    function addAsset(
        address asset,
        uint256 maxLTV,
        uint256 liquidationThreshold,
        uint256 liquidationBonus,
        uint256 supplyCap,
        uint256 borrowCap,
        uint256 reserveFactor,
        bool isBorrowable,
        bool isCollateral
    ) external onlyRole(RISK_MANAGER) {
        require(!assetParams[asset].isActive, "Asset already exists");
        require(maxLTV < liquidationThreshold, "LTV must be < liquidation threshold");
        require(liquidationThreshold <= 9500, "Threshold too high"); // max 95%

        assetParams[asset] = AssetRiskParams({
            maxLTV: maxLTV,
            liquidationThreshold: liquidationThreshold,
            liquidationBonus: liquidationBonus,
            supplyCap: supplyCap,
            borrowCap: borrowCap,
            reserveFactor: reserveFactor,
            isActive: true,
            isBorrowable: isBorrowable,
            isCollateral: isCollateral,
            lastUpdated: block.timestamp
        });

        supportedAssets.push(asset);

        emit AssetAdded(asset, assetParams[asset]);
    }

    /**
     * @notice อัพเดท parameter ของ asset
     */
    function updateLiquidationThreshold(
        address asset,
        uint256 newThreshold
    ) external onlyRole(RISK_MANAGER) {
        uint256 old = assetParams[asset].liquidationThreshold;
        assetParams[asset].liquidationThreshold = newThreshold;
        assetParams[asset].lastUpdated = block.timestamp;

        _recordChange(asset, "liquidationThreshold", old, newThreshold);
        emit ParamUpdated(asset, "liquidationThreshold", old, newThreshold);
    }

    function updateSupplyCap(
        address asset,
        uint256 newCap
    ) external onlyRole(RISK_MANAGER) {
        uint256 old = assetParams[asset].supplyCap;
        assetParams[asset].supplyCap = newCap;
        assetParams[asset].lastUpdated = block.timestamp;

        _recordChange(asset, "supplyCap", old, newCap);
        emit ParamUpdated(asset, "supplyCap", old, newCap);
    }

    /**
     * @notice ปิด asset ฉุกเฉิน (Guardian สามารถทำได้)
     */
    function deactivateAsset(address asset) external {
        require(
            hasRole(RISK_MANAGER, msg.sender) || hasRole(GUARDIAN, msg.sender),
            "Unauthorized"
        );
        assetParams[asset].isActive = false;
        assetParams[asset].lastUpdated = block.timestamp;
        emit AssetDeactivated(asset);
    }

    // ================== Internal ==================

    function _recordChange(
        address asset,
        string memory parameter,
        uint256 oldValue,
        uint256 newValue
    ) internal {
        paramHistory.push(ParamChange({
            asset: asset,
            parameter: parameter,
            oldValue: oldValue,
            newValue: newValue,
            changedBy: msg.sender,
            timestamp: block.timestamp
        }));
    }

    // ================== View Functions ==================

    function getAssetParams(address asset)
        external
        view
        returns (AssetRiskParams memory)
    {
        return assetParams[asset];
    }

    function getSupportedAssets() external view returns (address[] memory) {
        return supportedAssets;
    }

    function getParamHistoryLength() external view returns (uint256) {
        return paramHistory.length;
    }
}
```

---

## 2. RiskOracle

### 2.1 Health Monitoring System

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title RiskOracle
 * @notice ระบบ monitoring สุขภาพของ positions ทุก position
 * @dev คำนวณ health factor และ trigger warnings ที่ 120% ก่อน liquidation ที่ 110%
 */
contract RiskOracle is Ownable {
    // ================== Constants ==================
    uint256 public constant WARNING_THRESHOLD = 120;     // Warning ที่ 120%
    uint256 public constant LIQUIDATION_THRESHOLD = 110; // Liquidation ที่ 110%
    uint256 public constant SAFE_THRESHOLD = 150;        // Safe ที่ 150%
    uint256 public constant PRECISION = 1e18;

    // ================== Types ==================
    struct UserPosition {
        address collateralToken;
        address debtToken;
        uint256 collateralAmount;
        uint256 debtAmount;
        uint256 collateralDecimals;
        bool active;
    }

    struct HealthReport {
        uint256 healthFactor;       // collateral/debt * 100
        RiskLevel riskLevel;
        uint256 liquidationPrice;   // ราคา collateral ที่จะ trigger liquidation
        uint256 warningPrice;       // ราคาที่จะ trigger warning
        uint256 timestamp;
    }

    // ================== State Variables ==================
    mapping(address => UserPosition) public positions;
    mapping(address => HealthReport) public lastHealthReport;

    address[] public monitoredUsers;
    mapping(address => bool) public isMonitored;

    // Price feeds
    mapping(address => address) public priceFeeds; // token → Chainlink feed

    // Warning registry
    mapping(address => bool) public activeWarnings;
    address[] public usersWithWarnings;

    // ================== Events ==================
    event PositionRegistered(address indexed user, address collateral, address debt);
    event HealthReportUpdated(
        address indexed user,
        uint256 healthFactor,
        RiskLevel riskLevel
    );
    event WarningTriggered(
        address indexed user,
        uint256 healthFactor,
        uint256 liquidationPrice
    );
    event WarningCleared(address indexed user, uint256 newHealthFactor);
    event LiquidationAlert(
        address indexed user,
        uint256 healthFactor,
        uint256 collateralValue,
        uint256 debtValue
    );

    // ================== Errors ==================
    error PriceFeedNotSet(address token);
    error PositionNotRegistered(address user);
    error InvalidPosition();

    // ================== Constructor ==================
    constructor(address _owner) Ownable(_owner) {}

    // ================== Position Management ==================

    /**
     * @notice ลงทะเบียน position ให้ RiskOracle monitor
     * @param user address ของผู้ใช้
     * @param collateralToken collateral token
     * @param debtToken debt token
     * @param collateralDecimals decimals ของ collateral
     */
    function registerPosition(
        address user,
        address collateralToken,
        address debtToken,
        uint256 collateralDecimals
    ) external {
        // เฉพาะ vault contract เรียกได้ใน production
        positions[user] = UserPosition({
            collateralToken: collateralToken,
            debtToken: debtToken,
            collateralAmount: 0,
            debtAmount: 0,
            collateralDecimals: collateralDecimals,
            active: true
        });

        if (!isMonitored[user]) {
            isMonitored[user] = true;
            monitoredUsers.push(user);
        }

        emit PositionRegistered(user, collateralToken, debtToken);
    }

    /**
     * @notice อัพเดทข้อมูล position
     * @dev เรียกโดย vault เมื่อ position เปลี่ยน
     */
    function updatePosition(
        address user,
        uint256 newCollateral,
        uint256 newDebt
    ) external {
        UserPosition storage pos = positions[user];
        if (!pos.active) revert PositionNotRegistered(user);

        pos.collateralAmount = newCollateral;
        pos.debtAmount = newDebt;

        // Re-check health
        _assessHealth(user);
    }

    // ================== Health Assessment ==================

    /**
     * @notice ตรวจสอบสุขภาพของ position เดียว
     * @param user address ของผู้ใช้
     * @return report HealthReport ล่าสุด
     */
    function assessPositionHealth(address user)
        external
        returns (HealthReport memory report)
    {
        return _assessHealth(user);
    }

    /**
     * @notice ตรวจสอบ positions ทั้งหมด (batch)
     * @dev ควรเรียก off-chain หรือผ่าน keeper
     * @param startIdx เริ่มต้นที่ index ใด
     * @param batchSize จำนวน positions ที่จะตรวจ
     */
    function batchAssessHealth(uint256 startIdx, uint256 batchSize)
        external
        returns (address[] memory atRisk)
    {
        uint256 endIdx = startIdx + batchSize;
        if (endIdx > monitoredUsers.length) {
            endIdx = monitoredUsers.length;
        }

        address[] memory tempAtRisk = new address[](endIdx - startIdx);
        uint256 count;

        for (uint256 i = startIdx; i < endIdx; i++) {
            address user = monitoredUsers[i];
            HealthReport memory report = _assessHealth(user);

            if (report.riskLevel == RiskLevel.CRITICAL
                || report.riskLevel == RiskLevel.LIQUIDATABLE)
            {
                tempAtRisk[count++] = user;
            }
        }

        // Trim array
        atRisk = new address[](count);
        for (uint256 i = 0; i < count; i++) {
            atRisk[i] = tempAtRisk[i];
        }
    }

    /**
     * @notice Internal: คำนวณ health และ trigger events
     */
    function _assessHealth(address user)
        internal
        returns (HealthReport memory report)
    {
        UserPosition memory pos = positions[user];
        if (!pos.active) revert PositionNotRegistered(user);

        if (pos.debtAmount == 0) {
            report = HealthReport({
                healthFactor: type(uint256).max,
                riskLevel: RiskLevel.SAFE,
                liquidationPrice: 0,
                warningPrice: 0,
                timestamp: block.timestamp
            });
            lastHealthReport[user] = report;
            return report;
        }

        // ดึงราคา collateral
        uint256 collateralPrice = _getPrice(pos.collateralToken);

        // คำนวณ collateral value (normalize decimals)
        uint256 collateralValue = (pos.collateralAmount * collateralPrice)
            / (10 ** pos.collateralDecimals);

        // health factor = collateralValue * 100 / debt
        uint256 healthFactor = (collateralValue * 100) / pos.debtAmount;

        // คำนวณ liquidation price
        // liquidation occurs when: collateral * price / decimals * 100 / debt < 110
        // → price < debt * 110 * decimals / (collateral * 100)
        uint256 liquidationPrice = (pos.debtAmount * LIQUIDATION_THRESHOLD
            * (10 ** pos.collateralDecimals))
            / (pos.collateralAmount * 100);

        uint256 warningPrice = (pos.debtAmount * WARNING_THRESHOLD
            * (10 ** pos.collateralDecimals))
            / (pos.collateralAmount * 100);

        // กำหนด risk level
        RiskLevel riskLevel;
        if (healthFactor >= SAFE_THRESHOLD) {
            riskLevel = RiskLevel.SAFE;
        } else if (healthFactor >= WARNING_THRESHOLD) {
            riskLevel = RiskLevel.WARNING;
        } else if (healthFactor >= LIQUIDATION_THRESHOLD) {
            riskLevel = RiskLevel.CRITICAL;
        } else {
            riskLevel = RiskLevel.LIQUIDATABLE;
        }

        report = HealthReport({
            healthFactor: healthFactor,
            riskLevel: riskLevel,
            liquidationPrice: liquidationPrice,
            warningPrice: warningPrice,
            timestamp: block.timestamp
        });

        lastHealthReport[user] = report;

        emit HealthReportUpdated(user, healthFactor, riskLevel);

        // Trigger warnings
        if (riskLevel == RiskLevel.CRITICAL && !activeWarnings[user]) {
            activeWarnings[user] = true;
            usersWithWarnings.push(user);
            emit WarningTriggered(user, healthFactor, liquidationPrice);
        } else if (riskLevel == RiskLevel.LIQUIDATABLE) {
            emit LiquidationAlert(user, healthFactor, collateralValue, pos.debtAmount);
        } else if (riskLevel == RiskLevel.SAFE && activeWarnings[user]) {
            activeWarnings[user] = false;
            emit WarningCleared(user, healthFactor);
        }

        return report;
    }

    /**
     * @notice ดึงราคาจาก Chainlink oracle พร้อม staleness check
     */
    function _getPrice(address token) internal view returns (uint256) {
        address feed = priceFeeds[token];
        if (feed == address(0)) revert PriceFeedNotSet(token);

        AggregatorV3Interface chainlink = AggregatorV3Interface(feed);
        (
            uint80 roundId,
            int256 answer,
            ,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = chainlink.latestRoundData();

        require(answeredInRound >= roundId, "Stale price");
        require(block.timestamp - updatedAt <= 3600, "Price too old");
        require(answer > 0, "Invalid price");

        return uint256(answer);
    }

    // ================== Admin ==================

    function setPriceFeed(address token, address feed) external onlyOwner {
        priceFeeds[token] = feed;
    }

    // ================== View Functions ==================

    function getHealthReport(address user)
        external
        view
        returns (HealthReport memory)
    {
        return lastHealthReport[user];
    }

    function getUsersWithWarnings() external view returns (address[] memory) {
        return usersWithWarnings;
    }

    function getMonitoredUsersCount() external view returns (uint256) {
        return monitoredUsers.length;
    }
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

---

## 3. Stress Testing

### 3.1 หลักการ Stress Testing

Stress testing ช่วยตรวจสอบว่าโปรโตคอลสามารถรองรับสถานการณ์เลวร้ายได้หรือไม่:
- **Price Crash**: จำลองราคา crash -50%
- **Correlation Shock**: assets หลายตัว crash พร้อมกัน  
- **Liquidity Crisis**: liquidity หดตัว 80%
- **Mass Liquidation**: จำลอง cascade liquidations

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title StressTest
 * @notice Contract สำหรับ simulate stress scenarios
 * @dev จำลอง -50% price crash และตรวจสอบว่า positions liquidatable
 */
contract StressTest {
    // ================== Types ==================
    struct StressScenario {
        string name;
        int256 priceChangePercent;  // เปอร์เซ็นต์การเปลี่ยนราคา (negative = crash)
        uint256 liquidityReduction; // เปอร์เซ็นต์การลด liquidity
        bool correlatedCrash;       // crash หลาย assets พร้อมกัน
    }

    struct StressResult {
        uint256 totalPositions;
        uint256 liquidatableCount;
        uint256 totalDebtAtRisk;
        uint256 totalCollateralAtRisk;
        uint256 estimatedBadDebt;   // หนี้ที่จะ cover ด้วย collateral ไม่ได้
        uint256 protocolSolvencyRatio; // สัดส่วนที่ cover ได้
        bool systemSolvent;
    }

    // ================== Predefined Scenarios ==================

    // Scenario 1: 50% price crash (ตาม crypto bear market)
    StressScenario public priceCrash50 = StressScenario({
        name: "50% Price Crash",
        priceChangePercent: -50,
        liquidityReduction: 0,
        correlatedCrash: false
    });

    // Scenario 2: Black Thursday (ETH crash -40% in 1 day)
    StressScenario public blackThursday = StressScenario({
        name: "Black Thursday",
        priceChangePercent: -40,
        liquidityReduction: 80,
        correlatedCrash: true
    });

    // Scenario 3: Extreme crash (เช่น LUNA-style)
    StressScenario public extremeCrash = StressScenario({
        name: "Extreme Crash",
        priceChangePercent: -90,
        liquidityReduction: 95,
        correlatedCrash: true
    });

    // ================== Core Stress Test ==================

    /**
     * @notice จำลอง stress scenario บน positions ทั้งหมด
     * @param scenario สถานการณ์ที่จะทดสอบ
     * @param users รายชื่อ users ที่จะทดสอบ
     * @param collaterals จำนวน collateral แต่ละ user
     * @param debts จำนวนหนี้แต่ละ user
     * @param currentPrice ราคา collateral ปัจจุบัน
     * @param collateralDecimals decimals ของ collateral
     * @return result ผลลัพธ์การ stress test
     */
    function runStressTest(
        StressScenario memory scenario,
        address[] memory users,
        uint256[] memory collaterals,
        uint256[] memory debts,
        uint256 currentPrice,
        uint256 collateralDecimals
    ) external pure returns (StressResult memory result) {
        require(users.length == collaterals.length, "Array length mismatch");
        require(users.length == debts.length, "Array length mismatch");

        // คำนวณราคาหลัง stress
        uint256 stressedPrice = _applyPriceChange(
            currentPrice,
            scenario.priceChangePercent
        );

        result.totalPositions = users.length;
        uint256 totalCollateralValue;
        uint256 totalDebt;

        for (uint256 i = 0; i < users.length; i++) {
            if (debts[i] == 0) continue;

            uint256 collateralValue = (collaterals[i] * stressedPrice)
                / (10 ** collateralDecimals);

            uint256 healthFactor = (collateralValue * 100) / debts[i];
            totalCollateralValue += collateralValue;
            totalDebt += debts[i];

            // ถ้า health factor < 110% → liquidatable
            if (healthFactor < 110) {
                result.liquidatableCount++;
                result.totalDebtAtRisk += debts[i];
                result.totalCollateralAtRisk += collateralValue;

                // คำนวณ bad debt (หนี้ที่เกิน collateral)
                if (debts[i] > collateralValue) {
                    result.estimatedBadDebt += debts[i] - collateralValue;
                }
            }
        }

        // คำนวณ solvency ratio
        if (result.totalDebtAtRisk > 0) {
            result.protocolSolvencyRatio = (result.totalCollateralAtRisk * 100)
                / result.totalDebtAtRisk;
        } else {
            result.protocolSolvencyRatio = 100;
        }

        result.systemSolvent = result.estimatedBadDebt == 0;

        return result;
    }

    /**
     * @notice ทดสอบ 50% crash scenario โดยเฉพาะ
     */
    function testPriceCrash50(
        address[] memory users,
        uint256[] memory collaterals,
        uint256[] memory debts,
        uint256 currentPrice,
        uint256 collateralDecimals
    ) external pure returns (StressResult memory) {
        StressScenario memory scenario = StressScenario({
            name: "50% Price Crash",
            priceChangePercent: -50,
            liquidityReduction: 0,
            correlatedCrash: false
        });

        return StressTest(address(this)).runStressTest(
            scenario,
            users,
            collaterals,
            debts,
            currentPrice,
            collateralDecimals
        );
    }

    /**
     * @notice คำนวณราคาหลัง price change
     */
    function _applyPriceChange(
        uint256 price,
        int256 changePercent
    ) internal pure returns (uint256) {
        if (changePercent >= 0) {
            return price + (price * uint256(changePercent)) / 100;
        } else {
            uint256 decrease = (price * uint256(-changePercent)) / 100;
            return price > decrease ? price - decrease : 0;
        }
    }

    /**
     * @notice คำนวณว่าต้องมี insurance fund เท่าไรจึงจะ cover bad debt
     */
    function calculateRequiredInsuranceFund(
        address[] memory users,
        uint256[] memory collaterals,
        uint256[] memory debts,
        uint256 currentPrice,
        uint256 collateralDecimals,
        int256 worstCaseChangePercent
    ) external pure returns (uint256 requiredFund) {
        StressScenario memory worstCase = StressScenario({
            name: "Worst Case",
            priceChangePercent: worstCaseChangePercent,
            liquidityReduction: 100,
            correlatedCrash: true
        });

        StressResult memory result = StressTest(address(0)).runStressTest(
            worstCase,
            users,
            collaterals,
            debts,
            currentPrice,
            collateralDecimals
        );

        // ต้องมี insurance เท่ากับ bad debt + 20% buffer
        requiredFund = result.estimatedBadDebt * 120 / 100;
    }

    // ================== Simulation Helpers ==================

    /**
     * @notice จำลองผลลัพธ์ถ้า liquidate cascades
     * @dev แต่ละรอบ liquidation อาจกดราคาลงอีก ทำให้ cascade
     */
    function simulateCascadeLiquidation(
        address[] memory users,
        uint256[] memory collaterals,
        uint256[] memory debts,
        uint256 initialPrice,
        uint256 collateralDecimals,
        uint256 priceImpactPerLiquidation // bps ราคาตกต่อ liquidation
    ) external pure returns (
        uint256 totalRounds,
        uint256 finalPrice,
        uint256 totalBadDebt
    ) {
        uint256 currentPrice = initialPrice;
        bool[] memory liquidated = new bool[](users.length);
        totalRounds = 0;
        totalBadDebt = 0;

        bool anyLiquidated = true;
        while (anyLiquidated && totalRounds < 100) { // max 100 rounds
            anyLiquidated = false;
            totalRounds++;

            for (uint256 i = 0; i < users.length; i++) {
                if (liquidated[i] || debts[i] == 0) continue;

                uint256 colValue = (collaterals[i] * currentPrice)
                    / (10 ** collateralDecimals);
                uint256 hf = (colValue * 100) / debts[i];

                if (hf < 110) {
                    liquidated[i] = true;
                    anyLiquidated = true;

                    // คำนวณ bad debt ถ้า collateral < debt
                    if (colValue < debts[i]) {
                        totalBadDebt += debts[i] - colValue;
                    }

                    // Price impact จาก liquidation
                    uint256 priceImpact = (currentPrice * priceImpactPerLiquidation) / 10000;
                    currentPrice = currentPrice > priceImpact
                        ? currentPrice - priceImpact
                        : 0;
                }
            }
        }

        finalPrice = currentPrice;
    }
}
```

---

## 4. Circuit Breaker

### 4.1 หลักการ Circuit Breaker

Circuit breaker หยุดโปรโตคอลอัตโนมัติเมื่อพบสัญญาณอันตราย:
- **TVL Drop**: TVL ลดลง > 10% ใน 1 ชั่วโมง
- **Unusual Activity**: ปริมาณ withdrawal สูงผิดปกติ
- **Price Deviation**: ราคา oracle เบี่ยงจาก TWAP มากเกิน

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {AccessControl} from "@openzeppelin/contracts/access/AccessControl.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title CircuitBreaker
 * @notice ระบบ auto-pause เมื่อ TVL ลดลง > 10% ใน 1 ชั่วโมง
 * @dev ต้อง guardian enable ใหม่หลัง circuit breaker trigger
 */
contract CircuitBreaker is AccessControl, ReentrancyGuard {
    // ================== Roles ==================
    bytes32 public constant GUARDIAN_ROLE = keccak256("GUARDIAN_ROLE");
    bytes32 public constant REPORTER_ROLE = keccak256("REPORTER_ROLE");

    // ================== Constants ==================
    uint256 public constant TVL_DROP_THRESHOLD = 1000;   // 10% = 1000 bps
    uint256 public constant WINDOW_DURATION = 1 hours;
    uint256 public constant BASIS_POINTS = 10000;

    // Price deviation threshold: 5% from TWAP
    uint256 public constant PRICE_DEVIATION_THRESHOLD = 500;

    // ================== Types ==================
    struct TVLSnapshot {
        uint256 tvl;
        uint256 timestamp;
    }

    struct CircuitBreakerState {
        bool triggered;
        string reason;
        uint256 triggeredAt;
        address triggeredBy;
        uint256 tvlAtTrigger;
    }

    // ================== State Variables ==================
    CircuitBreakerState public state;

    // TVL history: circular buffer ของ snapshots ทุก 5 นาที
    TVLSnapshot[12] public tvlHistory; // 12 slots = 1 ชั่วโมง
    uint256 public historyHead;
    uint256 public currentTVL;

    // Withdrawal tracking
    mapping(uint256 => uint256) public hourlyWithdrawals; // hour → amount
    uint256 public withdrawalThreshold; // max withdrawal per hour

    // Guardian re-enable requirement
    uint256 public pauseCount;
    mapping(uint256 => address) public pauseResolvers;

    // ================== Events ==================
    event CircuitBreakerTriggered(
        string reason,
        uint256 tvlBefore,
        uint256 tvlAfter,
        address triggeredBy
    );
    event CircuitBreakerReset(address guardian, uint256 pauseCount);
    event TVLUpdated(uint256 oldTVL, uint256 newTVL);
    event WithdrawalThresholdBreached(
        uint256 amount,
        uint256 threshold,
        uint256 hour
    );

    // ================== Errors ==================
    error ProtocolPaused(string reason, uint256 pausedAt);
    error NotTriggered();
    error WithdrawalThresholdExceeded(uint256 attempted, uint256 remaining);

    // ================== Modifiers ==================
    modifier whenOperational() {
        if (state.triggered) {
            revert ProtocolPaused(state.reason, state.triggeredAt);
        }
        _;
    }

    // ================== Constructor ==================
    constructor(
        address admin,
        address guardian,
        uint256 initialTVL,
        uint256 _withdrawalThreshold
    ) {
        _grantRole(DEFAULT_ADMIN_ROLE, admin);
        _grantRole(GUARDIAN_ROLE, guardian);
        _grantRole(REPORTER_ROLE, admin);

        currentTVL = initialTVL;
        withdrawalThreshold = _withdrawalThreshold;

        // Initialize TVL history
        for (uint256 i = 0; i < 12; i++) {
            tvlHistory[i] = TVLSnapshot({
                tvl: initialTVL,
                timestamp: block.timestamp - (11 - i) * 5 minutes
            });
        }
    }

    // ================== TVL Monitoring ==================

    /**
     * @notice อัพเดท TVL และตรวจสอบ circuit breaker
     * @param newTVL TVL ใหม่
     */
    function updateTVL(uint256 newTVL)
        external
        onlyRole(REPORTER_ROLE)
    {
        uint256 oldTVL = currentTVL;
        currentTVL = newTVL;

        // บันทึก snapshot
        uint256 slotIndex = historyHead % 12;
        tvlHistory[slotIndex] = TVLSnapshot({
            tvl: newTVL,
            timestamp: block.timestamp
        });
        historyHead++;

        emit TVLUpdated(oldTVL, newTVL);

        // ตรวจสอบ TVL drop ใน 1 ชั่วโมงที่ผ่านมา
        _checkTVLDrop(newTVL);
    }

    /**
     * @notice ตรวจสอบการถอนเงินว่าเกิน threshold หรือไม่
     * @dev เรียกโดย vault เมื่อมี withdrawal
     */
    function recordWithdrawal(uint256 amount)
        external
        onlyRole(REPORTER_ROLE)
        whenOperational
    {
        uint256 currentHour = block.timestamp / WINDOW_DURATION;
        hourlyWithdrawals[currentHour] += amount;

        uint256 totalThisHour = hourlyWithdrawals[currentHour];

        if (totalThisHour > withdrawalThreshold) {
            emit WithdrawalThresholdBreached(
                amount,
                withdrawalThreshold,
                currentHour
            );
            _triggerCircuitBreaker(
                "Withdrawal threshold exceeded",
                msg.sender
            );
        }
    }

    // ================== Automatic Checks ==================

    /**
     * @notice ตรวจสอบ TVL drop > 10% ใน 1 ชั่วโมง
     */
    function _checkTVLDrop(uint256 currentTvl) internal {
        if (state.triggered) return;

        // หา TVL เมื่อ 1 ชั่วโมงที่แล้ว
        uint256 oneHourAgo = block.timestamp - WINDOW_DURATION;
        uint256 oldestValidTVL = _getTVLAt(oneHourAgo);

        if (oldestValidTVL == 0) return;

        // คำนวณ % drop
        if (currentTvl < oldestValidTVL) {
            uint256 dropBps = ((oldestValidTVL - currentTvl) * BASIS_POINTS)
                / oldestValidTVL;

            if (dropBps > TVL_DROP_THRESHOLD) {
                _triggerCircuitBreaker(
                    string(abi.encodePacked(
                        "TVL dropped >10% in 1 hour"
                    )),
                    address(this)
                );
            }
        }
    }

    /**
     * @notice ดึง TVL ที่เก็บไว้ใกล้ที่สุดกับ timestamp ที่กำหนด
     */
    function _getTVLAt(uint256 targetTimestamp)
        internal
        view
        returns (uint256)
    {
        uint256 closestTVL;
        uint256 closestDiff = type(uint256).max;

        for (uint256 i = 0; i < 12; i++) {
            TVLSnapshot memory snap = tvlHistory[i];
            if (snap.timestamp == 0) continue;

            uint256 diff = snap.timestamp >= targetTimestamp
                ? snap.timestamp - targetTimestamp
                : targetTimestamp - snap.timestamp;

            if (diff < closestDiff) {
                closestDiff = diff;
                closestTVL = snap.tvl;
            }
        }

        return closestTVL;
    }

    /**
     * @notice Trigger circuit breaker
     */
    function _triggerCircuitBreaker(
        string memory reason,
        address triggeredBy
    ) internal {
        if (state.triggered) return; // already triggered

        uint256 tvlBefore = _getTVLAt(block.timestamp - WINDOW_DURATION);

        state = CircuitBreakerState({
            triggered: true,
            reason: reason,
            triggeredAt: block.timestamp,
            triggeredBy: triggeredBy,
            tvlAtTrigger: currentTVL
        });

        emit CircuitBreakerTriggered(reason, tvlBefore, currentTVL, triggeredBy);
    }

    /**
     * @notice Guardian trigger circuit breaker manually
     */
    function manualTrigger(string calldata reason)
        external
        onlyRole(GUARDIAN_ROLE)
    {
        _triggerCircuitBreaker(reason, msg.sender);
    }

    // ================== Guardian Reset ==================

    /**
     * @notice Guardian reset circuit breaker หลังตรวจสอบแล้ว
     * @dev ต้องมี guardian approve เพื่อ re-enable
     */
    function resetCircuitBreaker()
        external
        onlyRole(GUARDIAN_ROLE)
    {
        if (!state.triggered) revert NotTriggered();

        pauseCount++;
        pauseResolvers[pauseCount] = msg.sender;

        state = CircuitBreakerState({
            triggered: false,
            reason: "",
            triggeredAt: 0,
            triggeredBy: address(0),
            tvlAtTrigger: 0
        });

        // Reset hourly withdrawal tracking
        uint256 currentHour = block.timestamp / WINDOW_DURATION;
        delete hourlyWithdrawals[currentHour];

        emit CircuitBreakerReset(msg.sender, pauseCount);
    }

    // ================== View Functions ==================

    function isOperational() external view returns (bool) {
        return !state.triggered;
    }

    function getState() external view returns (CircuitBreakerState memory) {
        return state;
    }

    function getCurrentHourWithdrawals() external view returns (uint256) {
        return hourlyWithdrawals[block.timestamp / WINDOW_DURATION];
    }

    function getRemainingWithdrawalCapacity()
        external
        view
        returns (uint256)
    {
        uint256 currentHour = block.timestamp / WINDOW_DURATION;
        uint256 used = hourlyWithdrawals[currentHour];
        return withdrawalThreshold > used ? withdrawalThreshold - used : 0;
    }

    function getTVLHistory()
        external
        view
        returns (TVLSnapshot[12] memory)
    {
        return tvlHistory;
    }
}
```

---

## 5. Insurance Fund

### 5.1 หลักการ Insurance Fund

Insurance Fund ป้องกัน bad debt ที่เกิดจาก liquidation ไม่ครบ:
- **Premium Collection**: เก็บ premium จาก yield
- **Claim Process**: ตรวจสอบและ approve claims
- **DAO Governance**: votes ขนาดใหญ่ต้องผ่าน DAO
- **Reinsurance Pool**: กระจาย risk ไปยัง reinsurer

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title InsuranceFund
 * @notice Insurance Fund สำหรับ cover bad debt ใน DeFi protocol
 * @dev รองรับ premium collection, claim process, DAO vote, reinsurance
 */
contract InsuranceFund is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ================== Constants ==================
    uint256 public constant LARGE_CLAIM_THRESHOLD = 100_000e18; // > 100k USD ต้อง DAO
    uint256 public constant DAO_VOTING_PERIOD = 3 days;
    uint256 public constant CLAIM_PROCESSING_DELAY = 24 hours;
    uint256 public constant REINSURANCE_TRIGGER = 8000; // 80% fund utilization
    uint256 public constant BASIS_POINTS = 10000;

    // ================== Types ==================
    enum ClaimStatus {
        PENDING,
        APPROVED,
        REJECTED,
        PAID,
        DAO_VOTING
    }

    struct Claim {
        address claimant;
        uint256 amount;
        string reason;
        ClaimStatus status;
        uint256 submittedAt;
        uint256 resolvedAt;
        address resolver;
        uint256 daoVoteId; // ถ้า claim ต้องผ่าน DAO
    }

    struct DAOVote {
        uint256 claimId;
        uint256 votesFor;
        uint256 votesAgainst;
        uint256 endTime;
        bool executed;
        mapping(address => bool) hasVoted;
    }

    struct ReinsuranceAgreement {
        address reinsurer;
        uint256 maxCoverage;        // coverage สูงสุดจาก reinsurer
        uint256 retentionAmount;    // Insurance Fund รับก่อนเท่าไร
        uint256 premiumRate;        // premium % ที่ส่งให้ reinsurer (bps)
        bool active;
    }

    // ================== State Variables ==================
    IERC20 public immutable fundToken; // token ที่ใช้ใน fund (เช่น USDC)

    // Fund balance tracking
    uint256 public fundBalance;
    uint256 public totalPremiumCollected;
    uint256 public totalClaimsPaid;

    // Claim management
    mapping(uint256 => Claim) public claims;
    uint256 public claimCount;
    mapping(uint256 => DAOVote) public daoVotes;
    uint256 public daoVoteCount;

    // Premium sources (protocols ที่ pay premium)
    mapping(address => uint256) public premiumRates; // protocol → bps of yield
    mapping(address => uint256) public lastPremiumPaid;

    // Reinsurance
    ReinsuranceAgreement[] public reinsuranceAgreements;
    uint256 public totalReinsuranceCoverage;

    // DAO token สำหรับ governance votes
    IERC20 public immutable governanceToken;
    uint256 public quorumRequired; // minimum votes needed

    // ================== Events ==================
    event PremiumCollected(
        address indexed protocol,
        uint256 amount,
        uint256 totalBalance
    );
    event ClaimSubmitted(
        uint256 indexed claimId,
        address indexed claimant,
        uint256 amount
    );
    event ClaimApproved(uint256 indexed claimId, address indexed approver);
    event ClaimRejected(uint256 indexed claimId, address indexed rejector, string reason);
    event ClaimPaid(uint256 indexed claimId, address indexed claimant, uint256 amount);
    event DAOVoteStarted(uint256 indexed voteId, uint256 indexed claimId, uint256 endTime);
    event DAOVoteCast(uint256 indexed voteId, address indexed voter, bool inFavor, uint256 weight);
    event DAOVoteExecuted(uint256 indexed voteId, bool passed);
    event ReinsuranceTrigger(uint256 indexed agreementId, uint256 amount);
    event ReinsuranceAdded(address indexed reinsurer, uint256 maxCoverage);

    // ================== Errors ==================
    error InsufficientFunds(uint256 available, uint256 required);
    error ClaimNotFound(uint256 claimId);
    error AlreadyVoted(address voter);
    error VotingEnded(uint256 voteId);
    error ClaimNotInDAOVoting(uint256 claimId);

    // ================== Constructor ==================
    constructor(
        address _fundToken,
        address _governanceToken,
        uint256 _quorumRequired,
        address _owner
    ) Ownable(_owner) {
        fundToken = IERC20(_fundToken);
        governanceToken = IERC20(_governanceToken);
        quorumRequired = _quorumRequired;
    }

    // ================== Premium Collection ==================

    /**
     * @notice รับ premium จาก DeFi protocol
     * @dev Protocol จ่าย % ของ yield เป็น premium
     * @param amount จำนวน premium
     */
    function collectPremium(uint256 amount) external nonReentrant {
        require(premiumRates[msg.sender] > 0, "Protocol not registered");

        fundToken.safeTransferFrom(msg.sender, address(this), amount);
        fundBalance += amount;
        totalPremiumCollected += amount;
        lastPremiumPaid[msg.sender] = block.timestamp;

        emit PremiumCollected(msg.sender, amount, fundBalance);
    }

    /**
     * @notice คำนวณ premium ที่ต้องจ่าย
     * @param protocol address ของ protocol
     * @param yieldGenerated yield ที่ protocol ได้รับในช่วงนั้น
     */
    function calculatePremium(
        address protocol,
        uint256 yieldGenerated
    ) external view returns (uint256) {
        uint256 rate = premiumRates[protocol];
        return (yieldGenerated * rate) / BASIS_POINTS;
    }

    // ================== Claim Process ==================

    /**
     * @notice ยื่น claim สำหรับ bad debt
     * @dev เฉพาะ vault/protocol ที่ register แล้วเรียกได้
     * @param amount จำนวนที่ claim
     * @param reason เหตุผล
     */
    function submitClaim(
        uint256 amount,
        string calldata reason
    ) external nonReentrant returns (uint256 claimId) {
        claimCount++;
        claimId = claimCount;

        ClaimStatus initialStatus;
        uint256 daoVoteId;

        if (amount >= LARGE_CLAIM_THRESHOLD) {
            // Claim ขนาดใหญ่ต้องผ่าน DAO
            initialStatus = ClaimStatus.DAO_VOTING;
            daoVoteId = _createDAOVote(claimId);
        } else {
            initialStatus = ClaimStatus.PENDING;
            daoVoteId = 0;
        }

        claims[claimId] = Claim({
            claimant: msg.sender,
            amount: amount,
            reason: reason,
            status: initialStatus,
            submittedAt: block.timestamp,
            resolvedAt: 0,
            resolver: address(0),
            daoVoteId: daoVoteId
        });

        emit ClaimSubmitted(claimId, msg.sender, amount);
        return claimId;
    }

    /**
     * @notice Approve claim ขนาดเล็ก (เฉพาะ owner)
     * @param claimId ID ของ claim
     */
    function approveClaim(uint256 claimId) external onlyOwner nonReentrant {
        Claim storage claim = claims[claimId];
        require(claim.claimant != address(0), "Claim not found");
        require(claim.status == ClaimStatus.PENDING, "Claim not pending");
        require(claim.amount < LARGE_CLAIM_THRESHOLD, "Must go through DAO");
        require(
            block.timestamp >= claim.submittedAt + CLAIM_PROCESSING_DELAY,
            "Processing delay not met"
        );

        claim.status = ClaimStatus.APPROVED;
        claim.resolvedAt = block.timestamp;
        claim.resolver = msg.sender;

        emit ClaimApproved(claimId, msg.sender);
    }

    /**
     * @notice Reject claim
     */
    function rejectClaim(
        uint256 claimId,
        string calldata reason
    ) external onlyOwner {
        Claim storage claim = claims[claimId];
        require(claim.status == ClaimStatus.PENDING, "Claim not pending");

        claim.status = ClaimStatus.REJECTED;
        claim.resolvedAt = block.timestamp;
        claim.resolver = msg.sender;

        emit ClaimRejected(claimId, msg.sender, reason);
    }

    /**
     * @notice จ่าย claim ที่ approved แล้ว
     * @param claimId ID ของ claim
     */
    function payClaim(uint256 claimId) external nonReentrant {
        Claim storage claim = claims[claimId];
        require(claim.status == ClaimStatus.APPROVED, "Claim not approved");
        require(msg.sender == claim.claimant, "Not claimant");

        uint256 amount = claim.amount;

        // ตรวจสอบ fund มีพอ
        if (fundBalance < amount) {
            // ลอง trigger reinsurance
            uint256 shortfall = amount - fundBalance;
            _triggerReinsurance(shortfall);
        }

        if (fundBalance < amount) {
            // จ่ายเท่าที่มี
            amount = fundBalance;
        }

        claim.status = ClaimStatus.PAID;
        fundBalance -= amount;
        totalClaimsPaid += amount;

        fundToken.safeTransfer(claim.claimant, amount);

        emit ClaimPaid(claimId, claim.claimant, amount);
    }

    // ================== DAO Governance ==================

    /**
     * @notice สร้าง DAO vote สำหรับ large claim
     */
    function _createDAOVote(uint256 claimId)
        internal
        returns (uint256 voteId)
    {
        daoVoteCount++;
        voteId = daoVoteCount;

        DAOVote storage vote = daoVotes[voteId];
        vote.claimId = claimId;
        vote.votesFor = 0;
        vote.votesAgainst = 0;
        vote.endTime = block.timestamp + DAO_VOTING_PERIOD;
        vote.executed = false;

        emit DAOVoteStarted(voteId, claimId, vote.endTime);
        return voteId;
    }

    /**
     * @notice ลงคะแนน DAO vote
     * @param voteId ID ของ vote
     * @param inFavor true = เห็นด้วย, false = ไม่เห็นด้วย
     */
    function castDAOVote(uint256 voteId, bool inFavor) external {
        DAOVote storage vote = daoVotes[voteId];
        require(vote.endTime > 0, "Vote not found");
        if (block.timestamp > vote.endTime) revert VotingEnded(voteId);
        if (vote.hasVoted[msg.sender]) revert AlreadyVoted(msg.sender);

        uint256 weight = governanceToken.balanceOf(msg.sender);
        require(weight > 0, "No governance tokens");

        vote.hasVoted[msg.sender] = true;

        if (inFavor) {
            vote.votesFor += weight;
        } else {
            vote.votesAgainst += weight;
        }

        emit DAOVoteCast(voteId, msg.sender, inFavor, weight);
    }

    /**
     * @notice Execute DAO vote หลัง voting period สิ้นสุด
     * @param voteId ID ของ vote
     */
    function executeDAOVote(uint256 voteId) external {
        DAOVote storage vote = daoVotes[voteId];
        require(!vote.executed, "Already executed");
        require(block.timestamp > vote.endTime, "Voting not ended");

        uint256 totalVotes = vote.votesFor + vote.votesAgainst;
        require(totalVotes >= quorumRequired, "Quorum not reached");

        vote.executed = true;

        bool passed = vote.votesFor > vote.votesAgainst;
        uint256 claimId = vote.claimId;

        if (passed) {
            claims[claimId].status = ClaimStatus.APPROVED;
            claims[claimId].resolvedAt = block.timestamp;
            emit ClaimApproved(claimId, address(this));
        } else {
            claims[claimId].status = ClaimStatus.REJECTED;
            claims[claimId].resolvedAt = block.timestamp;
            emit ClaimRejected(claimId, address(this), "DAO voted against");
        }

        emit DAOVoteExecuted(voteId, passed);
    }

    // ================== Reinsurance Pool ==================

    /**
     * @notice เพิ่ม reinsurance agreement
     * @param reinsurer address ของ reinsurer
     * @param maxCoverage coverage สูงสุด
     * @param retentionAmount จำนวนที่ insurance fund รับก่อน
     * @param premiumRate premium rate ที่ส่งให้ reinsurer (bps)
     */
    function addReinsurance(
        address reinsurer,
        uint256 maxCoverage,
        uint256 retentionAmount,
        uint256 premiumRate
    ) external onlyOwner {
        reinsuranceAgreements.push(ReinsuranceAgreement({
            reinsurer: reinsurer,
            maxCoverage: maxCoverage,
            retentionAmount: retentionAmount,
            premiumRate: premiumRate,
            active: true
        }));

        totalReinsuranceCoverage += maxCoverage;

        emit ReinsuranceAdded(reinsurer, maxCoverage);
    }

    /**
     * @notice Trigger reinsurance เมื่อ fund ใกล้หมด
     * @param shortfall จำนวนที่ขาด
     */
    function _triggerReinsurance(uint256 shortfall) internal {
        for (uint256 i = 0; i < reinsuranceAgreements.length; i++) {
            ReinsuranceAgreement memory agreement = reinsuranceAgreements[i];
            if (!agreement.active) continue;

            // ตรวจสอบว่า shortfall เกิน retention
            if (shortfall > agreement.retentionAmount) {
                uint256 reinsuranceClaim = shortfall - agreement.retentionAmount;
                if (reinsuranceClaim > agreement.maxCoverage) {
                    reinsuranceClaim = agreement.maxCoverage;
                }

                // ขอ funds จาก reinsurer
                try IERC20(fundToken).transferFrom(
                    agreement.reinsurer,
                    address(this),
                    reinsuranceClaim
                ) returns (bool success) {
                    if (success) {
                        fundBalance += reinsuranceClaim;
                        emit ReinsuranceTrigger(i, reinsuranceClaim);
                        break;
                    }
                } catch {
                    // Reinsurer ไม่มี funds
                    continue;
                }
            }
        }
    }

    /**
     * @notice ส่ง premium ให้ reinsurers ตาม agreements
     * @dev เรียกเป็น periodic เช่น monthly
     */
    function distributePremiums() external onlyOwner nonReentrant {
        for (uint256 i = 0; i < reinsuranceAgreements.length; i++) {
            ReinsuranceAgreement memory agreement = reinsuranceAgreements[i];
            if (!agreement.active) continue;

            uint256 premium = (fundBalance * agreement.premiumRate) / BASIS_POINTS;
            if (premium > 0 && fundBalance >= premium) {
                fundBalance -= premium;
                fundToken.safeTransfer(agreement.reinsurer, premium);
            }
        }
    }

    // ================== Admin Functions ==================

    function registerProtocol(address protocol, uint256 premiumRate) external onlyOwner {
        premiumRates[protocol] = premiumRate;
    }

    function updateQuorum(uint256 newQuorum) external onlyOwner {
        quorumRequired = newQuorum;
    }

    function setWithdrawalThreshold(uint256 amount) external onlyOwner {
        // สำหรับ circuit breaker integration
    }

    // Emergency fund injection
    function injectFunds(uint256 amount) external onlyOwner nonReentrant {
        fundToken.safeTransferFrom(msg.sender, address(this), amount);
        fundBalance += amount;
    }

    // ================== View Functions ==================

    function getClaim(uint256 claimId)
        external
        view
        returns (Claim memory)
    {
        return claims[claimId];
    }

    function getFundHealth() external view returns (
        uint256 balance,
        uint256 totalCoverage,
        uint256 utilizationBps,
        bool needsReinsurance
    ) {
        balance = fundBalance;
        totalCoverage = fundBalance + totalReinsuranceCoverage;
        utilizationBps = totalCoverage > 0
            ? (totalClaimsPaid * BASIS_POINTS) / totalCoverage
            : 0;
        needsReinsurance = utilizationBps >= REINSURANCE_TRIGGER;
    }

    function getDAOVote(uint256 voteId)
        external
        view
        returns (
            uint256 claimId,
            uint256 votesFor,
            uint256 votesAgainst,
            uint256 endTime,
            bool executed
        )
    {
        DAOVote storage vote = daoVotes[voteId];
        return (
            vote.claimId,
            vote.votesFor,
            vote.votesAgainst,
            vote.endTime,
            vote.executed
        );
    }

    function getReinsuranceCount() external view returns (uint256) {
        return reinsuranceAgreements.length;
    }
}
```

---

## Workshop: Integrated Risk Management System

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title IntegratedRiskManager
 * @notice Workshop: ระบบ Risk Management ครบวงจรรวมทุก components
 * @dev รวม RiskOracle + CircuitBreaker + InsuranceFund + StressTest
 */
contract IntegratedRiskManager is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ================== Components ==================
    RiskOracle public immutable riskOracle;
    CircuitBreaker public immutable circuitBreaker;
    InsuranceFund public immutable insuranceFund;

    // ================== Constants ==================
    uint256 public constant HEALTH_CHECK_INTERVAL = 5 minutes;
    uint256 public constant STRESS_TEST_INTERVAL = 1 days;
    uint256 public constant TVL_REPORT_INTERVAL = 5 minutes;

    // ================== State ==================
    uint256 public lastHealthCheck;
    uint256 public lastStressTest;
    uint256 public lastTVLReport;

    // Protocol TVL tracking
    uint256 public totalValueLocked;

    // Monitored protocols
    address[] public monitoredProtocols;
    mapping(address => uint256) public protocolTVL;

    // ================== Events ==================
    event RiskCheckCompleted(uint256 atRiskCount, uint256 timestamp);
    event StressTestCompleted(
        uint256 liquidatableCount,
        uint256 estimatedBadDebt,
        bool systemSolvent
    );
    event SystemHealthAlert(string message, uint256 severity);
    event TVLReported(uint256 tvl, uint256 timestamp);

    // ================== Constructor ==================
    constructor(
        address _riskOracle,
        address _circuitBreaker,
        address _insuranceFund,
        address _owner
    ) Ownable(_owner) {
        riskOracle = RiskOracle(_riskOracle);
        circuitBreaker = CircuitBreaker(_circuitBreaker);
        insuranceFund = InsuranceFund(_insuranceFund);
    }

    // ================== Keeper Functions (Automated) ==================

    /**
     * @notice Keeper function: ตรวจสอบ health ของทุก positions
     * @dev ควร call ทุก 5 นาทีโดย keeper bot (เช่น Gelato)
     */
    function performRiskCheck(uint256 startIdx, uint256 batchSize)
        external
    {
        require(
            block.timestamp >= lastHealthCheck + HEALTH_CHECK_INTERVAL,
            "Too frequent"
        );

        lastHealthCheck = block.timestamp;

        // Batch check positions
        address[] memory atRisk = riskOracle.batchAssessHealth(startIdx, batchSize);

        emit RiskCheckCompleted(atRisk.length, block.timestamp);

        // ถ้ามีหลาย positions วิกฤต → emit alert
        if (atRisk.length > 10) {
            emit SystemHealthAlert(
                "Multiple critical positions detected",
                atRisk.length > 50 ? 3 : 2 // severity 2-3
            );
        }
    }

    /**
     * @notice Keeper function: report TVL ให้ CircuitBreaker
     */
    function reportTVL() external {
        require(
            block.timestamp >= lastTVLReport + TVL_REPORT_INTERVAL,
            "Too frequent"
        );

        lastTVLReport = block.timestamp;

        // คำนวณ total TVL จากทุก protocols
        uint256 newTVL;
        for (uint256 i = 0; i < monitoredProtocols.length; i++) {
            newTVL += protocolTVL[monitoredProtocols[i]];
        }

        totalValueLocked = newTVL;

        // Report ให้ CircuitBreaker
        circuitBreaker.updateTVL(newTVL);

        emit TVLReported(newTVL, block.timestamp);
    }

    /**
     * @notice อัพเดท TVL ของ individual protocol
     * @dev เรียกโดย protocol เองเมื่อ TVL เปลี่ยน
     */
    function updateProtocolTVL(uint256 newTVL) external {
        protocolTVL[msg.sender] = newTVL;
    }

    // ================== Risk Reports ==================

    /**
     * @notice สร้าง comprehensive risk report
     * @return report string ของ risk report (simplified)
     */
    function generateRiskReport() external view returns (
        uint256 monitoredPositions,
        uint256 warningCount,
        uint256 tvl,
        bool circuitBreakerActive,
        uint256 insuranceFundBalance
    ) {
        monitoredPositions = riskOracle.getMonitoredUsersCount();
        warningCount = riskOracle.getUsersWithWarnings().length;
        tvl = totalValueLocked;
        circuitBreakerActive = !circuitBreaker.isOperational();
        (insuranceFundBalance,,,) = insuranceFund.getFundHealth();
    }

    // ================== Admin ==================

    function addMonitoredProtocol(address protocol) external onlyOwner {
        monitoredProtocols.push(protocol);
    }

    function emergencyPauseAll(string calldata reason) external onlyOwner {
        circuitBreaker.manualTrigger(reason);
        emit SystemHealthAlert(reason, 5); // severity 5 = critical
    }
}

// ============================================================
// Test Harness สำหรับ Workshop
// ============================================================

/**
 * @title RiskManagementTestHarness
 * @notice Test harness สำหรับทดสอบ risk management system
 */
contract RiskManagementTestHarness {
    StressTest public stressTest;

    constructor() {
        stressTest = new StressTest();
    }

    /**
     * @notice รัน stress test ครบทุก scenarios
     */
    function runAllScenarios(
        address[] memory users,
        uint256[] memory collaterals,
        uint256[] memory debts,
        uint256 currentPrice,
        uint256 collateralDecimals
    ) external view returns (
        StressTest.StressResult memory mild,
        StressTest.StressResult memory moderate,
        StressTest.StressResult memory severe
    ) {
        // Mild: -20%
        StressScenario memory mildScenario = StressScenario({
            name: "Mild -20%",
            priceChangePercent: -20,
            liquidityReduction: 0,
            correlatedCrash: false
        });

        // Moderate: -50%
        StressScenario memory moderateScenario = StressScenario({
            name: "Moderate -50%",
            priceChangePercent: -50,
            liquidityReduction: 30,
            correlatedCrash: true
        });

        // Severe: -80%
        StressScenario memory severeScenario = StressScenario({
            name: "Severe -80%",
            priceChangePercent: -80,
            liquidityReduction: 90,
            correlatedCrash: true
        });

        mild = stressTest.runStressTest(
            mildScenario, users, collaterals, debts, currentPrice, collateralDecimals
        );
        moderate = stressTest.runStressTest(
            moderateScenario, users, collaterals, debts, currentPrice, collateralDecimals
        );
        severe = stressTest.runStressTest(
            severeScenario, users, collaterals, debts, currentPrice, collateralDecimals
        );
    }

    /**
     * @notice ตรวจสอบว่า insurance fund มีพอสำหรับ worst case
     */
    function checkInsuranceSufficiency(
        address[] memory users,
        uint256[] memory collaterals,
        uint256[] memory debts,
        uint256 currentPrice,
        uint256 collateralDecimals,
        uint256 insuranceFundBalance
    ) external view returns (
        uint256 requiredFund,
        bool isSufficient,
        uint256 shortfall
    ) {
        StressScenario memory worstCase = StressScenario({
            name: "Worst Case -80%",
            priceChangePercent: -80,
            liquidityReduction: 95,
            correlatedCrash: true
        });

        StressTest.StressResult memory result = stressTest.runStressTest(
            worstCase, users, collaterals, debts, currentPrice, collateralDecimals
        );

        requiredFund = result.estimatedBadDebt * 120 / 100; // 120% of bad debt
        isSufficient = insuranceFundBalance >= requiredFund;
        shortfall = isSufficient ? 0 : requiredFund - insuranceFundBalance;
    }
}
```

---

## สรุป Part 75

- **Risk Framework**: DeFi มี 5 ประเภทความเสี่ยงหลัก ได้แก่ market risk, liquidity risk, smart contract risk, oracle risk, และ governance risk; แต่ละประเภทมีวิธีวัดและ mitigate ต่างกัน
- **RiskOracle**: ติดตาม health factor ทุก position; warning ที่ 120% ก่อน liquidation ที่ 110%; `batchAssessHealth()` ใช้สำหรับ keeper bots ตรวจสอบทุก positions
- **Stress Testing**: `StressTest.runStressTest()` จำลองสถานการณ์ต่างๆ ตั้งแต่ -20% ถึง -80%; ช่วยคำนวณ `estimatedBadDebt` และ `requiredInsuranceFund`
- **Circuit Breaker**: auto-pause เมื่อ TVL ลด > 10% ใน 1 ชั่วโมง หรือ withdrawal เกิน threshold; ต้อง guardian reset ด้วย `resetCircuitBreaker()` หลังตรวจสอบ
- **InsuranceFund**: เก็บ premium จาก protocol yield; claims < $100k ผ่าน owner; claims ≥ $100k ต้องผ่าน DAO vote 3 วัน; มี reinsurance pool สำรอง

## Next: Part 76 - Layer 2 Optimization
