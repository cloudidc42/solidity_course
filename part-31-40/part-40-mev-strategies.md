# Part 40: MEV Strategies and Protection

## สารบัญ
1. MEV Overview
2. Sandwich Attack Mechanics
3. JIT Liquidity
4. Backrunning
5. MEV Protection Patterns

---

## 1. MEV Overview

```
MEV (Maximal Extractable Value):
กำไรที่ validator/miner สามารถ extract ได้
โดยการเลือก ordering ของ transactions

Types of MEV:
1. Frontrunning: ส่ง tx เหมือนกันก่อน victim
2. Backrunning: ส่ง tx หลัง victim event
3. Sandwich: front + back ล้อม victim

Actors:
- Searchers: หา MEV opportunities ด้วย bots
- Builders: รวบ txs สร้าง blocks (Flashbots, MEV Boost)
- Validators: เลือก block ที่ให้ fee สูงสุด

ขนาดของ MEV:
- 2020-2024: มากกว่า $1.5B extracted
- ส่วนใหญ่มาจาก DEX arbitrage + sandwich attacks

MEV Protection Evolution:
- ไม่มี protection → ทุกอย่าง visible ใน mempool
- Flashbots → private transactions
- MEV Blocker RPC → penalize sandwichers
- Cowswap → batch auctions (sandwichproof by design)
- Arbitrum → fair sequencing (FCFS)
```

---

## 2. Sandwich Attack

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Sandwich Attack Mechanics:
 * 
 * Victim: swap 100 ETH → USDC ด้วย 1% slippage
 * 
 * Attacker:
 * 1. FRONTRUN: buy ETH (push price up)
 * 2. Victim executes at worse price
 * 3. BACKRUN: sell ETH (profit from price impact)
 * 
 * ตัวอย่าง:
 * Pool: 1000 ETH / 2,000,000 USDC
 * Price: 2000 USDC/ETH
 * 
 * Attacker front: buy 50 ETH → pool: 950/2,105,263 → price: 2216 USDC/ETH
 * Victim: swap 100 ETH → gets 179,271 USDC (instead of ~190,476)
 * Attacker back: sell 50 ETH → gets 107,000 USDC
 * Attacker profit: 107,000 - cost_of_50_ETH ≈ $7,000
 */

/**
 * Anti-sandwich defense: tight slippage + deadline
 */
contract SafeSwapper {
    
    IUniswapV2Router public immutable router;
    
    event SwapExecuted(
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 amountOut,
        address indexed user
    );
    
    constructor(address _router) {
        router = IUniswapV2Router(_router);
    }
    
    /**
     * Swap with tight slippage protection
     * 
     * @param amountIn amount to swap
     * @param minAmountOut minimum to receive (slippage tolerance)
     * @param path token path
     * @param deadline transaction must execute by this time
     */
    function safeSwap(
        uint256 amountIn,
        uint256 minAmountOut,  // Calculated off-chain with oracle price
        address[] calldata path,
        uint256 deadline
    ) external {
        require(deadline >= block.timestamp, "Expired");
        require(path.length >= 2, "Invalid path");
        
        IERC20(path[0]).transferFrom(msg.sender, address(this), amountIn);
        IERC20(path[0]).approve(address(router), amountIn);
        
        uint256[] memory amounts = router.swapExactTokensForTokens(
            amountIn,
            minAmountOut,  // Will revert if sandwich pushes price too far
            path,
            msg.sender,
            deadline
        );
        
        emit SwapExecuted(path[0], path[path.length-1], amountIn, amounts[amounts.length-1], msg.sender);
    }
    
    /**
     * Calculate max slippage based on trade size
     * ใหญ่กว่า → ใช้ tighter slippage
     */
    function getMaxSlippage(uint256 amountIn, uint256 poolLiquidity) 
        external 
        pure 
        returns (uint256 maxSlippageBps) 
    {
        // Impact = amountIn / poolLiquidity
        uint256 impact = amountIn * 10000 / poolLiquidity;
        
        // Add 20% buffer to expected impact
        maxSlippageBps = impact * 120 / 100;
        
        // Minimum 30bps, maximum 300bps
        if (maxSlippageBps < 30) maxSlippageBps = 30;
        if (maxSlippageBps > 300) maxSlippageBps = 300;
    }
}

interface IUniswapV2Router {
    function swapExactTokensForTokens(
        uint256 amountIn,
        uint256 amountOutMin,
        address[] calldata path,
        address to,
        uint256 deadline
    ) external returns (uint256[] memory amounts);
}
```

---

## 3. JIT Liquidity (Just-In-Time)

```
JIT Liquidity:
- Searcher เห็น large swap ใน mempool
- เพิ่ม liquidity เข้า V3 pool ก่อน swap (ใน block เดิม)
- Swap ผ่าน liquidity นั้น → ได้ fees
- ถอน liquidity ออกทันทีหลัง swap

