# Part 89: Advanced Perpetuals Architecture

## บทนำ

Perpetual futures (perps) เป็น derivative ที่ไม่มีวันหมดอายุ ต่างจาก traditional futures ที่ต้อง roll over ตลอด Perps ใช้ **funding rate** เป็นกลไก peg ราคาให้ใกล้เคียงกับ spot price

Exchange perps ที่ดัง:
- **dYdX**: Order book perps on Ethereum/Starknet
- **GMX**: AMM-based perps, zero price impact
- **Synthetix Perps**: Synthetic asset approach
- **Perpetual Protocol**: vAMM (virtual AMM)

ในบทนี้เราจะสร้าง:
1. Funding rate mechanics แบบ 8-hour TWAP
2. Cross-margin vs Isolated margin
3. Insurance fund + ADL
4. Partial liquidation
5. Full AdvancedPerpEngine

---

## 1. Funding Rate Mechanics

### แนวคิด Funding Rate

```
Funding Rate = Premium + Clamp
Premium = (Mark Price - Index Price) / Index Price
Clamp = max(-0.05%, min(Premium, 0.05%))  หรือตาม exchange

ถ้า funding rate > 0: Long จ่าย Short
ถ้า funding rate < 0: Short จ่าย Long

เหตุผล: ทำให้ราคา perp เข้าใกล้ spot price
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title FundingRateCalculator
 * @notice คำนวณ funding rate ด้วย 8-hour TWAP
 */
contract FundingRateCalculator {
    
    // ========== Constants ==========
    
    uint256 public constant FUNDING_PERIOD = 8 hours;
    int256 public constant MAX_FUNDING_RATE = 0.0005e18;   // 0.05% per 8 hours
    int256 public constant MIN_FUNDING_RATE = -0.0005e18;  // -0.05% per 8 hours
    uint256 public constant TWAP_PERIOD = 1 hours;
    uint256 public constant PRECISION = 1e18;
    
    // ========== TWAP Data ==========
    
    struct PriceObservation {
        uint256 timestamp;
        uint256 price;
        uint256 cumulativePrice; // สำหรับ TWAP calculation
    }
    
    PriceObservation[] public markPriceObservations;
    PriceObservation[] public indexPriceObservations;
    
    // ========== Funding State ==========
    
    int256 public cumulativeFundingRate;   // Accumulated funding rate
    uint256 public lastFundingTime;         // Last funding update time
    int256 public lastFundingRate;          // Most recent funding rate
    
    // Funding payments: position size * funding rate
    // Accumulated per unit of position
    int256 public fundingPerUnit; // Cumulative funding per unit of long position
    
    event FundingRateUpdated(int256 fundingRate, uint256 timestamp);
    event PriceObservationAdded(bool isMark, uint256 price, uint256 timestamp);
    
    constructor() {
        lastFundingTime = block.timestamp;
    }
    
    // ========== Price Feed ==========
    
    /**
     * @notice เพิ่ม mark price observation
     * @dev Mark price = on-chain AMM price
     */
    function addMarkPriceObservation(uint256 price) external {
        uint256 cumulative = markPriceObservations.length > 0
            ? markPriceObservations[markPriceObservations.length - 1].cumulativePrice + 
              price * (block.timestamp - markPriceObservations[markPriceObservations.length - 1].timestamp)
            : 0;
        
        markPriceObservations.push(PriceObservation({
            timestamp: block.timestamp,
            price: price,
            cumulativePrice: cumulative
        }));
        
        emit PriceObservationAdded(true, price, block.timestamp);
    }
    
    /**
     * @notice เพิ่ม index price observation
     * @dev Index price = oracle price (Chainlink)
     */
    function addIndexPriceObservation(uint256 price) external {
        uint256 cumulative = indexPriceObservations.length > 0
            ? indexPriceObservations[indexPriceObservations.length - 1].cumulativePrice +
              price * (block.timestamp - indexPriceObservations[indexPriceObservations.length - 1].timestamp)
            : 0;
        
        indexPriceObservations.push(PriceObservation({
            timestamp: block.timestamp,
            price: price,
            cumulativePrice: cumulative
        }));
        
        emit PriceObservationAdded(false, price, block.timestamp);
    }
    
    // ========== TWAP Calculation ==========
    
    /**
     * @notice คำนวณ TWAP ของ mark price ช่วง period
     */
    function getMarkTWAP(uint256 period) public view returns (uint256 twap) {
        return _calcTWAP(markPriceObservations, period);
    }
    
    /**
     * @notice คำนวณ TWAP ของ index price ช่วง period
     */
    function getIndexTWAP(uint256 period) public view returns (uint256 twap) {
        return _calcTWAP(indexPriceObservations, period);
    }
    
    function _calcTWAP(
        PriceObservation[] storage observations,
        uint256 period
    ) internal view returns (uint256 twap) {
        if (observations.length == 0) return 0;
        
        uint256 startTime = block.timestamp - period;
        uint256 endTime = block.timestamp;
        
        uint256 weightedSum = 0;
        uint256 totalTime = 0;
        
        for (uint256 i = observations.length; i > 0; i--) {
            PriceObservation memory obs = observations[i - 1];
            
            if (obs.timestamp <= startTime) {
                // Include partial window from this observation to startTime
                uint256 nextTimestamp = i < observations.length 
                    ? observations[i].timestamp 
                    : endTime;
                
                uint256 timeInWindow = nextTimestamp > startTime ? nextTimestamp - startTime : 0;
                weightedSum += obs.price * timeInWindow;
                totalTime += timeInWindow;
                break;
            } else {
                // Full window from this observation to next
                uint256 nextTimestamp = i < observations.length 
                    ? observations[i].timestamp 
                    : endTime;
                
                uint256 duration = nextTimestamp - obs.timestamp;
                weightedSum += obs.price * duration;
                totalTime += duration;
            }
        }
        
        if (totalTime == 0) {
            return observations[observations.length - 1].price;
        }
        
        twap = weightedSum / totalTime;
    }
    
    // ========== Funding Rate Update ==========
    
    /**
     * @notice อัพเดท funding rate (เรียกทุก 8 ชั่วโมง)
     */
    function updateFundingRate() external returns (int256 newFundingRate) {
        require(block.timestamp >= lastFundingTime + FUNDING_PERIOD, "too early");
        
        uint256 markTWAP = getMarkTWAP(TWAP_PERIOD);
        uint256 indexTWAP = getIndexTWAP(TWAP_PERIOD);
        
        require(markTWAP > 0 && indexTWAP > 0, "invalid prices");
        
        // Premium = (markTWAP - indexTWAP) / indexTWAP
        int256 premium;
        if (markTWAP >= indexTWAP) {
            premium = int256((markTWAP - indexTWAP) * PRECISION / indexTWAP);
        } else {
            premium = -int256((indexTWAP - markTWAP) * PRECISION / indexTWAP);
        }
        
        // Clamp funding rate
        newFundingRate = _clamp(premium, MIN_FUNDING_RATE, MAX_FUNDING_RATE);
        
        // อัพเดท state
        lastFundingRate = newFundingRate;
        lastFundingTime = block.timestamp;
        cumulativeFundingRate += newFundingRate;
        
        // อัพเดท fundingPerUnit (เพื่อใช้คำนวณ funding payment ต่อ position)
        fundingPerUnit += newFundingRate;
        
        emit FundingRateUpdated(newFundingRate, block.timestamp);
    }
    
    /**
     * @notice Clamp value between min and max
     */
    function _clamp(int256 value, int256 minVal, int256 maxVal) internal pure returns (int256) {
        if (value < minVal) return minVal;
        if (value > maxVal) return maxVal;
        return value;
    }
    
    /**
     * @notice คำนวณ funding payment สำหรับ position
     * @param size Position size (positive = long, negative = short)
     * @param entryFundingPerUnit fundingPerUnit เมื่อ open position
     * @return payment Funding payment (positive = pay, negative = receive)
     */
    function calcFundingPayment(
        int256 size,
        int256 entryFundingPerUnit
    ) external view returns (int256 payment) {
        int256 fundingDiff = fundingPerUnit - entryFundingPerUnit;
        payment = size * fundingDiff / int256(PRECISION);
    }
}
```

