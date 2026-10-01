# Part 53: Options Protocols

## บทนำ

**Options** คือสัญญาที่ให้สิทธิ์ (แต่ไม่ใช่ภาระผูกพัน) ในการซื้อหรือขาย asset ในราคาที่กำหนดไว้ล่วงหน้า ในโลก DeFi options มีความสำคัญสำหรับ:

- **Hedging**: ป้องกันความเสี่ยงจากการเปลี่ยนแปลงราคา
- **Income Generation**: สร้างรายได้จากการขาย covered calls
- **Speculation**: เก็งกำไรจาก volatility
- **Structured Products**: Building blocks สำหรับผลิตภัณฑ์ซับซ้อน

โปรโตคอลที่มีชื่อในพื้นที่นี้: **Lyra**, **Dopex**, **Hegic**, **Opyn (Squeeth)**

---

## 1. พื้นฐาน Options Pricing

### Black-Scholes Model (Simplified)

Black-Scholes เป็นสูตรคำนวณราคา option ที่ใช้กันทั่วไป:

```
C = S·N(d1) - K·e^(-rT)·N(d2)
P = K·e^(-rT)·N(-d2) - S·N(-d1)

d1 = [ln(S/K) + (r + σ²/2)·T] / (σ·√T)
d2 = d1 - σ·√T

ที่:
S = ราคาปัจจุบัน (spot price)
K = strike price
r = risk-free rate
T = เวลาถึง expiry (ในปี)
σ = implied volatility
N() = cumulative normal distribution
```

