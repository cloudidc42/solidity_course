# Part 87: DeFi Aggregators & Meta-Protocols

## บทนำ

DeFi Aggregators คือ layer ที่อยู่เหนือ protocol ต่างๆ โดยทำหน้าที่รวบรวมและ optimize liquidity จากหลาย sources ในการ trade เดียว สิ่งที่ทำให้ aggregator มีคุณค่าคือ:

1. **Price optimization**: หา best execution price จากหลาย DEX
2. **Split routing**: แบ่ง order ข้าม multiple pools เพื่อลด price impact
3. **Gas efficiency**: รวม operations หลายอย่างใน transaction เดียว
4. **Capital efficiency**: ใช้ flash loans เพื่อ arbitrage โดยไม่ต้องใช้ capital

ตัวอย่าง protocol ที่ดัง: 1inch, Paraswap, 0x Protocol, CoW Protocol

---

## 1. DEX Aggregator Architecture

### PathFinder - ค้นหา Best Route

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title PathFinder
 * @notice ค้นหา optimal swap path ข้าม DEX หลาย venues
 * @dev Supports Uniswap V2/V3, Curve, and custom pools
 */
contract PathFinder {
    
    struct Pool {
        address pool;           // Pool address
        address tokenIn;        // Input token
        address tokenOut;       // Output token
        uint256 fee;           // Fee in basis points
        PoolType poolType;     // Type ของ pool
    }
    
    enum PoolType {
        UniswapV2,
        UniswapV3,
        Curve,
        Balancer,
        Custom
    }
    
    struct Route {
        Pool[] pools;          // Sequence ของ pools
        uint256 amountIn;      // Input amount
        uint256 expectedOut;   // Expected output
        uint256 priceImpact;   // Price impact in basis points
        uint256 gasEstimate;   // Estimated gas
    }
    
    struct SplitRoute {
        Route[] routes;        // Multiple routes
        uint256[] splits;      // Percentage splits (basis points, sum = 10000)
    }
    
    // Pool registry
    mapping(address => mapping(address => Pool[])) public pools;
    
    // ========== Quote Functions ==========
    
    /**
     * @notice ดึง quote จาก Uniswap V2 style pool
     */
    function quoteUniV2(
        address pool,
        address tokenIn,
        uint256 amountIn
    ) public view returns (uint256 amountOut) {
        (uint256 reserve0, uint256 reserve1,) = IUniswapV2Pair(pool).getReserves();
        
        address token0 = IUniswapV2Pair(pool).token0();
        (uint256 reserveIn, uint256 reserveOut) = tokenIn == token0 
            ? (reserve0, reserve1) 
            : (reserve1, reserve0);
        
        // x * y = k formula with 0.3% fee
        uint256 amountInWithFee = amountIn * 997;
        amountOut = (amountInWithFee * reserveOut) / (reserveIn * 1000 + amountInWithFee);
    }
    
    /**
     * @notice ดึง quote จาก Uniswap V3 style pool
     */
    function quoteUniV3(
        address pool,
        address tokenIn,
        uint256 amountIn,
        uint24 fee
    ) public view returns (uint256 amountOut) {
        // Simplified V3 quote (ในความเป็นจริงใช้ quoter contract)
        (uint160 sqrtPriceX96,,,,,,) = IUniswapV3Pool(pool).slot0();
        
        // Convert sqrtPriceX96 to price
        uint256 price = (uint256(sqrtPriceX96) * uint256(sqrtPriceX96) * 1e18) >> 192;
        
        address token0 = IUniswapV3Pool(pool).token0();
        bool zeroForOne = tokenIn == token0;
        
        if (zeroForOne) {
            amountOut = amountIn * price / 1e18;
        } else {
            amountOut = amountIn * 1e18 / price;
        }
        
        // Apply fee
        amountOut = amountOut * (1e6 - fee) / 1e6;
    }
    
    /**
     * @notice ค้นหา best single route
     */
    function findBestRoute(
        address tokenIn,
        address tokenOut,
        uint256 amountIn
    ) external view returns (Route memory bestRoute) {
        Pool[] storage availablePools = pools[tokenIn][tokenOut];
        uint256 bestAmountOut = 0;
        
        for (uint256 i = 0; i < availablePools.length; i++) {
            uint256 amountOut;
            Pool memory pool = availablePools[i];
            
            if (pool.poolType == PoolType.UniswapV2) {
                amountOut = quoteUniV2(pool.pool, tokenIn, amountIn);
            } else if (pool.poolType == PoolType.UniswapV3) {
                amountOut = quoteUniV3(pool.pool, tokenIn, amountIn, uint24(pool.fee));
            }
            
            if (amountOut > bestAmountOut) {
                bestAmountOut = amountOut;
                
                Pool[] memory routePools = new Pool[](1);
                routePools[0] = pool;
                
                bestRoute = Route({
                    pools: routePools,
                    amountIn: amountIn,
                    expectedOut: amountOut,
                    priceImpact: _calcPriceImpact(pool, tokenIn, amountIn),
                    gasEstimate: _estimateGas(pool.poolType)
                });
            }
        }
    }
    
    /**
     * @notice ค้นหา optimal split route เพื่อลด price impact
     * @dev แบ่ง order ข้าม multiple pools
     */
    function findSplitRoute(
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 maxSplits
    ) external view returns (SplitRoute memory splitRoute) {
        Pool[] storage availablePools = pools[tokenIn][tokenOut];
        uint256 numPools = availablePools.length < maxSplits ? availablePools.length : maxSplits;
        
        Route[] memory routes = new Route[](numPools);
        uint256[] memory splits = new uint256[](numPools);
        
        // Simple equal split strategy (ในความเป็นจริงใช้ optimization algorithm)
        uint256 splitAmount = amountIn / numPools;
        uint256 equalSplit = 10000 / numPools;
        uint256 totalSplit = 0;
        
        for (uint256 i = 0; i < numPools; i++) {
            uint256 amount = (i == numPools - 1) ? (amountIn - splitAmount * (numPools - 1)) : splitAmount;
            uint256 split = (i == numPools - 1) ? (10000 - totalSplit) : equalSplit;
            
            Pool[] memory routePools = new Pool[](1);
            routePools[0] = availablePools[i];
            
            uint256 amountOut;
            if (availablePools[i].poolType == PoolType.UniswapV2) {
                amountOut = quoteUniV2(availablePools[i].pool, tokenIn, amount);
            }
            
            routes[i] = Route({
                pools: routePools,
                amountIn: amount,
                expectedOut: amountOut,
                priceImpact: _calcPriceImpact(availablePools[i], tokenIn, amount),
                gasEstimate: _estimateGas(availablePools[i].poolType)
            });
            
            splits[i] = split;
            totalSplit += split;
        }
        
        splitRoute = SplitRoute({ routes: routes, splits: splits });
    }
    
    // ========== Helper Functions ==========
    
    function _calcPriceImpact(Pool memory pool, address tokenIn, uint256 amountIn) 
        internal view returns (uint256) {
        if (pool.poolType == PoolType.UniswapV2) {
            (uint256 reserve0, uint256 reserve1,) = IUniswapV2Pair(pool.pool).getReserves();
            address token0 = IUniswapV2Pair(pool.pool).token0();
            uint256 reserveIn = tokenIn == token0 ? reserve0 : reserve1;
            
            // Price impact = amountIn / (reserveIn + amountIn) * 10000
            return amountIn * 10000 / (reserveIn + amountIn);
        }
        return 0;
    }
    
    function _estimateGas(PoolType poolType) internal pure returns (uint256) {
        if (poolType == PoolType.UniswapV2) return 80000;
        if (poolType == PoolType.UniswapV3) return 120000;
        if (poolType == PoolType.Curve) return 200000;
        return 100000;
    }
    
    /**
     * @notice Register pool ใน PathFinder
     */
    function registerPool(Pool calldata pool) external {
        pools[pool.tokenIn][pool.tokenOut].push(pool);
    }
}

