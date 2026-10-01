# Part 56: Advanced AMM Designs

## บทนำ

ใน Part ก่อนหน้านี้ เราได้เรียนรู้พื้นฐานของ AMM (Automated Market Maker) แบบ Constant Product (x*y=k) ซึ่งใช้ใน Uniswap V2
ในบทนี้ เราจะศึกษา AMM ที่ซับซ้อนและมีประสิทธิภาพสูงกว่า ได้แก่:

1. **StableSwap** (Curve-style) - สำหรับ stablecoin trading ที่มี slippage ต่ำ
2. **Concentrated Liquidity** (Uniswap V3 mechanics) - การจัดการ tick และ position
3. **Weighted Pool** (Balancer-style) - Pool ที่มี weight แบบกำหนดได้
4. **Virtual AMM** (สำหรับ perpetuals) - vAMM ที่ไม่ต้องการ real liquidity

---

## 1. StableSwap (Curve-style)

### ทฤษฎี StableSwap Invariant

Curve Protocol ใช้ invariant ที่รวมกัน:
- **Constant Sum** (x + y = k): slippage เป็นศูนย์ แต่สภาพคล่องหมดได้
- **Constant Product** (x * y = k): ทนทานกว่า แต่ slippage สูง

**StableSwap Invariant:**

```
A * n^n * Σx_i + D = A * n^n * D + D^(n+1) / (n^n * Π x_i)
```

เมื่อ:
- `A` คือ Amplification factor (ควบคุมว่าจะเอียงไปทาง constant sum มากแค่ไหน)
- `n` คือจำนวน tokens ใน pool
- `D` คือ invariant value (total liquidity)
- `x_i` คือ balance ของ token i

เมื่อ A → ∞ จะกลายเป็น constant sum, เมื่อ A = 0 จะกลายเป็น constant product

### การคำนวณ D (Newton's Method)

```
D_{n+1} = (A * n^n * S + n * D_P) * D / ((A * n^n - 1) * D + (n+1) * D_P)
```

เมื่อ:
- `S = Σx_i` คือผลรวม balances
- `D_P = D^(n+1) / (n^n * Π x_i)`

### Smart Contract: StableSwap

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title StableSwap
 * @notice Curve-style AMM สำหรับ stablecoin pairs
 * @dev Implements StableSwap invariant: A*n^n*Σx + D = A*n^n*D + D^(n+1)/(n^n*Πx)
 */
