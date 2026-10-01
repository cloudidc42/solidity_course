# Part 31: Perpetuals และ Derivatives

## สารบัญ
1. Perpetual Futures Basics
2. Funding Rate Mechanism
3. Margin และ Liquidation
4. Options Basics
5. Workshop: Mini Perpetuals

---

## 1. Perpetual Futures Basics

```
Perpetual Futures:
- Futures contract ไม่มีวันหมดอายุ
- ราคาติดตาม underlying asset ผ่าน funding rate
- ซื้อ Long: profit ถ้าราคาขึ้น
- ซื้อ Short: profit ถ้าราคาลง
- Leverage: ใช้ margin น้อยแต่ได้ exposure มาก

Key Metrics:
- Mark Price: ราคา fair value (oracle-based)
- Index Price: ราคา spot จาก CEXes
- Entry Price: ราคาที่เปิด position
- Leverage: e.g., 10x = ควบคุม $10k ด้วย $1k margin
- Margin Ratio: margin / position value
- Liquidation Price: ถ้าราคาถึงจุดนี้ → position ถูก liquidate

Protocol Examples:
- dYdX (v4 on Cosmos)
- GMX (Arbitrum)
- Perpetual Protocol
- Synthetix Perps
```

---

## 2. Funding Rate

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Funding Rate Mechanism:
 * 
 * ทุก 8 ชั่วโมง:
 * - ถ้า mark price > index price: longs จ่าย shorts
 * - ถ้า mark price < index price: shorts จ่าย longs
 * 
 * Funding = position_size × funding_rate
 * Funding Rate = (mark - index) / index × k
 * 
 * เป้าหมาย: ทำให้ mark price ≈ index price
 */
contract FundingRateCalculator {
    
    uint256 public constant FUNDING_INTERVAL = 8 hours;
    int256 public constant FUNDING_FACTOR = 1e15; // 0.001 or 0.1%
    
    int256 public cumulativeFundingRate;
    uint256 public lastFundingTime;
    
    IOracle public oracle;
    
    struct Position {
        address trader;
        bool isLong;
        uint256 size;         // position size in USD (18 decimals)
        uint256 entryPrice;   // entry mark price
        int256 fundingRateAtEntry; // cumulative funding at entry
        uint256 margin;       // collateral deposited
    }
    
    mapping(bytes32 => Position) public positions;
    
    constructor(address _oracle) {
        oracle = IOracle(_oracle);
        lastFundingTime = block.timestamp;
    }
    
    function calculateFundingRate() public view returns (int256) {
        uint256 markPrice = oracle.getMarkPrice();
        uint256 indexPrice = oracle.getIndexPrice();
        
        if (indexPrice == 0) return 0;
        
        // fundingRate = (mark - index) / index × factor
        int256 premium = (int256(markPrice) - int256(indexPrice)) * FUNDING_FACTOR / int256(indexPrice);
        
        // Clamp to ±0.3% per interval
        int256 maxFunding = 3e15; // 0.3%
        if (premium > maxFunding) return maxFunding;
        if (premium < -maxFunding) return -maxFunding;
        
        return premium;
    }
    
    function settleFunding() external {
        if (block.timestamp < lastFundingTime + FUNDING_INTERVAL) return;
        
        int256 fundingRate = calculateFundingRate();
        cumulativeFundingRate += fundingRate;
        lastFundingTime = block.timestamp;
    }
    
    // Calculate pending funding for a position
    function pendingFunding(bytes32 positionId) external view returns (int256) {
        Position storage pos = positions[positionId];
        
        int256 fundingDiff = cumulativeFundingRate - pos.fundingRateAtEntry;
        
        // Longs pay positive funding, receive negative
        // Shorts receive positive funding, pay negative
        int256 funding = int256(pos.size) * fundingDiff / 1e18;
        
        return pos.isLong ? -funding : funding;
    }
}