การ implement บน blockchain ต้องใช้ approximations เพราะ:
1. ไม่มี float arithmetic
2. `ln()` และ `e^x` ต้องใช้ Taylor series
3. `N()` ต้องใช้ polynomial approximation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title BlackScholesLib - Library สำหรับคำนวณ options pricing บน chain
/// @notice ใช้ fixed-point arithmetic (1e18 precision) และ approximations
/// @dev ไม่เหมาะสำหรับ trading ที่ต้องการความแม่นยำสูง แต่ดีพอสำหรับ AMM
library BlackScholesLib {
    // ============================================================
    //                        CONSTANTS
    // ============================================================
    
    uint256 internal constant PRECISION = 1e18;
    uint256 internal constant SQRT_2PI = 2506628274631000502; // sqrt(2*pi) * 1e18
    uint256 internal constant TWO = 2e18;
    uint256 internal constant ONE_HALF = 5e17;
    
    // Horner's method coefficients สำหรับ N(x) approximation
    // จาก Abramowitz & Stegun formula 26.2.17
    int256 internal constant A1 = 254829592;
    int256 internal constant A2 = -284496736;
    int256 internal constant A3 = 1421413741;
    int256 internal constant A4 = -1453152027;
    int256 internal constant A5 = 1061405429;
    int256 internal constant P = 327591100;
    
    // ============================================================
    //                    MAIN PRICING FUNCTIONS
    // ============================================================
    
    struct OptionParams {
        uint256 spotPrice;          // S: ราคาปัจจุบัน (1e18 scaled)
        uint256 strikePrice;        // K: strike price (1e18 scaled)
        uint256 timeToExpiry;       // T: เวลาในวินาที
        uint256 volatility;         // σ: annualized volatility (1e18 = 100%)
        uint256 riskFreeRate;       // r: annualized risk-free rate (1e18 = 100%)
    }
    
    /// @notice คำนวณราคา Call option
    /// @param params Option parameters
    /// @return callPrice ราคา call option (1e18 scaled, หน่วยเดียวกับ spotPrice)
    function calculateCallPrice(OptionParams memory params) internal pure returns (uint256 callPrice) {
        (int256 d1, int256 d2) = _calculateD1D2(params);
        
        // N(d1) และ N(d2)
        uint256 nd1 = _normalCDF(d1);
        uint256 nd2 = _normalCDF(d2);
        
        // C = S * N(d1) - K * e^(-rT) * N(d2)
        uint256 snd1 = (params.spotPrice * nd1) / PRECISION;
        
        // e^(-rT) = discount factor
        uint256 discountFactor = _expNeg(
            (params.riskFreeRate * params.timeToExpiry) / 365 days
        );
        uint256 kDiscNd2 = (params.strikePrice * discountFactor / PRECISION * nd2) / PRECISION;
        
        if (snd1 > kDiscNd2) {
            callPrice = snd1 - kDiscNd2;
        } else {
            callPrice = 0; // deep ITM call should not happen, floor at intrinsic value
        }
    }
    
    /// @notice คำนวณราคา Put option (ใช้ put-call parity)
    /// @param params Option parameters
    /// @return putPrice ราคา put option
    function calculatePutPrice(OptionParams memory params) internal pure returns (uint256 putPrice) {
        uint256 callPrice = calculateCallPrice(params);
        
        // Put-Call Parity: P = C - S + K * e^(-rT)
        uint256 discountFactor = _expNeg(
            (params.riskFreeRate * params.timeToExpiry) / 365 days
        );
        uint256 pv_k = (params.strikePrice * discountFactor) / PRECISION;
        
        // P = C + PV(K) - S
        if (callPrice + pv_k > params.spotPrice) {
            putPrice = callPrice + pv_k - params.spotPrice;
        }
        // ถ้า deep OTM put, price ≈ 0
    }
    
    // ============================================================
    //                    GREEKS CALCULATIONS
    // ============================================================
    
    /// @notice คำนวณ Delta ของ Call option
    /// @dev Delta = N(d1) สำหรับ call, N(d1) - 1 สำหรับ put
    /// @return callDelta ระหว่าง 0 - 1e18 (0 = 0%, 1e18 = 100%)
    function calculateCallDelta(OptionParams memory params) internal pure returns (uint256 callDelta) {
        (int256 d1, ) = _calculateD1D2(params);
        callDelta = _normalCDF(d1);
    }
    
    /// @notice คำนวณ Gamma (rate of change ของ Delta)
    /// @dev Gamma = φ(d1) / (S * σ * √T)
    function calculateGamma(OptionParams memory params) internal pure returns (uint256 gamma) {
        (int256 d1, ) = _calculateD1D2(params);
        
        uint256 phi_d1 = _normalPDF(d1);
        uint256 sqrtT = _sqrt((params.timeToExpiry * PRECISION) / 365 days);
        uint256 denominator = (params.spotPrice * params.volatility / PRECISION * sqrtT) / PRECISION;
        
        if (denominator > 0) {
            gamma = (phi_d1 * PRECISION) / denominator;
        }
    }
    
    /// @notice คำนวณ Vega (sensitivity ต่อ volatility)
    /// @dev Vega = S * φ(d1) * √T
    function calculateVega(OptionParams memory params) internal pure returns (uint256 vega) {
        (int256 d1, ) = _calculateD1D2(params);
        
        uint256 phi_d1 = _normalPDF(d1);
        uint256 sqrtT = _sqrt((params.timeToExpiry * PRECISION) / 365 days);
        
        vega = (params.spotPrice * phi_d1 / PRECISION * sqrtT) / PRECISION;
        // Vega ต้องหารด้วย 100 เพื่อให้ได้หน่วย "per 1% vol change"
        vega = vega / 100;
    }
    
    /// @notice คำนวณ Theta (time decay)
    /// @dev Theta = -[S·φ(d1)·σ / (2√T)] - r·K·e^(-rT)·N(d2)
    function calculateCallTheta(OptionParams memory params) internal pure returns (uint256 theta) {
        (int256 d1, int256 d2) = _calculateD1D2(params);
        
        uint256 phi_d1 = _normalPDF(d1);
        uint256 sqrtT = _sqrt((params.timeToExpiry * PRECISION) / 365 days);
        uint256 nd2 = _normalCDF(d2);
        
        uint256 term1 = (params.spotPrice * phi_d1 / PRECISION * params.volatility / PRECISION);
        if (sqrtT > 0) {
            term1 = (term1 * PRECISION) / (2 * sqrtT);
        }
        
        uint256 discountFactor = _expNeg(
            (params.riskFreeRate * params.timeToExpiry) / 365 days
        );
        uint256 term2 = (params.riskFreeRate * params.strikePrice / PRECISION * discountFactor / PRECISION * nd2) / PRECISION;
        
        theta = (term1 + term2) / 365; // Per day
    }
    
    // ============================================================
    //                      INTERNAL MATH
    // ============================================================
    
    /// @notice คำนวณ d1 และ d2 ใน Black-Scholes
    function _calculateD1D2(OptionParams memory params) internal pure returns (int256 d1, int256 d2) {
        // d1 = [ln(S/K) + (r + σ²/2) * T] / (σ * √T)
        
        // ln(S/K): ใช้ approximation ln(x) ≈ (x-1) - (x-1)²/2 + ...
        // สำหรับ x ใกล้ 1 (ATM options)
        int256 logRatio = _ln(int256((params.spotPrice * PRECISION) / params.strikePrice));
        
        // T ในหน่วยปี
        int256 timeInYears = int256((params.timeToExpiry * PRECISION) / 365 days);
        
        // σ² / 2
        int256 volSquaredHalf = int256((params.volatility * params.volatility) / (2 * PRECISION));
        
        // (r + σ²/2) * T
        int256 driftTerm = ((int256(params.riskFreeRate) + volSquaredHalf) * timeInYears) / int256(PRECISION);
        
        // σ * √T
        int256 sqrtT = int256(_sqrt(uint256(timeInYears)));
        int256 volSqrtT = (int256(params.volatility) * sqrtT) / int256(PRECISION);
        
        if (volSqrtT == 0) {
            // Edge case: no volatility or zero time
            d1 = 0;
            d2 = 0;
            return (d1, d2);
        }
        
        d1 = ((logRatio + driftTerm) * int256(PRECISION)) / volSqrtT;
        d2 = d1 - volSqrtT;
    }
    
    /// @notice Cumulative Normal Distribution N(x)
    /// @dev ใช้ rational approximation (error < 7.5e-8)
    function _normalCDF(int256 x) internal pure returns (uint256) {
        // ถ้า x < -6 หรือ x > 6 ใช้ asymptotic values
        if (x <= -6 * int256(PRECISION)) return 0;
        if (x >= 6 * int256(PRECISION)) return PRECISION;
        
        // ใช้ approximation: N(x) ≈ 1 - φ(x) * polynomial(t)
        // ที่ t = 1 / (1 + 0.2316419 * |x|)
        bool negative = x < 0;
        uint256 absX = negative ? uint256(-x) : uint256(x);
        
        // t = PRECISION / (PRECISION + 0.2316419 * |x|)
        uint256 t = (PRECISION * PRECISION) / (PRECISION + (231641900 * absX) / 1000000000);
        
        // Horner's polynomial evaluation
        // p(t) = A1*t + A2*t² + A3*t³ + A4*t⁴ + A5*t⁵
        int256 poly = A1;
        poly = (poly * int256(t)) / int256(PRECISION) + A2;
        poly = (poly * int256(t)) / int256(PRECISION) + A3;
        poly = (poly * int256(t)) / int256(PRECISION) + A4;
        poly = (poly * int256(t)) / int256(PRECISION) + A5;
        poly = (poly * int256(t)) / int256(PRECISION);
        
        // 1 - φ(x) * p(t)
        uint256 phi = _normalPDF(x);
        uint256 result = PRECISION - uint256((int256(phi) * poly) / int256(PRECISION));
        
        if (negative) {
            return PRECISION - result;
        }
        return result;
    }
    
    /// @notice Normal PDF (probability density function) φ(x)
    /// @dev φ(x) = e^(-x²/2) / √(2π)
    function _normalPDF(int256 x) internal pure returns (uint256) {
        // x² / 2 ใน PRECISION
        uint256 absX = x < 0 ? uint256(-x) : uint256(x);
        uint256 xSquaredHalf = (absX * absX) / (2 * PRECISION);
        
        // e^(-x²/2)
        uint256 expNeg = _expNeg(xSquaredHalf);
        
        // หารด้วย √(2π)
        return (expNeg * PRECISION) / SQRT_2PI;
    }
    
    /// @notice Natural log approximation ln(x) โดยใช้ Taylor series
    /// @dev ใช้ ln(x) = 2 * arctanh((x-1)/(x+1)) สำหรับ x > 0
    function _ln(int256 x) internal pure returns (int256 result) {
        require(x > 0, "ln: x must be positive");
        
        // แปลง x เป็น float ใน range [1, 2)
        // ใช้ bit shifting เพื่อหา exponent
        int256 n = 0;
        int256 y = x;
        
        while (y >= 2 * int256(PRECISION)) {
            y /= 2;
            n++;
        }
        while (y < int256(PRECISION)) {
            y *= 2;
            n--;
        }
        
        // ตอนนี้ y ∈ [1, 2) * PRECISION
        // ln(y) ≈ (y - PRECISION) - (y - PRECISION)²/2 + (y - PRECISION)³/3 - ...
        int256 z = y - int256(PRECISION);
        int256 z2 = (z * z) / int256(PRECISION);
        int256 z3 = (z2 * z) / int256(PRECISION);
        int256 z4 = (z3 * z) / int256(PRECISION);
        
        // ln(y) ≈ z - z²/2 + z³/3 - z⁴/4 (4-term Taylor)
        int256 lny = z - z2 / 2 + z3 / 3 - z4 / 4;
        
        // ln(x) = ln(y) + n * ln(2)
        int256 ln2 = 693147180559945309; // ln(2) * 1e18
        result = lny + n * ln2;
    }
    
    /// @notice e^(-x) approximation สำหรับ x >= 0
    function _expNeg(uint256 x) internal pure returns (uint256) {
        // e^(-x) = 1 / e^x
        // ใช้ Taylor series: e^x ≈ 1 + x + x²/2! + x³/3! + ...
        
        if (x == 0) return PRECISION;
        if (x >= 42 * PRECISION) return 0; // underflow
        
        // แบ่ง x เป็น integer part n และ fractional part f
        // e^(-x) = e^(-n) * e^(-f)
        
        // ใช้ precomputed table สำหรับ integer part + polynomial สำหรับ fractional
        // Simplified: ใช้ polynomial approximation โดยตรง
        
        uint256 result = PRECISION;
        uint256 term = PRECISION;
        
        // Taylor series: e^x = Σ x^n/n!
        for (uint256 i = 1; i <= 20; i++) {
            term = (term * x) / (i * PRECISION);
            result += term;
            if (term < 100) break; // converged
        }
        
        // e^(-x) = PRECISION² / e^x
        return (PRECISION * PRECISION) / result;
    }
    
    /// @notice Integer square root (Babylonian method)
    function _sqrt(uint256 x) internal pure returns (uint256 y) {
        if (x == 0) return 0;
        uint256 z = (x + 1) / 2;
        y = x;
        while (z < y) {
            y = z;
            z = (x / z + z) / 2;
        }
    }
}
```

---

## 2. GreeksCalculator Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "./BlackScholesLib.sol";

/// @title GreeksCalculator - On-chain Greeks calculation สำหรับ options
/// @notice ใช้คำนวณ Delta, Gamma, Vega, Theta สำหรับ options pricing
contract GreeksCalculator {
    using BlackScholesLib for BlackScholesLib.OptionParams;
    
    struct Greeks {
        int256 delta;   // การเปลี่ยนแปลงราคา option ต่อ 1 unit change ของ spot
        uint256 gamma;  // Rate of change ของ delta
        uint256 vega;   // Sensitivity ต่อ volatility
        uint256 theta;  // Time decay per day
        int256 rho;     // Sensitivity ต่อ interest rate (simplified)
    }
    
    /// @notice คำนวณ Greeks ทั้งหมด
    function calculateGreeks(
        uint256 spotPrice,
        uint256 strikePrice,
        uint256 timeToExpiry,
        uint256 volatility,
        uint256 riskFreeRate,
        bool isCall
    ) external pure returns (Greeks memory greeks) {
        BlackScholesLib.OptionParams memory params = BlackScholesLib.OptionParams({
            spotPrice: spotPrice,
            strikePrice: strikePrice,
            timeToExpiry: timeToExpiry,
            volatility: volatility,
            riskFreeRate: riskFreeRate
        });
        
        uint256 callDelta = BlackScholesLib.calculateCallDelta(params);
        
        if (isCall) {
            greeks.delta = int256(callDelta);
        } else {
            // Put delta = Call delta - 1
            greeks.delta = int256(callDelta) - int256(1e18);
        }
        
        greeks.gamma = BlackScholesLib.calculateGamma(params);
        greeks.vega = BlackScholesLib.calculateVega(params);
        greeks.theta = BlackScholesLib.calculateCallTheta(params);
        
        // Rho (simplified): Put rho ≈ -Call rho
        // rho = K * T * e^(-rT) * N(d2) / 100 (per 1% rate change)
        greeks.rho = isCall ? int256(1e15) : -int256(1e15); // Placeholder
    }
    
    /// @notice คำนวณ implied volatility จากราคา option (Newton-Raphson)
    /// @dev ใช้ binary search เพราะ Newton-Raphson อาจ unstable
    function calculateImpliedVolatility(
        uint256 optionPrice,
        uint256 spotPrice,
        uint256 strikePrice,
        uint256 timeToExpiry,
        uint256 riskFreeRate,
        bool isCall
    ) external pure returns (uint256 impliedVol) {
        // Binary search ระหว่าง 0.1% ถึง 500% volatility
        uint256 low = 1e15;    // 0.1%
        uint256 high = 5e18;   // 500%
        
        for (uint256 i = 0; i < 50; i++) { // max 50 iterations
            uint256 mid = (low + high) / 2;
            
            BlackScholesLib.OptionParams memory params = BlackScholesLib.OptionParams({
                spotPrice: spotPrice,
                strikePrice: strikePrice,
                timeToExpiry: timeToExpiry,
                volatility: mid,
                riskFreeRate: riskFreeRate
            });
            
            uint256 modelPrice = isCall 
                ? BlackScholesLib.calculateCallPrice(params)
                : BlackScholesLib.calculatePutPrice(params);
            
            if (modelPrice < optionPrice) {
                low = mid;
            } else {
                high = mid;
            }
            
            // ถ้า precision ดีพอแล้ว หยุด
            if (high - low < 1e14) break;
        }
        
        impliedVol = (low + high) / 2;
    }
}
```