// Interfaces
interface IUniswapV2Pair {
    function getReserves() external view returns (uint112 reserve0, uint112 reserve1, uint32 blockTimestampLast);
    function token0() external view returns (address);
    function token1() external view returns (address);
}

interface IUniswapV3Pool {
    function slot0() external view returns (
        uint160 sqrtPriceX96,
        int24 tick,
        uint16 observationIndex,
        uint16 observationCardinality,
        uint16 observationCardinalityNext,
        uint8 feeProtocol,
        bool unlocked
    );
    function token0() external view returns (address);
}
```

---

## 2. Aggregator Router - Execute Best Path

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title AggregatorRouter
 * @notice Execute swaps ข้าม multiple DEX ด้วย optimal routing
 * @dev รองรับ single path, split path, multi-hop
 */
contract AggregatorRouter is ReentrancyGuard, Ownable {
    using SafeERC20 for IERC20;
    
    // ========== Data Structures ==========
    
    struct SwapParams {
        address tokenIn;
        address tokenOut;
        uint256 amountIn;
        uint256 minAmountOut;  // Slippage protection
        address recipient;
        uint256 deadline;
        bytes routeData;       // Encoded routing information
    }
    
    struct SingleSwapRoute {
        address pool;
        address tokenIn;
        address tokenOut;
        uint256 fee;
        uint8 poolType;        // 0=UniV2, 1=UniV3, 2=Curve
    }
    
    struct MultiHopRoute {
        SingleSwapRoute[] hops;
    }
    
    struct SplitSwapRoute {
        SingleSwapRoute[] routes;
        uint256[] percentages;  // Sum must be 10000
    }
    
    // ========== State ==========
    
    // Protocol fee (basis points)
    uint256 public protocolFee = 10; // 0.1%
    address public feeRecipient;
    
    // Whitelisted pools สำหรับ security
    mapping(address => bool) public whitelistedPools;
    
    // ========== Events ==========
    
    event Swapped(
        address indexed user,
        address indexed tokenIn,
        address indexed tokenOut,
        uint256 amountIn,
        uint256 amountOut,
        uint256 fee
    );
    
    event PoolWhitelisted(address indexed pool, bool status);
    
    // ========== Constructor ==========
    
    constructor(address _feeRecipient) Ownable(msg.sender) {
        feeRecipient = _feeRecipient;
    }
    
    // ========== Main Swap Functions ==========
    
    /**
     * @notice Execute single-pool swap
     */
    function swapSingle(SwapParams calldata params) external nonReentrant returns (uint256 amountOut) {
        require(block.timestamp <= params.deadline, "expired");
        
        SingleSwapRoute memory route = abi.decode(params.routeData, (SingleSwapRoute));
        require(whitelistedPools[route.pool], "pool not whitelisted");
        
        // Transfer tokens from user
        IERC20(params.tokenIn).safeTransferFrom(msg.sender, address(this), params.amountIn);
        
        // Deduct protocol fee
        uint256 fee = params.amountIn * protocolFee / 10000;
        uint256 amountInAfterFee = params.amountIn - fee;
        
        // Execute swap
        amountOut = _executeSwap(route, amountInAfterFee);
        
        require(amountOut >= params.minAmountOut, "insufficient output");
        
        // Transfer fee
        if (fee > 0) {
            IERC20(params.tokenIn).safeTransfer(feeRecipient, fee);
        }
        
        // Transfer output to recipient
        IERC20(params.tokenOut).safeTransfer(params.recipient, amountOut);
        
        emit Swapped(msg.sender, params.tokenIn, params.tokenOut, params.amountIn, amountOut, fee);
    }
    
    /**
     * @notice Execute multi-hop swap (A → B → C)
     */
    function swapMultiHop(SwapParams calldata params) external nonReentrant returns (uint256 amountOut) {
        require(block.timestamp <= params.deadline, "expired");
        
        MultiHopRoute memory route = abi.decode(params.routeData, (MultiHopRoute));
        require(route.hops.length >= 2, "need at least 2 hops");
        
        // Transfer tokens from user
        IERC20(params.tokenIn).safeTransferFrom(msg.sender, address(this), params.amountIn);
        
        // Deduct protocol fee
        uint256 fee = params.amountIn * protocolFee / 10000;
        uint256 currentAmount = params.amountIn - fee;
        
        // Execute each hop
        for (uint256 i = 0; i < route.hops.length; i++) {
            require(whitelistedPools[route.hops[i].pool], "pool not whitelisted");
            currentAmount = _executeSwap(route.hops[i], currentAmount);
        }
        
        amountOut = currentAmount;
        require(amountOut >= params.minAmountOut, "insufficient output");
        
        // Transfer fee  
        if (fee > 0) {
            IERC20(params.tokenIn).safeTransfer(feeRecipient, fee);
        }
        
        // Transfer final output
        IERC20(params.tokenOut).safeTransfer(params.recipient, amountOut);
        
        emit Swapped(msg.sender, params.tokenIn, params.tokenOut, params.amountIn, amountOut, fee);
    }
    
    /**
     * @notice Execute split route swap (50% pool A, 50% pool B)
     */
    function swapSplitRoute(SwapParams calldata params) external nonReentrant returns (uint256 totalAmountOut) {
        require(block.timestamp <= params.deadline, "expired");
        
        SplitSwapRoute memory splitRoute = abi.decode(params.routeData, (SplitSwapRoute));
        
        // Validate splits sum to 10000
        uint256 totalPercentage = 0;
        for (uint256 i = 0; i < splitRoute.percentages.length; i++) {
            totalPercentage += splitRoute.percentages[i];
        }
        require(totalPercentage == 10000, "splits must sum to 10000");
        require(splitRoute.routes.length == splitRoute.percentages.length, "array length mismatch");
        
        // Transfer tokens from user
        IERC20(params.tokenIn).safeTransferFrom(msg.sender, address(this), params.amountIn);
        
        // Deduct protocol fee
        uint256 fee = params.amountIn * protocolFee / 10000;
        uint256 amountAfterFee = params.amountIn - fee;
        
        // Execute each split
        for (uint256 i = 0; i < splitRoute.routes.length; i++) {
            require(whitelistedPools[splitRoute.routes[i].pool], "pool not whitelisted");
            
            uint256 splitAmount = amountAfterFee * splitRoute.percentages[i] / 10000;
            
            // Handle last split to avoid rounding issues
            if (i == splitRoute.routes.length - 1) {
                splitAmount = IERC20(params.tokenIn).balanceOf(address(this)) - fee;
            }
            
            uint256 splitOut = _executeSwap(splitRoute.routes[i], splitAmount);
            totalAmountOut += splitOut;
        }
        
        require(totalAmountOut >= params.minAmountOut, "insufficient output");
        
        // Transfer fee
        if (fee > 0) {
            IERC20(params.tokenIn).safeTransfer(feeRecipient, fee);
        }
        
        // Transfer output
        IERC20(params.tokenOut).safeTransfer(params.recipient, totalAmountOut);
        
        emit Swapped(msg.sender, params.tokenIn, params.tokenOut, params.amountIn, totalAmountOut, fee);
    }
    
    // ========== Internal Execution ==========
    
    /**
     * @notice Execute swap on specific pool
     */
    function _executeSwap(SingleSwapRoute memory route, uint256 amountIn) internal returns (uint256 amountOut) {
        IERC20(route.tokenIn).forceApprove(route.pool, amountIn);
        
        if (route.poolType == 0) {
            // Uniswap V2
            amountOut = _swapUniV2(route.pool, route.tokenIn, route.tokenOut, amountIn);
        } else if (route.poolType == 1) {
            // Uniswap V3
            amountOut = _swapUniV3(route.pool, route.tokenIn, route.tokenOut, amountIn, uint24(route.fee));
        } else if (route.poolType == 2) {
            // Curve
            amountOut = _swapCurve(route.pool, route.tokenIn, route.tokenOut, amountIn);
        }
    }
    
    function _swapUniV2(
        address pool,
        address tokenIn,
        address tokenOut,
        uint256 amountIn
    ) internal returns (uint256 amountOut) {
        IERC20(tokenIn).safeTransfer(pool, amountIn);
        
        (uint256 reserve0, uint256 reserve1,) = IUniswapV2Pair(pool).getReserves();
        address token0 = IUniswapV2Pair(pool).token0();
        
        (uint256 reserveIn, uint256 reserveOut) = tokenIn == token0 
            ? (reserve0, reserve1) 
            : (reserve1, reserve0);
        
        uint256 amountInWithFee = amountIn * 997;
        amountOut = (amountInWithFee * reserveOut) / (reserveIn * 1000 + amountInWithFee);
        
        (uint256 amount0Out, uint256 amount1Out) = tokenIn == token0 
            ? (uint256(0), amountOut) 
            : (amountOut, uint256(0));
        
        IUniswapV2Pair(pool).swap(amount0Out, amount1Out, address(this), "");
    }
    
    function _swapUniV3(
        address pool,
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint24 fee
    ) internal returns (uint256 amountOut) {
        bool zeroForOne = tokenIn < tokenOut;
        
        (int256 amount0, int256 amount1) = IUniswapV3Pool(pool).swap(
            address(this),
            zeroForOne,
            int256(amountIn),
            zeroForOne ? 4295128740 : 1461446703485210103287273052203988822378723970341,
            abi.encode(tokenIn, fee)
        );
        
        amountOut = uint256(-(zeroForOne ? amount1 : amount0));
    }
    
    function _swapCurve(
        address pool,
        address tokenIn,
        address tokenOut,
        uint256 amountIn
    ) internal returns (uint256 amountOut) {
        // Curve stable swap
        int128 i = ICurvePool(pool).coins(0) == tokenIn ? int128(0) : int128(1);
        int128 j = i == 0 ? int128(1) : int128(0);
        
        amountOut = ICurvePool(pool).exchange(i, j, amountIn, 0);
    }
    
    // ========== Admin Functions ==========
    
    function whitelistPool(address pool, bool status) external onlyOwner {
        whitelistedPools[pool] = status;
        emit PoolWhitelisted(pool, status);
    }
    
    function setProtocolFee(uint256 _fee) external onlyOwner {
        require(_fee <= 100, "fee too high"); // Max 1%
        protocolFee = _fee;
    }
    
    function setFeeRecipient(address _recipient) external onlyOwner {
        feeRecipient = _recipient;
    }
}

interface IUniswapV3Pool {
    function swap(
        address recipient,
        bool zeroForOne,
        int256 amountSpecified,
        uint160 sqrtPriceLimitX96,
        bytes calldata data
    ) external returns (int256 amount0, int256 amount1);
    function token0() external view returns (address);
    function slot0() external view returns (uint160, int24, uint16, uint16, uint16, uint8, bool);
}

interface ICurvePool {
    function coins(uint256 i) external view returns (address);
    function exchange(int128 i, int128 j, uint256 dx, uint256 min_dy) external returns (uint256);
}
```

