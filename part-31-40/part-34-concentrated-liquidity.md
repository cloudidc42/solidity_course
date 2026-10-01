# Part 34: Concentrated Liquidity (Uniswap V3)

## สารบัญ
1. V3 vs V2 Comparison
2. Price Ticks และ Ranges
3. Position Management
4. Fee Calculation
5. Workshop: LP Strategy

---

## 1. V3 vs V2

```
Uniswap V2:
- Liquidity กระจายทั่ว price range (0, ∞)
- Capital efficiency ต่ำ
- ราคาส่วนใหญ่อยู่แคบๆ แต่ liquidity อยู่กระจาย

Uniswap V3:
- LP เลือก price range ได้ (e.g., $1800-$2200 สำหรับ ETH)
- Capital efficiency สูงมาก (10-4000x)
- ได้ fee เฉพาะเมื่อราคาอยู่ใน range

Example:
- V2: LP ใส่ $10,000 → ครอบคลุม price ทั้งหมด
- V3: LP ใส่ $10,000 ใน $1800-$2200 ETH range
  → ราคาปัจจุบัน $2000 = เหมือนใส่ $130,000 ใน V2!

Fee Tiers:
- 0.01%: stable-stable pairs (USDC/USDT)
- 0.05%: stable-volatile (ETH/USDC)
- 0.30%: standard (most pairs)
- 1.00%: exotic pairs

Ticks:
- ราคาแบ่งเป็น discrete "ticks"
- Tick i → price = 1.0001^i
- tickSpacing: ขั้นต่ำระหว่าง ticks (ขึ้นกับ fee tier)
```

---

## 2. Tick Math

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Tick Math:
 * price = 1.0001^tick
 * sqrtPriceX96 = sqrt(price) × 2^96
 * 
 * ใช้ Q96 fixed-point arithmetic
 */
library TickMath {
    
    int24 public constant MIN_TICK = -887272;
    int24 public constant MAX_TICK = 887272;
    
    // Get sqrtPriceX96 from tick
    function getSqrtRatioAtTick(int24 tick) internal pure returns (uint160 sqrtPriceX96) {
        uint256 absTick = tick < 0 ? uint256(-int256(tick)) : uint256(int256(tick));
        require(absTick <= uint256(int256(MAX_TICK)), "T");
        
        uint256 ratio = absTick & 0x1 != 0 ? 0xfffcb933bd6fad37aa2d162d1a594001 : 0x100000000000000000000000000000000;
        
        // Precomputed powers of sqrt(1.0001)
        if (absTick & 0x2 != 0) ratio = (ratio * 0xfff97272373d413259a46990580e213a) >> 128;
        if (absTick & 0x4 != 0) ratio = (ratio * 0xfff2e50f5f656932ef12357cf3c7fdcc) >> 128;
        if (absTick & 0x8 != 0) ratio = (ratio * 0xffe5caca7e10e4e61c3624eaa0941cd0) >> 128;
        if (absTick & 0x10 != 0) ratio = (ratio * 0xffcb9843d60f6159c9db58835c926644) >> 128;
        // ... (more precomputed values)
        
        if (tick > 0) ratio = type(uint256).max / ratio;
        
        sqrtPriceX96 = uint160((ratio >> 32) + (ratio % (1 << 32) == 0 ? 0 : 1));
    }
    
    // Get tick from sqrtPriceX96
    function getTickAtSqrtRatio(uint160 sqrtPriceX96) internal pure returns (int24 tick) {
        require(sqrtPriceX96 >= getSqrtRatioAtTick(MIN_TICK) && sqrtPriceX96 < getSqrtRatioAtTick(MAX_TICK), "R");
        
        uint256 ratio = uint256(sqrtPriceX96) << 32;
        
        uint256 r = ratio;
        uint256 msb = 0;
        
        assembly {
            let f := shl(7, gt(r, 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF))
            msb := or(msb, f)
            r := shr(f, r)
        }
        // ... Binary search for tick
        
        int256 log_2 = (int256(msb) - 128) << 64;
        // ... more computation
        
        int256 log_sqrt10001 = log_2 * 255738958999603826347141;
        int24 tickLow = int24((log_sqrt10001 - 3402992956809132418596140100660247210) >> 128);
        int24 tickHigh = int24((log_sqrt10001 + 291339464771989622907191584880176258498) >> 128);
        
        tick = tickLow == tickHigh ? tickLow :
            getSqrtRatioAtTick(tickHigh) <= sqrtPriceX96 ? tickHigh : tickLow;
    }
}