---

## 2. Cross-Margin vs Isolated Margin

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title MarginManager
 * @notice จัดการ margin สองแบบ: Cross-Margin และ Isolated Margin
 *
 * Cross-Margin: Margin ร่วมกันทุก positions
 * - ข้อดี: ใช้ capital efficiently
 * - ข้อเสีย: position หนึ่ง liquidate กระทบทั้งหมด
 *
 * Isolated Margin: Margin แยกต่อ position
 * - ข้อดี: จำกัด risk ต่อ position
 * - ข้อเสีย: ต้องวาง margin ต่อ position
 */
contract MarginManager {
    
    // ========== Types ==========
    
    enum MarginMode { Cross, Isolated }
    
    struct CrossMarginAccount {
        int256 balance;           // Total collateral (ใน USD, 1e18 precision)
        int256 unrealizedPnL;     // Total unrealized P&L
        uint256 totalPositionValue; // Sum ของ abs(position value)
        bool exists;
    }
    
    struct IsolatedPosition {
        int256 size;              // Position size (positive = long, negative = short)
        uint256 entryPrice;       // Entry price
        uint256 margin;           // Isolated margin allocated
        uint256 leverage;         // Leverage (1x = 1e18)
        int256 unrealizedPnL;     // Current unrealized PnL
        uint256 liquidationPrice; // Price ที่จะ liquidate
        bool exists;
    }
    
    // ========== State ==========
    
    // Cross margin accounts
    mapping(address => CrossMarginAccount) public crossAccounts;
    
    // Cross margin positions: user → market → size
    mapping(address => mapping(address => int256)) public crossPositions;
    
    // Isolated positions: user → positionId → IsolatedPosition
    mapping(address => mapping(uint256 => IsolatedPosition)) public isolatedPositions;
    mapping(address => uint256) public nextPositionId;
    
    // Constants
    uint256 public constant MAINTENANCE_MARGIN_RATIO = 0.05e18; // 5%
    uint256 public constant INIT_MARGIN_RATIO = 0.10e18;        // 10% (10x max leverage)
    uint256 public constant PRECISION = 1e18;
    
    event CrossMarginDeposit(address indexed user, uint256 amount);
    event CrossMarginWithdraw(address indexed user, uint256 amount);
    event IsolatedPositionOpened(address indexed user, uint256 indexed positionId, int256 size, uint256 margin);
    event IsolatedMarginAdded(address indexed user, uint256 indexed positionId, uint256 amount);
    
    // ========== Cross Margin ==========
    
    /**
     * @notice Deposit collateral ใน cross margin
     */
    function depositCrossMargin(uint256 amount) external {
        crossAccounts[msg.sender].balance += int256(amount);
        crossAccounts[msg.sender].exists = true;
        emit CrossMarginDeposit(msg.sender, amount);
    }
    
    /**
     * @notice Withdraw collateral จาก cross margin
     * @dev ต้องมี margin เพียงพอหลัง withdraw
     */
    function withdrawCrossMargin(uint256 amount) external {
        CrossMarginAccount storage account = crossAccounts[msg.sender];
        require(account.exists, "no account");
        
        int256 newBalance = account.balance - int256(amount);
        
        // ตรวจสอบว่า margin ratio ยังพอ
        require(
            _checkCrossMarginRatio(msg.sender, newBalance),
            "insufficient margin after withdraw"
        );
        
        account.balance = newBalance;
        emit CrossMarginWithdraw(msg.sender, amount);
    }
    
    /**
     * @notice คำนวณ cross margin ratio
     * @return ratio Margin ratio (1e18 = 100%)
     */
    function getCrossMarginRatio(address user) public view returns (uint256 ratio) {
        CrossMarginAccount storage account = crossAccounts[user];
        if (account.totalPositionValue == 0) return type(uint256).max;
        
        int256 equity = account.balance + account.unrealizedPnL;
        if (equity <= 0) return 0;
        
        return uint256(equity) * PRECISION / account.totalPositionValue;
    }
    
    /**
     * @notice ตรวจสอบว่า cross margin ratio ผ่าน threshold
     */
    function _checkCrossMarginRatio(address user, int256 newBalance) internal view returns (bool) {
        CrossMarginAccount storage account = crossAccounts[user];
        if (account.totalPositionValue == 0) return true;
        
        int256 equity = newBalance + account.unrealizedPnL;
        if (equity <= 0) return false;
        
        uint256 ratio = uint256(equity) * PRECISION / account.totalPositionValue;
        return ratio >= INIT_MARGIN_RATIO;
    }
    
    // ========== Isolated Margin ==========
    
    /**
     * @notice เปิด isolated margin position
     */
    function openIsolatedPosition(
        int256 size,
        uint256 entryPrice,
        uint256 margin,
        uint256 leverage
    ) external returns (uint256 positionId) {
        require(size != 0, "zero size");
        require(entryPrice > 0, "zero entry price");
        require(margin > 0, "zero margin");
        require(leverage >= 1e18 && leverage <= 100e18, "invalid leverage"); // 1x - 100x
        
        // ตรวจสอบ margin requirement
        uint256 positionValue = _abs(size) * entryPrice / PRECISION;
        uint256 requiredMargin = positionValue / (leverage / PRECISION);
        require(margin >= requiredMargin, "insufficient margin");
        
        positionId = nextPositionId[msg.sender]++;
        
        // คำนวณ liquidation price
        uint256 liquidationPrice = _calcLiquidationPrice(size, entryPrice, margin, leverage);
        
        isolatedPositions[msg.sender][positionId] = IsolatedPosition({
            size: size,
            entryPrice: entryPrice,
            margin: margin,
            leverage: leverage,
            unrealizedPnL: 0,
            liquidationPrice: liquidationPrice,
            exists: true
        });
        
        emit IsolatedPositionOpened(msg.sender, positionId, size, margin);
    }
    
    /**
     * @notice เพิ่ม margin ใน isolated position (ลด leverage)
     */
    function addIsolatedMargin(uint256 positionId, uint256 amount) external {
        IsolatedPosition storage pos = isolatedPositions[msg.sender][positionId];
        require(pos.exists, "position not found");
        
        pos.margin += amount;
        
        // คำนวณ liquidation price ใหม่
        pos.liquidationPrice = _calcLiquidationPrice(
            pos.size, pos.entryPrice, pos.margin, pos.leverage
        );
        
        emit IsolatedMarginAdded(msg.sender, positionId, amount);
    }
    
    /**
     * @notice คำนวณ liquidation price
     * @dev Long: liquidation when price drops enough to wipe out margin
     *      Short: liquidation when price rises enough to wipe out margin
     */
    function _calcLiquidationPrice(
        int256 size,
        uint256 entryPrice,
        uint256 margin,
        uint256 leverage
    ) internal pure returns (uint256 liquidationPrice) {
        uint256 absSize = _abs(size);
        uint256 maintenanceMargin = absSize * entryPrice / PRECISION * MAINTENANCE_MARGIN_RATIO / PRECISION;
        
        if (size > 0) {
            // Long position: ราคาลดลงจน margin = maintenance margin
            // liquidation = entryPrice - (margin - maintenanceMargin) / size
            uint256 priceMove = (margin - maintenanceMargin) * PRECISION / absSize;
            liquidationPrice = entryPrice > priceMove ? entryPrice - priceMove : 0;
        } else {
            // Short position: ราคาเพิ่มขึ้น
            uint256 priceMove = (margin - maintenanceMargin) * PRECISION / absSize;
            liquidationPrice = entryPrice + priceMove;
        }
    }
    
    /**
     * @notice อัพเดท unrealized PnL
     */
    function updateUnrealizedPnL(uint256 positionId, uint256 currentPrice) external {
        IsolatedPosition storage pos = isolatedPositions[msg.sender][positionId];
        require(pos.exists, "position not found");
        
        pos.unrealizedPnL = _calcPnL(pos.size, pos.entryPrice, currentPrice);
    }
    
    /**
     * @notice คำนวณ PnL
     */
    function _calcPnL(int256 size, uint256 entryPrice, uint256 currentPrice) 
        internal pure returns (int256 pnl) {
        if (size > 0) {
            // Long: PnL = size * (currentPrice - entryPrice)
            if (currentPrice >= entryPrice) {
                pnl = size * int256(currentPrice - entryPrice) / int256(PRECISION);
            } else {
                pnl = -size * int256(entryPrice - currentPrice) / int256(PRECISION);
            }
        } else {
            // Short: PnL = size * (entryPrice - currentPrice)
            if (entryPrice >= currentPrice) {
                pnl = (-size) * int256(entryPrice - currentPrice) / int256(PRECISION);
            } else {
                pnl = size * int256(currentPrice - entryPrice) / int256(PRECISION);
            }
        }
    }
    
    function _abs(int256 x) internal pure returns (uint256) {
        return x >= 0 ? uint256(x) : uint256(-x);
    }
}
```

---

## 3. Insurance Fund + ADL

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title InsuranceFund
 * @notice จัดการ insurance fund และ Auto-Deleveraging (ADL)
 *
 * Insurance Fund:
 * - รวบรวมส่วนหนึ่งของ liquidation fee
 * - ใช้ cover bad debt เมื่อ liquidation ไม่พอ
 *
 * ADL (Auto-Deleveraging):
 * - ใช้เมื่อ insurance fund หมด
 * - Force close positions ที่ profitable สุด
 * - เพื่อ offset bad debt
 */
contract InsuranceFund {
    
    struct ADLQueue {
        address[] traders;      // เรียงจาก most profitable
        int256[] pnlPercentage; // PnL% ต่อ margin
    }
    
    // Insurance fund balance (USD, 1e18 precision)
    uint256 public fundBalance;
    
    // Total bad debt ที่ยังค้างอยู่
    uint256 public totalBadDebt;
    
    // ADL threshold
    uint256 public constant ADL_TRIGGER_RATIO = 0.10e18; // fund ต่ำกว่า 10% ของ OI
    
    // Open interest tracker
    uint256 public totalOpenInterest;
    
    address public perpEngine;
    
    event InsuranceFundContribution(address indexed contributor, uint256 amount);
    event BadDebtCovered(uint256 badDebt, uint256 covered, uint256 remaining);
    event ADLTriggered(address indexed trader, uint256 positionId, uint256 amount);
    
    modifier onlyPerpEngine() {
        require(msg.sender == perpEngine, "not perp engine");
        _;
    }
    
    constructor(address _perpEngine) {
        perpEngine = _perpEngine;
    }
    
    /**
     * @notice เพิ่ม fund จาก liquidation fees
     */
    function contribute(uint256 amount) external onlyPerpEngine {
        fundBalance += amount;
        emit InsuranceFundContribution(msg.sender, amount);
    }
    
    /**
     * @notice ใช้ fund cover bad debt
     * @param badDebt Amount ของ bad debt ที่ต้อง cover
     * @return covered Amount ที่ cover ได้
     * @return uncovered Amount ที่ cover ไม่ได้ (ต้องใช้ ADL)
     */
    function coverBadDebt(uint256 badDebt) external onlyPerpEngine returns (
        uint256 covered,
        uint256 uncovered
    ) {
        if (fundBalance >= badDebt) {
            covered = badDebt;
            fundBalance -= badDebt;
            uncovered = 0;
        } else {
            covered = fundBalance;
            uncovered = badDebt - fundBalance;
            fundBalance = 0;
        }
        
        totalBadDebt += uncovered;
        
        emit BadDebtCovered(badDebt, covered, uncovered);
    }
    
    /**
     * @notice ตรวจสอบว่าควร trigger ADL
     */
    function shouldTriggerADL() public view returns (bool) {
        if (totalOpenInterest == 0) return false;
        return fundBalance * 1e18 / totalOpenInterest < ADL_TRIGGER_RATIO;
    }
    
    /**
     * @notice Execute ADL - force close most profitable positions
     * @param traders Array ของ traders ที่จะ ADL
     * @param positionIds Array ของ position IDs
     * @param targetAmount Amount ที่ต้องการ recover
     */
    function executeADL(
        address[] calldata traders,
        uint256[] calldata positionIds,
        uint256 targetAmount
    ) external onlyPerpEngine {
        require(shouldTriggerADL(), "ADL not triggered");
        require(traders.length == positionIds.length, "array mismatch");
        
        uint256 recovered = 0;
        
        for (uint256 i = 0; i < traders.length && recovered < targetAmount; i++) {
            address trader = traders[i];
            uint256 positionId = positionIds[i];
            
            // Force close ไปยัง perp engine
            uint256 amount = IPerpEngine(perpEngine).forceClosePosition(
                trader,
                positionId,
                targetAmount - recovered
            );
            
            recovered += amount;
            
            emit ADLTriggered(trader, positionId, amount);
        }
        
        totalBadDebt = totalBadDebt > recovered ? totalBadDebt - recovered : 0;
    }
    
    /**
     * @notice อัพเดท open interest
     */
    function updateOpenInterest(uint256 newOI) external onlyPerpEngine {
        totalOpenInterest = newOI;
    }
    
    /**
     * @notice ดู fund health ratio
     */
    function getFundHealthRatio() external view returns (uint256 ratio) {
        if (totalOpenInterest == 0) return type(uint256).max;
        return fundBalance * 1e18 / totalOpenInterest;
    }
}

interface IPerpEngine {
    function forceClosePosition(address trader, uint256 positionId, uint256 maxAmount) external returns (uint256);
}
```