---

## 3. Yield Aggregator Router

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

/**
 * @title YieldAggregator
 * @notice Auto-rebalance หา best yield ข้าม Aave/Compound/Curve
 * @dev คล้าย Yearn Finance แต่ simplified
 */
contract YieldAggregator {
    using SafeERC20 for IERC20;
    
    struct Strategy {
        address protocol;       // Protocol address
        address asset;          // Underlying asset
        uint256 allocation;     // Current allocation (basis points)
        uint256 lastAPY;        // Last known APY (in basis points)
        bool active;            // Whether strategy is active
    }
    
    struct UserPosition {
        uint256 shares;         // Shares owned
        uint256 depositTime;    // When deposited
    }
    
    // ========== State ==========
    
    IERC20 public immutable asset;
    
    Strategy[] public strategies;
    
    mapping(address => UserPosition) public positions;
    
    uint256 public totalShares;
    uint256 public totalAssets;
    
    // Rebalance parameters
    uint256 public constant REBALANCE_THRESHOLD = 200; // 2% APY difference
    uint256 public constant MAX_SLIPPAGE = 100;        // 1%
    
    event Deposited(address indexed user, uint256 amount, uint256 shares);
    event Withdrawn(address indexed user, uint256 shares, uint256 amount);
    event Rebalanced(uint256 indexed strategyFrom, uint256 indexed strategyTo, uint256 amount);
    event APYUpdated(uint256 indexed strategyIndex, uint256 newAPY);
    
    constructor(address _asset) {
        asset = IERC20(_asset);
    }
    
    // ========== User Functions ==========
    
    /**
     * @notice Deposit assets เข้า yield aggregator
     */
    function deposit(uint256 amount) external returns (uint256 shares) {
        require(amount > 0, "zero amount");
        
        // Calculate shares
        shares = totalShares == 0 ? amount : (amount * totalShares / totalAssets);
        
        // Transfer assets
        asset.safeTransferFrom(msg.sender, address(this), amount);
        
        // Update state
        positions[msg.sender].shares += shares;
        totalShares += shares;
        totalAssets += amount;
        
        // Deploy to best strategy
        _deployToStrategy(amount);
        
        emit Deposited(msg.sender, amount, shares);
    }
    
    /**
     * @notice Withdraw assets จาก yield aggregator
     */
    function withdraw(uint256 shares) external returns (uint256 amount) {
        require(shares <= positions[msg.sender].shares, "insufficient shares");
        
        // Calculate amount
        amount = shares * totalAssets / totalShares;
        
        // Update state
        positions[msg.sender].shares -= shares;
        totalShares -= shares;
        totalAssets -= amount;
        
        // Withdraw from strategies
        _withdrawFromStrategies(amount);
        
        // Transfer to user
        asset.safeTransfer(msg.sender, amount);
        
        emit Withdrawn(msg.sender, shares, amount);
    }
    
    // ========== Strategy Management ==========
    
    /**
     * @notice เปรียบเทียบ APY และ rebalance ถ้าต้องการ
     */
    function rebalance() external {
        // อัพเดท APY จากแต่ละ protocol
        _updateAPYs();
        
        // หา best และ worst strategy
        (uint256 bestIdx, uint256 worstIdx) = _findBestAndWorstStrategy();
        
        // Rebalance ถ้า APY difference เกิน threshold
        if (strategies[bestIdx].lastAPY > strategies[worstIdx].lastAPY + REBALANCE_THRESHOLD) {
            uint256 amountToMove = _getStrategyBalance(worstIdx) / 2; // Move 50%
            
            if (amountToMove > 0) {
                _withdrawFromSpecificStrategy(worstIdx, amountToMove);
                _depositToSpecificStrategy(bestIdx, amountToMove);
                
                emit Rebalanced(worstIdx, bestIdx, amountToMove);
            }
        }
    }
    
    /**
     * @notice ดึง APY จาก Aave
     */
    function _getAaveAPY(address aavePool, address token) internal view returns (uint256) {
        // Aave returns ray (27 decimals) - convert to basis points
        (,,,,uint128 currentLiquidityRate,,,,,,,) = IAavePool(aavePool).getReserveData(token);
        // ray = 1e27, basis points = 1e4
        // APY = (1 + rate/1e27)^(365*24*3600) - 1 (simplified to linear)
        return uint256(currentLiquidityRate) / 1e23; // Approximate conversion to bps
    }
    
    /**
     * @notice ดึง APY จาก Compound
     */
    function _getCompoundAPY(address cToken) internal view returns (uint256) {
        uint256 supplyRate = ICToken(cToken).supplyRatePerBlock();
        // Compound uses per-block rate, ~2102400 blocks/year (Ethereum)
        uint256 blocksPerYear = 2102400;
        // APY in basis points (simplified)
        return supplyRate * blocksPerYear / 1e14;
    }
    
    /**
     * @notice Deploy assets ไปยัง best strategy
     */
    function _deployToStrategy(uint256 amount) internal {
        if (strategies.length == 0) return;
        
        uint256 bestAPY = 0;
        uint256 bestIdx = 0;
        
        for (uint256 i = 0; i < strategies.length; i++) {
            if (strategies[i].active && strategies[i].lastAPY > bestAPY) {
                bestAPY = strategies[i].lastAPY;
                bestIdx = i;
            }
        }
        
        _depositToSpecificStrategy(bestIdx, amount);
    }
    
    function _depositToSpecificStrategy(uint256 idx, uint256 amount) internal {
        Strategy memory strat = strategies[idx];
        
        asset.forceApprove(strat.protocol, amount);
        
        // Protocol-specific deposit
        // (simplified - implement for each protocol)
        ILendingProtocol(strat.protocol).deposit(address(asset), amount, address(this), 0);
    }
    
    function _withdrawFromStrategies(uint256 amount) internal {
        // Withdraw proportionally from all active strategies
        for (uint256 i = 0; i < strategies.length && amount > 0; i++) {
            if (!strategies[i].active) continue;
            
            uint256 stratBalance = _getStrategyBalance(i);
            if (stratBalance == 0) continue;
            
            uint256 withdrawAmount = amount < stratBalance ? amount : stratBalance;
            _withdrawFromSpecificStrategy(i, withdrawAmount);
            amount -= withdrawAmount;
        }
    }
    
    function _withdrawFromSpecificStrategy(uint256 idx, uint256 amount) internal {
        Strategy memory strat = strategies[idx];
        ILendingProtocol(strat.protocol).withdraw(address(asset), amount, address(this));
    }
    
    function _getStrategyBalance(uint256 idx) internal view returns (uint256) {
        // Query protocol for current balance (simplified)
        return ILendingProtocol(strategies[idx].protocol).balanceOf(address(this), address(asset));
    }
    
    function _updateAPYs() internal {
        for (uint256 i = 0; i < strategies.length; i++) {
            // Update each strategy's APY
            // (implement per protocol)
        }
    }
    
    function _findBestAndWorstStrategy() internal view returns (uint256 bestIdx, uint256 worstIdx) {
        uint256 bestAPY = 0;
        uint256 worstAPY = type(uint256).max;
        
        for (uint256 i = 0; i < strategies.length; i++) {
            if (!strategies[i].active) continue;
            
            if (strategies[i].lastAPY > bestAPY) {
                bestAPY = strategies[i].lastAPY;
                bestIdx = i;
            }
            if (strategies[i].lastAPY < worstAPY) {
                worstAPY = strategies[i].lastAPY;
                worstIdx = i;
            }
        }
    }
    
    /**
     * @notice เพิ่ม strategy
     */
    function addStrategy(address protocol, address stratAsset) external {
        strategies.push(Strategy({
            protocol: protocol,
            asset: stratAsset,
            allocation: 0,
            lastAPY: 0,
            active: true
        }));
    }
    
    /**
     * @notice คำนวณ current APY ทั้งหมด (weighted average)
     */
    function getCurrentAPY() external view returns (uint256) {
        if (totalAssets == 0) return 0;
        
        uint256 weightedAPY = 0;
        for (uint256 i = 0; i < strategies.length; i++) {
            if (!strategies[i].active) continue;
            uint256 balance = _getStrategyBalance(i);
            weightedAPY += strategies[i].lastAPY * balance;
        }
        
        return weightedAPY / totalAssets;
    }
}

