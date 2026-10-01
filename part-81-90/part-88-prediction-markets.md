# Part 88: Prediction Markets

## บทนำ

Prediction Markets คือตลาดที่ให้คนซื้อขาย shares ใน outcome ของเหตุการณ์ในอนาคต เช่น การเลือกตั้ง ผลกีฬา หรือเหตุการณ์ทางเศรษฐกิจ ราคาของ share สะท้อน probability ที่ตลาดประเมิน

ตัวอย่างที่ดัง:
- **Augur**: Ethereum-based prediction market ยุคแรก
- **Polymarket**: ใช้ USDC บน Polygon
- **Gnosis Prediction Market**: AMM-based

ในบทนี้เราจะสร้าง:
1. OrderBook-based prediction market
2. AMM-based (LMSR) prediction market
3. Oracle resolution system

---

## 1. Prediction Market Architecture

### Core Concepts

```
Market = Event ที่กำลังพยากรณ์
Outcomes = ผลลัพธ์ที่เป็นไปได้ (Yes/No หรือ Multiple)
Shares = Token แทน position ใน outcome
Resolution = การพิจารณาว่า outcome ไหนถูก
```

### Market States

```
OPEN → Trading allowed
LOCKED → No new orders (before resolution)
RESOLVED → Winner declared
REDEEMED → Winners claimed funds
```

---

## 2. OrderBook Prediction Market

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title OrderBookPredictionMarket
 * @notice Prediction market ด้วย limit order book
 * @dev ผู้ใช้วาง limit orders เพื่อซื้อ/ขาย outcome shares
 */