contract StableSwap is ERC20, ReentrancyGuard, Ownable {
    // ============ Constants ============
    uint256 public constant N_COINS = 2;
    uint256 public constant PRECISION = 1e18;
    uint256 public constant A_PRECISION = 100;
    uint256 public constant MAX_A = 1_000_000;
    uint256 public constant MAX_A_CHANGE = 10;    // max 10x change per ramp
    uint256 public constant MIN_RAMP_TIME = 86400; // 1 day minimum
    uint256 public constant FEE_DENOMINATOR = 1e10;
    uint256 public constant MAX_FEE = 5e7; // 0.5%

    // ============ State Variables ============
    address[N_COINS] public coins;
    uint256[N_COINS] public balances;
    uint256[N_COINS] public rates; // precision multipliers

    uint256 public fee;           // swap fee in FEE_DENOMINATOR units
    uint256 public adminFee;      // portion of fee going to admin
    uint256 public adminFeeAccumulated;

    // Amplification factor with ramp support
    uint256 public initialA;
    uint256 public futureA;
    uint256 public initialATime;
    uint256 public futureATime;

    // ============ Events ============
    event TokenExchange(
        address indexed buyer,
        int128 soldId,
        uint256 tokensSold,
        int128 boughtId,
        uint256 tokensBought
    );
    event AddLiquidity(
        address indexed provider,
        uint256[N_COINS] tokenAmounts,
        uint256[N_COINS] fees,
        uint256 invariant,
        uint256 tokenSupply
    );
    event RemoveLiquidity(
        address indexed provider,
        uint256[N_COINS] tokenAmounts,
        uint256 tokenSupply
    );
    event RampA(
        uint256 oldA,
        uint256 newA,
        uint256 initialTime,
        uint256 futureTime
    );

    // ============ Constructor ============
    constructor(
        address[N_COINS] memory _coins,
        uint256[N_COINS] memory _rates,
        uint256 _A,
        uint256 _fee,
        address _owner
    ) ERC20("Curve LP Token", "CRV-LP") Ownable(_owner) {
        require(_A > 0 && _A <= MAX_A, "StableSwap: invalid A");
        require(_fee <= MAX_FEE, "StableSwap: fee too high");

        coins = _coins;
        rates = _rates;
        fee = _fee;
        adminFee = 5000000000; // 50% of fee

        uint256 ampFactor = _A * A_PRECISION;
        initialA = ampFactor;
        futureA = ampFactor;
        initialATime = block.timestamp;
        futureATime = block.timestamp;
    }

    // ============ View Functions ============

    /**
     * @notice คำนวณค่า A ปัจจุบัน (รองรับ linear ramping)
     */
    function _A() internal view returns (uint256) {
        uint256 t1 = futureATime;
        uint256 A1 = futureA;

        if (block.timestamp < t1) {
            uint256 A0 = initialA;
            uint256 t0 = initialATime;
            // Linear interpolation
            if (A1 > A0) {
                return A0 + (A1 - A0) * (block.timestamp - t0) / (t1 - t0);
            } else {
                return A0 - (A0 - A1) * (block.timestamp - t0) / (t1 - t0);
            }
        }
        return A1;
    }

    /**
     * @notice คำนวณ D invariant ด้วย Newton's iteration method
     * @param xp Normalized balances (adjusted for precision)
     * @param amp Amplification factor
     * @return D invariant value
     */
    function getD(uint256[N_COINS] memory xp, uint256 amp) public pure returns (uint256) {
        uint256 S = 0;
        for (uint256 i = 0; i < N_COINS; i++) {
            S += xp[i];
        }
        if (S == 0) return 0;

        uint256 Dprev = 0;
        uint256 D = S;
        uint256 Ann = amp * N_COINS;

        for (uint256 i = 0; i < 255; i++) {
            uint256 D_P = D;
            for (uint256 j = 0; j < N_COINS; j++) {
                // D_P = D_P * D / (xp[j] * N_COINS)
                D_P = D_P * D / (xp[j] * N_COINS);
            }
            Dprev = D;
            // Newton's step:
            // D = (Ann * S + D_P * N_COINS) * D / ((Ann - 1) * D + (N_COINS + 1) * D_P)
            D = (Ann * S / A_PRECISION + D_P * N_COINS) * D /
                ((Ann - A_PRECISION) * D / A_PRECISION + (N_COINS + 1) * D_P);

            if (D > Dprev) {
                if (D - Dprev <= 1) return D;
            } else {
                if (Dprev - D <= 1) return D;
            }
        }
        revert("StableSwap: D did not converge");
    }

    /**
     * @notice คำนวณ balance ของ coin j หลังจาก exchange
     * @param i Index ของ coin ที่ส่งเข้า
     * @param j Index ของ coin ที่รับออก
     * @param x Amount ของ coin i ใหม่ (หลัง exchange)
     * @param xp Current normalized balances
     * @return y Balance ใหม่ของ coin j
     */
    function getY(
        uint256 i,
        uint256 j,
        uint256 x,
        uint256[N_COINS] memory xp
    ) internal view returns (uint256) {
        uint256 amp = _A();
        uint256 D = getD(xp, amp);

        uint256 Ann = amp * N_COINS;
        uint256 c = D;
        uint256 S_ = 0;

        uint256 _x = 0;
        for (uint256 k = 0; k < N_COINS; k++) {
            if (k == i) {
                _x = x;
            } else if (k != j) {
                _x = xp[k];
            } else {
                continue;
            }
            S_ += _x;
            c = c * D / (_x * N_COINS);
        }

        c = c * D * A_PRECISION / (Ann * N_COINS);
        uint256 b = S_ + D * A_PRECISION / Ann;

        uint256 y_prev = 0;
        uint256 y = D;

        for (uint256 k = 0; k < 255; k++) {
            y_prev = y;
            y = (y * y + c) / (2 * y + b - D);

            if (y > y_prev) {
                if (y - y_prev <= 1) return y;
            } else {
                if (y_prev - y <= 1) return y;
            }
        }
        revert("StableSwap: y did not converge");
    }

    /**
     * @notice ดึง normalized balances
     */
    function _xp() internal view returns (uint256[N_COINS] memory xp) {
        for (uint256 i = 0; i < N_COINS; i++) {
            xp[i] = balances[i] * rates[i] / PRECISION;
        }
    }

    /**
     * @notice คำนวณจำนวน tokens ที่ได้รับจากการ swap
     * @param i Index ของ input token
     * @param j Index ของ output token
     * @param dx Amount ของ input token
     * @return Output token amount (after fee)
     */
    function getDy(uint256 i, uint256 j, uint256 dx) public view returns (uint256) {
        uint256[N_COINS] memory xp = _xp();
        uint256 x = xp[i] + dx * rates[i] / PRECISION;
        uint256 y = getY(i, j, x, xp);

        uint256 dy = xp[j] - y - 1;
        uint256 _fee = dy * fee / FEE_DENOMINATOR;
        return (dy - _fee) * PRECISION / rates[j];
    }

    // ============ Core Exchange Function ============

    /**
     * @notice แลกเปลี่ยน token i เป็น token j
     * @param i Index ของ input token
     * @param j Index ของ output token
     * @param dx Amount ของ input token
     * @param minDy Minimum output amount (slippage protection)
     */
    function exchange(
        int128 i,
        int128 j,
        uint256 dx,
        uint256 minDy
    ) external nonReentrant returns (uint256) {
        require(i >= 0 && i < int128(int256(N_COINS)), "StableSwap: invalid i");
        require(j >= 0 && j < int128(int256(N_COINS)), "StableSwap: invalid j");
        require(i != j, "StableSwap: same token");
        require(dx > 0, "StableSwap: zero input");

        uint256 ui = uint256(int256(i));
        uint256 uj = uint256(int256(j));

        uint256[N_COINS] memory xp = _xp();
        uint256 x = xp[ui] + dx * rates[ui] / PRECISION;
        uint256 y = getY(ui, uj, x, xp);

        uint256 dy = xp[uj] - y - 1;
        uint256 dyFee = dy * fee / FEE_DENOMINATOR;
        uint256 dyAdminFee = dyFee * adminFee / FEE_DENOMINATOR;

        // อัปเดต balances
        balances[ui] += dx;
        balances[uj] -= (dy - dyFee) * PRECISION / rates[uj];
        adminFeeAccumulated += dyAdminFee * PRECISION / rates[uj];

        uint256 dyOut = (dy - dyFee) * PRECISION / rates[uj];
        require(dyOut >= minDy, "StableSwap: slippage");

        // Transfer tokens
        IERC20(coins[ui]).transferFrom(msg.sender, address(this), dx);
        IERC20(coins[uj]).transfer(msg.sender, dyOut);

        emit TokenExchange(msg.sender, i, dx, j, dyOut);
        return dyOut;
    }

    // ============ Liquidity Functions ============

    /**
     * @notice เพิ่ม liquidity เข้า pool
     * @param amounts จำนวน token แต่ละชนิดที่เพิ่ม
     * @param minMintAmount Minimum LP tokens ที่ต้องได้รับ
     */
    function addLiquidity(
        uint256[N_COINS] calldata amounts,
        uint256 minMintAmount
    ) external nonReentrant returns (uint256) {
        uint256 amp = _A();
        uint256 totalSupply_ = totalSupply();

        // คำนวณ D ก่อนเพิ่ม liquidity
        uint256[N_COINS] memory oldBalances = balances;
        uint256[N_COINS] memory xpOld = _xp();
        uint256 D0 = totalSupply_ == 0 ? 0 : getD(xpOld, amp);

        // อัปเดต balances ชั่วคราว
        uint256[N_COINS] memory newBalances;
        for (uint256 i = 0; i < N_COINS; i++) {
            newBalances[i] = oldBalances[i] + amounts[i];
        }

        uint256[N_COINS] memory xpNew;
        for (uint256 i = 0; i < N_COINS; i++) {
            xpNew[i] = newBalances[i] * rates[i] / PRECISION;
        }
        uint256 D1 = getD(xpNew, amp);
        require(D1 > D0, "StableSwap: D1 not greater than D0");

        uint256[N_COINS] memory fees;
        uint256 mintAmount;

        if (totalSupply_ > 0) {
            // คิด fee สำหรับ imbalanced deposits
            uint256 _fee = fee * N_COINS / (4 * (N_COINS - 1));

            for (uint256 i = 0; i < N_COINS; i++) {
                uint256 idealBalance = D1 * oldBalances[i] / D0;
                uint256 difference;
                if (idealBalance > newBalances[i]) {
                    difference = idealBalance - newBalances[i];
                } else {
                    difference = newBalances[i] - idealBalance;
                }
                fees[i] = _fee * difference / FEE_DENOMINATOR;
                balances[i] = newBalances[i] - fees[i] * adminFee / FEE_DENOMINATOR;
                newBalances[i] -= fees[i];
            }

            for (uint256 i = 0; i < N_COINS; i++) {
                xpNew[i] = newBalances[i] * rates[i] / PRECISION;
            }
            uint256 D2 = getD(xpNew, amp);
            mintAmount = totalSupply_ * (D2 - D0) / D0;
        } else {
            // First liquidity provision
            for (uint256 i = 0; i < N_COINS; i++) {
                balances[i] = amounts[i];
            }
            mintAmount = D1;
        }

        require(mintAmount >= minMintAmount, "StableSwap: slippage");

        // Transfer tokens in
        for (uint256 i = 0; i < N_COINS; i++) {
            if (amounts[i] > 0) {
                IERC20(coins[i]).transferFrom(msg.sender, address(this), amounts[i]);
            }
        }

        _mint(msg.sender, mintAmount);
        emit AddLiquidity(msg.sender, amounts, fees, D1, totalSupply() );
        return mintAmount;
    }

    /**
     * @notice ถอน liquidity แบบสมดุล
     * @param lpAmount จำนวน LP tokens ที่เผา
     * @param minAmounts Minimum amounts ที่ต้องได้รับ
     */
    function removeLiquidity(
        uint256 lpAmount,
        uint256[N_COINS] calldata minAmounts
    ) external nonReentrant returns (uint256[N_COINS] memory amounts) {
        uint256 totalSupply_ = totalSupply();
        require(lpAmount > 0 && lpAmount <= totalSupply_, "StableSwap: invalid amount");

        for (uint256 i = 0; i < N_COINS; i++) {
            amounts[i] = balances[i] * lpAmount / totalSupply_;
            require(amounts[i] >= minAmounts[i], "StableSwap: slippage");
            balances[i] -= amounts[i];
            IERC20(coins[i]).transfer(msg.sender, amounts[i]);
        }

        _burn(msg.sender, lpAmount);
        emit RemoveLiquidity(msg.sender, amounts, totalSupply());
    }

    // ============ Admin Functions ============

    /**
     * @notice เริ่ม ramp A parameter อย่างช้าๆ
     */
    function rampA(uint256 futureA_, uint256 futureATime_) external onlyOwner {
        require(block.timestamp >= initialATime + MIN_RAMP_TIME, "StableSwap: too soon");
        require(futureATime_ >= block.timestamp + MIN_RAMP_TIME, "StableSwap: ramp too short");

        uint256 currentA = _A();
        uint256 futureAScaled = futureA_ * A_PRECISION;

        require(futureAScaled > 0 && futureAScaled <= MAX_A * A_PRECISION, "StableSwap: invalid future A");
        require(
            (futureAScaled >= currentA && futureAScaled <= currentA * MAX_A_CHANGE) ||
            (futureAScaled < currentA && futureAScaled * MAX_A_CHANGE >= currentA),
            "StableSwap: A change too large"
        );

        initialA = currentA;
        futureA = futureAScaled;
        initialATime = block.timestamp;
        futureATime = futureATime_;

        emit RampA(currentA, futureAScaled, block.timestamp, futureATime_);
    }

    /**
     * @notice ดึง spot price ระหว่าง coin i และ coin j
     */
    function spotPrice(uint256 i, uint256 j) external view returns (uint256) {
        uint256[N_COINS] memory xp = _xp();
        // สำหรับ stable swap, spot price ≈ rates[i]/rates[j] เมื่ออยู่ใกล้ equilibrium
        // แต่คำนวณที่แม่นยำกว่าโดยใช้ dx เล็กๆ
        uint256 dx = xp[i] / 1e6; // 0.0001% ของ balance
        if (dx == 0) dx = 1;

        uint256 x = xp[i] + dx;
        uint256 y = getY(i, j, x, xp);
        uint256 dy = xp[j] - y;

        // price = dy/dx, adjusted for rates
        return dy * PRECISION / dx;
    }

    function getVirtualPrice() external view returns (uint256) {
        uint256 D = getD(_xp(), _A());
        uint256 supply = totalSupply();
        if (supply == 0) return PRECISION;
        return D * PRECISION / supply;
    }
}
```

---

## 2. Concentrated Liquidity V3 Mechanics

### ทฤษฎี Concentrated Liquidity

ใน Uniswap V3 liquidity ถูกกระจายใน price range `[p_lower, p_upper]` แทนที่จะเป็น 0 ถึง ∞

**Tick System:**
```
price = 1.0001^tick
tick = floor(log(price) / log(1.0001))
```

**Liquidity Formula:**
```
L = sqrt(x * y)
x = L * (sqrt(p_upper) - sqrt(p)) / (sqrt(p) * sqrt(p_upper))
y = L * (sqrt(p) - sqrt(p_lower))
```

### Smart Contract: Tick & Position Management

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/utils/math/Math.sol";

/**
 * @title TickMath
 * @notice คำนวณ sqrt price จาก tick
 */
library TickMath {
    int24 public constant MIN_TICK = -887272;
    int24 public constant MAX_TICK = 887272;

    uint160 public constant MIN_SQRT_RATIO = 4295128739;
    uint160 public constant MAX_SQRT_RATIO = 1461446703485210103287273052203988822378723970342;

    /**
     * @notice แปลง tick เป็น sqrtPriceX96
     * @dev Q64.96 fixed point format
     */
    function getSqrtRatioAtTick(int24 tick) internal pure returns (uint160 sqrtPriceX96) {
        uint256 absTick = tick < 0 ? uint256(-int256(tick)) : uint256(int256(tick));
        require(absTick <= uint256(int256(MAX_TICK)), "T");

        uint256 ratio = absTick & 0x1 != 0
            ? 0xfffcb933bd6fad37aa2d162d1a594001
            : 0x100000000000000000000000000000000;

        if (absTick & 0x2 != 0) ratio = (ratio * 0xfff97272373d413259a46990580e213a) >> 128;
        if (absTick & 0x4 != 0) ratio = (ratio * 0xfff2e50f5f656932ef12357cf3c7fdcc) >> 128;
        if (absTick & 0x8 != 0) ratio = (ratio * 0xffe5caca7e10e4e61c3624eaa0941cd0) >> 128;
        if (absTick & 0x10 != 0) ratio = (ratio * 0xffcb9843d60f6159c9db58835c926644) >> 128;
        if (absTick & 0x20 != 0) ratio = (ratio * 0xff973b41fa98c081472e6896dfb254c0) >> 128;
        if (absTick & 0x40 != 0) ratio = (ratio * 0xff2ea16466c96a3843ec78b326b52861) >> 128;
        if (absTick & 0x80 != 0) ratio = (ratio * 0xfe5dee046a99a2a811c461f1969c3053) >> 128;
        if (absTick & 0x100 != 0) ratio = (ratio * 0xfcbe86c7900a88aedcffc83b479aa3a4) >> 128;

        if (tick > 0) ratio = type(uint256).max / ratio;

        sqrtPriceX96 = uint160((ratio >> 32) + (ratio % (1 << 32) == 0 ? 0 : 1));
    }

    /**
     * @notice แปลง sqrtPriceX96 เป็น tick
     */
    function getTickAtSqrtRatio(uint160 sqrtPriceX96) internal pure returns (int24 tick) {
        require(
            sqrtPriceX96 >= MIN_SQRT_RATIO && sqrtPriceX96 < MAX_SQRT_RATIO,
            "R"
        );

        uint256 ratio = uint256(sqrtPriceX96) << 32;
        uint256 r = ratio;
        uint256 msb = 0;

        assembly {
            let f := shl(7, gt(r, 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF))
            msb := or(msb, f)
            r := shr(f, r)
        }
        assembly {
            let f := shl(6, gt(r, 0xFFFFFFFFFFFFFFFF))
            msb := or(msb, f)
            r := shr(f, r)
        }

        int256 log_2 = (int256(msb) - 128) << 64;
        assembly {
            r := shr(127, mul(r, r))
            let f := shr(128, r)
            log_2 := or(log_2, shl(63, f))
            r := shr(f, r)
        }

        int256 log_sqrt10001 = log_2 * 255738958999603826347141;
        int24 tickLow = int24((log_sqrt10001 - 3402992956809132418596140100660247210) >> 128);
        int24 tickHi = int24((log_sqrt10001 + 291339464771989622907027621153398088495) >> 128);

        tick = tickLow == tickHi
            ? tickLow
            : getSqrtRatioAtTick(tickHi) <= sqrtPriceX96
                ? tickHi
                : tickLow;
    }
}

/**
 * @title TickBitmap
 * @notice จัดการ bitmap สำหรับ initialized ticks
 */
library TickBitmap {
    /**
     * @notice คำนวณ word position และ bit position จาก tick
     */
    function position(int24 tick) private pure returns (int16 wordPos, uint8 bitPos) {
        wordPos = int16(tick >> 8);
        bitPos = uint8(int8(tick % 256));
    }

    /**
     * @notice กำหนดหรือยกเลิก tick ใน bitmap
     */
    function flipTick(
        mapping(int16 => uint256) storage self,
        int24 tick,
        int24 tickSpacing
    ) internal {
        require(tick % tickSpacing == 0, "TickBitmap: invalid tick");
        (int16 wordPos, uint8 bitPos) = position(tick / tickSpacing);
        uint256 mask = 1 << bitPos;
        self[wordPos] ^= mask;
    }

    /**
     * @notice หา tick ที่ initialized ถัดไปหรือก่อนหน้า
     */
    function nextInitializedTickWithinOneWord(
        mapping(int16 => uint256) storage self,
        int24 tick,
        int24 tickSpacing,
        bool lte
    ) internal view returns (int24 next, bool initialized) {
        int24 compressed = tick / tickSpacing;
        if (tick < 0 && tick % tickSpacing != 0) compressed--;

        if (lte) {
            (int16 wordPos, uint8 bitPos) = position(compressed);
            uint256 mask = (1 << bitPos) - 1 + (1 << bitPos);
            uint256 masked = self[wordPos] & mask;

            initialized = masked != 0;
            next = initialized
                ? (compressed - int24(uint24(bitPos - _msb(masked)))) * tickSpacing
                : (compressed - int24(uint24(bitPos))) * tickSpacing;
        } else {
            (int16 wordPos, uint8 bitPos) = position(compressed + 1);
            uint256 mask = ~((1 << bitPos) - 1);
            uint256 masked = self[wordPos] & mask;

            initialized = masked != 0;
            next = initialized
                ? (compressed + 1 + int24(uint24(_lsb(masked) - bitPos))) * tickSpacing
                : (compressed + 1 + int24(uint24(type(uint8).max - bitPos))) * tickSpacing;
        }
    }

    function _msb(uint256 x) private pure returns (uint8 r) {
        require(x > 0, "TickBitmap: zero input");
        if (x >= 0x100000000000000000000000000000000) { x >>= 128; r += 128; }
        if (x >= 0x10000000000000000) { x >>= 64; r += 64; }
        if (x >= 0x100000000) { x >>= 32; r += 32; }
        if (x >= 0x10000) { x >>= 16; r += 16; }
        if (x >= 0x100) { x >>= 8; r += 8; }
        if (x >= 0x10) { x >>= 4; r += 4; }
        if (x >= 0x4) { x >>= 2; r += 2; }
        if (x >= 0x2) r += 1;
    }

    function _lsb(uint256 x) private pure returns (uint8 r) {
        require(x > 0, "TickBitmap: zero input");
        r = 255;
        if (x & type(uint128).max > 0) { r -= 128; } else { x >>= 128; }
        if (x & type(uint64).max > 0) { r -= 64; } else { x >>= 64; }
        if (x & type(uint32).max > 0) { r -= 32; } else { x >>= 32; }
        if (x & type(uint16).max > 0) { r -= 16; } else { x >>= 16; }
        if (x & type(uint8).max > 0) { r -= 8; } else { x >>= 8; }
        if (x & 0xf > 0) { r -= 4; } else { x >>= 4; }
        if (x & 0x3 > 0) { r -= 2; } else { x >>= 2; }
        if (x & 0x1 > 0) r -= 1;
    }
}

/**
 * @title ConcentratedLiquidityPool
 * @notice Simplified Uniswap V3-style concentrated liquidity pool
 */
contract ConcentratedLiquidityPool {
    using TickBitmap for mapping(int16 => uint256);

    // ============ Structs ============
    struct Slot0 {
        uint160 sqrtPriceX96;
        int24 tick;
        uint16 observationIndex;
        uint128 liquidity;
    }

    struct Position {
        uint128 liquidity;
        uint256 feeGrowthInside0LastX128;
        uint256 feeGrowthInside1LastX128;
        uint128 tokensOwed0;
        uint128 tokensOwed1;
    }

    struct TickInfo {
        uint128 liquidityGross;        // ปริมาณ liquidity รวมที่อ้างอิง tick นี้
        int128 liquidityNet;           // liquidity ที่เพิ่ม/ลดเมื่อผ่าน tick นี้
        uint256 feeGrowthOutside0X128; // fee ที่เกิดขึ้นนอก tick นี้ (token0)
        uint256 feeGrowthOutside1X128; // fee ที่เกิดขึ้นนอก tick นี้ (token1)
        bool initialized;
    }

    // ============ Constants ============
    uint256 public constant Q96 = 0x1000000000000000000000000;
    uint256 public constant Q128 = 0x100000000000000000000000000000000;

    // ============ State ============
    Slot0 public slot0;
    address public immutable token0;
    address public immutable token1;
    uint24 public immutable fee;
    int24 public immutable tickSpacing;

    uint256 public feeGrowthGlobal0X128;
    uint256 public feeGrowthGlobal1X128;

    mapping(bytes32 => Position) public positions;
    mapping(int24 => TickInfo) public ticks;
    mapping(int16 => uint256) public tickBitmap;

    // ============ Events ============
    event Mint(
        address sender,
        address indexed owner,
        int24 indexed tickLower,
        int24 indexed tickUpper,
        uint128 amount,
        uint256 amount0,
        uint256 amount1
    );
    event Collect(
        address indexed owner,
        address recipient,
        int24 indexed tickLower,
        int24 indexed tickUpper,
        uint128 amount0,
        uint128 amount1
    );
    event Swap(
        address indexed sender,
        address indexed recipient,
        int256 amount0,
        int256 amount1,
        uint160 sqrtPriceX96,
        uint128 liquidity,
        int24 tick
    );

    constructor(
        address _token0,
        address _token1,
        uint24 _fee,
        int24 _tickSpacing,
        uint160 _sqrtPriceX96
    ) {
        token0 = _token0;
        token1 = _token1;
        fee = _fee;
        tickSpacing = _tickSpacing;

        slot0.sqrtPriceX96 = _sqrtPriceX96;
        slot0.tick = TickMath.getTickAtSqrtRatio(_sqrtPriceX96);
    }

    // ============ Position Key ============

    function _positionKey(
        address owner,
        int24 tickLower,
        int24 tickUpper
    ) internal pure returns (bytes32) {
        return keccak256(abi.encodePacked(owner, tickLower, tickUpper));
    }

    // ============ Fee Growth Inside Calculation ============

    /**
     * @notice คำนวณ fee growth ที่เกิดขึ้นภายใน tick range
     * @dev นี่คือหัวใจสำคัญของ V3 fee accounting
     */
    function _getFeeGrowthInside(
        int24 tickLower,
        int24 tickUpper,
        int24 tickCurrent,
        uint256 feeGrowthGlobal0X128_,
        uint256 feeGrowthGlobal1X128_
    ) internal view returns (
        uint256 feeGrowthInside0X128,
        uint256 feeGrowthInside1X128
    ) {
        TickInfo storage lower = ticks[tickLower];
        TickInfo storage upper = ticks[tickUpper];

        uint256 feeGrowthBelow0X128;
        uint256 feeGrowthBelow1X128;
        if (tickCurrent >= tickLower) {
            feeGrowthBelow0X128 = lower.feeGrowthOutside0X128;
            feeGrowthBelow1X128 = lower.feeGrowthOutside1X128;
        } else {
            // tick ปัจจุบันอยู่ต่ำกว่า tickLower
            feeGrowthBelow0X128 = feeGrowthGlobal0X128_ - lower.feeGrowthOutside0X128;
            feeGrowthBelow1X128 = feeGrowthGlobal1X128_ - lower.feeGrowthOutside1X128;
        }

        uint256 feeGrowthAbove0X128;
        uint256 feeGrowthAbove1X128;
        if (tickCurrent < tickUpper) {
            feeGrowthAbove0X128 = upper.feeGrowthOutside0X128;
            feeGrowthAbove1X128 = upper.feeGrowthOutside1X128;
        } else {
            // tick ปัจจุบันอยู่สูงกว่าหรือเท่ากับ tickUpper
            feeGrowthAbove0X128 = feeGrowthGlobal0X128_ - upper.feeGrowthOutside0X128;
            feeGrowthAbove1X128 = feeGrowthGlobal1X128_ - upper.feeGrowthOutside1X128;
        }

        // fee inside = global - below - above (arithmetic works modulo 2^256)
        unchecked {
            feeGrowthInside0X128 = feeGrowthGlobal0X128_ - feeGrowthBelow0X128 - feeGrowthAbove0X128;
            feeGrowthInside1X128 = feeGrowthGlobal1X128_ - feeGrowthBelow1X128 - feeGrowthAbove1X128;
        }
    }

    /**
     * @notice อัปเดต position และคำนวณ fees ที่เก็บได้
     */
    function _updatePosition(
        address owner,
        int24 tickLower,
        int24 tickUpper,
        int128 liquidityDelta,
        int24 tick
    ) internal returns (Position storage position) {
        bytes32 posKey = _positionKey(owner, tickLower, tickUpper);
        position = positions[posKey];

        (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128) = _getFeeGrowthInside(
            tickLower,
            tickUpper,
            tick,
            feeGrowthGlobal0X128,
            feeGrowthGlobal1X128
        );

        // คำนวณ fees ที่สะสมได้ตั้งแต่ครั้งล่าสุด
        if (position.liquidity > 0) {
            unchecked {
                position.tokensOwed0 += uint128(
                    (feeGrowthInside0X128 - position.feeGrowthInside0LastX128) *
                    position.liquidity / Q128
                );
                position.tokensOwed1 += uint128(
                    (feeGrowthInside1X128 - position.feeGrowthInside1LastX128) *
                    position.liquidity / Q128
                );
            }
        }

        position.feeGrowthInside0LastX128 = feeGrowthInside0X128;
        position.feeGrowthInside1LastX128 = feeGrowthInside1X128;

        if (liquidityDelta != 0) {
            if (liquidityDelta > 0) {
                position.liquidity += uint128(liquidityDelta);
            } else {
                position.liquidity -= uint128(-liquidityDelta);
            }
        }
    }

    /**
     * @notice ดึง fees ที่สะสมได้
     */
    function collectFees(
        address recipient,
        int24 tickLower,
        int24 tickUpper,
        uint128 amount0Requested,
        uint128 amount1Requested
    ) external returns (uint128 amount0, uint128 amount1) {
        Position storage position = _updatePosition(
            msg.sender, tickLower, tickUpper, 0, slot0.tick
        );

        amount0 = amount0Requested < position.tokensOwed0
            ? amount0Requested
            : position.tokensOwed0;
        amount1 = amount1Requested < position.tokensOwed1
            ? amount1Requested
            : position.tokensOwed1;

        position.tokensOwed0 -= amount0;
        position.tokensOwed1 -= amount1;

        if (amount0 > 0) IERC20(token0).transfer(recipient, amount0);
        if (amount1 > 0) IERC20(token1).transfer(recipient, amount1);

        emit Collect(msg.sender, recipient, tickLower, tickUpper, amount0, amount1);
    }
}
```