interface IOracle {
    function getMarkPrice() external view returns (uint256);
    function getIndexPrice() external view returns (uint256);
}
```

---

## 3. Margin และ Position Management

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Perpetual Position Management
 */
contract PerpetualEngine {
    
    IERC20 public immutable collateralToken; // USDC
    IOracle public immutable oracle;
    
    uint256 public constant MAINTENANCE_MARGIN_RATIO = 500; // 5%
    uint256 public constant INITIAL_MARGIN_RATIO = 1000;    // 10%
    uint256 public constant LIQUIDATION_FEE = 100;          // 1%
    uint256 public constant BASIS_POINTS = 10000;
    
    struct Position {
        address trader;
        bool isLong;
        uint256 size;       // notional value in USDC
        uint256 collateral; // margin deposited
        uint256 entryPrice;
        int256 realisedPnl;
    }
    
    mapping(bytes32 => Position) public positions;
    mapping(address => bytes32[]) public traderPositions;
    
    uint256 public openInterestLong;
    uint256 public openInterestShort;
    
    event PositionOpened(bytes32 indexed posId, address indexed trader, bool isLong, uint256 size, uint256 price);
    event PositionClosed(bytes32 indexed posId, address indexed trader, int256 pnl);
    event PositionLiquidated(bytes32 indexed posId, address indexed liquidator);
    
    error InsufficientMargin(uint256 required, uint256 provided);
    error PositionHealthy(uint256 marginRatio);
    error NotTrader();
    
    constructor(address _collateral, address _oracle) {
        collateralToken = IERC20(_collateral);
        oracle = IOracle(_oracle);
    }
    
    function openPosition(
        bool isLong,
        uint256 collateralAmount,
        uint256 leverage // e.g., 10 = 10x
    ) external returns (bytes32 posId) {
        require(leverage >= 1 && leverage <= 50, "Invalid leverage");
        require(collateralAmount > 0, "No collateral");
        
        uint256 size = collateralAmount * leverage; // notional value
        
        // Check initial margin requirement
        uint256 initialMarginRequired = (size * INITIAL_MARGIN_RATIO) / BASIS_POINTS;
        if (collateralAmount < initialMarginRequired) {
            revert InsufficientMargin(initialMarginRequired, collateralAmount);
        }
        
        collateralToken.transferFrom(msg.sender, address(this), collateralAmount);
        
        uint256 markPrice = oracle.getMarkPrice();
        
        posId = keccak256(abi.encodePacked(msg.sender, block.timestamp, isLong, size));
        
        positions[posId] = Position({
            trader: msg.sender,
            isLong: isLong,
            size: size,
            collateral: collateralAmount,
            entryPrice: markPrice,
            realisedPnl: 0
        });
        
        traderPositions[msg.sender].push(posId);
        
        if (isLong) {
            openInterestLong += size;
        } else {
            openInterestShort += size;
        }
        
        emit PositionOpened(posId, msg.sender, isLong, size, markPrice);
    }
    
    function closePosition(bytes32 posId) external {
        Position storage pos = positions[posId];
        require(pos.trader == msg.sender, "Not trader");
        
        uint256 currentPrice = oracle.getMarkPrice();
        int256 pnl = _calculatePnl(pos, currentPrice);
        
        uint256 collateral = pos.collateral;
        
        if (pos.isLong) {
            openInterestLong -= pos.size;
        } else {
            openInterestShort -= pos.size;
        }
        
        delete positions[posId];
        
        // Calculate payout
        int256 payout = int256(collateral) + pnl;
        
        if (payout > 0) {
            collateralToken.transfer(msg.sender, uint256(payout));
        }
        
        emit PositionClosed(posId, msg.sender, pnl);
    }
    
    function liquidate(bytes32 posId) external {
        Position storage pos = positions[posId];
        
        uint256 currentPrice = oracle.getMarkPrice();
        
        // Check if position is liquidatable
        int256 pnl = _calculatePnl(pos, currentPrice);
        uint256 remainingMargin = pnl >= 0
            ? pos.collateral + uint256(pnl)
            : pos.collateral > uint256(-pnl)
                ? pos.collateral - uint256(-pnl)
                : 0;
        
        uint256 marginRatio = (remainingMargin * BASIS_POINTS) / pos.size;
        
        if (marginRatio >= MAINTENANCE_MARGIN_RATIO) {
            revert PositionHealthy(marginRatio);
        }
        
        // Pay liquidator
        uint256 liquidationFee = (pos.size * LIQUIDATION_FEE) / BASIS_POINTS;
        
        if (pos.isLong) {
            openInterestLong -= pos.size;
        } else {
            openInterestShort -= pos.size;
        }
        
        delete positions[posId];
        
        collateralToken.transfer(msg.sender, liquidationFee);
        
        emit PositionLiquidated(posId, msg.sender);
    }
    
    function _calculatePnl(Position storage pos, uint256 currentPrice) internal view returns (int256) {
        if (pos.size == 0) return 0;
        
        int256 priceDiff = int256(currentPrice) - int256(pos.entryPrice);
        
        // Long: profit when price goes up
        // Short: profit when price goes down
        if (pos.isLong) {
            return (int256(pos.size) * priceDiff) / int256(pos.entryPrice);
        } else {
            return -(int256(pos.size) * priceDiff) / int256(pos.entryPrice);
        }
    }
    
    function getMarginRatio(bytes32 posId) external view returns (uint256) {
        Position storage pos = positions[posId];
        uint256 currentPrice = oracle.getMarkPrice();
        
        int256 pnl = _calculatePnl(pos, currentPrice);
        uint256 remainingMargin = pnl >= 0
            ? pos.collateral + uint256(pnl)
            : pos.collateral > uint256(-pnl)
                ? pos.collateral - uint256(-pnl)
                : 0;
        
        return (remainingMargin * BASIS_POINTS) / pos.size;
    }
    
    function getLiquidationPrice(bytes32 posId) external view returns (uint256) {
        Position storage pos = positions[posId];
        
        // At liquidation: remaining margin = maintenance margin
        // maintenance_margin = size × maintenance_margin_ratio / BASIS_POINTS
        // collateral + pnl = maintenance_margin
        
        uint256 maintenanceMargin = (pos.size * MAINTENANCE_MARGIN_RATIO) / BASIS_POINTS;
        
        if (pos.isLong) {
            // pnl = size × (price - entry) / entry
            // collateral + size × (liqPrice - entry) / entry = maintenanceMargin
            // liqPrice = entry × (maintenanceMargin - collateral + size) / size
            if (pos.collateral <= maintenanceMargin) return 0;
            return pos.entryPrice * (pos.size - pos.collateral + maintenanceMargin) / pos.size;
        } else {
            // pnl = size × (entry - price) / entry
            // collateral + size × (entry - liqPrice) / entry = maintenanceMargin
            return pos.entryPrice * (pos.size + pos.collateral - maintenanceMargin) / pos.size;
        }
    }
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
}
```