ผล:
- Searcher: ได้ fees จาก swap
- Swapper: ได้ better price (slippage ต่ำกว่า)
- Regular LPs: ได้ fees น้อยลง (JIT LP ดูดไป)

ทั้ง sandwich (harm to swapper) และ JIT (benefit to swapper)
เป็นคนละ category ของ MEV

Uniswap V4 แก้ด้วย hooks:
- JIT hook: ให้ LPs สามารถ commit ล่วงหน้า
- ป้องกัน last-second JIT ที่ไม่ fair
```

---

## 4. Backrunning Strategies

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Backrunning:
 * ส่ง transaction หลัง event ที่สร้าง opportunity
 * 
 * Examples:
 * 1. หลัง oracle update → liquidate undercollateral positions
 * 2. หลัง large swap → rebalance arbitrage
 * 3. หลัง new token list → buy at low price
 */
contract BackrunLiquidator {
    
    ILendingPool public immutable lendingPool;
    IUniswapV2Router public immutable router;
    
    address public immutable owner;
    
    struct LiquidationParams {
        address collateralAsset;
        address debtAsset;
        address user;
        uint256 debtToCover;
        bool receiveAToken;
    }
    
    constructor(address _lendingPool, address _router) {
        lendingPool = ILendingPool(_lendingPool);
        router = IUniswapV2Router(_router);
        owner = msg.sender;
    }
    
    /**
     * Atomic liquidation + sell collateral
     * Called as backrun after oracle price drops
     */
    function liquidateAndSell(LiquidationParams calldata params) external {
        require(msg.sender == owner, "Not owner");
        
        // Check if position is liquidatable
        (,,,,, uint256 healthFactor) = lendingPool.getUserAccountData(params.user);
        require(healthFactor < 1e18, "Not liquidatable");
        
        // Pre-approve debt repayment
        IERC20(params.debtAsset).approve(address(lendingPool), params.debtToCover);
        
        // Execute liquidation: repay debt, receive collateral
        lendingPool.liquidationCall(
            params.collateralAsset,
            params.debtAsset,
            params.user,
            params.debtToCover,
            params.receiveAToken
        );
        
        // Sell received collateral for profit
        uint256 collateralBalance = IERC20(params.collateralAsset).balanceOf(address(this));
        
        if (collateralBalance > 0) {
            IERC20(params.collateralAsset).approve(address(router), collateralBalance);
            
            address[] memory path = new address[](2);
            path[0] = params.collateralAsset;
            path[1] = params.debtAsset; // sell for same token we used to liquidate
            
            router.swapExactTokensForTokens(
                collateralBalance,
                0, // No slippage protection (MEV bot, instant)
                path,
                owner,
                block.timestamp
            );
        }
    }
    
    function withdraw(address token) external {
        require(msg.sender == owner);
        uint256 bal = IERC20(token).balanceOf(address(this));
        if (bal > 0) IERC20(token).transfer(owner, bal);
    }
}

interface ILendingPool {
    function getUserAccountData(address user) external view returns (
        uint256 totalCollateralBase,
        uint256 totalDebtBase,
        uint256 availableBorrowsBase,
        uint256 currentLiquidationThreshold,
        uint256 ltv,
        uint256 healthFactor
    );
    
    function liquidationCall(
        address collateralAsset,
        address debtAsset,
        address user,
        uint256 debtToCover,
        bool receiveAToken
    ) external;
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
    function approve(address, uint256) external returns (bool);
    function balanceOf(address) external view returns (uint256);
}
```

---

## 5. MEV Protection Patterns

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Private Mempool + Commit-Reveal:
 * ป้องกัน frontrunning ด้วยการซ่อน transaction details
 */
contract CommitRevealDEX {
    
    struct Commitment {
        bytes32 commitHash;
        uint256 revealDeadline;
        bool revealed;
    }
    
    mapping(bytes32 => Commitment) public commitments;
    
    uint256 public constant REVEAL_WINDOW = 5 minutes;
    
    event Committed(bytes32 indexed commitId, address indexed user);
    event Revealed(bytes32 indexed commitId, address tokenIn, address tokenOut, uint256 amount);
    
    // Step 1: Commit (hide trade details)
    function commit(bytes32 commitHash) external returns (bytes32 commitId) {
        commitId = keccak256(abi.encodePacked(msg.sender, commitHash, block.number));
        
        commitments[commitId] = Commitment({
            commitHash: commitHash,
            revealDeadline: block.timestamp + REVEAL_WINDOW,
            revealed: false
        });
        
        emit Committed(commitId, msg.sender);
    }
    
    // Step 2: Reveal and execute (after commitment is mined)
    function revealAndSwap(
        bytes32 commitId,
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 minAmountOut,
        uint256 salt
    ) external {
        Commitment storage c = commitments[commitId];
        
        require(!c.revealed, "Already revealed");
        require(block.timestamp <= c.revealDeadline, "Reveal window expired");
        
        // Verify commitment matches
        bytes32 expectedHash = keccak256(
            abi.encodePacked(msg.sender, tokenIn, tokenOut, amountIn, minAmountOut, salt)
        );
        require(c.commitHash == expectedHash, "Invalid reveal");
        
        c.revealed = true;
        
        // Execute swap (now visible, but commitment already mined)
        _executeSwap(tokenIn, tokenOut, amountIn, minAmountOut, msg.sender);
        
        emit Revealed(commitId, tokenIn, tokenOut, amountIn);
    }
    
    function _executeSwap(
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 minAmountOut,
        address recipient
    ) internal {
        // AMM swap logic
    }
}