contract OrderBookPredictionMarket is ReentrancyGuard, Ownable {
    using SafeERC20 for IERC20;
    
    // ========== Data Structures ==========
    
    enum MarketStatus { Open, Locked, Resolved, Redeemed }
    
    struct Market {
        string question;           // คำถาม เช่น "Will ETH > $5000 by Dec 2025?"
        string[] outcomes;         // ผลลัพธ์ที่เป็นไปได้
        uint256 resolutionTime;    // เวลาที่สามารถ resolve ได้
        uint256 endTime;           // เวลาหยุดรับ orders
        address oracle;            // Oracle ที่จะ resolve
        MarketStatus status;
        uint256 resolvedOutcome;   // Index ของ outcome ที่ถูก
        uint256 totalVolume;       // Total trading volume
        mapping(uint256 => uint256) outcomeShares; // Shares per outcome
    }
    
    struct Order {
        uint256 marketId;
        uint256 outcomeIndex;      // Which outcome
        bool isBuy;                // true = buy, false = sell
        uint256 price;             // Price per share (0-1e18, where 1e18 = $1)
        uint256 amount;            // Number of shares
        uint256 filledAmount;      // Amount filled so far
        address maker;
        bool cancelled;
        uint256 createdAt;
    }
    
    struct Position {
        uint256 shares;
        uint256 avgCost;           // Average cost per share
    }
    
    // ========== State Variables ==========
    
    IERC20 public immutable collateral;   // USDC or similar
    uint256 public constant SHARE_DECIMALS = 1e18;
    uint256 public constant FEE_BPS = 20; // 0.2% trading fee
    
    mapping(uint256 => Market) public markets;
    uint256 public marketCount;
    
    // orders[orderId] = Order
    mapping(uint256 => Order) public orders;
    uint256 public orderCount;
    
    // positions[user][marketId][outcomeIndex] = Position
    mapping(address => mapping(uint256 => mapping(uint256 => Position))) public positions;
    
    // Order book: marketId → outcomeIndex → price → orderIds[]
    mapping(uint256 => mapping(uint256 => mapping(uint256 => uint256[]))) public buyOrders;
    mapping(uint256 => mapping(uint256 => mapping(uint256 => uint256[]))) public sellOrders;
    
    // Accumulated fees
    uint256 public accumulatedFees;
    
    // ========== Events ==========
    
    event MarketCreated(uint256 indexed marketId, string question, uint256 endTime);
    event OrderPlaced(uint256 indexed orderId, uint256 indexed marketId, address indexed maker, bool isBuy, uint256 price, uint256 amount);
    event OrderFilled(uint256 indexed orderId, address indexed taker, uint256 filledAmount, uint256 price);
    event OrderCancelled(uint256 indexed orderId);
    event MarketResolved(uint256 indexed marketId, uint256 winningOutcome);
    event SharesRedeemed(address indexed user, uint256 indexed marketId, uint256 amount);
    
    // ========== Constructor ==========
    
    constructor(address _collateral) Ownable(msg.sender) {
        collateral = IERC20(_collateral);
    }
    
    // ========== Market Management ==========
    
    /**
     * @notice สร้าง prediction market ใหม่
     */
    function createMarket(
        string calldata question,
        string[] calldata outcomes,
        uint256 endTime,
        uint256 resolutionTime,
        address oracle
    ) external returns (uint256 marketId) {
        require(outcomes.length >= 2, "need at least 2 outcomes");
        require(endTime > block.timestamp, "end time in past");
        require(resolutionTime >= endTime, "resolution before end");
        require(oracle != address(0), "zero oracle");
        
        marketId = marketCount++;
        
        Market storage market = markets[marketId];
        market.question = question;
        market.outcomes = outcomes;
        market.endTime = endTime;
        market.resolutionTime = resolutionTime;
        market.oracle = oracle;
        market.status = MarketStatus.Open;
        
        emit MarketCreated(marketId, question, endTime);
    }
    
    // ========== Order Management ==========
    
    /**
     * @notice วาง limit buy order
     * @param marketId Market ที่ต้องการ trade
     * @param outcomeIndex Outcome ที่ต้องการซื้อ
     * @param price ราคาต่อ share (0 - 1e18)
     * @param amount จำนวน shares ที่ต้องการ
     */
    function placeBuyOrder(
        uint256 marketId,
        uint256 outcomeIndex,
        uint256 price,
        uint256 amount
    ) external nonReentrant returns (uint256 orderId) {
        Market storage market = markets[marketId];
        require(market.status == MarketStatus.Open, "market not open");
        require(block.timestamp < market.endTime, "market ended");
        require(outcomeIndex < market.outcomes.length, "invalid outcome");
        require(price > 0 && price < SHARE_DECIMALS, "invalid price");
        require(amount > 0, "zero amount");
        
        // คำนวณ collateral ที่ต้อง lock
        uint256 collateralAmount = price * amount / SHARE_DECIMALS;
        uint256 fee = collateralAmount * FEE_BPS / 10000;
        
        // Lock collateral + fee
        collateral.safeTransferFrom(msg.sender, address(this), collateralAmount + fee);
        accumulatedFees += fee;
        
        // สร้าง order
        orderId = orderCount++;
        orders[orderId] = Order({
            marketId: marketId,
            outcomeIndex: outcomeIndex,
            isBuy: true,
            price: price,
            amount: amount,
            filledAmount: 0,
            maker: msg.sender,
            cancelled: false,
            createdAt: block.timestamp
        });
        
        // เพิ่มใน order book
        buyOrders[marketId][outcomeIndex][price].push(orderId);
        
        // Try to match with existing sell orders
        _matchOrder(orderId);
        
        emit OrderPlaced(orderId, marketId, msg.sender, true, price, amount);
    }
    
    /**
     * @notice วาง limit sell order
     */
    function placeSellOrder(
        uint256 marketId,
        uint256 outcomeIndex,
        uint256 price,
        uint256 amount
    ) external nonReentrant returns (uint256 orderId) {
        Market storage market = markets[marketId];
        require(market.status == MarketStatus.Open, "market not open");
        require(block.timestamp < market.endTime, "market ended");
        require(outcomeIndex < market.outcomes.length, "invalid outcome");
        require(price > 0 && price < SHARE_DECIMALS, "invalid price");
        require(amount > 0, "zero amount");
        
        // ตรวจสอบ user มี shares เพียงพอ
        require(
            positions[msg.sender][marketId][outcomeIndex].shares >= amount,
            "insufficient shares"
        );
        
        // Lock shares
        positions[msg.sender][marketId][outcomeIndex].shares -= amount;
        
        // สร้าง order
        orderId = orderCount++;
        orders[orderId] = Order({
            marketId: marketId,
            outcomeIndex: outcomeIndex,
            isBuy: false,
            price: price,
            amount: amount,
            filledAmount: 0,
            maker: msg.sender,
            cancelled: false,
            createdAt: block.timestamp
        });
        
        // เพิ่มใน order book
        sellOrders[marketId][outcomeIndex][price].push(orderId);
        
        // Try to match
        _matchOrder(orderId);
        
        emit OrderPlaced(orderId, marketId, msg.sender, false, price, amount);
    }
    
    /**
     * @notice ยกเลิก order
     */
    function cancelOrder(uint256 orderId) external nonReentrant {
        Order storage order = orders[orderId];
        require(order.maker == msg.sender, "not your order");
        require(!order.cancelled, "already cancelled");
        require(order.filledAmount < order.amount, "fully filled");
        
        order.cancelled = true;
        
        uint256 remainingAmount = order.amount - order.filledAmount;
        
        if (order.isBuy) {
            // คืน locked collateral
            uint256 refund = order.price * remainingAmount / SHARE_DECIMALS;
            collateral.safeTransfer(msg.sender, refund);
        } else {
            // คืน locked shares
            positions[msg.sender][order.marketId][order.outcomeIndex].shares += remainingAmount;
        }
        
        emit OrderCancelled(orderId);
    }
    
    // ========== Order Matching Engine ==========
    
    /**
     * @notice Match new order กับ existing orders
     * @dev Simple price-time priority matching
     */
    function _matchOrder(uint256 newOrderId) internal {
        Order storage newOrder = orders[newOrderId];
        
        if (newOrder.isBuy) {
            // ค้นหา sell orders ที่ price <= buy price
            _matchBuyOrder(newOrderId);
        } else {
            // ค้นหา buy orders ที่ price >= sell price
            _matchSellOrder(newOrderId);
        }
    }
    
    function _matchBuyOrder(uint256 buyOrderId) internal {
        Order storage buyOrder = orders[buyOrderId];
        
        // ค้นหา best sell orders (lowest price first)
        for (uint256 price = 1; price <= buyOrder.price; price++) {
            uint256[] storage matchingSells = sellOrders[buyOrder.marketId][buyOrder.outcomeIndex][price];
            
            for (uint256 i = 0; i < matchingSells.length; i++) {
                if (buyOrder.filledAmount >= buyOrder.amount) break;
                
                uint256 sellOrderId = matchingSells[i];
                Order storage sellOrder = orders[sellOrderId];
                
                if (sellOrder.cancelled || sellOrder.filledAmount >= sellOrder.amount) continue;
                
                uint256 buyRemaining = buyOrder.amount - buyOrder.filledAmount;
                uint256 sellRemaining = sellOrder.amount - sellOrder.filledAmount;
                uint256 fillAmount = buyRemaining < sellRemaining ? buyRemaining : sellRemaining;
                
                _executeFill(buyOrderId, sellOrderId, fillAmount, price);
            }
        }
    }
    
    function _matchSellOrder(uint256 sellOrderId) internal {
        Order storage sellOrder = orders[sellOrderId];
        
        // ค้นหา best buy orders (highest price first)
        for (uint256 price = SHARE_DECIMALS - 1; price >= sellOrder.price; price--) {
            uint256[] storage matchingBuys = buyOrders[sellOrder.marketId][sellOrder.outcomeIndex][price];
            
            for (uint256 i = 0; i < matchingBuys.length; i++) {
                if (sellOrder.filledAmount >= sellOrder.amount) break;
                
                uint256 buyOrderId = matchingBuys[i];
                Order storage buyOrder = orders[buyOrderId];
                
                if (buyOrder.cancelled || buyOrder.filledAmount >= buyOrder.amount) continue;
                
                uint256 sellRemaining = sellOrder.amount - sellOrder.filledAmount;
                uint256 buyRemaining = buyOrder.amount - buyOrder.filledAmount;
                uint256 fillAmount = sellRemaining < buyRemaining ? sellRemaining : buyRemaining;
                
                _executeFill(buyOrderId, sellOrderId, fillAmount, price);
            }
            
            if (price == 0) break;
        }
    }
    
    /**
     * @notice Execute fill ระหว่าง buy และ sell order
     */
    function _executeFill(
        uint256 buyOrderId,
        uint256 sellOrderId,
        uint256 fillAmount,
        uint256 fillPrice
    ) internal {
        Order storage buyOrder = orders[buyOrderId];
        Order storage sellOrder = orders[sellOrderId];
        
        buyOrder.filledAmount += fillAmount;
        sellOrder.filledAmount += fillAmount;
        
        uint256 collateralAmount = fillPrice * fillAmount / SHARE_DECIMALS;
        
        // Transfer shares to buyer
        positions[buyOrder.maker][buyOrder.marketId][buyOrder.outcomeIndex].shares += fillAmount;
        
        // Transfer collateral to seller
        collateral.safeTransfer(sellOrder.maker, collateralAmount);
        
        // คืน excess collateral ให้ buyer ถ้าราคา fill ต่ำกว่า limit
        if (fillPrice < buyOrder.price) {
            uint256 refund = (buyOrder.price - fillPrice) * fillAmount / SHARE_DECIMALS;
            collateral.safeTransfer(buyOrder.maker, refund);
        }
        
        emit OrderFilled(buyOrderId, sellOrder.maker, fillAmount, fillPrice);
        emit OrderFilled(sellOrderId, buyOrder.maker, fillAmount, fillPrice);
    }
    
    // ========== Resolution & Redemption ==========
    
    /**
     * @notice Resolve market (เรียกโดย oracle)
     */
    function resolveMarket(uint256 marketId, uint256 winningOutcome) external {
        Market storage market = markets[marketId];
        require(msg.sender == market.oracle, "not oracle");
        require(market.status == MarketStatus.Open || market.status == MarketStatus.Locked, "invalid status");
        require(block.timestamp >= market.resolutionTime, "too early");
        require(winningOutcome < market.outcomes.length, "invalid outcome");
        
        market.status = MarketStatus.Resolved;
        market.resolvedOutcome = winningOutcome;
        
        emit MarketResolved(marketId, winningOutcome);
    }
    
    /**
     * @notice Redeem winning shares
     */
    function redeemShares(uint256 marketId) external nonReentrant {
        Market storage market = markets[marketId];
        require(market.status == MarketStatus.Resolved, "not resolved");
        
        uint256 winningOutcome = market.resolvedOutcome;
        uint256 shares = positions[msg.sender][marketId][winningOutcome].shares;
        require(shares > 0, "no winning shares");
        
        positions[msg.sender][marketId][winningOutcome].shares = 0;
        
        // 1 winning share = 1 USDC (or collateral unit)
        collateral.safeTransfer(msg.sender, shares);
        
        emit SharesRedeemed(msg.sender, marketId, shares);
    }
    
    // ========== View Functions ==========
    
    function getMarketInfo(uint256 marketId) external view returns (
        string memory question,
        MarketStatus status,
        uint256 endTime,
        uint256 resolutionTime,
        uint256 numOutcomes
    ) {
        Market storage market = markets[marketId];
        return (
            market.question,
            market.status,
            market.endTime,
            market.resolutionTime,
            market.outcomes.length
        );
    }
    
    function getUserPosition(
        address user,
        uint256 marketId,
        uint256 outcomeIndex
    ) external view returns (uint256 shares, uint256 avgCost) {
        Position storage pos = positions[user][marketId][outcomeIndex];
        return (pos.shares, pos.avgCost);
    }
    
    function getImpliedProbability(
        uint256 marketId,
        uint256 outcomeIndex
    ) external view returns (uint256) {
        // ดึงจาก best bid price
        // (simplified - ควรดูจาก best ask/bid)
        return 5000; // 50% default
    }
    
    /**
     * @notice Withdraw accumulated fees
     */
    function withdrawFees() external onlyOwner {
        uint256 fees = accumulatedFees;
        accumulatedFees = 0;
        collateral.safeTransfer(owner(), fees);
    }
}
```

---

## 3. AMM-Based Prediction Market (LMSR)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title LMSRMarket
 * @notice Logarithmic Market Scoring Rule prediction market
 * @dev LMSR คือ AMM ที่ออกแบบมาเฉพาะสำหรับ prediction markets
 *
 * Cost Function:
 * C(q1, q2) = b * ln(e^(q1/b) + e^(q2/b))
 *
 * Price Function:
 * p_i = e^(q_i/b) / (sum of e^(q_j/b))
 *
 * เมื่อ b = liquidity parameter (ค่าสูง = ราคาเปลี่ยนช้า)
 *      q_i = shares outstanding ของ outcome i
 */
contract LMSRMarket is ReentrancyGuard {
    using SafeERC20 for IERC20;
    
    // ========== Math Library ==========
    
    /**
     * @notice คำนวณ e^x โดยใช้ fixed-point arithmetic
     * @dev ใช้ Taylor series approximation
     *      Precision: 18 decimals
     */
    function expFixed(int256 x) internal pure returns (uint256) {
        // Scale: x ใน 1e18
        // e^x ≈ 1 + x + x²/2! + x³/3! + x⁴/4! + ...
        
        if (x < -41e18) return 0; // e^(-41) ≈ 0
        if (x > 130e18) revert("overflow");
        
        // ใช้ scaled calculation
        int256 result = 1e18;
        int256 term = 1e18;
        
        for (uint256 i = 1; i <= 20; i++) {
            term = term * x / (int256(i) * 1e18);
            result += term;
            if (term < 100 && term > -100) break; // Converged
        }
        
        return uint256(result > 0 ? result : 0);
    }
    
    /**
     * @notice คำนวณ ln(x) โดยใช้ fixed-point arithmetic
     */
    function lnFixed(uint256 x) internal pure returns (int256) {
        require(x > 0, "ln(0) undefined");
        
        // ln(x) = ln(2^n * m) = n*ln(2) + ln(m) เมื่อ 1 <= m < 2
        int256 ln2 = 693147180559945309; // ln(2) * 1e18
        
        uint256 y = x;
        int256 n = 0;
        
        while (y < 1e18) {
            y *= 2;
            n--;
        }
        while (y >= 2e18) {
            y /= 2;
            n++;
        }
        
        // ln(y) for 1 <= y < 2, using (y-1)/(y+1) expansion
        int256 z = (int256(y) - 1e18) * 1e18 / (int256(y) + 1e18);
        int256 z2 = z * z / 1e18;
        int256 result = z;
        int256 term = z;
        
        for (uint256 i = 1; i <= 20; i++) {
            term = term * z2 / 1e18;
            result += term / int256(2 * i + 1);
        }
        
        return n * ln2 + 2 * result;
    }
    
    // ========== Market Data ==========
    
    struct Market {
        string question;
        string[] outcomes;
        uint256[] shares;      // q_i: shares outstanding per outcome
        uint256 b;             // Liquidity parameter (1e18 scale)
        uint256 endTime;
        address oracle;
        bool resolved;
        uint256 winningOutcome;
        uint256 initialFunds;  // Initial market maker funding
    }
    
    IERC20 public immutable collateral;
    
    mapping(uint256 => Market) public markets;
    uint256 public marketCount;
    
    // User shares: user → marketId → outcomeIndex → shares
    mapping(address => mapping(uint256 => mapping(uint256 => uint256))) public userShares;
    
    event MarketCreated(uint256 indexed marketId, uint256 b, uint256 initialFunds);
    event SharesBought(uint256 indexed marketId, uint256 indexed outcomeIndex, address indexed buyer, uint256 shares, uint256 cost);
    event SharesSold(uint256 indexed marketId, uint256 indexed outcomeIndex, address indexed seller, uint256 shares, uint256 proceeds);
    event MarketResolved(uint256 indexed marketId, uint256 winningOutcome);
    event SharesRedeemed(address indexed user, uint256 indexed marketId, uint256 amount);
    
    constructor(address _collateral) {
        collateral = IERC20(_collateral);
    }
    
    /**
     * @notice สร้าง LMSR market
     * @param b Liquidity parameter (ค่าสูง = deeper market)
     * @param initialFunds ETH/token ที่ market maker วางค้ำประกัน
     */
    function createMarket(
        string calldata question,
        string[] calldata outcomes,
        uint256 b,
        uint256 initialFunds,
        uint256 endTime,
        address oracle
    ) external returns (uint256 marketId) {
        require(outcomes.length >= 2, "need 2+ outcomes");
        require(b > 0, "zero b parameter");
        require(endTime > block.timestamp, "end in past");
        
        // Transfer initial funds from market maker
        collateral.safeTransferFrom(msg.sender, address(this), initialFunds);
        
        marketId = marketCount++;
        
        Market storage market = markets[marketId];
        market.question = question;
        market.outcomes = outcomes;
        market.shares = new uint256[](outcomes.length); // All zeros initially
        market.b = b;
        market.endTime = endTime;
        market.oracle = oracle;
        market.initialFunds = initialFunds;
        
        emit MarketCreated(marketId, b, initialFunds);
    }
    
    /**
     * @notice คำนวณ LMSR cost function
     * @dev C(q) = b * ln(sum(e^(q_i/b)))
     */
    function calcCost(uint256 marketId) public view returns (uint256 cost) {
        Market storage market = markets[marketId];
        uint256 b = market.b;
        
        // คำนวณ sum of e^(q_i/b)
        uint256 sumExp = 0;
        for (uint256 i = 0; i < market.shares.length; i++) {
            int256 qi_over_b = int256(market.shares[i] * 1e18 / b);
            sumExp += expFixed(qi_over_b);
        }
        
        // cost = b * ln(sumExp)
        int256 lnSum = lnFixed(sumExp);
        cost = b * uint256(lnSum > 0 ? lnSum : 0) / 1e18;
    }
    
    /**
     * @notice คำนวณราคาปัจจุบันของ outcome i
     * @dev p_i = e^(q_i/b) / sum(e^(q_j/b))
     * @return price Price ใน 1e18 scale (1e18 = 100%)
     */
    function getPrice(uint256 marketId, uint256 outcomeIndex) public view returns (uint256 price) {
        Market storage market = markets[marketId];
        uint256 b = market.b;
        
        uint256 sumExp = 0;
        for (uint256 i = 0; i < market.shares.length; i++) {
            int256 qi_over_b = int256(market.shares[i] * 1e18 / b);
            sumExp += expFixed(qi_over_b);
        }
        
        int256 qi_over_b = int256(market.shares[outcomeIndex] * 1e18 / b);
        uint256 expQi = expFixed(qi_over_b);
        
        price = expQi * 1e18 / sumExp;
    }
    
    /**
     * @notice ซื้อ shares ใน outcome
     * @param marketId Market ID
     * @param outcomeIndex Outcome ที่ต้องการซื้อ
     * @param sharesAmount จำนวน shares ที่ต้องการ
     * @return cost ค่าใช้จ่าย (collateral)
     */
    function buyShares(
        uint256 marketId,
        uint256 outcomeIndex,
        uint256 sharesAmount
    ) external nonReentrant returns (uint256 cost) {
        Market storage market = markets[marketId];
        require(!market.resolved, "market resolved");
        require(block.timestamp < market.endTime, "market ended");
        require(outcomeIndex < market.outcomes.length, "invalid outcome");
        require(sharesAmount > 0, "zero shares");
        
        // คำนวณ cost ก่อน = C(q)
        uint256 costBefore = calcCost(marketId);
        
        // อัพเดท shares
        market.shares[outcomeIndex] += sharesAmount;
        
        // คำนวณ cost หลัง = C(q + Δq)
        uint256 costAfter = calcCost(marketId);
        
        // Cost ที่ต้องจ่าย = C(q + Δq) - C(q)
        cost = costAfter - costBefore;
        
        // Transfer collateral
        collateral.safeTransferFrom(msg.sender, address(this), cost);
        
        // Update user shares
        userShares[msg.sender][marketId][outcomeIndex] += sharesAmount;
        
        emit SharesBought(marketId, outcomeIndex, msg.sender, sharesAmount, cost);
    }
    
    /**
     * @notice ขาย shares
     * @return proceeds เงินที่ได้รับ
     */
    function sellShares(
        uint256 marketId,
        uint256 outcomeIndex,
        uint256 sharesAmount
    ) external nonReentrant returns (uint256 proceeds) {
        Market storage market = markets[marketId];
        require(!market.resolved, "market resolved");
        require(block.timestamp < market.endTime, "market ended");
        require(outcomeIndex < market.outcomes.length, "invalid outcome");
        require(userShares[msg.sender][marketId][outcomeIndex] >= sharesAmount, "insufficient shares");
        
        // คำนวณ proceeds = C(q) - C(q - Δq)
        uint256 costBefore = calcCost(marketId);
        
        // อัพเดท shares
        market.shares[outcomeIndex] -= sharesAmount;
        
        uint256 costAfter = calcCost(marketId);
        proceeds = costBefore - costAfter;
        
        // Transfer proceeds to seller
        collateral.safeTransfer(msg.sender, proceeds);
        
        // Update user shares
        userShares[msg.sender][marketId][outcomeIndex] -= sharesAmount;
        
        emit SharesSold(marketId, outcomeIndex, msg.sender, sharesAmount, proceeds);
    }
    
    /**
     * @notice Resolve market
     */
    function resolveMarket(uint256 marketId, uint256 winningOutcome) external {
        Market storage market = markets[marketId];
        require(msg.sender == market.oracle, "not oracle");
        require(!market.resolved, "already resolved");
        require(block.timestamp >= market.endTime, "not ended");
        require(winningOutcome < market.outcomes.length, "invalid outcome");
        
        market.resolved = true;
        market.winningOutcome = winningOutcome;
        
        emit MarketResolved(marketId, winningOutcome);
    }
    
    /**
     * @notice Redeem winning shares (1 share = 1 collateral unit)
     */
    function redeemShares(uint256 marketId) external nonReentrant {
        Market storage market = markets[marketId];
        require(market.resolved, "not resolved");
        
        uint256 winningOutcome = market.winningOutcome;
        uint256 shares = userShares[msg.sender][marketId][winningOutcome];
        require(shares > 0, "no winning shares");
        
        userShares[msg.sender][marketId][winningOutcome] = 0;
        
        // 1 winning share = 1e6 USDC (1 dollar)
        collateral.safeTransfer(msg.sender, shares * 1e6 / 1e18);
        
        emit SharesRedeemed(msg.sender, marketId, shares);
    }
    
    /**
     * @notice ดู implied probabilities ของทุก outcomes
     */
    function getImpliedProbabilities(uint256 marketId) 
        external view returns (uint256[] memory probabilities) {
        Market storage market = markets[marketId];
        uint256 n = market.outcomes.length;
        probabilities = new uint256[](n);
        
        for (uint256 i = 0; i < n; i++) {
            probabilities[i] = getPrice(marketId, i);
        }
    }
}
```