interface IAavePool {
    function getReserveData(address asset) external view returns (
        uint256 configuration,
        uint128 liquidityIndex,
        uint128 currentLiquidityRate,
        uint128 variableBorrowIndex,
        uint128 currentVariableBorrowRate,
        uint128 currentStableBorrowRate,
        uint40 lastUpdateTimestamp,
        uint16 id,
        address aTokenAddress,
        address stableDebtTokenAddress,
        address variableDebtTokenAddress,
        address interestRateStrategyAddress,
        uint128 accruedToTreasury,
        uint128 unbacked,
        uint128 isolationModeTotalDebt
    );
    function deposit(address asset, uint256 amount, address onBehalfOf, uint16 referralCode) external;
    function withdraw(address asset, uint256 amount, address to) external returns (uint256);
}

interface ICToken {
    function supplyRatePerBlock() external view returns (uint256);
}

interface ILendingProtocol {
    function deposit(address asset, uint256 amount, address onBehalfOf, uint16 referralCode) external;
    function withdraw(address asset, uint256 amount, address to) external;
    function balanceOf(address user, address asset) external view returns (uint256);
}
```

---

## 4. Gas-Efficient Multicall Aggregation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title Multicall3
 * @notice Aggregate multiple calls ใน single transaction
 * @dev Compatible กับ Multicall3 standard (MakerDAO)
 *      Reference: https://github.com/mds1/multicall
 */
contract Multicall3 {
    
    struct Call {
        address target;
        bytes callData;
    }
    
    struct Call3 {
        address target;
        bool allowFailure;
        bytes callData;
    }
    
    struct Call3Value {
        address target;
        bool allowFailure;
        uint256 value;
        bytes callData;
    }
    
    struct Result {
        bool success;
        bytes returnData;
    }
    
    /**
     * @notice Aggregate calls - revert ถ้าใดๆ fails
     * @param calls Array of Call structs
     * @return blockNumber Current block number
     * @return returnData Array of return data
     */
    function aggregate(Call[] calldata calls) 
        external payable 
        returns (uint256 blockNumber, bytes[] memory returnData) {
        blockNumber = block.number;
        uint256 length = calls.length;
        returnData = new bytes[](length);
        
        Call calldata call;
        for (uint256 i = 0; i < length;) {
            bool success;
            call = calls[i];
            (success, returnData[i]) = call.target.call(call.callData);
            require(success, "Multicall3: call failed");
            unchecked { ++i; }
        }
    }
    
    /**
     * @notice tryAggregate - ไม่ revert เมื่อ call fail
     * @param requireSuccess ถ้า true จะ revert เมื่อ fail
     */
    function tryAggregate(bool requireSuccess, Call[] calldata calls) 
        external payable 
        returns (Result[] memory returnData) {
        uint256 length = calls.length;
        returnData = new Result[](length);
        
        Call calldata call;
        for (uint256 i = 0; i < length;) {
            Result memory result = returnData[i];
            call = calls[i];
            (result.success, result.returnData) = call.target.call(call.callData);
            if (requireSuccess) require(result.success, "Multicall3: call failed");
            unchecked { ++i; }
        }
    }
    
    /**
     * @notice aggregate3 - รองรับ allowFailure per call
     */
    function aggregate3(Call3[] calldata calls) 
        external payable 
        returns (Result[] memory returnData) {
        uint256 length = calls.length;
        returnData = new Result[](length);
        
        Call3 calldata calli;
        for (uint256 i = 0; i < length;) {
            Result memory result = returnData[i];
            calli = calls[i];
            (result.success, result.returnData) = calli.target.call(calli.callData);
            assembly {
                // Revert ถ้า allowFailure = false และ success = false
                if iszero(or(calldataload(add(calli, 0x20)), mload(result))) {
                    let ptr := mload(0x40)
                    mstore(ptr, 0x08c379a000000000000000000000000000000000000000000000000000000000)
                    mstore(add(ptr, 0x04), 0x0000000000000000000000000000000000000000000000000000000000000020)
                    mstore(add(ptr, 0x24), 0x0000000000000000000000000000000000000000000000000000000000000017)
                    mstore(add(ptr, 0x44), 0x4d756c746963616c6c333a2063616c6c206661696c656400000000000000000)
                    revert(ptr, 0x64)
                }
            }
            unchecked { ++i; }
        }
    }
    
    /**
     * @notice aggregate3Value - รองรับ ETH value per call
     */
    function aggregate3Value(Call3Value[] calldata calls) 
        external payable 
        returns (Result[] memory returnData) {
        uint256 valAccumulator;
        uint256 length = calls.length;
        returnData = new Result[](length);
        
        Call3Value calldata calli;
        for (uint256 i = 0; i < length;) {
            Result memory result = returnData[i];
            calli = calls[i];
            uint256 val = calli.value;
            
            // Check value accumulation ไม่เกิน msg.value
            unchecked { valAccumulator += val; }
            require(valAccumulator <= msg.value, "Multicall3: value mismatch");
            
            (result.success, result.returnData) = calli.target.call{value: val}(calli.callData);
            
            if (!calli.allowFailure) {
                require(result.success, "Multicall3: call failed");
            }
            
            unchecked { ++i; }
        }
        
        // คืน ETH ที่เหลือ
        unchecked {
            uint256 surplus = msg.value - valAccumulator;
            if (surplus > 0) {
                (bool success,) = msg.sender.call{value: surplus}("");
                require(success, "Multicall3: ETH refund failed");
            }
        }
    }
    
    // ========== View Helpers ==========
    
    function getBlockHash(uint256 blockNumber) external view returns (bytes32 blockHash) {
        blockHash = blockhash(blockNumber);
    }
    
    function getBlockNumber() external view returns (uint256 blockNumber) {
        blockNumber = block.number;
    }
    
    function getCurrentBlockCoinbase() external view returns (address coinbase) {
        coinbase = block.coinbase;
    }
    
    function getCurrentBlockTimestamp() external view returns (uint256 timestamp) {
        timestamp = block.timestamp;
    }
    
    function getEthBalance(address addr) external view returns (uint256 balance) {
        balance = addr.balance;
    }
    
    function getLastBlockHash() external view returns (bytes32 blockHash) {
        unchecked { blockHash = blockhash(block.number - 1); }
    }
    
    function getBasefee() external view returns (uint256 basefee) {
        basefee = block.basefee;
    }
    
    function getChainId() external view returns (uint256 chainid) {
        chainid = block.chainid;
    }
}
```