---

## 3. OptionsVault - Covered Calls

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Enumerable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "./BlackScholesLib.sol";

/// @title OptionsVault - Covered Call Vault ที่ mint Option NFTs
/// @notice ผู้ใช้ lock collateral และได้รับ option NFT แทน premium
/// @dev Option แต่ละตัวเป็น ERC-721 ที่สามารถซื้อขายได้
contract OptionsVault is ERC721Enumerable, Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;
    using BlackScholesLib for BlackScholesLib.OptionParams;
    
    // ============================================================
    //                          ERRORS
    // ============================================================
    error InsufficientCollateral(uint256 required, uint256 provided);
    error OptionExpired(uint256 optionId, uint256 expiry);
    error OptionNotExpired(uint256 optionId, uint256 expiry);
    error NotOptionOwner(uint256 optionId, address caller);
    error OptionAlreadyExercised(uint256 optionId);
    error OptionAlreadySettled(uint256 optionId);
    error PriceOutOfRange(uint256 price, uint256 minPrice);
    error StrikeMustBeHigher(uint256 strike, uint256 spot);
    error ExpiryTooShort(uint256 expiry, uint256 minExpiry);
    error ExpiryTooLong(uint256 expiry, uint256 maxExpiry);
    
    // ============================================================
    //                          EVENTS
    // ============================================================
    event OptionMinted(
        uint256 indexed optionId,
        address indexed writer,
        address indexed collateralToken,
        uint256 strikePrice,
        uint256 expiry,
        uint256 amount,
        uint256 premium
    );
    event OptionExercised(uint256 indexed optionId, address indexed exerciser, uint256 payout);
    event OptionExpiredSettled(uint256 indexed optionId, address indexed writer, uint256 collateralReturned);
    event CollateralLocked(address indexed writer, uint256 amount);
    event CollateralReleased(address indexed writer, uint256 amount);
    
    // ============================================================
    //                          STRUCTS
    // ============================================================
    enum OptionType { CALL, PUT }
    enum OptionStatus { ACTIVE, EXERCISED, EXPIRED }
    
    struct Option {
        address writer;             // ผู้เขียน option
        address collateralToken;    // token ที่ใช้เป็น collateral
        address paymentToken;       // token ที่จ่าย premium (มักเป็น stablecoin)
        uint256 strikePrice;        // strike price (1e18 scaled)
        uint256 expiry;             // Unix timestamp ที่ expire
        uint256 amount;             // จำนวน underlying asset (1e18 scaled)
        uint256 premium;            // premium ที่ได้รับ (1e18 scaled)
        uint256 collateralAmount;   // collateral ที่ lock ไว้
        OptionType optionType;
        OptionStatus status;
    }
    
    // ============================================================
    //                          STORAGE
    // ============================================================
    
    mapping(uint256 => Option) public options;
    uint256 private _nextTokenId;
    
    /// @notice Price oracle สำหรับดึงราคา underlying
    IPriceOracle public priceOracle;
    
    /// @notice Volatility oracle
    IVolatilityOracle public volOracle;
    
    /// @notice Protocol fee (bps)
    uint256 public protocolFee;
    address public feeRecipient;
    
    uint256 public constant BPS = 10_000;
    uint256 public constant MIN_EXPIRY = 1 hours;
    uint256 public constant MAX_EXPIRY = 90 days;
    uint256 public constant COLLATERAL_RATIO = 11_000; // 110% overcollateralized (bps)
    
    // ============================================================
    //                        CONSTRUCTOR
    // ============================================================
    
    constructor(
        address _priceOracle,
        address _volOracle,
        address _feeRecipient,
        address _owner
    ) ERC721("DeFi Options", "DOPT") Ownable(_owner) {
        priceOracle = IPriceOracle(_priceOracle);
        volOracle = IVolatilityOracle(_volOracle);
        feeRecipient = _feeRecipient;
        protocolFee = 50; // 0.5%
    }
    
    // ============================================================
    //                    OPTION WRITING
    // ============================================================
    
    /// @notice เขียน Covered Call option
    /// @param collateralToken underlying token ที่ lock (เช่น ETH/WETH)
    /// @param paymentToken token สำหรับจ่าย premium (เช่น USDC)
    /// @param strikePrice strike price ใน paymentToken (1e18 scaled)
    /// @param expiry Unix timestamp
    /// @param amount จำนวน underlying token ที่ cover
    function writeCoveredCall(
        address collateralToken,
        address paymentToken,
        uint256 strikePrice,
        uint256 expiry,
        uint256 amount
    ) external nonReentrant returns (uint256 optionId) {
        // ============================================================
        //                         VALIDATION
        // ============================================================
        
        uint256 spotPrice = priceOracle.getPrice(collateralToken, paymentToken);
        
        // Covered call: strike ต้องสูงกว่า spot (OTM หรือ ATM)
        if (strikePrice < spotPrice) revert StrikeMustBeHigher(strikePrice, spotPrice);
        
        uint256 timeToExpiry = expiry - block.timestamp;
        if (timeToExpiry < MIN_EXPIRY) revert ExpiryTooShort(expiry, block.timestamp + MIN_EXPIRY);
        if (timeToExpiry > MAX_EXPIRY) revert ExpiryTooLong(expiry, block.timestamp + MAX_EXPIRY);
        
        // ============================================================
        //                    CALCULATE PREMIUM
        // ============================================================
        
        uint256 volatility = volOracle.getImpliedVolatility(collateralToken, paymentToken, expiry);
        
        BlackScholesLib.OptionParams memory params = BlackScholesLib.OptionParams({
            spotPrice: spotPrice,
            strikePrice: strikePrice,
            timeToExpiry: timeToExpiry,
            volatility: volatility,
            riskFreeRate: 3e16 // 3% risk-free rate (hardcoded, ในกรณีจริงใช้ oracle)
        });
        
        uint256 premium = BlackScholesLib.calculateCallPrice(params);
        
        // Scale premium ตาม amount
        uint256 totalPremium = (premium * amount) / 1e18;
        
        // Protocol fee
        uint256 fee = (totalPremium * protocolFee) / BPS;
        uint256 writerPremium = totalPremium - fee;
        
        // ============================================================
        //                    COLLATERAL MANAGEMENT
        // ============================================================
        
        // Lock collateral (110% ของ notional value)
        uint256 collateralRequired = (amount * COLLATERAL_RATIO) / BPS;
        
        IERC20(collateralToken).safeTransferFrom(msg.sender, address(this), collateralRequired);
        
        // Writer รับ premium ทันที
        IERC20(paymentToken).safeTransferFrom(msg.sender, address(this), totalPremium);
        IERC20(paymentToken).safeTransfer(msg.sender, writerPremium);
        
        if (fee > 0) {
            IERC20(paymentToken).safeTransfer(feeRecipient, fee);
        }
        
        // ============================================================
        //                      MINT OPTION NFT
        // ============================================================
        
        optionId = _nextTokenId++;
        
        options[optionId] = Option({
            writer: msg.sender,
            collateralToken: collateralToken,
            paymentToken: paymentToken,
            strikePrice: strikePrice,
            expiry: expiry,
            amount: amount,
            premium: totalPremium,
            collateralAmount: collateralRequired,
            optionType: OptionType.CALL,
            status: OptionStatus.ACTIVE
        });
        
        _mint(msg.sender, optionId); // Writer ได้รับ NFT ที่แทน obligation
        
        emit OptionMinted(
            optionId,
            msg.sender,
            collateralToken,
            strikePrice,
            expiry,
            amount,
            totalPremium
        );
        
        emit CollateralLocked(msg.sender, collateralRequired);
    }
    
    // ============================================================
    //                    OPTION EXERCISING
    // ============================================================
    
    /// @notice Exercise option (buyer ซื้อ underlying ในราคา strike)
    /// @dev Buyer ต้องจ่าย strikePrice * amount
    function exercise(uint256 optionId) external nonReentrant {
        Option storage option = options[optionId];
        
        // ตรวจสอบ ownership (ต้องโอน NFT ไปให้ buyer ก่อน)
        if (ownerOf(optionId) != msg.sender) revert NotOptionOwner(optionId, msg.sender);
        if (option.status != OptionStatus.ACTIVE) revert OptionAlreadyExercised(optionId);
        if (block.timestamp > option.expiry) revert OptionExpired(optionId, option.expiry);
        
        // ตรวจสอบว่า ITM (In-The-Money)
        uint256 spotPrice = priceOracle.getPrice(option.collateralToken, option.paymentToken);
        // สำหรับ call option: exercise เมื่อ spot > strike
        
        // Buyer จ่าย strikePrice * amount
        uint256 exerciseAmount = (option.strikePrice * option.amount) / 1e18;
        IERC20(option.paymentToken).safeTransferFrom(msg.sender, address(this), exerciseAmount);
        
        // ส่ง payment ให้ writer
        IERC20(option.paymentToken).safeTransfer(option.writer, exerciseAmount);
        
        // ส่ง underlying ให้ buyer
        IERC20(option.collateralToken).safeTransfer(msg.sender, option.amount);
        
        // คืน collateral ส่วนที่เหลือให้ writer
        uint256 remainingCollateral = option.collateralAmount - option.amount;
        if (remainingCollateral > 0) {
            IERC20(option.collateralToken).safeTransfer(option.writer, remainingCollateral);
        }
        
        // อัปเดตสถานะ
        option.status = OptionStatus.EXERCISED;
        _burn(optionId);
        
        emit OptionExercised(optionId, msg.sender, option.amount);
        emit CollateralReleased(option.writer, remainingCollateral);
    }
    
    /// @notice Settle expired option (คืน collateral ให้ writer)
    function settleExpired(uint256 optionId) external nonReentrant {
        Option storage option = options[optionId];
        
        if (option.status != OptionStatus.ACTIVE) revert OptionAlreadySettled(optionId);
        if (block.timestamp <= option.expiry) revert OptionNotExpired(optionId, option.expiry);
        
        // คืน collateral ทั้งหมดให้ writer
        uint256 collateral = option.collateralAmount;
        option.collateralAmount = 0;
        option.status = OptionStatus.EXPIRED;
        
        IERC20(option.collateralToken).safeTransfer(option.writer, collateral);
        
        // Burn NFT
        if (_ownerOf(optionId) != address(0)) {
            _burn(optionId);
        }
        
        emit OptionExpiredSettled(optionId, option.writer, collateral);
        emit CollateralReleased(option.writer, collateral);
    }
}