/**
 * Liquidity Math:
 * จำนวน token ที่ต้องใส่สำหรับ liquidity ใน range
 */
library LiquidityMath {
    
    uint256 internal constant Q96 = 0x1000000000000000000000000;
    
    // Amount of token0 for liquidity in range [sqrtLower, sqrtUpper]
    function getAmount0(
        uint128 liquidity,
        uint160 sqrtRatioA,
        uint160 sqrtRatioB
    ) internal pure returns (uint256 amount0) {
        if (sqrtRatioA > sqrtRatioB) (sqrtRatioA, sqrtRatioB) = (sqrtRatioB, sqrtRatioA);
        
        return (uint256(liquidity) << 96) * (sqrtRatioB - sqrtRatioA) / sqrtRatioB / sqrtRatioA;
    }
    
    // Amount of token1 for liquidity in range
    function getAmount1(
        uint128 liquidity,
        uint160 sqrtRatioA,
        uint160 sqrtRatioB
    ) internal pure returns (uint256 amount1) {
        if (sqrtRatioA > sqrtRatioB) (sqrtRatioA, sqrtRatioB) = (sqrtRatioB, sqrtRatioA);
        
        return (uint256(liquidity) * (sqrtRatioB - sqrtRatioA)) / Q96;
    }
    
    // Calculate liquidity from token amounts and price range
    function getLiquidityForAmounts(
        uint160 sqrtRatioX96,
        uint160 sqrtRatioAX96,
        uint160 sqrtRatioBX96,
        uint256 amount0,
        uint256 amount1
    ) internal pure returns (uint128 liquidity) {
        if (sqrtRatioAX96 > sqrtRatioBX96) (sqrtRatioAX96, sqrtRatioBX96) = (sqrtRatioBX96, sqrtRatioAX96);
        
        if (sqrtRatioX96 <= sqrtRatioAX96) {
            // Price below range: all token0
            liquidity = uint128(amount0 * sqrtRatioAX96 / Q96 * sqrtRatioBX96 / (sqrtRatioBX96 - sqrtRatioAX96));
        } else if (sqrtRatioX96 < sqrtRatioBX96) {
            // Price in range: calculate min
            uint128 liquidity0 = uint128(amount0 * sqrtRatioX96 / Q96 * sqrtRatioBX96 / (sqrtRatioBX96 - sqrtRatioX96));
            uint128 liquidity1 = uint128(amount1 * Q96 / (sqrtRatioX96 - sqrtRatioAX96));
            liquidity = liquidity0 < liquidity1 ? liquidity0 : liquidity1;
        } else {
            // Price above range: all token1
            liquidity = uint128(amount1 * Q96 / (sqrtRatioBX96 - sqrtRatioAX96));
        }
    }
}
```

---

## 3. V3 Position NFT (Simplified)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * V3 Position Manager (Simplified)
 * 
 * ใน V3 จริง LP position ถือเป็น NFT
 * NonfungiblePositionManager: mint NFT เมื่อ add liquidity
 */
contract V3PositionManager {
    
    struct Position {
        address token0;
        address token1;
        uint24 fee;
        int24 tickLower;
        int24 tickUpper;
        uint128 liquidity;
        uint256 feeGrowthInside0LastX128;
        uint256 feeGrowthInside1LastX128;
        uint128 tokensOwed0;
        uint128 tokensOwed1;
    }
    
    mapping(uint256 => Position) public positions;
    uint256 public nextTokenId;
    mapping(uint256 => address) public ownerOf;
    
    IV3Pool public pool;
    
    event IncreaseLiquidity(uint256 indexed tokenId, uint128 liquidity, uint256 amount0, uint256 amount1);
    event DecreaseLiquidity(uint256 indexed tokenId, uint128 liquidity, uint256 amount0, uint256 amount1);
    event Collect(uint256 indexed tokenId, address recipient, uint256 amount0, uint256 amount1);
    
    struct MintParams {
        address token0;
        address token1;
        uint24 fee;
        int24 tickLower;
        int24 tickUpper;
        uint256 amount0Desired;
        uint256 amount1Desired;
        uint256 amount0Min;
        uint256 amount1Min;
        address recipient;
        uint256 deadline;
    }
    
    constructor(address _pool) {
        pool = IV3Pool(_pool);
    }
    
    function mint(MintParams calldata params) external returns (
        uint256 tokenId,
        uint128 liquidity,
        uint256 amount0,
        uint256 amount1
    ) {
        require(block.timestamp <= params.deadline, "Expired");
        
        // Calculate liquidity from amounts
        (uint160 sqrtPriceX96,,,,,,) = pool.slot0();
        
        liquidity = LiquidityMath.getLiquidityForAmounts(
            sqrtPriceX96,
            TickMath.getSqrtRatioAtTick(params.tickLower),
            TickMath.getSqrtRatioAtTick(params.tickUpper),
            params.amount0Desired,
            params.amount1Desired
        );
        
        // Add liquidity to pool
        (amount0, amount1) = pool.mint(
            address(this),
            params.tickLower,
            params.tickUpper,
            liquidity,
            abi.encode(msg.sender)
        );
        
        require(amount0 >= params.amount0Min && amount1 >= params.amount1Min, "Slippage");
        
        // Mint NFT
        tokenId = nextTokenId++;
        ownerOf[tokenId] = params.recipient;
        
        positions[tokenId] = Position({
            token0: params.token0,
            token1: params.token1,
            fee: params.fee,
            tickLower: params.tickLower,
            tickUpper: params.tickUpper,
            liquidity: liquidity,
            feeGrowthInside0LastX128: 0,
            feeGrowthInside1LastX128: 0,
            tokensOwed0: 0,
            tokensOwed1: 0
        });
        
        emit IncreaseLiquidity(tokenId, liquidity, amount0, amount1);
    }
    
    function decreaseLiquidity(
        uint256 tokenId,
        uint128 liquidityToRemove,
        uint256 amount0Min,
        uint256 amount1Min,
        uint256 deadline
    ) external returns (uint256 amount0, uint256 amount1) {
        require(block.timestamp <= deadline, "Expired");
        require(ownerOf[tokenId] == msg.sender, "Not owner");
        
        Position storage pos = positions[tokenId];
        require(pos.liquidity >= liquidityToRemove, "Insufficient liquidity");
        
        (amount0, amount1) = pool.burn(pos.tickLower, pos.tickUpper, liquidityToRemove);
        
        require(amount0 >= amount0Min && amount1 >= amount1Min, "Slippage");
        
        pos.liquidity -= liquidityToRemove;
        pos.tokensOwed0 += uint128(amount0);
        pos.tokensOwed1 += uint128(amount1);
        
        emit DecreaseLiquidity(tokenId, liquidityToRemove, amount0, amount1);
    }
    
    function collect(
        uint256 tokenId,
        address recipient,
        uint128 amount0Max,
        uint128 amount1Max
    ) external returns (uint256 amount0, uint256 amount1) {
        require(ownerOf[tokenId] == msg.sender, "Not owner");
        
        Position storage pos = positions[tokenId];
        
        amount0 = amount0Max > pos.tokensOwed0 ? pos.tokensOwed0 : amount0Max;
        amount1 = amount1Max > pos.tokensOwed1 ? pos.tokensOwed1 : amount1Max;
        
        pool.collect(recipient, pos.tickLower, pos.tickUpper, uint128(amount0), uint128(amount1));
        
        pos.tokensOwed0 -= uint128(amount0);
        pos.tokensOwed1 -= uint128(amount1);
        
        emit Collect(tokenId, recipient, amount0, amount1);
    }
}

interface IV3Pool {
    function slot0() external view returns (
        uint160 sqrtPriceX96,
        int24 tick,
        uint16 observationIndex,
        uint16 observationCardinality,
        uint16 observationCardinalityNext,
        uint8 feeProtocol,
        bool unlocked
    );
    
    function mint(address recipient, int24 tickLower, int24 tickUpper, uint128 amount, bytes calldata data)
        external returns (uint256 amount0, uint256 amount1);
    
    function burn(int24 tickLower, int24 tickUpper, uint128 amount)
        external returns (uint256 amount0, uint256 amount1);
    
    function collect(address recipient, int24 tickLower, int24 tickUpper, uint128 amount0Requested, uint128 amount1Requested)
        external returns (uint128 amount0, uint128 amount1);
}
```

---

## สรุป Part 34

Concentrated Liquidity ที่เรียนรู้:
- ✅ V2 vs V3 capital efficiency
- ✅ Tick system และ price ranges
- ✅ sqrtPriceX96 Q96 arithmetic
- ✅ Position NFT management
- ✅ Fee collection

## Quiz

1. Concentrated liquidity เพิ่ม capital efficiency อย่างไร?
2. Tick คืออะไร? price = 1.0001^tick หมายความอะไร?
3. LP ใน V3 อาจขาดทุนได้อย่างไร (impermanent loss)?
4. ทำไม V3 LP position ถึงเป็น NFT ไม่ใช่ fungible token?

---

## Next: Part 35 - Protocol Security and Hardening