---

## 5. Price Impact Minimization

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title PriceImpactOptimizer
 * @notice คำนวณ optimal split สำหรับ minimize price impact
 * @dev ใช้ calculus-based optimization approach
 */
contract PriceImpactOptimizer {
    
    struct PoolInfo {
        address pool;
        uint256 reserveIn;
        uint256 reserveOut;
        uint256 fee;    // ใน basis points
    }
    
    /**
     * @notice คำนวณ optimal split ข้าม N pools
     * @dev Minimize total price impact โดยการแจก amount ตาม pool depth
     * @param pools Array ของ pool information
     * @param totalAmount Total amount ที่ต้องการ swap
     * @return amounts Array ของ amount ต่อ pool
     */
    function calcOptimalSplit(
        PoolInfo[] calldata pools,
        uint256 totalAmount
    ) external pure returns (uint256[] memory amounts) {
        uint256 n = pools.length;
        amounts = new uint256[](n);
        
        if (n == 0) return amounts;
        if (n == 1) {
            amounts[0] = totalAmount;
            return amounts;
        }
        
        // Weighted split based on pool liquidity
        // Deeper pools = higher weight (กระจาย order ไปที่ pool ที่ใหญ่กว่า)
        uint256 totalLiquidity = 0;
        for (uint256 i = 0; i < n; i++) {
            totalLiquidity += pools[i].reserveIn;
        }
        
        uint256 allocated = 0;
        for (uint256 i = 0; i < n; i++) {
            if (i == n - 1) {
                amounts[i] = totalAmount - allocated;
            } else {
                amounts[i] = totalAmount * pools[i].reserveIn / totalLiquidity;
                allocated += amounts[i];
            }
        }
        
        return amounts;
    }
    
    /**
     * @notice คำนวณ price impact สำหรับ AMM swap
     * @param reserveIn Current reserve of input token
     * @param amountIn Input amount
     * @return impact Price impact ใน basis points
     */
    function calcPriceImpact(
        uint256 reserveIn,
        uint256 amountIn
    ) public pure returns (uint256 impact) {
        // Impact = amountIn / (reserveIn * 2) * 10000
        // Approximation สำหรับ constant product AMM
        impact = amountIn * 10000 / (reserveIn * 2 + amountIn);
    }
    
    /**
     * @notice คำนวณ output amount จาก constant product formula
     * @param reserveIn Reserve ของ input token
     * @param reserveOut Reserve ของ output token
     * @param amountIn Input amount
     * @param feeBps Fee ใน basis points
     */
    function getAmountOut(
        uint256 reserveIn,
        uint256 reserveOut,
        uint256 amountIn,
        uint256 feeBps
    ) public pure returns (uint256 amountOut) {
        require(amountIn > 0, "insufficient input");
        require(reserveIn > 0 && reserveOut > 0, "insufficient liquidity");
        
        uint256 amountInWithFee = amountIn * (10000 - feeBps);
        uint256 numerator = amountInWithFee * reserveOut;
        uint256 denominator = reserveIn * 10000 + amountInWithFee;
        amountOut = numerator / denominator;
    }
    
    /**
     * @notice หา max amount ที่ swap ได้โดยไม่เกิน max price impact
     */
    function findMaxAmountForImpact(
        uint256 reserveIn,
        uint256 maxImpactBps
    ) public pure returns (uint256 maxAmount) {
        // impact = x / (2R + x) * 10000 = maxImpact
        // x = maxImpact * 2R / (10000 - maxImpact)
        maxAmount = maxImpactBps * reserveIn * 2 / (10000 - maxImpactBps);
    }
    
    /**
     * @notice คำนวณ effective price ของ swap
     */
    function getEffectivePrice(
        uint256 reserveIn,
        uint256 reserveOut,
        uint256 amountIn,
        uint256 feeBps
    ) external pure returns (uint256 effectivePrice) {
        uint256 amountOut = getAmountOut(reserveIn, reserveOut, amountIn, feeBps);
        // Price = amountOut / amountIn (scaled by 1e18)
        effectivePrice = amountOut * 1e18 / amountIn;
    }
}
```

---

## Workshop: สร้าง Full DEX Aggregator System

### Workshop 1: Deploy และ Test Aggregator

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";

contract AggregatorTest is Test {
    
    AggregatorRouter public router;
    MockUniV2Pool public pool1;
    MockUniV2Pool public pool2;
    MockERC20 public tokenA;
    MockERC20 public tokenB;
    
    address public user = address(0xUSER);
    
    function setUp() public {
        // Deploy tokens
        tokenA = new MockERC20("Token A", "TOKA", 18);
        tokenB = new MockERC20("Token B", "TOKB", 18);
        
        // Deploy pools with different liquidity depths
        pool1 = new MockUniV2Pool(address(tokenA), address(tokenB));
        pool2 = new MockUniV2Pool(address(tokenA), address(tokenB));
        
        // Add liquidity to pools
        tokenA.mint(address(pool1), 1000000e18);
        tokenB.mint(address(pool1), 1000000e18);
        tokenA.mint(address(pool2), 500000e18);
        tokenB.mint(address(pool2), 550000e18); // Different price
        
        // Deploy router
        router = new AggregatorRouter(address(this));
        
        // Whitelist pools
        router.whitelistPool(address(pool1), true);
        router.whitelistPool(address(pool2), true);
        
        // Setup user
        tokenA.mint(user, 10000e18);
        vm.prank(user);
        tokenA.approve(address(router), type(uint256).max);
    }
    
    function test_singleSwap() public {
        uint256 amountIn = 1000e18;
        uint256 expectedOut = _getExpectedOut(address(pool1), address(tokenA), amountIn);
        
        AggregatorRouter.SingleSwapRoute memory route = AggregatorRouter.SingleSwapRoute({
            pool: address(pool1),
            tokenIn: address(tokenA),
            tokenOut: address(tokenB),
            fee: 30,      // 0.3%
            poolType: 0   // UniV2
        });
        
        AggregatorRouter.SwapParams memory params = AggregatorRouter.SwapParams({
            tokenIn: address(tokenA),
            tokenOut: address(tokenB),
            amountIn: amountIn,
            minAmountOut: expectedOut * 99 / 100, // 1% slippage
            recipient: user,
            deadline: block.timestamp + 1 hours,
            routeData: abi.encode(route)
        });
        
        uint256 balBefore = tokenB.balanceOf(user);
        
        vm.prank(user);
        uint256 amountOut = router.swapSingle(params);
        
        uint256 balAfter = tokenB.balanceOf(user);
        
        assertEq(balAfter - balBefore, amountOut);
        assertGe(amountOut, params.minAmountOut);
        
        console.log("Amount In:", amountIn);
        console.log("Amount Out:", amountOut);
        console.log("Effective Rate:", amountOut * 1e18 / amountIn);
    }
    
    function test_splitSwap_reducePriceImpact() public {
        uint256 amountIn = 100000e18; // Large amount
        
        // Compare single vs split
        uint256 singleOut = _getExpectedOut(address(pool1), address(tokenA), amountIn);
        
        // Split: 60% pool1, 40% pool2
        uint256 split1 = amountIn * 60 / 100;
        uint256 split2 = amountIn - split1;
        uint256 out1 = _getExpectedOut(address(pool1), address(tokenA), split1);
        uint256 out2 = _getExpectedOut(address(pool2), address(tokenA), split2);
        uint256 splitOut = out1 + out2;
        
        console.log("Single swap output:", singleOut);
        console.log("Split swap output:", splitOut);
        console.log("Improvement:", splitOut > singleOut ? splitOut - singleOut : 0);
        
        // Split route should be better for large amounts
        assertTrue(splitOut >= singleOut, "split should be >= single for large amounts");
    }
    
    function test_multicall_batch() public {
        Multicall3 multicall = new Multicall3();
        
        // Batch multiple read calls
        Multicall3.Call[] memory calls = new Multicall3.Call[](3);
        calls[0] = Multicall3.Call({
            target: address(tokenA),
            callData: abi.encodeCall(IERC20.balanceOf, (user))
        });
        calls[1] = Multicall3.Call({
            target: address(tokenB),
            callData: abi.encodeCall(IERC20.balanceOf, (user))
        });
        calls[2] = Multicall3.Call({
            target: address(multicall),
            callData: abi.encodeCall(Multicall3.getBlockNumber, ())
        });
        
        (uint256 blockNum, bytes[] memory results) = multicall.aggregate(calls);
        
        uint256 balA = abi.decode(results[0], (uint256));
        uint256 balB = abi.decode(results[1], (uint256));
        uint256 currentBlock = abi.decode(results[2], (uint256));
        
        assertEq(currentBlock, blockNum);
        console.log("TokenA balance:", balA);
        console.log("TokenB balance:", balB);
        console.log("Block number:", currentBlock);
    }
    
    function _getExpectedOut(address pool, address tokenIn, uint256 amountIn) 
        internal view returns (uint256) {
        (uint256 r0, uint256 r1,) = IUniswapV2Pair(pool).getReserves();
        address t0 = IUniswapV2Pair(pool).token0();
        (uint256 rIn, uint256 rOut) = tokenIn == t0 ? (r0, r1) : (r1, r0);
        uint256 amountInFee = amountIn * 997;
        return amountInFee * rOut / (rIn * 1000 + amountInFee);
    }
}
```