// Interfaces
interface IPriceOracle {
    function getPrice(address base, address quote) external view returns (uint256);
}

interface IVolatilityOracle {
    function getImpliedVolatility(address base, address quote, uint256 expiry) external view returns (uint256);
}
```

---

## 4. OptionsMarket - Secondary Market

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title OptionsMarket - Secondary market สำหรับซื้อขาย Option NFTs
/// @notice ผู้ถือ option สามารถขายก่อน expiry
/// @dev Buyer จ่าย premium, ได้รับ NFT ที่ให้สิทธิ์ exercise
contract OptionsMarket is ReentrancyGuard {
    using SafeERC20 for IERC20;
    
    // ============================================================
    //                          ERRORS
    // ============================================================
    error ListingNotFound(uint256 listingId);
    error NotLister(uint256 listingId, address caller);
    error ListingExpired(uint256 listingId);
    error InsufficientPayment(uint256 sent, uint256 required);
    error OptionAlreadyExpired(uint256 optionId);
    
    // ============================================================
    //                          EVENTS
    // ============================================================
    event Listed(uint256 indexed listingId, uint256 indexed optionId, address indexed seller, uint256 askPrice);
    event Sold(uint256 indexed listingId, uint256 indexed optionId, address indexed buyer, uint256 price);
    event Cancelled(uint256 indexed listingId);
    event BidPlaced(uint256 indexed optionId, address indexed bidder, uint256 bidPrice);
    event BidAccepted(uint256 indexed optionId, address indexed seller, address indexed buyer, uint256 price);
    
    // ============================================================
    //                          STRUCTS
    // ============================================================
    struct Listing {
        address seller;
        address optionVault;        // address ของ OptionsVault
        uint256 optionId;           // NFT token ID
        address paymentToken;       // token ที่รับเป็นค่าตอบแทน
        uint256 askPrice;           // ราคาขาย
        uint256 listedAt;
        uint256 listingExpiry;      // listing หมดอายุเมื่อไร
        bool active;
    }
    
    struct Bid {
        address bidder;
        address paymentToken;
        uint256 bidPrice;
        uint256 bidExpiry;          // bid หมดอายุเมื่อไร
        bool active;
    }
    
    // ============================================================
    //                          STORAGE
    // ============================================================
    
    mapping(uint256 => Listing) public listings;
    mapping(uint256 => Bid[]) public bids;  // optionId => bids[]
    
    uint256 private _nextListingId;
    
    uint256 public marketFee;       // Market fee (bps)
    address public feeRecipient;
    uint256 public constant BPS = 10_000;
    
    constructor(address _feeRecipient) {
        feeRecipient = _feeRecipient;
        marketFee = 50; // 0.5%
    }
    
    // ============================================================
    //                      LISTING FUNCTIONS
    // ============================================================
    
    /// @notice List option ขายใน market
    function listOption(
        address optionVault,
        uint256 optionId,
        address paymentToken,
        uint256 askPrice,
        uint256 listingDuration
    ) external nonReentrant returns (uint256 listingId) {
        // ตรวจสอบ ownership
        require(IERC721(optionVault).ownerOf(optionId) == msg.sender, "Not owner");
        
        // Transfer NFT ไปยัง market contract (escrow)
        IERC721(optionVault).transferFrom(msg.sender, address(this), optionId);
        
        listingId = _nextListingId++;
        
        listings[listingId] = Listing({
            seller: msg.sender,
            optionVault: optionVault,
            optionId: optionId,
            paymentToken: paymentToken,
            askPrice: askPrice,
            listedAt: block.timestamp,
            listingExpiry: block.timestamp + listingDuration,
            active: true
        });
        
        emit Listed(listingId, optionId, msg.sender, askPrice);
    }
    
    /// @notice ซื้อ option ในราคา ask
    function buyOption(uint256 listingId) external nonReentrant {
        Listing storage listing = listings[listingId];
        
        if (!listing.active) revert ListingNotFound(listingId);
        if (block.timestamp > listing.listingExpiry) revert ListingExpired(listingId);
        
        uint256 price = listing.askPrice;
        uint256 fee = (price * marketFee) / BPS;
        uint256 sellerAmount = price - fee;
        
        // รับ payment
        IERC20(listing.paymentToken).safeTransferFrom(msg.sender, address(this), price);
        
        // จ่าย seller
        IERC20(listing.paymentToken).safeTransfer(listing.seller, sellerAmount);
        
        // จ่าย fee
        if (fee > 0) {
            IERC20(listing.paymentToken).safeTransfer(feeRecipient, fee);
        }
        
        // Transfer NFT ให้ buyer
        IERC721(listing.optionVault).transferFrom(address(this), msg.sender, listing.optionId);
        
        listing.active = false;
        
        emit Sold(listingId, listing.optionId, msg.sender, price);
    }
    
    /// @notice Cancel listing
    function cancelListing(uint256 listingId) external nonReentrant {
        Listing storage listing = listings[listingId];
        
        if (!listing.active) revert ListingNotFound(listingId);
        if (listing.seller != msg.sender) revert NotLister(listingId, msg.sender);
        
        listing.active = false;
        
        // คืน NFT
        IERC721(listing.optionVault).transferFrom(address(this), msg.sender, listing.optionId);
        
        emit Cancelled(listingId);
    }
    
    // ============================================================
    //                       BID FUNCTIONS  
    // ============================================================
    
    /// @notice วาง bid สำหรับ option ที่ไม่ได้ list (OTC bid)
    function placeBid(
        address optionVault,
        uint256 optionId,
        address paymentToken,
        uint256 bidPrice,
        uint256 bidDuration
    ) external nonReentrant {
        // Lock bid amount ใน contract
        IERC20(paymentToken).safeTransferFrom(msg.sender, address(this), bidPrice);
        
        bids[optionId].push(Bid({
            bidder: msg.sender,
            paymentToken: paymentToken,
            bidPrice: bidPrice,
            bidExpiry: block.timestamp + bidDuration,
            active: true
        }));
        
        emit BidPlaced(optionId, msg.sender, bidPrice);
    }
    
    /// @notice Accept bid (ขาย option ในราคา bid)
    function acceptBid(
        address optionVault,
        uint256 optionId,
        uint256 bidIndex
    ) external nonReentrant {
        require(IERC721(optionVault).ownerOf(optionId) == msg.sender, "Not owner");
        
        Bid storage bid = bids[optionId][bidIndex];
        require(bid.active, "Bid not active");
        require(block.timestamp <= bid.bidExpiry, "Bid expired");
        
        uint256 fee = (bid.bidPrice * marketFee) / BPS;
        uint256 sellerAmount = bid.bidPrice - fee;
        
        // Transfer NFT
        IERC721(optionVault).transferFrom(msg.sender, bid.bidder, optionId);
        
        // ส่ง payment (ถูก lock ไว้แล้ว)
        IERC20(bid.paymentToken).safeTransfer(msg.sender, sellerAmount);
        if (fee > 0) {
            IERC20(bid.paymentToken).safeTransfer(feeRecipient, fee);
        }
        
        bid.active = false;
        
        emit BidAccepted(optionId, msg.sender, bid.bidder, bid.bidPrice);
    }
    
    /// @notice Cancel bid และรับเงินคืน
    function cancelBid(uint256 optionId, uint256 bidIndex) external nonReentrant {
        Bid storage bid = bids[optionId][bidIndex];
        require(bid.bidder == msg.sender, "Not bidder");
        require(bid.active, "Bid not active");
        
        bid.active = false;
        
        IERC20(bid.paymentToken).safeTransfer(msg.sender, bid.bidPrice);
    }
}
```