/**
 * EIP-7702 + Private Transactions Protection:
 * ส่งผ่าน Flashbots RPC เพื่อ bypass mempool
 * 
 * ทำบน off-chain infrastructure:
 * 1. ส่ง tx ไปที่ https://relay.flashbots.net
 * 2. Flashbots bundler จัด package เป็น bundle
 * 3. Bundle ส่งตรงไปหา builder
 * 4. ไม่ผ่าน public mempool!
 */

/**
 * Cow Protocol / Batch Auction:
 * รวบรวม orders แล้วหา optimal clearing price
 * ทุกคนซื้อ/ขายที่ราคาเดียวกัน = no sandwich possible
 */
contract BatchAuctionSettlement {
    
    struct Order {
        address sellToken;
        address buyToken;
        uint256 sellAmount;
        uint256 buyAmount;  // minimum to receive
        uint256 validUntil;
        address owner;
    }
    
    mapping(bytes32 => Order) public orders;
    mapping(bytes32 => bool) public filledOrders;
    
    address public solver; // Cowswap solver submits settlements
    
    event OrderPlaced(bytes32 indexed orderId, address indexed owner);
    event BatchSettled(bytes32[] orderIds, uint256 clearingPrice);
    
    modifier onlySolver() {
        require(msg.sender == solver, "Not solver");
        _;
    }
    
    function placeOrder(
        address sellToken,
        address buyToken,
        uint256 sellAmount,
        uint256 buyAmount,
        uint256 validUntil
    ) external returns (bytes32 orderId) {
        orderId = keccak256(abi.encodePacked(
            msg.sender, sellToken, buyToken, sellAmount, buyAmount, block.timestamp
        ));
        
        orders[orderId] = Order({
            sellToken: sellToken,
            buyToken: buyToken,
            sellAmount: sellAmount,
            buyAmount: buyAmount,
            validUntil: validUntil,
            owner: msg.sender
        });
        
        // Transfer tokens to contract
        IERC20(sellToken).transferFrom(msg.sender, address(this), sellAmount);
        
        emit OrderPlaced(orderId, msg.sender);
    }
    
    /**
     * Solver submits optimal settlement:
     * - clearingPrice: price where all orders match
     * - All buyers pay same price
     * - Surplus goes to solver as reward
     */
    function settleOrders(
        bytes32[] calldata orderIds,
        uint256 clearingPrice
    ) external onlySolver {
        for (uint256 i; i < orderIds.length; i++) {
            bytes32 id = orderIds[i];
            Order storage o = orders[id];
            
            require(!filledOrders[id], "Already filled");
            require(block.timestamp <= o.validUntil, "Expired");
            
            // Calculate what buyer receives at clearing price
            uint256 received = o.sellAmount * clearingPrice / 1e18;
            require(received >= o.buyAmount, "Below limit");
            
            filledOrders[id] = true;
            
            // Transfer bought tokens to user
            IERC20(o.buyToken).transfer(o.owner, received);
        }
        
        emit BatchSettled(orderIds, clearingPrice);
    }
}
```

---

## สรุป Part 40

MEV และ Protection ที่เรียนรู้:
- ✅ MEV types (frontrun, backrun, sandwich)
- ✅ Sandwich attack mechanics
- ✅ JIT liquidity
- ✅ Backrunning (liquidation bots)
- ✅ Protection: commit-reveal, private mempool, batch auctions
- ✅ Cowswap settlement model

## Quiz

1. Sandwich attack ทำกำไรได้อย่างไร?
2. JIT liquidity ต่างจาก sandwich อย่างไร?
3. Commit-reveal ป้องกัน frontrunning อย่างไร?
4. Batch auction ทำให้ sandwich เป็นไปไม่ได้อย่างไร?

---

## ยินดีด้วย! จบ Section 4 (Parts 31-40)

Section ถัดไป: Parts 41-50 - Full-Stack DeFi Development