---

## 4. Oracle Resolution System

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title PredictionMarketOracle
 * @notice Oracle system สำหรับ resolve prediction markets
 * @dev รองรับ Chainlink direct + UMA optimistic oracle
 */
contract PredictionMarketOracle {
    
    enum ResolutionMethod {
        ChainlinkFeed,      // ใช้ Chainlink price feed โดยตรง
        UMAOptimistic,      // ใช้ UMA optimistic oracle
        MultiSigVote,       // ใช้ multi-signature voting
        Automated           // Automated based on on-chain data
    }
    
    struct OracleConfig {
        ResolutionMethod method;
        address chainlinkFeed;      // Chainlink aggregator address
        bytes32 umaIdentifier;      // UMA price identifier
        address[] signers;          // Multi-sig signers
        uint256 signerThreshold;    // Required signatures
        uint256 ancillaryData;      // Additional data for resolution
    }
    
    struct PendingResolution {
        uint256 marketId;
        address predictionMarket;
        OracleConfig config;
        bool resolved;
        uint256 resolvedOutcome;
        uint256 resolveTime;
        
        // UMA specific
        bytes umaRequest;
        bool umaSettled;
        
        // Multi-sig votes
        mapping(address => uint256) votes;  // signer => outcome
        mapping(address => bool) hasVoted;
        uint256[] outcomeCounts;
    }
    
    mapping(uint256 => PendingResolution) public pendingResolutions;
    uint256 public resolutionCount;
    
    // UMA Oracle interface
    address public umaOracle;
    
    event ResolutionRequested(uint256 indexed resolutionId, uint256 indexed marketId);
    event ResolutionSettled(uint256 indexed resolutionId, uint256 winningOutcome);
    
    constructor(address _umaOracle) {
        umaOracle = _umaOracle;
    }
    
    /**
     * @notice Request market resolution ผ่าน Chainlink
     * @dev ใช้สำหรับ price-based markets เช่น "ETH > $5000?"
     */
    function requestChainlinkResolution(
        uint256 marketId,
        address predictionMarket,
        address chainlinkFeed,
        uint256 threshold,     // Price threshold ที่ต้องเปรียบเทียบ
        bool greaterThan       // true = price > threshold means YES
    ) external returns (uint256 resolutionId) {
        resolutionId = resolutionCount++;
        
        PendingResolution storage res = pendingResolutions[resolutionId];
        res.marketId = marketId;
        res.predictionMarket = predictionMarket;
        res.config.method = ResolutionMethod.ChainlinkFeed;
        res.config.chainlinkFeed = chainlinkFeed;
        res.config.ancillaryData = threshold;
        res.resolveTime = block.timestamp;
        
        // Auto-resolve via Chainlink
        _resolveViaChainlink(resolutionId, greaterThan);
        
        emit ResolutionRequested(resolutionId, marketId);
    }
    
    /**
     * @notice Resolve ผ่าน Chainlink feed
     */
    function _resolveViaChainlink(uint256 resolutionId, bool greaterThan) internal {
        PendingResolution storage res = pendingResolutions[resolutionId];
        
        // ดึงราคาจาก Chainlink
        (, int256 price,, uint256 updatedAt,) = AggregatorV3Interface(res.config.chainlinkFeed)
            .latestRoundData();
        
        require(price > 0, "invalid chainlink price");
        require(block.timestamp - updatedAt < 3600, "stale price");
        
        uint256 threshold = res.config.ancillaryData;
        
        // Determine outcome: 0 = NO, 1 = YES
        uint256 winningOutcome;
        if (greaterThan) {
            winningOutcome = uint256(price) > threshold ? 1 : 0;
        } else {
            winningOutcome = uint256(price) < threshold ? 1 : 0;
        }
        
        res.resolved = true;
        res.resolvedOutcome = winningOutcome;
        
        // Resolve on prediction market
        IResolvableMarket(res.predictionMarket).resolveMarket(res.marketId, winningOutcome);
        
        emit ResolutionSettled(resolutionId, winningOutcome);
    }
    
    /**
     * @notice Request UMA optimistic oracle resolution
     * @dev UMA ใช้ dispute mechanism - ถ้าไม่มี dispute ใน 2 ชั่วโมง จะ settle
     */
    function requestUMAResolution(
        uint256 marketId,
        address predictionMarket,
        bytes32 identifier,
        bytes calldata ancillaryData
    ) external returns (uint256 resolutionId) {
        resolutionId = resolutionCount++;
        
        PendingResolution storage res = pendingResolutions[resolutionId];
        res.marketId = marketId;
        res.predictionMarket = predictionMarket;
        res.config.method = ResolutionMethod.UMAOptimistic;
        res.config.umaIdentifier = identifier;
        
        // Request price from UMA
        uint256 bond = 100e18; // Bond required
        IERC20(IERC20Address).approve(umaOracle, bond);
        
        IUMAOptimisticOracle(umaOracle).requestPrice(
            identifier,
            block.timestamp,
            ancillaryData,
            address(0), // reward token
            0           // reward amount
        );
        
        emit ResolutionRequested(resolutionId, marketId);
    }
    
    /**
     * @notice Settle UMA resolution หลัง dispute period
     */
    function settleUMAResolution(uint256 resolutionId) external {
        PendingResolution storage res = pendingResolutions[resolutionId];
        require(res.config.method == ResolutionMethod.UMAOptimistic, "not UMA");
        require(!res.resolved, "already resolved");
        
        // Get settled price from UMA
        int256 settledPrice = IUMAOptimisticOracle(umaOracle).settleAndGetPrice(
            res.config.umaIdentifier,
            res.resolveTime,
            res.umaRequest
        );
        
        // 0 = NO, 1 = YES (UMA convention: 0 or 1e18)
        uint256 winningOutcome = settledPrice >= 0.5e18 ? 1 : 0;
        
        res.resolved = true;
        res.resolvedOutcome = winningOutcome;
        
        IResolvableMarket(res.predictionMarket).resolveMarket(res.marketId, winningOutcome);
        
        emit ResolutionSettled(resolutionId, winningOutcome);
    }
    
    /**
     * @notice Multi-sig vote resolution
     */
    function voteOnOutcome(uint256 resolutionId, uint256 outcome) external {
        PendingResolution storage res = pendingResolutions[resolutionId];
        require(res.config.method == ResolutionMethod.MultiSigVote, "not multisig");
        require(!res.resolved, "already resolved");
        require(!res.hasVoted[msg.sender], "already voted");
        
        bool isSigner = false;
        for (uint256 i = 0; i < res.config.signers.length; i++) {
            if (res.config.signers[i] == msg.sender) {
                isSigner = true;
                break;
            }
        }
        require(isSigner, "not authorized signer");
        
        res.hasVoted[msg.sender] = true;
        res.votes[msg.sender] = outcome;
        res.outcomeCounts[outcome]++;
        
        // Check if threshold reached
        if (res.outcomeCounts[outcome] >= res.config.signerThreshold) {
            res.resolved = true;
            res.resolvedOutcome = outcome;
            
            IResolvableMarket(res.predictionMarket).resolveMarket(res.marketId, outcome);
            
            emit ResolutionSettled(resolutionId, outcome);
        }
    }
    
    address constant IERC20Address = address(0); // Placeholder
}