### Workshop 2: Yield Aggregator Integration

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @notice Test Yield Aggregator rebalancing logic
 */
contract YieldAggregatorTest is Test {
    
    YieldAggregator public aggregator;
    MockERC20 public usdc;
    MockAave public aave;
    MockCompound public compound;
    
    address user = address(0xUSER);
    
    function setUp() public {
        usdc = new MockERC20("USD Coin", "USDC", 6);
        aave = new MockAave();
        compound = new MockCompound();
        
        aggregator = new YieldAggregator(address(usdc));
        
        // Add strategies
        aggregator.addStrategy(address(aave), address(usdc));
        aggregator.addStrategy(address(compound), address(usdc));
        
        // Fund user
        usdc.mint(user, 100000e6);
        vm.prank(user);
        usdc.approve(address(aggregator), type(uint256).max);
    }
    
    function test_deposit_and_rebalance() public {
        // Set APY: Aave = 5%, Compound = 3%
        aave.setAPY(500);     // 5% in bps
        compound.setAPY(300); // 3% in bps
        
        uint256 depositAmount = 10000e6;
        
        vm.prank(user);
        uint256 shares = aggregator.deposit(depositAmount);
        
        assertGt(shares, 0);
        
        // Change APY: now Compound is better
        aave.setAPY(200);     // 2%
        compound.setAPY(600); // 6%
        
        // Rebalance should move funds to Compound
        aggregator.rebalance();
        
        // Verify rebalance occurred
        // (check balances in each protocol)
    }
}
```

---

## สรุป Part 87

- **PathFinder**: ค้นหา best route โดยเปรียบเทียบ quotes จากหลาย pools รองรับ UniV2/V3/Curve
- **AggregatorRouter**: Execute single, multi-hop, และ split route swaps พร้อม slippage protection
- **YieldAggregator**: Auto-rebalance ระหว่าง lending protocols ตาม APY comparison
- **Multicall3**: Batch multiple calls ใน single transaction รองรับ allowFailure และ ETH value
- **PriceImpactOptimizer**: คำนวณ optimal split เพื่อลด price impact สำหรับ large orders

## Next: Part 88 - Prediction Markets