---

## 3. Balancer-style Weighted Pool

### ทฤษฎี Weighted AMM

Balancer ใช้ invariant:
```
V = Π (B_i ^ W_i)
```

เมื่อ:
- `B_i` คือ balance ของ token i
- `W_i` คือ weight ของ token i (Σ W_i = 1)

**Spot Price:**
```
SP_io = (B_i / W_i) / (B_o / W_o)
```

### Smart Contract: WeightedPool

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title FixedPoint
 * @notice Fixed-point math library (18 decimals)
 */
library FixedPoint {
    uint256 internal constant ONE = 1e18;
    uint256 internal constant MAX_POW_RELATIVE_ERROR = 10000; // 10^(-14)

    /**
     * @notice x^y computation using ln/exp
     * @dev Approximation valid for 0.7 < x < 1.3
     */
    function pow(uint256 x, uint256 y) internal pure returns (uint256) {
        if (y == 0) return ONE;
        if (x == ONE) return ONE;
        if (y == ONE) return x;

        // ln(x) * y
        int256 ln_x = _ln(int256(x));
        int256 result_exp = ln_x * int256(y) / int256(ONE);
        return uint256(_exp(result_exp));
    }

    function _ln(int256 x) private pure returns (int256) {
        // Taylor series approximation for ln around 1
        int256 ONE_INT = int256(ONE);
        require(x > 0, "ln: negative");

        // Normalize x to [0.5, 2]
        int256 a = 0;
        while (x >= 2 * ONE_INT) { x /= 2; a += int256(ONE); }
        while (x < ONE_INT / 2) { x *= 2; a -= int256(ONE); }

        int256 z = (x - ONE_INT) * ONE_INT / (x + ONE_INT);
        int256 z2 = z * z / ONE_INT;

        int256 sum = z;
        int256 zn = z * z2 / ONE_INT;
        sum += zn / 3;
        zn = zn * z2 / ONE_INT;
        sum += zn / 5;
        zn = zn * z2 / ONE_INT;
        sum += zn / 7;

        return 2 * sum + a * 693147180559945309 / ONE_INT; // 2*sum + a*ln(2)
    }

    function _exp(int256 x) private pure returns (int256) {
        int256 ONE_INT = int256(ONE);
        // e^x approximation
        int256 sum = ONE_INT;
        int256 xi = x;
        int256 factorial = ONE_INT;

        for (uint256 i = 1; i <= 12; i++) {
            factorial = factorial * int256(i);
            sum += xi / factorial;
            xi = xi * x / ONE_INT;
        }
        return sum;
    }

    function mulDown(uint256 a, uint256 b) internal pure returns (uint256) {
        uint256 product = a * b;
        return product / ONE;
    }

    function divDown(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b != 0, "FixedPoint: zero division");
        uint256 aInflated = a * ONE;
        return aInflated / b;
    }
}