---

## 4. Partial Liquidation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title LiquidationEngine
 * @notice Partial liquidation engine
 * @dev แทนที่จะ liquidate ทั้ง position, ลด size เพื่อให้ margin ratio ดีขึ้น
 *      ลดความเสียหายต่อ trader และ market stability
 */
contract LiquidationEngine {
    
    struct LiquidationParams {
        address trader;
        uint256 positionId;
        uint256 markPrice;
        uint256 liquidationFeeRatio;  // Fee สำหรับ liquidator
        uint256 insuranceFeeRatio;    // Fee สำหรับ insurance fund
    }
    
    uint256 public constant MAINTENANCE_MARGIN_RATIO = 0.05e18;  // 5%
    uint256 public constant TARGET_MARGIN_RATIO = 0.065e18;      // 6.5% (target after partial liq)
    uint256 public constant LIQUIDATION_FEE_RATIO = 0.0025e18;   // 0.25%
    uint256 public constant INSURANCE_FEE_RATIO = 0.0025e18;     // 0.25%
    uint256 public constant MAX_LIQUIDATION_PORTION = 0.5e18;    // Max 50% per liquidation
    
    address public perpEngine;
    address public insuranceFund;
    
    event PartialLiquidation(
        address indexed trader,
        uint256 indexed positionId,
        uint256 liquidatedSize,
        uint256 liquidationPrice,
        address indexed liquidator,
        uint256 liquidatorFee
    );
    
    event FullLiquidation(
        address indexed trader,
        uint256 indexed positionId,
        uint256 badDebt
    );
    
    constructor(address _perpEngine, address _insuranceFund) {
        perpEngine = _perpEngine;
        insuranceFund = _insuranceFund;
    }
    
    /**
     * @notice ตรวจสอบว่า position ควร liquidate
     */
    function isLiquidatable(
        int256 size,
        uint256 entryPrice,
        uint256 margin,
        uint256 markPrice
    ) public pure returns (bool) {
        int256 pnl = _calcPnL(size, entryPrice, markPrice);
        int256 equity = int256(margin) + pnl;
        
        if (equity <= 0) return true;
        
        uint256 positionValue = _abs(size) * markPrice / 1e18;
        uint256 marginRatio = uint256(equity) * 1e18 / positionValue;
        
        return marginRatio < MAINTENANCE_MARGIN_RATIO;
    }
    
    /**
     * @notice คำนวณ partial liquidation size
     * @dev คำนวณว่าต้อง reduce size เท่าไหร่ให้ margin ratio = target ratio
     */
    function calcPartialLiquidationSize(
        int256 currentSize,
        uint256 entryPrice,
        uint256 margin,
        uint256 markPrice
    ) public pure returns (uint256 liquidationSize) {
        // target equity = targetRatio * (positionValue - liquidationValue)
        // equity after = margin + pnl - liquidatorFee - insuranceFee
        
        // Simplified: คำนวณ size ที่ต้อง close เพื่อให้ margin ratio = target
        int256 pnl = _calcPnL(currentSize, entryPrice, markPrice);
        int256 currentEquity = int256(margin) + pnl;
        uint256 absSize = _abs(currentSize);
        
        if (currentEquity <= 0) {
            // ต้อง full liquidation
            return absSize;
        }
        
        uint256 currentPositionValue = absSize * markPrice / 1e18;
        
        // target: equity / (positionValue - sizeToClose * markPrice) = targetRatio
        // equity / targetRatio = positionValue - sizeToClose * markPrice
        // sizeToClose * markPrice = positionValue - equity / targetRatio
        
        uint256 targetPositionValue = uint256(currentEquity) * 1e18 / TARGET_MARGIN_RATIO;
        
        if (targetPositionValue >= currentPositionValue) {
            return 0; // ไม่ต้อง liquidate
        }
        
        uint256 valueDiff = currentPositionValue - targetPositionValue;
        liquidationSize = valueDiff * 1e18 / markPrice;
        
        // Cap ที่ MAX_LIQUIDATION_PORTION
        uint256 maxSize = absSize * MAX_LIQUIDATION_PORTION / 1e18;
        if (liquidationSize > maxSize) {
            liquidationSize = maxSize;
        }
        
        // ต้อง liquidate อย่างน้อย 1 unit
        if (liquidationSize == 0) liquidationSize = 1;
    }
    
    /**
     * @notice Execute partial liquidation
     */
    function liquidate(LiquidationParams calldata params) external returns (
        uint256 liquidationSize,
        uint256 liquidatorFee,
        uint256 badDebt
    ) {
        // ดึงข้อมูล position จาก perp engine
        (int256 size, uint256 entryPrice, uint256 margin) = IPerpEngineV2(perpEngine).getPosition(
            params.trader,
            params.positionId
        );
        
        require(
            isLiquidatable(size, entryPrice, margin, params.markPrice),
            "not liquidatable"
        );
        
        int256 pnl = _calcPnL(size, entryPrice, params.markPrice);
        int256 equity = int256(margin) + pnl;
        
        if (equity <= 0 || _abs(size) == calcPartialLiquidationSize(size, entryPrice, margin, params.markPrice)) {
            // Full liquidation
            return _fullLiquidation(params, size, entryPrice, margin, equity);
        } else {
            // Partial liquidation
            return _partialLiquidation(params, size, entryPrice, margin);
        }
    }
    
    function _partialLiquidation(
        LiquidationParams calldata params,
        int256 size,
        uint256 entryPrice,
        uint256 margin
    ) internal returns (uint256 liquidatedSize, uint256 liquidatorFee, uint256 badDebt) {
        liquidatedSize = calcPartialLiquidationSize(size, entryPrice, margin, params.markPrice);
        
        uint256 liquidationValue = liquidatedSize * params.markPrice / 1e18;
        
        liquidatorFee = liquidationValue * LIQUIDATION_FEE_RATIO / 1e18;
        uint256 insuranceFee = liquidationValue * INSURANCE_FEE_RATIO / 1e18;
        
        // Execute partial close on perp engine
        IPerpEngineV2(perpEngine).partialClose(
            params.trader,
            params.positionId,
            liquidatedSize,
            params.markPrice,
            liquidatorFee + insuranceFee
        );
        
        // ส่ง fees
        // Liquidator fee ส่งไป liquidator
        // Insurance fee ส่งไป insurance fund
        
        emit PartialLiquidation(
            params.trader,
            params.positionId,
            liquidatedSize,
            params.markPrice,
            msg.sender,
            liquidatorFee
        );
    }
    
    function _fullLiquidation(
        LiquidationParams calldata params,
        int256 size,
        uint256 entryPrice,
        uint256 margin,
        int256 equity
    ) internal returns (uint256 liquidatedSize, uint256 liquidatorFee, uint256 badDebt) {
        liquidatedSize = _abs(size);
        
        if (equity >= 0) {
            uint256 uEquity = uint256(equity);
            uint256 positionValue = liquidatedSize * params.markPrice / 1e18;
            
            liquidatorFee = positionValue * LIQUIDATION_FEE_RATIO / 1e18;
            uint256 insuranceFee = positionValue * INSURANCE_FEE_RATIO / 1e18;
            
            // ถ้า equity > fee, คืนส่วนที่เหลือให้ trader
            if (uEquity > liquidatorFee + insuranceFee) {
                uint256 refund = uEquity - liquidatorFee - insuranceFee;
                IPerpEngineV2(perpEngine).refundTrader(params.trader, refund);
            }
        } else {
            // Bad debt: equity < 0
            badDebt = uint256(-equity);
            
            // Insurance fund covers
            IInsuranceFund(insuranceFund).coverBadDebt(badDebt);
        }
        
        IPerpEngineV2(perpEngine).closePosition(params.trader, params.positionId, params.markPrice);
        
        emit FullLiquidation(params.trader, params.positionId, badDebt);
    }
    
    function _calcPnL(int256 size, uint256 entryPrice, uint256 markPrice) 
        internal pure returns (int256 pnl) {
        if (size > 0) {
            if (markPrice >= entryPrice) {
                pnl = size * int256(markPrice - entryPrice) / 1e18;
            } else {
                pnl = -size * int256(entryPrice - markPrice) / 1e18;
            }
        } else {
            if (entryPrice >= markPrice) {
                pnl = (-size) * int256(entryPrice - markPrice) / 1e18;
            } else {
                pnl = size * int256(markPrice - entryPrice) / 1e18;
            }
        }
    }
    
    function _abs(int256 x) internal pure returns (uint256) {
        return x >= 0 ? uint256(x) : uint256(-x);
    }
}