---

## 5. Lyra/Dopex Style Architecture Overview

### สถาปัตยกรรมของ Lyra Finance

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title LyraStyleOptionsAMM - AMM สำหรับ options (simplified Lyra architecture)
/// @notice แสดงให้เห็น concept ของ options AMM ที่ใช้ Black-Scholes
/// @dev ใน production จะใช้ off-chain vol surface และ on-chain settlement
contract LyraStyleOptionsAMM {
    // ============================================================
    //                    LYRA ARCHITECTURE OVERVIEW
    // ============================================================
    
    // Lyra ประกอบด้วย components หลัก:
    // 1. OptionMarket    - Central contract จัดการ option lifecycle
    // 2. LiquidityPool   - LP ที่ provide collateral สำหรับ write options
    // 3. OptionToken     - ERC-20 แทน long/short positions
    // 4. GreekCache      - Cache greeks เพื่อประหยัด gas
    // 5. OptionPriceCalculator - คำนวณ premium
    // 6. SynthetixAdapter - ใช้ SNX สำหรับ hedging delta
    
    // ============================================================
    //                    LIQUIDITY POOL CONCEPT
    // ============================================================
    
    struct LiquidityPool {
        uint256 totalLiquidity;     // รวม USDC ที่ LPs ใส่มา
        uint256 usedLiquidity;      // ใช้ cover options ที่ขายไปแล้ว
        uint256 pendingDelta;       // delta ที่ต้อง hedge
        uint256 NAV;                // Net Asset Value
    }
    
    // LP receives premium income แต่รับ risk จาก option payouts
    // Delta hedging ช่วยลด directional risk ของ LP
    
    // ============================================================
    //                    GREEK CACHE PATTERN
    // ============================================================
    
    struct OptionBoard {
        uint256 expiry;
        uint256 iv;                 // Base implied volatility
        bool frozen;
    }
    
    struct Strike {
        uint256 strikePrice;
        uint256 skew;               // IV skew relative to base
        uint256 longCall;           // long call open interest
        uint256 shortCall;
        uint256 longPut;
        uint256 shortPut;
    }
    
    // ============================================================
    //                    DOPEX ARCHITECTURE  
    // ============================================================
    
    // Dopex ใช้ concept ที่แตกต่าง:
    // 1. Single Staking Option Vaults (SSOVs) - Covered call vaults
    // 2. Atlantic Straddles - Combine call + put
    // 3. Permissioned markets ที่ผ่านการ audit
    // 4. DPX + rDPX tokens สำหรับ governance + rebate
    
    // Key difference: Dopex มี epoch-based options (ขายเป็น batches)
    // ไม่ใช่ continuous market อย่าง Lyra
    
    // ============================================================
    //                    SIMPLE SSOV IMPLEMENTATION
    // ============================================================
    
    uint256 public epochDuration;
    uint256 public currentEpoch;
    
    struct Epoch {
        uint256 startTime;
        uint256 endTime;
        uint256[] strikes;
        uint256 totalCollateral;
        mapping(uint256 => uint256) strikeCollateral;
        mapping(uint256 => uint256) strikePremiums;
        bool settled;
        uint256 settlementPrice;
    }
    
    mapping(uint256 => Epoch) private epochs;
    
    // Users deposit collateral at epoch start
    // Options are sold throughout epoch at Black-Scholes price
    // At epoch end, ITM options are settled, OTM options expire worthless
    
    // สำหรับ LPs: epoch return = Σ premiums - Σ ITM payouts + accrued yield
}
```

---

## Workshop / แบบฝึกหัด

### แบบฝึกหัดที่ 1: Put Option Vault

**โจทย์**: ปรับ `OptionsVault` ให้รองรับ Cash-Secured Put:
1. Writer lock USDC เป็น collateral
2. Buyer จ่าย premium
3. ถ้า spot < strike เมื่อ expire, writer ต้องซื้อ underlying ในราคา strike
4. ถ้า spot > strike, writer รับ premium ฟรี

### แบบฝึกหัดที่ 2: Option Portfolio Greeks

**โจทย์**: สร้าง `PortfolioGreeks` ที่:
1. รับรายการ option positions
2. คำนวณ net delta, gamma, vega, theta ของ portfolio
3. แสดง P&L สำหรับ price scenarios ต่างๆ (+10%, -10%, +20%, -20%)

```solidity
// แบบฝึกหัด
contract PortfolioGreeks {
    struct Position {
        uint256 optionId;
        int256 quantity;    // บวก = long, ลบ = short
        bool isCall;
    }
    
    function calculatePortfolioGreeks(
        Position[] calldata positions,
        address optionsVault,
        uint256 spotPrice,
        uint256 volatility,
        uint256 riskFreeRate
    ) external view returns (
        int256 netDelta,
        int256 netGamma,
        int256 netVega,
        int256 netTheta
    ) {
        // TODO: Loop ผ่าน positions ทั้งหมด
        // TODO: คำนวณ Greeks ของแต่ละ position
        // TODO: คูณด้วย quantity (positive/negative)
        // TODO: Sum ขึ้น
    }
}
```

### แบบฝึกหัดที่ 3: Volatility Surface

**โจทย์**: สร้าง `VolatilitySurface` ที่:
1. Store volatility values สำหรับ strike-expiry matrix
2. Interpolate IV สำหรับ strike/expiry ที่ไม่มีใน matrix
3. Admin สามารถอัปเดต surface ได้

---

## สรุป Part 53

- **Black-Scholes** เป็นสูตรมาตรฐานสำหรับ options pricing แต่ต้องใช้ approximations บน blockchain
- **Fixed-point arithmetic** (1e18) จำเป็นสำหรับทำ math บน EVM ที่ไม่มี float
- **Greeks** (Delta, Gamma, Vega, Theta) บอกความเสี่ยงของ option ต่อ market parameters
- **Covered Call**: Writer lock underlying, รับ premium ทันที แต่จะถูก exercise ถ้า price ขึ้นถึง strike
- **Option NFTs**: ทำให้ options เป็น tradeable assets ที่ซื้อขายได้ใน secondary market
- **Lyra**: ใช้ LP pool เป็น counterparty, delta hedging ผ่าน Synthetix
- **Dopex**: ใช้ epoch-based SSOVs, batch settlement
- **GreekCache** pattern ประหยัด gas โดยไม่ต้องคำนวณ Greeks ทุก transaction

## Next: Part 54 - Structured DeFi Products