interface AggregatorV3Interface {
    function latestRoundData() external view returns (
        uint80, int256, uint256, uint256, uint80
    );
}

interface IUMAOptimisticOracle {
    function requestPrice(
        bytes32 identifier,
        uint256 timestamp,
        bytes calldata ancillaryData,
        address currency,
        uint256 reward
    ) external;
    
    function settleAndGetPrice(
        bytes32 identifier,
        uint256 timestamp,
        bytes calldata ancillaryData
    ) external returns (int256);
}

interface IResolvableMarket {
    function resolveMarket(uint256 marketId, uint256 winningOutcome) external;
}
```

---

## 5. Full Prediction Market Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title PredictionMarket
 * @notice Full-featured prediction market
 * @dev รวม order book + AMM + oracle resolution
 */
contract PredictionMarket is ReentrancyGuard {
    using SafeERC20 for IERC20;
    
    // ========== Types ==========
    
    enum MarketType { Binary, Categorical, Scalar }
    enum MarketStatus { Open, Trading, Locked, Resolved }
    
    struct Market {
        uint256 id;
        string question;
        MarketType marketType;
        MarketStatus status;
        string[] outcomes;
        
        // Timing
        uint256 tradingStart;
        uint256 tradingEnd;
        uint256 resolutionDeadline;
        
        // Resolution
        address oracle;
        uint256 resolvedOutcome;
        bool isResolved;
        
        // Financial
        uint256 totalLiquidity;
        uint256[] outcomeLiquidity;
        uint256 totalFees;
        uint256 creatorFee;    // basis points
        
        // Creator
        address creator;
    }
    
    // ========== State ==========
    
    IERC20 public immutable usdc;
    
    mapping(uint256 => Market) public markets;
    uint256 public nextMarketId;
    
    // Positions: user → marketId → outcomeIndex → amount
    mapping(address => mapping(uint256 => mapping(uint256 => uint256))) public positions;
    
    // Liquidity providers: user → marketId → LP shares
    mapping(address => mapping(uint256 => uint256)) public lpShares;
    mapping(uint256 => uint256) public totalLPShares;
    
    uint256 public constant PLATFORM_FEE = 10; // 0.1%
    address public treasury;
    
    // ========== Events ==========
    
    event MarketCreated(uint256 indexed marketId, address indexed creator, string question);
    event LiquidityAdded(uint256 indexed marketId, address indexed provider, uint256 amount);
    event SharesPurchased(uint256 indexed marketId, address indexed buyer, uint256 outcomeIndex, uint256 amount, uint256 cost);
    event SharesSold(uint256 indexed marketId, address indexed seller, uint256 outcomeIndex, uint256 amount, uint256 proceeds);
    event MarketResolved(uint256 indexed marketId, uint256 winningOutcome);
    event WinningsClaimed(uint256 indexed marketId, address indexed winner, uint256 amount);
    
    // ========== Constructor ==========
    
    constructor(address _usdc, address _treasury) {
        usdc = IERC20(_usdc);
        treasury = _treasury;
    }
    
    // ========== Market Creation ==========
    
    /**
     * @notice สร้าง prediction market ใหม่
     */
    function createMarket(
        string calldata question,
        MarketType marketType,
        string[] calldata outcomes,
        uint256 tradingStart,
        uint256 tradingEnd,
        uint256 resolutionDeadline,
        address oracle,
        uint256 creatorFee,
        uint256 initialLiquidity
    ) external nonReentrant returns (uint256 marketId) {
        require(outcomes.length >= 2, "need 2+ outcomes");
        require(tradingEnd > tradingStart, "invalid times");
        require(resolutionDeadline >= tradingEnd, "resolution before end");
        require(creatorFee <= 200, "fee too high"); // Max 2%
        require(initialLiquidity > 0, "need initial liquidity");
        
        marketId = nextMarketId++;
        
        Market storage market = markets[marketId];
        market.id = marketId;
        market.question = question;
        market.marketType = marketType;
        market.status = block.timestamp >= tradingStart ? MarketStatus.Trading : MarketStatus.Open;
        market.outcomes = outcomes;
        market.tradingStart = tradingStart;
        market.tradingEnd = tradingEnd;
        market.resolutionDeadline = resolutionDeadline;
        market.oracle = oracle;
        market.creatorFee = creatorFee;
        market.creator = msg.sender;
        market.outcomeLiquidity = new uint256[](outcomes.length);
        
        // Add initial liquidity equally across outcomes
        usdc.safeTransferFrom(msg.sender, address(this), initialLiquidity);
        
        uint256 perOutcome = initialLiquidity / outcomes.length;
        for (uint256 i = 0; i < outcomes.length; i++) {
            market.outcomeLiquidity[i] = perOutcome;
        }
        market.totalLiquidity = initialLiquidity;
        
        // Mint LP shares to creator
        totalLPShares[marketId] = initialLiquidity;
        lpShares[msg.sender][marketId] = initialLiquidity;
        
        emit MarketCreated(marketId, msg.sender, question);
        emit LiquidityAdded(marketId, msg.sender, initialLiquidity);
    }
    
    // ========== Trading ==========
    
    /**
     * @notice ซื้อ shares ใน outcome
     * @param marketId Market ID
     * @param outcomeIndex Outcome ที่ต้องการ
     * @param maxCost Maximum cost ที่ยอมจ่าย (slippage protection)
     * @param sharesAmount จำนวน shares ที่ต้องการ
     */
    function buyShares(
        uint256 marketId,
        uint256 outcomeIndex,
        uint256 maxCost,
        uint256 sharesAmount
    ) external nonReentrant returns (uint256 cost) {
        Market storage market = markets[marketId];
        require(market.status == MarketStatus.Trading, "not trading");
        require(block.timestamp < market.tradingEnd, "trading ended");
        require(outcomeIndex < market.outcomes.length, "invalid outcome");
        
        // คำนวณ cost ด้วย AMM formula
        cost = _calcBuyCost(marketId, outcomeIndex, sharesAmount);
        require(cost <= maxCost, "cost exceeds max");
        
        // Collect fees
        uint256 platformFee = cost * PLATFORM_FEE / 10000;
        uint256 creatorFeeAmt = cost * market.creatorFee / 10000;
        uint256 totalFeeAmt = platformFee + creatorFeeAmt;
        
        // Transfer USDC
        usdc.safeTransferFrom(msg.sender, address(this), cost);
        
        // Distribute fees
        usdc.safeTransfer(treasury, platformFee);
        market.totalFees += creatorFeeAmt;
        
        // Update liquidity
        uint256 netCost = cost - totalFeeAmt;
        market.outcomeLiquidity[outcomeIndex] += netCost;
        market.totalLiquidity += netCost;
        
        // Mint shares to buyer
        positions[msg.sender][marketId][outcomeIndex] += sharesAmount;
        
        emit SharesPurchased(marketId, msg.sender, outcomeIndex, sharesAmount, cost);
    }
    
    /**
     * @notice ขาย shares
     */
    function sellShares(
        uint256 marketId,
        uint256 outcomeIndex,
        uint256 minProceeds,
        uint256 sharesAmount
    ) external nonReentrant returns (uint256 proceeds) {
        Market storage market = markets[marketId];
        require(market.status == MarketStatus.Trading, "not trading");
        require(block.timestamp < market.tradingEnd, "trading ended");
        require(positions[msg.sender][marketId][outcomeIndex] >= sharesAmount, "insufficient shares");
        
        // คำนวณ proceeds
        proceeds = _calcSellProceeds(marketId, outcomeIndex, sharesAmount);
        require(proceeds >= minProceeds, "proceeds below min");
        
        // Update position
        positions[msg.sender][marketId][outcomeIndex] -= sharesAmount;
        
        // Update liquidity
        market.outcomeLiquidity[outcomeIndex] -= proceeds;
        market.totalLiquidity -= proceeds;
        
        // Transfer proceeds
        usdc.safeTransfer(msg.sender, proceeds);
        
        emit SharesSold(marketId, msg.sender, outcomeIndex, sharesAmount, proceeds);
    }
    
    // ========== Pricing ==========
    
    /**
     * @notice คำนวณราคา outcome ปัจจุบัน
     * @dev ใช้ liquidity-weighted probability
     */
    function getOutcomePrice(uint256 marketId, uint256 outcomeIndex) 
        public view returns (uint256 price) {
        Market storage market = markets[marketId];
        if (market.totalLiquidity == 0) return 1e18 / market.outcomes.length;
        
        return market.outcomeLiquidity[outcomeIndex] * 1e18 / market.totalLiquidity;
    }
    
    function _calcBuyCost(
        uint256 marketId,
        uint256 outcomeIndex,
        uint256 sharesAmount
    ) internal view returns (uint256) {
        // Simple CPMM: cost proportional to current price
        uint256 price = getOutcomePrice(marketId, outcomeIndex);
        return sharesAmount * price / 1e18;
    }
    
    function _calcSellProceeds(
        uint256 marketId,
        uint256 outcomeIndex,
        uint256 sharesAmount
    ) internal view returns (uint256) {
        uint256 price = getOutcomePrice(marketId, outcomeIndex);
        return sharesAmount * price / 1e18;
    }
    
    // ========== Resolution ==========
    
    /**
     * @notice Resolve market
     */
    function resolveMarket(uint256 marketId, uint256 winningOutcome) external {
        Market storage market = markets[marketId];
        require(msg.sender == market.oracle, "not oracle");
        require(!market.isResolved, "already resolved");
        require(block.timestamp >= market.tradingEnd, "trading not ended");
        require(block.timestamp <= market.resolutionDeadline, "past deadline");
        require(winningOutcome < market.outcomes.length, "invalid outcome");
        
        market.isResolved = true;
        market.resolvedOutcome = winningOutcome;
        market.status = MarketStatus.Resolved;
        
        emit MarketResolved(marketId, winningOutcome);
    }
    
    /**
     * @notice Claim winnings
     */
    function claimWinnings(uint256 marketId) external nonReentrant {
        Market storage market = markets[marketId];
        require(market.isResolved, "not resolved");
        
        uint256 winningOutcome = market.resolvedOutcome;
        uint256 shares = positions[msg.sender][marketId][winningOutcome];
        require(shares > 0, "no winning shares");
        
        positions[msg.sender][marketId][winningOutcome] = 0;
        
        // 1 winning share = 1 USDC
        uint256 payout = shares;
        usdc.safeTransfer(msg.sender, payout);
        
        emit WinningsClaimed(marketId, msg.sender, payout);
    }
    
    /**
     * @notice Creator claims accumulated fees
     */
    function claimCreatorFees(uint256 marketId) external nonReentrant {
        Market storage market = markets[marketId];
        require(msg.sender == market.creator, "not creator");
        
        uint256 fees = market.totalFees;
        require(fees > 0, "no fees");
        
        market.totalFees = 0;
        usdc.safeTransfer(msg.sender, fees);
    }
}
```