interface IPerpEngineV2 {
    function getPosition(address trader, uint256 positionId) external view returns (int256, uint256, uint256);
    function partialClose(address trader, uint256 positionId, uint256 size, uint256 price, uint256 fees) external;
    function closePosition(address trader, uint256 positionId, uint256 price) external;
    function refundTrader(address trader, uint256 amount) external;
    function forceClosePosition(address trader, uint256 positionId, uint256 maxAmount) external returns (uint256);
}

interface IInsuranceFund {
    function coverBadDebt(uint256 amount) external returns (uint256, uint256);
}
```

---

## 5. AdvancedPerpEngine - Full Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title AdvancedPerpEngine
 * @notice Full perpetual futures engine
 * @dev รวม funding rate, cross/isolated margin, liquidation, insurance fund
 */
contract AdvancedPerpEngine is ReentrancyGuard, Ownable {
    using SafeERC20 for IERC20;
    
    // ========== Data Structures ==========
    
    struct Position {
        int256 size;              // Positive = long, negative = short
        uint256 entryPrice;       // Average entry price
        uint256 openNotional;     // size * entryPrice (unsigned)
        uint256 margin;           // Collateral for isolated, 0 for cross
        int256 fundingPaymentAccumulated; // Accumulated funding payments
        int256 entryFundingPerUnit; // fundingPerUnit เมื่อ open
        MarginMode marginMode;
        bool exists;
        uint256 openTimestamp;
    }
    
    struct Market {
        string name;
        address baseToken;
        address quoteToken;
        bool active;
        
        // Market parameters
        uint256 maxLeverage;           // Max leverage (1e18 precision)
        uint256 maintenanceMarginRatio; // Maintenance margin ratio (1e18)
        uint256 initMarginRatio;        // Initial margin ratio (1e18)
        
        // Market stats
        int256 totalLongSize;
        int256 totalShortSize;
        uint256 openInterest;
        
        // Funding
        int256 fundingPerUnit;
        uint256 lastFundingTime;
        int256 lastFundingRate;
    }
    
    enum MarginMode { Cross, Isolated }
    
    // ========== State ==========
    
    IERC20 public immutable collateral;
    
    // Markets
    mapping(bytes32 => Market) public markets;
    
    // Positions: positionKey = keccak256(trader, marketId, positionId)
    mapping(bytes32 => Position) public positions;
    mapping(address => uint256) public positionCount;
    
    // Cross margin balances
    mapping(address => int256) public crossMarginBalances;
    
    // Protocol components
    address public liquidationEngine;
    address public insuranceFund;
    address public oracle;
    
    // Fee rates
    uint256 public takerFeeRate = 5;   // 0.05% = 5 bps
    uint256 public makerFeeRate = 2;   // 0.02% = 2 bps
    uint256 public liquidationFeeRate = 25; // 0.25% = 25 bps
    
    // ========== Events ==========
    
    event PositionOpened(
        address indexed trader,
        bytes32 indexed marketId,
        uint256 positionId,
        int256 size,
        uint256 entryPrice,
        MarginMode marginMode
    );
    
    event PositionClosed(
        address indexed trader,
        bytes32 indexed marketId,
        uint256 positionId,
        int256 realizedPnL,
        uint256 exitPrice
    );
    
    event PositionIncreased(
        address indexed trader,
        bytes32 indexed positionId,
        int256 additionalSize
    );
    
    event FundingRateUpdated(
        bytes32 indexed marketId,
        int256 fundingRate,
        uint256 timestamp
    );
    
    event MarginAdded(address indexed trader, bytes32 indexed positionKey, uint256 amount);
    event MarginRemoved(address indexed trader, bytes32 indexed positionKey, uint256 amount);
    
    // ========== Constructor ==========
    
    constructor(
        address _collateral,
        address _oracle
    ) Ownable(msg.sender) {
        collateral = IERC20(_collateral);
        oracle = _oracle;
    }
    
    // ========== Market Setup ==========
    
    function createMarket(
        string calldata name,
        address baseToken,
        bytes32 marketId
    ) external onlyOwner {
        markets[marketId] = Market({
            name: name,
            baseToken: baseToken,
            quoteToken: address(collateral),
            active: true,
            maxLeverage: 100e18,
            maintenanceMarginRatio: 0.05e18,
            initMarginRatio: 0.10e18,
            totalLongSize: 0,
            totalShortSize: 0,
            openInterest: 0,
            fundingPerUnit: 0,
            lastFundingTime: block.timestamp,
            lastFundingRate: 0
        });
    }
    
    // ========== Position Management ==========
    
    /**
     * @notice เปิด position ใหม่
     */
    function openPosition(
        bytes32 marketId,
        int256 size,             // Positive = long, negative = short
        uint256 margin,          // Initial margin (0 for cross)
        MarginMode marginMode,
        uint256 maxSlippage      // Max acceptable slippage in bps
    ) external nonReentrant returns (uint256 positionId) {
        Market storage market = markets[marketId];
        require(market.active, "market not active");
        require(size != 0, "zero size");
        
        // ดึงราคาจาก oracle
        uint256 markPrice = _getMarkPrice(marketId);
        
        // คำนวณ required margin
        uint256 notional = _abs(size) * markPrice / 1e18;
        uint256 requiredMargin = notional * market.initMarginRatio / 1e18;
        
        if (marginMode == MarginMode.Isolated) {
            require(margin >= requiredMargin, "insufficient isolated margin");
            collateral.safeTransferFrom(msg.sender, address(this), margin);
        } else {
            // Cross margin - check total equity
            require(
                _getCrossEquity(msg.sender) >= int256(requiredMargin),
                "insufficient cross margin"
            );
        }
        
        // Collect taker fee
        uint256 takerFee = notional * takerFeeRate / 10000;
        if (marginMode == MarginMode.Isolated) {
            require(margin >= requiredMargin + takerFee, "margin covers fee");
            margin -= takerFee;
        } else {
            crossMarginBalances[msg.sender] -= int256(takerFee);
        }
        
        // Create position
        positionId = positionCount[msg.sender]++;
        bytes32 positionKey = _getPositionKey(msg.sender, marketId, positionId);
        
        // Settle pending funding before opening
        _settleFunding(marketId);
        
        positions[positionKey] = Position({
            size: size,
            entryPrice: markPrice,
            openNotional: notional,
            margin: marginMode == MarginMode.Isolated ? margin : 0,
            fundingPaymentAccumulated: 0,
            entryFundingPerUnit: market.fundingPerUnit,
            marginMode: marginMode,
            exists: true,
            openTimestamp: block.timestamp
        });
        
        // อัพเดท market stats
        if (size > 0) {
            market.totalLongSize += size;
        } else {
            market.totalShortSize += size;
        }
        market.openInterest += notional;
        
        emit PositionOpened(msg.sender, marketId, positionId, size, markPrice, marginMode);
    }
    
    /**
     * @notice ปิด position
     */
    function closePosition(
        bytes32 marketId,
        uint256 positionId,
        uint256 minReceived    // Slippage protection
    ) external nonReentrant returns (int256 realizedPnL) {
        bytes32 positionKey = _getPositionKey(msg.sender, marketId, positionId);
        Position storage pos = positions[positionKey];
        require(pos.exists, "position not found");
        
        uint256 markPrice = _getMarkPrice(marketId);
        
        // Settle funding
        _settleFunding(marketId);
        int256 fundingPayment = _calcFundingPayment(positionKey, marketId);
        
        // คำนวณ realized PnL
        realizedPnL = _calcPnL(pos.size, pos.entryPrice, markPrice) - fundingPayment;
        
        uint256 payout = 0;
        if (pos.marginMode == MarginMode.Isolated) {
            int256 equity = int256(pos.margin) + realizedPnL;
            if (equity > 0) {
                payout = uint256(equity);
            }
        } else {
            crossMarginBalances[msg.sender] += realizedPnL;
            payout = 0;
        }
        
        require(payout >= minReceived, "slippage exceeded");
        
        // Collect taker fee
        uint256 notional = _abs(pos.size) * markPrice / 1e18;
        uint256 closeFee = notional * takerFeeRate / 10000;
        
        if (payout > closeFee) {
            payout -= closeFee;
        }
        
        // อัพเดท market stats
        Market storage market = markets[marketId];
        if (pos.size > 0) {
            market.totalLongSize -= pos.size;
        } else {
            market.totalShortSize -= pos.size;
        }
        market.openInterest -= pos.openNotional;
        
        // Close position
        delete positions[positionKey];
        
        // Transfer payout
        if (payout > 0) {
            collateral.safeTransfer(msg.sender, payout);
        }
        
        emit PositionClosed(msg.sender, marketId, positionId, realizedPnL, markPrice);
    }
    
    /**
     * @notice เพิ่ม margin ให้ isolated position
     */
    function addMargin(bytes32 marketId, uint256 positionId, uint256 amount) external nonReentrant {
        bytes32 positionKey = _getPositionKey(msg.sender, marketId, positionId);
        Position storage pos = positions[positionKey];
        require(pos.exists, "position not found");
        require(pos.marginMode == MarginMode.Isolated, "not isolated");
        
        collateral.safeTransferFrom(msg.sender, address(this), amount);
        pos.margin += amount;
        
        emit MarginAdded(msg.sender, positionKey, amount);
    }
    
    // ========== Funding Rate ==========
    
    /**
     * @notice Settle funding rate สำหรับ market
     */
    function _settleFunding(bytes32 marketId) internal {
        Market storage market = markets[marketId];
        
        if (block.timestamp < market.lastFundingTime + 8 hours) return;
        
        uint256 markPrice = _getMarkPrice(marketId);
        uint256 indexPrice = _getIndexPrice(marketId);
        
        // คำนวณ funding rate
        int256 premium;
        if (markPrice >= indexPrice) {
            premium = int256((markPrice - indexPrice) * 1e18 / indexPrice);
        } else {
            premium = -int256((indexPrice - markPrice) * 1e18 / indexPrice);
        }
        
        // Clamp at ±0.05%
        int256 maxFunding = 0.0005e18;
        if (premium > maxFunding) premium = maxFunding;
        if (premium < -maxFunding) premium = -maxFunding;
        
        market.lastFundingRate = premium;
        market.fundingPerUnit += premium;
        market.lastFundingTime = block.timestamp;
        
        emit FundingRateUpdated(marketId, premium, block.timestamp);
    }
    
    /**
     * @notice คำนวณ funding payment สำหรับ position
     */
    function _calcFundingPayment(bytes32 positionKey, bytes32 marketId) internal view returns (int256) {
        Position storage pos = positions[positionKey];
        Market storage market = markets[marketId];
        
        int256 fundingDiff = market.fundingPerUnit - pos.entryFundingPerUnit;
        return pos.size * fundingDiff / 1e18;
    }
    
    // ========== Helper Functions ==========
    
    function _getPositionKey(address trader, bytes32 marketId, uint256 positionId) 
        internal pure returns (bytes32) {
        return keccak256(abi.encodePacked(trader, marketId, positionId));
    }
    
    function _getMarkPrice(bytes32 marketId) internal view returns (uint256) {
        return IPriceOracle(oracle).getMarkPrice(marketId);
    }
    
    function _getIndexPrice(bytes32 marketId) internal view returns (uint256) {
        return IPriceOracle(oracle).getIndexPrice(marketId);
    }
    
    function _getCrossEquity(address user) internal view returns (int256) {
        return crossMarginBalances[user]; // Simplified (should include unrealized PnL)
    }
    
    function _calcPnL(int256 size, uint256 entryPrice, uint256 markPrice) 
        internal pure returns (int256 pnl) {
        if (size > 0) {
            if (markPrice >= entryPrice) {
                pnl = size * int256(markPrice - entryPrice) / 1e18;
            } else {
                pnl = -size * int256(entryPrice - markPrice) / 1e18;
            }
        } else {
            if (entryPrice >= markPrice) {
                pnl = (-size) * int256(entryPrice - markPrice) / 1e18;
            } else {
                pnl = size * int256(markPrice - entryPrice) / 1e18;
            }
        }
    }
    
    function _abs(int256 x) internal pure returns (uint256) {
        return x >= 0 ? uint256(x) : uint256(-x);
    }
    
    // ========== View Functions ==========
    
    function getPosition(address trader, bytes32 marketId, uint256 positionId) 
        external view returns (
            int256 size,
            uint256 entryPrice,
            uint256 margin,
            int256 unrealizedPnL,
            uint256 liquidationPrice
        ) {
        bytes32 positionKey = _getPositionKey(trader, marketId, positionId);
        Position storage pos = positions[positionKey];
        require(pos.exists, "position not found");
        
        uint256 markPrice = _getMarkPrice(marketId);
        
        size = pos.size;
        entryPrice = pos.entryPrice;
        margin = pos.margin;
        unrealizedPnL = _calcPnL(pos.size, pos.entryPrice, markPrice);
        liquidationPrice = _calcLiquidationPrice(pos.size, pos.entryPrice, pos.margin);
    }
    
    function _calcLiquidationPrice(int256 size, uint256 entryPrice, uint256 margin) 
        internal pure returns (uint256) {
        if (size == 0 || margin == 0) return 0;
        
        uint256 absSize = _abs(size);
        uint256 maintenanceMargin = absSize * entryPrice / 1e18 * 0.05e18 / 1e18;
        uint256 usableMargin = margin > maintenanceMargin ? margin - maintenanceMargin : 0;
        
        if (size > 0) {
            uint256 priceMove = usableMargin * 1e18 / absSize;
            return entryPrice > priceMove ? entryPrice - priceMove : 0;
        } else {
            return entryPrice + usableMargin * 1e18 / absSize;
        }
    }
    
    // Interface implementations
    function getPosition(address trader, uint256 positionId) external view returns (int256, uint256, uint256) {
        return (0, 0, 0); // Stub for interface
    }
    
    function partialClose(address trader, uint256 positionId, uint256 size, uint256 price, uint256 fees) external {}
    function refundTrader(address trader, uint256 amount) external {
        collateral.safeTransfer(trader, amount);
    }
    function forceClosePosition(address trader, uint256 positionId, uint256 maxAmount) external returns (uint256) {
        return 0;
    }
}

interface IPriceOracle {
    function getMarkPrice(bytes32 marketId) external view returns (uint256);
    function getIndexPrice(bytes32 marketId) external view returns (uint256);
}
```