/**
 * @title WeightedPool
 * @notice Balancer-style weighted pool with configurable token weights
 */
contract WeightedPool is ERC20, ReentrancyGuard, Ownable {
    using FixedPoint for uint256;

    // ============ Constants ============
    uint256 public constant MIN_WEIGHT = 0.01e18;  // 1%
    uint256 public constant MAX_WEIGHT = 0.99e18;  // 99%
    uint256 public constant MIN_TOKENS = 2;
    uint256 public constant MAX_TOKENS = 8;
    uint256 public constant ONE = 1e18;
    uint256 public constant FEE_DENOMINATOR = 1e18;
    uint256 public constant MAX_SWAP_FEE = 0.1e18; // 10%

    // ============ State Variables ============
    address[] public tokens;
    uint256[] public normalizedWeights; // sum = 1e18
    uint256[] public balances;
    uint256 public swapFeePercentage;

    // ============ Events ============
    event SwapExecuted(
        address indexed tokenIn,
        address indexed tokenOut,
        uint256 amountIn,
        uint256 amountOut
    );
    event LiquidityAdded(address indexed provider, uint256[] amounts, uint256 lpMinted);
    event LiquidityRemoved(address indexed provider, uint256 lpBurned, uint256[] amounts);

    constructor(
        address[] memory _tokens,
        uint256[] memory _weights,
        uint256 _swapFeePercentage,
        address _owner
    ) ERC20("Balancer LP", "BAL-LP") Ownable(_owner) {
        require(_tokens.length >= MIN_TOKENS && _tokens.length <= MAX_TOKENS, "WeightedPool: invalid token count");
        require(_tokens.length == _weights.length, "WeightedPool: length mismatch");
        require(_swapFeePercentage <= MAX_SWAP_FEE, "WeightedPool: fee too high");

        uint256 weightSum = 0;
        for (uint256 i = 0; i < _weights.length; i++) {
            require(_weights[i] >= MIN_WEIGHT, "WeightedPool: weight too low");
            require(_weights[i] <= MAX_WEIGHT, "WeightedPool: weight too high");
            weightSum += _weights[i];
        }
        require(weightSum == ONE, "WeightedPool: weights must sum to 1");

        tokens = _tokens;
        normalizedWeights = _weights;
        balances = new uint256[](_tokens.length);
        swapFeePercentage = _swapFeePercentage;
    }

    // ============ View Functions ============

    /**
     * @notice คำนวณ spot price ระหว่าง token in และ token out
     * @dev SP = (balanceIn / weightIn) / (balanceOut / weightOut)
     * @param tokenIn Address ของ input token
     * @param tokenOut Address ของ output token
     * @return Spot price (not including fees)
     */
    function getSpotPrice(address tokenIn, address tokenOut) public view returns (uint256) {
        (uint256 indexIn, uint256 indexOut) = _getTokenIndexes(tokenIn, tokenOut);

        uint256 numer = balances[indexIn].divDown(normalizedWeights[indexIn]);
        uint256 denom = balances[indexOut].divDown(normalizedWeights[indexOut]);

        return numer.divDown(denom);
    }

    /**
     * @notice คำนวณ output amount จาก swap
     * @dev Balancer weighted swap formula:
     *      Ao = Bo * (1 - (Bi / (Bi + Ai))^(Wi/Wo))
     */
    function calcOutGivenIn(
        uint256 balanceIn,
        uint256 weightIn,
        uint256 balanceOut,
        uint256 weightOut,
        uint256 amountIn
    ) public pure returns (uint256 amountOut) {
        // Apply fee: effectiveAmountIn = amountIn * (1 - fee)
        uint256 base = balanceIn.divDown(balanceIn + amountIn);
        uint256 exponent = weightIn.divDown(weightOut);
        uint256 power = FixedPoint.pow(base, exponent);

        amountOut = balanceOut.mulDown(ONE - power);
    }

    /**
     * @notice คำนวณ input amount ที่ต้องการเพื่อได้ output ที่กำหนด
     * @dev Ai = Bi * ((Bo / (Bo - Ao))^(Wo/Wi) - 1)
     */
    function calcInGivenOut(
        uint256 balanceIn,
        uint256 weightIn,
        uint256 balanceOut,
        uint256 weightOut,
        uint256 amountOut
    ) public pure returns (uint256 amountIn) {
        require(amountOut < balanceOut, "WeightedPool: insufficient liquidity");

        uint256 base = balanceOut.divDown(balanceOut - amountOut);
        uint256 exponent = weightOut.divDown(weightIn);
        uint256 power = FixedPoint.pow(base, exponent);

        amountIn = balanceIn.mulDown(power - ONE);
    }

    // ============ Swap ============

    /**
     * @notice แลกเปลี่ยน token แบบ exact input
     * @param tokenIn Input token address
     * @param tokenOut Output token address
     * @param amountIn จำนวน input tokens
     * @param minAmountOut Minimum output tokens (slippage protection)
     */
    function swapExactIn(
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 minAmountOut
    ) external nonReentrant returns (uint256 amountOut) {
        (uint256 indexIn, uint256 indexOut) = _getTokenIndexes(tokenIn, tokenOut);

        // หัก fee
        uint256 amountInAfterFee = amountIn.mulDown(ONE - swapFeePercentage);

        amountOut = calcOutGivenIn(
            balances[indexIn],
            normalizedWeights[indexIn],
            balances[indexOut],
            normalizedWeights[indexOut],
            amountInAfterFee
        );

        require(amountOut >= minAmountOut, "WeightedPool: slippage");

        // อัปเดต balances
        balances[indexIn] += amountIn;
        balances[indexOut] -= amountOut;

        IERC20(tokenIn).transferFrom(msg.sender, address(this), amountIn);
        IERC20(tokenOut).transfer(msg.sender, amountOut);

        emit SwapExecuted(tokenIn, tokenOut, amountIn, amountOut);
    }

    // ============ Liquidity ============

    /**
     * @notice เพิ่ม liquidity แบบ proportional
     * @param amounts จำนวน token แต่ละชนิด
     * @param minBptOut Minimum LP tokens
     */
    function joinPool(
        uint256[] calldata amounts,
        uint256 minBptOut
    ) external nonReentrant returns (uint256 bptOut) {
        require(amounts.length == tokens.length, "WeightedPool: invalid amounts");
        uint256 totalSupply_ = totalSupply();

        if (totalSupply_ == 0) {
            // Initial join: BPT = invariant
            uint256 invariant = ONE;
            for (uint256 i = 0; i < tokens.length; i++) {
                require(amounts[i] > 0, "WeightedPool: zero amount");
                invariant = invariant.mulDown(FixedPoint.pow(amounts[i], normalizedWeights[i]));
            }
            bptOut = invariant;
        } else {
            // Proportional join
            uint256 ratio = type(uint256).max;
            for (uint256 i = 0; i < tokens.length; i++) {
                if (balances[i] > 0) {
                    uint256 tokenRatio = amounts[i].divDown(balances[i]);
                    if (tokenRatio < ratio) ratio = tokenRatio;
                }
            }
            bptOut = totalSupply_.mulDown(ratio);
        }

        require(bptOut >= minBptOut, "WeightedPool: slippage");

        for (uint256 i = 0; i < tokens.length; i++) {
            if (amounts[i] > 0) {
                balances[i] += amounts[i];
                IERC20(tokens[i]).transferFrom(msg.sender, address(this), amounts[i]);
            }
        }

        _mint(msg.sender, bptOut);
        emit LiquidityAdded(msg.sender, amounts, bptOut);
    }

    /**
     * @notice ถอน liquidity แบบ proportional
     */
    function exitPool(
        uint256 bptIn,
        uint256[] calldata minAmountsOut
    ) external nonReentrant returns (uint256[] memory amountsOut) {
        uint256 totalSupply_ = totalSupply();
        require(bptIn > 0 && bptIn <= totalSupply_, "WeightedPool: invalid bptIn");

        amountsOut = new uint256[](tokens.length);
        uint256 ratio = bptIn.divDown(totalSupply_);

        for (uint256 i = 0; i < tokens.length; i++) {
            amountsOut[i] = balances[i].mulDown(ratio);
            require(amountsOut[i] >= minAmountsOut[i], "WeightedPool: slippage");
            balances[i] -= amountsOut[i];
            IERC20(tokens[i]).transfer(msg.sender, amountsOut[i]);
        }

        _burn(msg.sender, bptIn);
        emit LiquidityRemoved(msg.sender, bptIn, amountsOut);
    }

    // ============ Helper Functions ============

    function _getTokenIndexes(
        address tokenA,
        address tokenB
    ) internal view returns (uint256 indexA, uint256 indexB) {
        bool foundA = false;
        bool foundB = false;

        for (uint256 i = 0; i < tokens.length; i++) {
            if (tokens[i] == tokenA) { indexA = i; foundA = true; }
            if (tokens[i] == tokenB) { indexB = i; foundB = true; }
        }

        require(foundA && foundB, "WeightedPool: token not found");
        require(indexA != indexB, "WeightedPool: same token");
    }

    function getTokens() external view returns (address[] memory, uint256[] memory, uint256[] memory) {
        return (tokens, normalizedWeights, balances);
    }
}
```

---

## 4. Virtual AMM (vAMM for Perpetuals)

### ทฤษฎี vAMM

vAMM ใช้ mechanics ของ x*y=k แต่ไม่มี real liquidity - ราคาถูกกำหนดโดย virtual reserves

**ข้อดี:**
- ไม่ต้องการ liquidity providers
- ราคาตอบสนองต่อ trading pressure โดยตรง
- ง่ายต่อการกำหนด funding rate

### Smart Contract: vAMM

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

/**
 * @title VirtualAMM
 * @notice Virtual AMM for perpetual futures trading
 * @dev ราคาถูกกำหนดโดย virtual x*y=k โดยไม่มี real liquidity
 *      Traders deposit collateral และเทรดกับ virtual reserves
 */
contract VirtualAMM is Ownable, ReentrancyGuard {
    // ============ Enums ============
    enum Side { LONG, SHORT }

    // ============ Structs ============
    struct Position {
        address trader;
        Side side;
        uint256 margin;          // collateral deposited
        uint256 openNotional;    // USD value at open
        uint256 size;            // position size in base asset
        uint256 entryPrice;      // entry price
        int256  unrealizedPnL;   // unrealized profit/loss
        uint256 lastFundingIndex;
    }

    struct AMMLiveState {
        uint256 quoteAssetReserve;  // y - USD side (virtual)
        uint256 baseAssetReserve;   // x - asset side (virtual)
        uint256 k;                  // invariant = x * y
    }

    // ============ Constants ============
    uint256 public constant PRICE_PRECISION = 1e18;
    uint256 public constant FUNDING_PERIOD = 8 hours;
    uint256 public constant MAX_LEVERAGE = 20;
    uint256 public constant LIQUIDATION_THRESHOLD = 0.05e18; // 5% margin ratio

    // ============ State ============
    IERC20 public immutable collateral; // USDC or similar

    AMMLiveState public amm;
    mapping(address => Position) public positions;
    mapping(address => uint256) public collateralBalance;

    // Funding rate tracking
    int256 public fundingRate;          // per FUNDING_PERIOD
    uint256 public fundingIndex;        // cumulative funding
    uint256 public lastFundingTime;

    // Oracle for index price (external)
    address public priceOracle;
    uint256 public indexPrice;

    // Protocol stats
    uint256 public totalLongOpenInterest;
    uint256 public totalShortOpenInterest;

    // ============ Events ============
    event PositionOpened(
        address indexed trader,
        Side side,
        uint256 margin,
        uint256 size,
        uint256 entryPrice,
        uint256 leverage
    );
    event PositionClosed(
        address indexed trader,
        int256 realizedPnL,
        uint256 exitPrice
    );
    event PositionLiquidated(
        address indexed trader,
        address indexed liquidator,
        uint256 liquidationFee
    );
    event FundingPaid(
        int256 fundingRate,
        uint256 timestamp
    );
    event VirtualKUpdated(uint256 oldK, uint256 newK);

    constructor(
        address _collateral,
        uint256 _initialBaseReserve,
        uint256 _initialQuoteReserve,
        address _oracle
    ) Ownable(msg.sender) {
        collateral = IERC20(_collateral);
        amm.baseAssetReserve = _initialBaseReserve;
        amm.quoteAssetReserve = _initialQuoteReserve;
        amm.k = _initialBaseReserve * _initialQuoteReserve;
        priceOracle = _oracle;
        lastFundingTime = block.timestamp;
        indexPrice = _initialQuoteReserve * PRICE_PRECISION / _initialBaseReserve;
    }

    // ============ Price Functions ============

    /**
     * @notice ราคาปัจจุบันจาก vAMM (mark price)
     */
    function getMarkPrice() public view returns (uint256) {
        return amm.quoteAssetReserve * PRICE_PRECISION / amm.baseAssetReserve;
    }

    /**
     * @notice คำนวณราคาสำหรับเปิด position ขนาดหนึ่ง
     * @param side LONG หรือ SHORT
     * @param quoteAmount จำนวน USD notional
     * @return baseAmount จำนวน base asset ที่ได้
     * @return entryPrice ราคาเฉลี่ย
     */
    function getPositionPrice(
        Side side,
        uint256 quoteAmount
    ) public view returns (uint256 baseAmount, uint256 entryPrice) {
        if (side == Side.LONG) {
            // Trader buys base asset (pushes price up)
            // New quoteReserve = quoteReserve + quoteAmount
            // New baseReserve = k / (quoteReserve + quoteAmount)
            uint256 newQuoteReserve = amm.quoteAssetReserve + quoteAmount;
            uint256 newBaseReserve = amm.k / newQuoteReserve;
            baseAmount = amm.baseAssetReserve - newBaseReserve;
        } else {
            // Trader sells base asset (pushes price down)
            // New quoteReserve = quoteReserve - quoteAmount
            uint256 newQuoteReserve = amm.quoteAssetReserve - quoteAmount;
            uint256 newBaseReserve = amm.k / newQuoteReserve;
            baseAmount = newBaseReserve - amm.baseAssetReserve;
        }

        require(baseAmount > 0, "vAMM: zero output");
        entryPrice = quoteAmount * PRICE_PRECISION / baseAmount;
    }

    // ============ Position Management ============

    /**
     * @notice เปิด position
     * @param side LONG หรือ SHORT
     * @param marginAmount จำนวน collateral ที่ deposit
     * @param leverage leverage ที่ใช้ (1-20x)
     * @param minBaseAmount Minimum base asset ที่ต้องได้ (slippage)
     */
    function openPosition(
        Side side,
        uint256 marginAmount,
        uint256 leverage,
        uint256 minBaseAmount
    ) external nonReentrant {
        require(leverage >= 1 && leverage <= MAX_LEVERAGE, "vAMM: invalid leverage");
        require(marginAmount > 0, "vAMM: zero margin");
        require(positions[msg.sender].size == 0, "vAMM: position already open");

        // คำนวณ notional size
        uint256 quoteAmount = marginAmount * leverage;

        (uint256 baseAmount, uint256 entryPrice) = getPositionPrice(side, quoteAmount);
        require(baseAmount >= minBaseAmount, "vAMM: slippage");

        // Transfer collateral
        collateral.transferFrom(msg.sender, address(this), marginAmount);
        collateralBalance[address(this)] += marginAmount;

        // อัปเดต virtual reserves
        _settleVAMM(side, quoteAmount, baseAmount);

        // บันทึก position
        positions[msg.sender] = Position({
            trader: msg.sender,
            side: side,
            margin: marginAmount,
            openNotional: quoteAmount,
            size: baseAmount,
            entryPrice: entryPrice,
            unrealizedPnL: 0,
            lastFundingIndex: fundingIndex
        });

        // อัปเดต open interest
        if (side == Side.LONG) {
            totalLongOpenInterest += quoteAmount;
        } else {
            totalShortOpenInterest += quoteAmount;
        }

        emit PositionOpened(
            msg.sender, side, marginAmount, baseAmount, entryPrice, leverage
        );
    }

    /**
     * @notice ปิด position
     * @param minQuoteAmount Minimum USD ที่ต้องได้ (slippage)
     */
    function closePosition(uint256 minQuoteAmount) external nonReentrant {
        Position storage pos = positions[msg.sender];
        require(pos.size > 0, "vAMM: no position");

        // คำนวณ close price
        uint256 quoteAmount;
        if (pos.side == Side.LONG) {
            // ปิด LONG: ขาย base asset คืน → quoteReserve ลด
            uint256 newBaseReserve = amm.baseAssetReserve + pos.size;
            uint256 newQuoteReserve = amm.k / newBaseReserve;
            quoteAmount = amm.quoteAssetReserve - newQuoteReserve;
        } else {
            // ปิด SHORT: ซื้อ base asset คืน → quoteReserve เพิ่ม
            uint256 newBaseReserve = amm.baseAssetReserve - pos.size;
            uint256 newQuoteReserve = amm.k / newBaseReserve;
            quoteAmount = newQuoteReserve - amm.quoteAssetReserve;
        }

        require(quoteAmount >= minQuoteAmount, "vAMM: slippage");

        // คำนวณ PnL
        int256 pnl;
        if (pos.side == Side.LONG) {
            pnl = int256(quoteAmount) - int256(pos.openNotional);
        } else {
            pnl = int256(pos.openNotional) - int256(quoteAmount);
        }

        // คำนวณ funding payment
        int256 fundingPayment = _calcFundingPayment(pos);
        int256 totalPnL = pnl - fundingPayment;

        // คำนวณ margin ที่ได้คืน
        uint256 marginReturn;
        if (totalPnL >= 0) {
            marginReturn = pos.margin + uint256(totalPnL);
        } else {
            uint256 loss = uint256(-totalPnL);
            marginReturn = loss >= pos.margin ? 0 : pos.margin - loss;
        }

        // อัปเดต virtual reserves
        Side closingSide = pos.side == Side.LONG ? Side.SHORT : Side.LONG;
        _settleVAMM(closingSide, quoteAmount, pos.size);

        // อัปเดต open interest
        if (pos.side == Side.LONG) {
            totalLongOpenInterest -= pos.openNotional;
        } else {
            totalShortOpenInterest -= pos.openNotional;
        }

        // คืน margin
        uint256 exitPrice = quoteAmount * PRICE_PRECISION / pos.size;
        delete positions[msg.sender];

        if (marginReturn > 0) {
            collateral.transfer(msg.sender, marginReturn);
        }

        emit PositionClosed(msg.sender, totalPnL, exitPrice);
    }

    /**
     * @notice Liquidate position ที่มี margin ratio ต่ำเกินไป
     */
    function liquidate(address trader) external nonReentrant {
        Position storage pos = positions[trader];
        require(pos.size > 0, "vAMM: no position");
        require(_isLiquidatable(trader), "vAMM: not liquidatable");

        uint256 liquidationFee = pos.margin * 5 / 100; // 5% fee to liquidator
        if (liquidationFee > pos.margin) liquidationFee = pos.margin;

        // ปิด position โดยไม่คิด slippage protection
        uint256 quoteAmount;
        if (pos.side == Side.LONG) {
            uint256 newBaseReserve = amm.baseAssetReserve + pos.size;
            uint256 newQuoteReserve = amm.k / newBaseReserve;
            quoteAmount = amm.quoteAssetReserve - newQuoteReserve;
        } else {
            uint256 newBaseReserve = amm.baseAssetReserve - pos.size;
            uint256 newQuoteReserve = amm.k / newBaseReserve;
            quoteAmount = newQuoteReserve - amm.quoteAssetReserve;
        }

        Side closingSide = pos.side == Side.LONG ? Side.SHORT : Side.LONG;
        _settleVAMM(closingSide, quoteAmount, pos.size);

        if (pos.side == Side.LONG) {
            totalLongOpenInterest -= pos.openNotional;
        } else {
            totalShortOpenInterest -= pos.openNotional;
        }

        delete positions[trader];
        collateral.transfer(msg.sender, liquidationFee);

        emit PositionLiquidated(trader, msg.sender, liquidationFee);
    }

    // ============ Funding Rate ============

    /**
     * @notice คำนวณและอัปเดต funding rate
     * @dev Funding = (markPrice - indexPrice) / indexPrice
     *      Longs จ่าย Shorts เมื่อ mark > index
     */
    function settleFunding() external {
        require(
            block.timestamp >= lastFundingTime + FUNDING_PERIOD,
            "vAMM: too early"
        );

        uint256 markPrice = getMarkPrice();

        // Funding rate = premium / indexPrice
        int256 premium = int256(markPrice) - int256(indexPrice);
        fundingRate = premium * 1e18 / int256(indexPrice);

        // Limit funding rate to ±0.5% per period
        int256 maxRate = 0.005e18;
        if (fundingRate > maxRate) fundingRate = maxRate;
        if (fundingRate < -maxRate) fundingRate = -maxRate;

        fundingIndex = uint256(int256(fundingIndex) + fundingRate);
        lastFundingTime = block.timestamp;

        emit FundingPaid(fundingRate, block.timestamp);
    }

    /**
     * @notice Admin ปรับ virtual K เพื่อเพิ่ม/ลด depth
     */
    function adjustK(uint256 newBaseReserve, uint256 newQuoteReserve) external onlyOwner {
        require(newBaseReserve > 0 && newQuoteReserve > 0, "vAMM: invalid reserves");

        // คำนวณ mark price ใหม่ต้องเท่ากับเดิม
        uint256 currentMarkPrice = getMarkPrice();
        uint256 newMarkPrice = newQuoteReserve * PRICE_PRECISION / newBaseReserve;
        require(
            newMarkPrice > currentMarkPrice * 99 / 100 &&
            newMarkPrice < currentMarkPrice * 101 / 100,
            "vAMM: price drift too large"
        );

        uint256 oldK = amm.k;
        amm.baseAssetReserve = newBaseReserve;
        amm.quoteAssetReserve = newQuoteReserve;
        amm.k = newBaseReserve * newQuoteReserve;

        emit VirtualKUpdated(oldK, amm.k);
    }

    // ============ Internal Functions ============

    function _settleVAMM(Side side, uint256 quoteAmount, uint256 baseAmount) internal {
        if (side == Side.LONG) {
            amm.quoteAssetReserve += quoteAmount;
            amm.baseAssetReserve -= baseAmount;
        } else {
            amm.quoteAssetReserve -= quoteAmount;
            amm.baseAssetReserve += baseAmount;
        }
    }

    function _calcFundingPayment(Position storage pos) internal view returns (int256) {
        int256 indexDiff = int256(fundingIndex) - int256(pos.lastFundingIndex);
        if (pos.side == Side.LONG) {
            return int256(pos.openNotional) * indexDiff / 1e18;
        } else {
            return -(int256(pos.openNotional) * indexDiff / 1e18);
        }
    }

    function _isLiquidatable(address trader) internal view returns (bool) {
        Position storage pos = positions[trader];
        if (pos.size == 0) return false;

        // คำนวณ current value ของ position
        uint256 currentMarkPrice = getMarkPrice();
        int256 pnl;
        if (pos.side == Side.LONG) {
            pnl = int256(pos.size * currentMarkPrice / PRICE_PRECISION) - int256(pos.openNotional);
        } else {
            pnl = int256(pos.openNotional) - int256(pos.size * currentMarkPrice / PRICE_PRECISION);
        }

        int256 remainingMargin = int256(pos.margin) + pnl;
        if (remainingMargin <= 0) return true;

        uint256 marginRatio = uint256(remainingMargin) * PRICE_PRECISION / pos.openNotional;
        return marginRatio < LIQUIDATION_THRESHOLD;
    }

    // ============ View Functions ============

    function getPositionInfo(address trader) external view returns (
        Position memory position,
        uint256 currentMarkPrice,
        int256 unrealizedPnL,
        uint256 marginRatio
    ) {
        position = positions[trader];
        currentMarkPrice = getMarkPrice();

        if (position.size > 0) {
            uint256 currentValue = position.size * currentMarkPrice / PRICE_PRECISION;
            if (position.side == Side.LONG) {
                unrealizedPnL = int256(currentValue) - int256(position.openNotional);
            } else {
                unrealizedPnL = int256(position.openNotional) - int256(currentValue);
            }

            int256 remainingMargin = int256(position.margin) + unrealizedPnL;
            if (remainingMargin > 0) {
                marginRatio = uint256(remainingMargin) * PRICE_PRECISION / position.openNotional;
            }
        }
    }

    function setIndexPrice(uint256 newIndexPrice) external {
        require(msg.sender == priceOracle, "vAMM: not oracle");
        indexPrice = newIndexPrice;
    }
}
```