---

## 4. Options Basics

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * European Options (Simple)
 * 
 * Call Option: สิทธิ์ซื้อ asset ที่ strike price ก่อน expiry
 * Put Option: สิทธิ์ขาย asset ที่ strike price ก่อน expiry
 * 
 * Payoff at expiry:
 * Call: max(spot - strike, 0)
 * Put: max(strike - spot, 0)
 * 
 * Option Pricing (Black-Scholes simplified):
 * Premium ขึ้นอยู่กับ:
 * - Intrinsic value: payoff ณ ปัจจุบัน
 * - Time value: เวลาที่เหลือ
 * - Volatility: ความผันผวนของ underlying
 */
contract SimpleOptions {
    
    enum OptionType { Call, Put }
    
    struct Option {
        OptionType optionType;
        address underlying;     // asset ที่ option อ้างอิง
        uint256 strikePrice;    // in USD (18 decimals)
        uint256 expiry;
        uint256 premium;        // cost to buy option
        address writer;
        address buyer;
        bool exercised;
        bool settled;
    }
    
    IERC20 public immutable usdc;
    IOracle public immutable oracle;
    
    uint256 public optionCount;
    mapping(uint256 => Option) public options;
    
    // Writer posts collateral
    mapping(uint256 => uint256) public collateral;
    
    event OptionCreated(uint256 indexed optionId, OptionType optionType, uint256 strike, uint256 expiry);
    event OptionBought(uint256 indexed optionId, address indexed buyer, uint256 premium);
    event OptionExercised(uint256 indexed optionId, address indexed exerciser, uint256 payout);
    
    constructor(address _usdc, address _oracle) {
        usdc = IERC20(_usdc);
        oracle = IOracle(_oracle);
    }
    
    // Writer creates option (posts collateral)
    function writeOption(
        OptionType optionType,
        uint256 strikePrice,
        uint256 expiry,
        uint256 premium,
        uint256 notional // USD value of underlying
    ) external returns (uint256 optionId) {
        require(expiry > block.timestamp, "Expiry in past");
        
        optionId = optionCount++;
        
        // Writer posts collateral (max loss)
        uint256 requiredCollateral;
        if (optionType == OptionType.Call) {
            // Call writer: unlimited downside but capped at notional
            requiredCollateral = notional;
        } else {
            // Put writer: max loss = strikePrice × quantity
            requiredCollateral = strikePrice;
        }
        
        usdc.transferFrom(msg.sender, address(this), requiredCollateral);
        collateral[optionId] = requiredCollateral;
        
        options[optionId] = Option({
            optionType: optionType,
            underlying: address(0), // simplified
            strikePrice: strikePrice,
            expiry: expiry,
            premium: premium,
            writer: msg.sender,
            buyer: address(0),
            exercised: false,
            settled: false
        });
        
        emit OptionCreated(optionId, optionType, strikePrice, expiry);
    }
    
    // Buyer purchases option
    function buyOption(uint256 optionId) external {
        Option storage opt = options[optionId];
        
        require(opt.buyer == address(0), "Already sold");
        require(block.timestamp < opt.expiry, "Expired");
        
        usdc.transferFrom(msg.sender, opt.writer, opt.premium);
        opt.buyer = msg.sender;
        
        emit OptionBought(optionId, msg.sender, opt.premium);
    }
    
    // Exercise option at expiry
    function exercise(uint256 optionId) external {
        Option storage opt = options[optionId];
        
        require(msg.sender == opt.buyer, "Not buyer");
        require(block.timestamp >= opt.expiry, "Not expired");
        require(!opt.exercised, "Already exercised");
        
        uint256 spotPrice = oracle.getMarkPrice();
        
        uint256 payout;
        if (opt.optionType == OptionType.Call) {
            // Call payoff: max(spot - strike, 0)
            if (spotPrice > opt.strikePrice) {
                payout = spotPrice - opt.strikePrice;
            }
        } else {
            // Put payoff: max(strike - spot, 0)
            if (opt.strikePrice > spotPrice) {
                payout = opt.strikePrice - spotPrice;
            }
        }
        
        opt.exercised = true;
        opt.settled = true;
        
        if (payout > 0 && payout <= collateral[optionId]) {
            usdc.transfer(opt.buyer, payout);
            usdc.transfer(opt.writer, collateral[optionId] - payout);
        } else {
            usdc.transfer(opt.writer, collateral[optionId]);
        }
        
        emit OptionExercised(optionId, msg.sender, payout);
    }
    
    // Expire worthless if not exercised
    function expireOption(uint256 optionId) external {
        Option storage opt = options[optionId];
        
        require(block.timestamp >= opt.expiry + 1 days, "Too soon");
        require(!opt.settled, "Already settled");
        
        opt.settled = true;
        usdc.transfer(opt.writer, collateral[optionId]);
    }
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
}
```

---

## สรุป Part 31

Perpetuals & Derivatives ที่เรียนรู้:
- ✅ Perpetual futures mechanics
- ✅ Funding rate (mark vs index)
- ✅ Margin management + liquidation
- ✅ PnL calculation (Long/Short)
- ✅ Options (Call/Put, payoff)

## Quiz

1. Funding rate ทำหน้าที่อะไรใน perpetuals?
2. Liquidation price คำนวณอย่างไร?
3. Call option payoff คืออะไร?
4. Open interest คืออะไร?

---

## Next: Part 32 - Real World Assets (RWA)