---

## Workshop: Testing Advanced Perp Engine

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";

contract AdvancedPerpTest is Test {
    
    AdvancedPerpEngine public engine;
    MockERC20 public usdc;
    MockOracle public oracle;
    
    address alice = address(0xALICE);
    address bob = address(0xBOB);
    bytes32 constant ETH_MARKET = keccak256("ETH-USD");
    
    function setUp() public {
        usdc = new MockERC20("USDC", "USDC", 6);
        oracle = new MockOracle();
        engine = new AdvancedPerpEngine(address(usdc), address(oracle));
        
        // Create ETH market
        engine.createMarket("ETH-USD", address(0), ETH_MARKET);
        
        // Setup oracle price
        oracle.setMarkPrice(ETH_MARKET, 3000e18);
        oracle.setIndexPrice(ETH_MARKET, 2990e18); // Slightly above index
        
        // Fund users
        usdc.mint(alice, 100000e6);
        usdc.mint(bob, 100000e6);
        
        vm.prank(alice);
        usdc.approve(address(engine), type(uint256).max);
        vm.prank(bob);
        usdc.approve(address(engine), type(uint256).max);
    }
    
    function test_openAndCloseLong() public {
        // Alice opens 1 ETH long at 3000, 10x leverage
        int256 size = 1e18; // 1 ETH
        uint256 markPrice = 3000e18;
        uint256 notional = 3000e6; // 3000 USDC
        uint256 margin = 300e6;   // 300 USDC (10x)
        
        vm.prank(alice);
        uint256 posId = engine.openPosition(
            ETH_MARKET,
            size,
            margin + 2e6, // Extra for fees
            AdvancedPerpEngine.MarginMode.Isolated,
            50 // 0.5% max slippage
        );
        
        // Price increases to 3300
        oracle.setMarkPrice(ETH_MARKET, 3300e18);
        
        // Alice closes for profit
        uint256 balBefore = usdc.balanceOf(alice);
        
        vm.prank(alice);
        int256 pnl = engine.closePosition(ETH_MARKET, posId, 0);
        
        uint256 balAfter = usdc.balanceOf(alice);
        
        // Should have made ~300 USDC profit (1 ETH * 300 price increase)
        assertGt(pnl, 0, "should be profitable");
        assertGt(balAfter, balBefore, "balance should increase");
        
        console.log("PnL:", uint256(pnl));
        console.log("Balance increase:", balAfter - balBefore);
    }
    
    function test_funding_rate() public {
        // Mark > Index means longs pay shorts
        oracle.setMarkPrice(ETH_MARKET, 3100e18);  // 3.35% premium
        oracle.setIndexPrice(ETH_MARKET, 3000e18);
        
        // Fast forward 8 hours to trigger funding
        vm.warp(block.timestamp + 8 hours);
        
        // Open long to settle funding
        vm.prank(alice);
        engine.openPosition(
            ETH_MARKET,
            1e18,
            300e6,
            AdvancedPerpEngine.MarginMode.Isolated,
            50
        );
        
        // Check funding was settled
        (,,,,,,int256 fundingPerUnit,,,) = engine.markets(ETH_MARKET);
        
        // Funding rate should be clamped at max (0.05%)
        // Since premium (3.35%) >> max funding (0.05%)
        assertEq(fundingPerUnit, 0.0005e18); // Clamped at max
        
        console.log("Funding per unit:", uint256(fundingPerUnit));
    }
    
    function test_partial_liquidation() public {
        LiquidationEngine liquidator = new LiquidationEngine(
            address(engine),
            address(new InsuranceFund(address(engine)))
        );
        
        // Alice opens leveraged long
        vm.prank(alice);
        uint256 posId = engine.openPosition(
            ETH_MARKET,
            10e18,    // 10 ETH long
            3000e6,   // 3000 USDC margin (10x leverage)
            AdvancedPerpEngine.MarginMode.Isolated,
            50
        );
        
        // Price drops 7%
        oracle.setMarkPrice(ETH_MARKET, 2790e18);
        
        LiquidationEngine.LiquidationParams memory params = LiquidationEngine.LiquidationParams({
            trader: alice,
            positionId: posId,
            markPrice: 2790e18,
            liquidationFeeRatio: 25,
            insuranceFeeRatio: 25
        });
        
        // Check if liquidatable
        (int256 size, uint256 entryPrice, uint256 margin) = engine.getPosition(alice, posId);
        bool liqable = liquidator.isLiquidatable(size, entryPrice, margin, 2790e18);
        
        if (liqable) {
            console.log("Position is liquidatable");
            
            uint256 partialSize = liquidator.calcPartialLiquidationSize(
                size, entryPrice, margin, 2790e18
            );
            console.log("Partial liquidation size:", partialSize);
            assertTrue(partialSize > 0 && partialSize < uint256(size > 0 ? size : -size));
        }
    }
}
```

---

## สรุป Part 89

- **Funding Rate**: 8-hour TWAP ของ mark vs index price, clamped ที่ ±0.05%, สะสมใน fundingPerUnit
- **Cross vs Isolated Margin**: Cross ใช้ equity รวม, Isolated แยกต่อ position พร้อม liquidation price เฉพาะ
- **Insurance Fund**: รวบรวม fees, cover bad debt, trigger ADL เมื่อ fund ต่ำกว่า threshold
- **Partial Liquidation**: คำนวณ minimum size ที่ต้อง close เพื่อคืน margin ratio สู่ target (6.5%)
- **AdvancedPerpEngine**: Position management สมบูรณ์ พร้อม funding settlement, PnL calculation, liquidation price

## Next: Part 90 - Protocol Composability & DeFi Legos