---

## Workshop และ Exercises

### Exercise 1: StableSwap - A Parameter Sensitivity

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title StableSwapAnalyzer
 * @notice วิเคราะห์ผลกระทบของ Amplification factor
 */
contract StableSwapAnalyzer {
    uint256 constant A_PRECISION = 100;
    uint256 constant N_COINS = 2;

    /**
     * @notice เปรียบเทียบ slippage ที่ A ต่างกัน
     * @param balanceA Balance ของ token A
     * @param balanceB Balance ของ token B
     * @param tradeAmount จำนวนที่ trade
     * @param ampFactor Amplification factor (e.g., 100, 1000, 10000)
     */
    function analyzeSlippage(
        uint256 balanceA,
        uint256 balanceB,
        uint256 tradeAmount,
        uint256 ampFactor
    ) external pure returns (
        uint256 outputAmount,
        uint256 effectivePrice,
        uint256 priceImpact,
        uint256 spotPrice
    ) {
        uint256[N_COINS] memory xp = [balanceA, balanceB];
        uint256 D = _getD(xp, ampFactor * A_PRECISION);

        uint256 x = xp[0] + tradeAmount;
        uint256 y = _getY(0, 1, x, xp, ampFactor * A_PRECISION, D);

        outputAmount = xp[1] - y;
        spotPrice = balanceA * 1e18 / balanceB;
        effectivePrice = tradeAmount * 1e18 / outputAmount;

        // Price impact = (effectivePrice - spotPrice) / spotPrice
        if (effectivePrice > spotPrice) {
            priceImpact = (effectivePrice - spotPrice) * 10000 / spotPrice;
        }
    }

    function _getD(uint256[2] memory xp, uint256 amp) internal pure returns (uint256) {
        uint256 S = xp[0] + xp[1];
        if (S == 0) return 0;

        uint256 D = S;
        uint256 Ann = amp * N_COINS;

        for (uint256 i = 0; i < 255; i++) {
            uint256 D_P = D * D / (xp[0] * N_COINS) * D / (xp[1] * N_COINS);
            uint256 Dprev = D;
            D = (Ann * S / A_PRECISION + D_P * N_COINS) * D /
                ((Ann - A_PRECISION) * D / A_PRECISION + (N_COINS + 1) * D_P);

            if (D > Dprev ? D - Dprev <= 1 : Dprev - D <= 1) return D;
        }
        revert("Did not converge");
    }

    function _getY(
        uint256 i,
        uint256 j,
        uint256 x,
        uint256[2] memory xp,
        uint256 amp,
        uint256 D
    ) internal pure returns (uint256) {
        uint256 Ann = amp * N_COINS;
        uint256 c = D;
        uint256 S_ = x;

        c = c * D / (x * N_COINS);
        c = c * D * A_PRECISION / (Ann * N_COINS);
        uint256 b = S_ + D * A_PRECISION / Ann;

        uint256 y = D;
        for (uint256 k = 0; k < 255; k++) {
            uint256 y_prev = y;
            y = (y * y + c) / (2 * y + b - D);
            if (y > y_prev ? y - y_prev <= 1 : y_prev - y <= 1) return y;
        }
        revert("y did not converge");
    }
}
```

### Exercise 2: Weighted Pool - Portfolio Management

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title PortfolioCalculator
 * @notice คำนวณ portfolio value และ rebalancing cost
 */
contract PortfolioCalculator {
    uint256 constant ONE = 1e18;

    /**
     * @notice คำนวณ portfolio value ด้วย weighted invariant
     */
    function calcPortfolioValue(
        uint256[] memory balances,
        uint256[] memory weights,
        uint256[] memory prices
    ) external pure returns (uint256 totalValue) {
        require(balances.length == weights.length, "length mismatch");
        require(balances.length == prices.length, "length mismatch");

        for (uint256 i = 0; i < balances.length; i++) {
            totalValue += balances[i] * prices[i] / ONE;
        }
    }

    /**
     * @notice คำนวณว่าต้อง trade เท่าไรเพื่อ rebalance
     */
    function calcRebalanceAmounts(
        uint256[] memory currentBalances,
        uint256[] memory targetWeights,
        uint256[] memory prices
    ) external pure returns (int256[] memory tradeDiffs) {
        uint256 n = currentBalances.length;
        tradeDiffs = new int256[](n);

        // คำนวณ total portfolio value
        uint256 totalValue = 0;
        for (uint256 i = 0; i < n; i++) {
            totalValue += currentBalances[i] * prices[i] / ONE;
        }

        // คำนวณ target balances
        for (uint256 i = 0; i < n; i++) {
            uint256 targetValue = totalValue * targetWeights[i] / ONE;
            uint256 targetBalance = targetValue * ONE / prices[i];
            tradeDiffs[i] = int256(targetBalance) - int256(currentBalances[i]);
        }
    }
}
```