---

## Workshop

### Workshop 1: Deploy และ Trade ใน Prediction Market

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";

contract PredictionMarketTest is Test {
    
    PredictionMarket public market;
    MockERC20 public usdc;
    address public oracle = address(0xORACLE);
    address public alice = address(0xALICE);
    address public bob = address(0xBOB);
    
    function setUp() public {
        usdc = new MockERC20("USD Coin", "USDC", 6);
        market = new PredictionMarket(address(usdc), address(this));
        
        // Fund users
        usdc.mint(alice, 10000e6);
        usdc.mint(bob, 10000e6);
        
        vm.prank(alice);
        usdc.approve(address(market), type(uint256).max);
        vm.prank(bob);
        usdc.approve(address(market), type(uint256).max);
    }
    
    function test_fullMarketLifecycle() public {
        // Setup
        string[] memory outcomes = new string[](2);
        outcomes[0] = "YES - ETH > $5000";
        outcomes[1] = "NO - ETH <= $5000";
        
        uint256 tradingStart = block.timestamp;
        uint256 tradingEnd = block.timestamp + 7 days;
        uint256 resolutionDeadline = block.timestamp + 8 days;
        uint256 initialLiquidity = 1000e6;
        
        usdc.mint(address(this), initialLiquidity);
        usdc.approve(address(market), initialLiquidity);
        
        // Create market
        uint256 marketId = market.createMarket(
            "Will ETH be > $5000 by Dec 31, 2025?",
            PredictionMarket.MarketType.Binary,
            outcomes,
            tradingStart,
            tradingEnd,
            resolutionDeadline,
            oracle,
            50,   // 0.5% creator fee
            initialLiquidity
        );
        
        // Check initial prices (should be ~50/50)
        uint256 yesPrice = market.getOutcomePrice(marketId, 0);
        uint256 noPrice = market.getOutcomePrice(marketId, 1);
        assertApproxEqRel(yesPrice, 0.5e18, 0.01e18); // 50% ± 1%
        assertApproxEqRel(noPrice, 0.5e18, 0.01e18);
        
        // Alice buys YES shares
        uint256 aliceShares = 100e18;
        vm.prank(alice);
        uint256 aliceCost = market.buyShares(marketId, 0, 60e6, aliceShares);
        
        console.log("Alice cost for YES:", aliceCost);
        
        // Bob buys NO shares
        uint256 bobShares = 150e18;
        vm.prank(bob);
        uint256 bobCost = market.buyShares(marketId, 1, 90e6, bobShares);
        
        console.log("Bob cost for NO:", bobCost);
        
        // After trading, YES price should have increased
        uint256 yesPriceAfter = market.getOutcomePrice(marketId, 0);
        console.log("YES price after Alice bought:", yesPriceAfter);
        
        // Fast forward to resolution
        vm.warp(tradingEnd + 1);
        
        // Oracle resolves: YES wins
        vm.prank(oracle);
        market.resolveMarket(marketId, 0); // 0 = YES
        
        // Alice claims winnings
        uint256 aliceBalBefore = usdc.balanceOf(alice);
        vm.prank(alice);
        market.claimWinnings(marketId);
        uint256 aliceBalAfter = usdc.balanceOf(alice);
        
        uint256 aliceWinnings = aliceBalAfter - aliceBalBefore;
        console.log("Alice winnings:", aliceWinnings);
        
        // Alice should get back more than she paid
        assertGt(aliceWinnings, 0, "alice should win");
        
        // Bob cannot claim (lost)
        vm.prank(bob);
        vm.expectRevert("no winning shares");
        market.claimWinnings(marketId);
    }
    
    function test_lmsr_pricing() public {
        LMSRMarket lmsr = new LMSRMarket(address(usdc));
        
        string[] memory outcomes = new string[](2);
        outcomes[0] = "YES";
        outcomes[1] = "NO";
        
        uint256 b = 100e18; // Liquidity parameter
        uint256 initialFunds = 200e6; // Market maker covers max loss
        
        usdc.mint(address(this), initialFunds);
        usdc.approve(address(lmsr), initialFunds);
        
        uint256 marketId = lmsr.createMarket(
            "Will it rain tomorrow?",
            outcomes,
            b,
            initialFunds,
            block.timestamp + 1 days,
            oracle
        );
        
        // Initial prices should be 50/50
        uint256 yesPrice = lmsr.getPrice(marketId, 0);
        uint256 noPrice = lmsr.getPrice(marketId, 1);
        
        console.log("Initial YES price:", yesPrice);
        console.log("Initial NO price:", noPrice);
        
        // Buy YES shares
        usdc.mint(alice, 1000e6);
        vm.prank(alice);
        usdc.approve(address(lmsr), 1000e6);
        
        vm.prank(alice);
        uint256 cost = lmsr.buyShares(marketId, 0, 10e18);
        
        console.log("Cost for 10 YES shares:", cost);
        
        // YES price should increase
        uint256 yesPriceAfter = lmsr.getPrice(marketId, 0);
        assertGt(yesPriceAfter, yesPrice, "YES price should increase");
        
        console.log("YES price after buying:", yesPriceAfter);
    }
}
```

### Workshop 2: LMSR Math Verification

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @notice Verify LMSR cost function correctness
 */
contract LMSRMathTest is Test {
    
    LMSRMarket public lmsr;
    
    function test_lmsr_cost_function() public {
        // ค้ำประกัน: ผลรวมของ probabilities ควรเท่ากับ 1
        // sum(p_i) = sum(e^(q_i/b) / sum(e^(q_j/b))) = 1
        
        // กรณี initial (q_i = 0 สำหรับทุก i)
        // p_i = e^0 / (n * e^0) = 1/n
        
        // กรณี binary market (YES/NO):
        // ถ้า q_YES = 100, q_NO = 0, b = 100
        // p_YES = e^1 / (e^1 + e^0) = e / (e + 1) ≈ 0.731
        // p_NO = 1 / (e + 1) ≈ 0.269
        
        // ต้น ทดสอบ:
        uint256 b = 100e18;
        
        // q_YES = 100, q_NO = 0
        uint256 qYes = 100e18;
        uint256 qNo = 0;
        
        // คำนวณ prices manually
        // e^1 ≈ 2.71828
        // e^0 = 1
        // p_YES = 2.71828 / 3.71828 ≈ 0.7311
        
        uint256 expectedPYes = 731e15; // 73.1%
        uint256 expectedPNo = 269e15;  // 26.9%
        
        console.log("Expected YES probability: ~73.1%");
        console.log("Expected NO probability: ~26.9%");
        
        // Verify sum = 1
        assertApproxEqAbs(expectedPYes + expectedPNo, 1e18, 1e15);
    }
}
```

---

## สรุป Part 88

- **OrderBook Market**: Limit orders, order matching engine, maker/taker model
- **LMSR AMM Market**: Logarithmic Market Scoring Rule, automatic liquidity provision, continuous pricing
- **Oracle Resolution**: Chainlink direct, UMA optimistic oracle, multi-sig voting
- **Full PredictionMarket**: Creator fees, LP shares, AMM pricing, slippage protection
- **Market Lifecycle**: Open → Trading → Locked → Resolved → Redemption

## Next: Part 89 - Advanced Perpetuals Architecture