### Exercise 3: vAMM - Risk Management System

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title RiskManager
 * @notice ระบบ risk management สำหรับ vAMM
 */
contract RiskManager {
    uint256 constant PRECISION = 1e18;

    struct RiskMetrics {
        uint256 totalOpenInterest;
        uint256 longShortImbalance; // basis points
        uint256 utilizationRate;    // basis points
        uint256 recommendedFundingRate; // basis points per period
    }

    /**
     * @notice คำนวณ risk metrics ของ protocol
     */
    function calcRiskMetrics(
        uint256 longOI,
        uint256 shortOI,
        uint256 maxOI
    ) external pure returns (RiskMetrics memory metrics) {
        metrics.totalOpenInterest = longOI + shortOI;

        if (longOI + shortOI > 0) {
            uint256 imbalance = longOI > shortOI
                ? longOI - shortOI
                : shortOI - longOI;
            metrics.longShortImbalance = imbalance * 10000 / (longOI + shortOI);
        }

        if (maxOI > 0) {
            metrics.utilizationRate = metrics.totalOpenInterest * 10000 / maxOI;
        }

        // Funding rate ที่แนะนำเพื่อ rebalance imbalance
        // สูงสุด 50 bps (0.5%) ต่อ period
        if (longOI > shortOI) {
            uint256 excess = longOI - shortOI;
            metrics.recommendedFundingRate = excess * 50 / (longOI + 1);
            if (metrics.recommendedFundingRate > 50) metrics.recommendedFundingRate = 50;
        }
    }

    /**
     * @notice คำนวณ liquidation price ของ position
     */
    function getLiquidationPrice(
        bool isLong,
        uint256 entryPrice,
        uint256 leverage,
        uint256 maintenanceMarginRate // basis points
    ) external pure returns (uint256 liquidationPrice) {
        // Maintenance margin ratio = 0.5% default
        uint256 mmr = maintenanceMarginRate; // e.g., 50 = 0.5%
        uint256 liquidationThreshold = (10000 - 10000 / leverage + mmr);

        if (isLong) {
            // Long liquidation: price drops by (1 - 1/leverage + mmr)
            liquidationPrice = entryPrice * (10000 - liquidationThreshold) / 10000;
        } else {
            // Short liquidation: price rises by (1 - 1/leverage + mmr)
            liquidationPrice = entryPrice * (10000 + liquidationThreshold) / 10000;
        }
    }
}
```

---

## สรุป Part 56

- **StableSwap (Curve-style)**: ใช้ invariant `A*n^n*Σx + D = A*n^n*D + D^(n+1)/(n^n*Πx)` ที่รวม constant sum และ constant product เข้าด้วยกัน โดย Amplification factor A ควบคุมว่าจะเอียงไปทางไหน
- **getD function**: ใช้ Newton's Method iteration เพื่อแก้หา D invariant ซึ่งแทน total liquidity
- **Concentrated Liquidity (V3)**: จัดการ ticks ด้วย bitmap, คำนวณ feeGrowthInside ด้วยการ track feeGrowthOutside สองข้างของ tick range
- **Weighted Pool (Balancer)**: ใช้ invariant `V = Π(B_i^W_i)` ที่ยืดหยุ่นกว่า สามารถมีหลาย token และ weight ที่กำหนดได้
- **Virtual AMM**: ใช้ x*y=k mechanics แต่บน virtual reserves โดยไม่มี real liquidity - เหมาะสำหรับ perpetuals ที่ต้องการ price discovery

## Next: Part 57 - Token Vesting & Distribution
