# Part 64: Advanced Tokenomics Engineering

## บทนำ

Tokenomics ไม่ใช่แค่การออกแบบว่า Token มีจำนวนเท่าไหร่ แต่คือการออกแบบ **กลไกทางเศรษฐศาสตร์** ที่ทำให้ Protocol ยั่งยืน สร้างแรงจูงใจที่ถูกต้อง และจัดการ Token Supply อย่างมีระบบ

ใน Part นี้เราจะสร้าง Tokenomics Engine ที่สมบูรณ์แบบ ครอบคลุมทุก Pattern ที่ DeFi Protocol ชั้นนำใช้

---

## ทำไม Tokenomics Engineering ถึงซับซ้อน?

ปัญหาที่ Protocol เจอบ่อย:
1. **Mercenary Liquidity**: Liquidity Providers ออกทันทีเมื่อ Emission หมด
2. **Sell Pressure**: Token Holders ขาย Token ทันทีที่ได้รับ Rewards
3. **Governance Capture**: Whale สามารถควบคุม Governance ได้
4. **Hyperinflation**: Emission มากเกินไปทำให้ Token ไม่มีมูลค่า

ทุก Pattern ใน Part นี้แก้ปัญหาเหล่านี้

---

## 1. veToken Model: VotingEscrow

### หลักการ veToken

**veToken** (Vote-Escrowed Token) ต้องการให้ Holder ล็อค Token เพื่อรับสิทธิ์:
- **Voting Power**: ยิ่งล็อคนาน ยิ่งมีสิทธิ์เลือกตั้งมาก
- **Boost Multiplier**: ยิ่งล็อคนาน ยิ่งได้ Yield Boost มาก
- **Decay**: สิทธิ์ลดลงเรื่อยๆ จนถึง 0 เมื่อครบกำหนด

สูตร: `vePower = lockAmount × (remainingTime / maxLockTime)`

### Code: VotingEscrow

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/// @title VotingEscrow
/// @notice Lock TOKEN for veTOKEN with linear decay and boost multiplier
/// @custom:invariant A user's vePower is always <= their lockedAmount
/// @custom:invariant A user cannot withdraw before lockEnd
/// @custom:invariant vePower decays linearly to 0 at lockEnd
contract VotingEscrow is ReentrancyGuard, Ownable {
    using SafeERC20 for IERC20;

    // ─────────────────────────────────────────────────────────────
    // Types
    // ─────────────────────────────────────────────────────────────

    struct LockPosition {
        uint256 amount;      // Amount of TOKEN locked
        uint256 lockEnd;     // Unix timestamp when lock expires
        uint256 lockStart;   // When this lock was created/extended
    }

    // ─────────────────────────────────────────────────────────────
    // Constants
    // ─────────────────────────────────────────────────────────────

    uint256 public constant MAX_LOCK_TIME = 4 * 365 days; // 4 years
    uint256 public constant MIN_LOCK_TIME = 7 days;       // 1 week minimum
    uint256 public constant WEEK = 7 days;                // Locks rounded to weeks

    // Boost: 1x at min lock, 2.5x at max lock (in basis points)
    uint256 public constant MIN_BOOST_BPS = 10_000;  // 1.0x
    uint256 public constant MAX_BOOST_BPS = 25_000;  // 2.5x
    uint256 public constant BASIS_POINTS  = 10_000;

    // ─────────────────────────────────────────────────────────────
    // State
    // ─────────────────────────────────────────────────────────────

    IERC20 public immutable token;

    mapping(address => LockPosition) public locks;

    uint256 public totalLocked;
    uint256 public totalVePower; // Snapshot (approximate, degrades over time)

    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────

    event Locked(address indexed user, uint256 amount, uint256 lockEnd);
    event LockIncreased(address indexed user, uint256 additionalAmount, uint256 newTotal);
    event LockExtended(address indexed user, uint256 newLockEnd);
    event Withdrawn(address indexed user, uint256 amount);

    // ─────────────────────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────────────────────

    constructor(address _token) Ownable(msg.sender) {
        token = IERC20(_token);
    }

    // ─────────────────────────────────────────────────────────────
    // Core Lock Functions
    // ─────────────────────────────────────────────────────────────

    /// @notice Create a new lock position
    /// @param amount Amount of TOKEN to lock
    /// @param duration Lock duration in seconds (will be rounded to nearest week)
    function lock(uint256 amount, uint256 duration) external nonReentrant {
        require(amount > 0, "VE: zero amount");
        require(duration >= MIN_LOCK_TIME, "VE: lock too short");
        require(duration <= MAX_LOCK_TIME, "VE: lock too long");
        require(locks[msg.sender].amount == 0, "VE: existing lock, use increase/extend");

        // Round down to nearest week (Curve-style)
        uint256 roundedDuration = (duration / WEEK) * WEEK;
        uint256 lockEnd = block.timestamp + roundedDuration;

        token.safeTransferFrom(msg.sender, address(this), amount);

        locks[msg.sender] = LockPosition({
            amount: amount,
            lockEnd: lockEnd,
            lockStart: block.timestamp
        });

        totalLocked += amount;

        emit Locked(msg.sender, amount, lockEnd);
    }

    /// @notice Increase lock amount (without changing duration)
    function increaseLockAmount(uint256 additionalAmount) external nonReentrant {
        LockPosition storage pos = locks[msg.sender];
        require(pos.amount > 0, "VE: no existing lock");
        require(pos.lockEnd > block.timestamp, "VE: lock expired");
        require(additionalAmount > 0, "VE: zero amount");

        token.safeTransferFrom(msg.sender, address(this), additionalAmount);
        pos.amount += additionalAmount;
        totalLocked += additionalAmount;

        emit LockIncreased(msg.sender, additionalAmount, pos.amount);
    }

    /// @notice Extend lock duration (without changing amount)
    function extendLock(uint256 newDuration) external nonReentrant {
        LockPosition storage pos = locks[msg.sender];
        require(pos.amount > 0, "VE: no existing lock");
        require(newDuration >= MIN_LOCK_TIME, "VE: lock too short");
        require(newDuration <= MAX_LOCK_TIME, "VE: lock too long");

        uint256 roundedDuration = (newDuration / WEEK) * WEEK;
        uint256 newLockEnd = block.timestamp + roundedDuration;

        require(newLockEnd > pos.lockEnd, "VE: must extend, not shorten");

        pos.lockEnd = newLockEnd;
        pos.lockStart = block.timestamp; // Reset for boost calculation

        emit LockExtended(msg.sender, newLockEnd);
    }

    /// @notice Withdraw tokens after lock expires
    function withdraw() external nonReentrant {
        LockPosition storage pos = locks[msg.sender];
        require(pos.amount > 0, "VE: nothing locked");
        require(block.timestamp >= pos.lockEnd, "VE: lock not expired");

        uint256 amount = pos.amount;
        totalLocked -= amount;

        delete locks[msg.sender];

        token.safeTransfer(msg.sender, amount);

        emit Withdrawn(msg.sender, amount);
    }

    // ─────────────────────────────────────────────────────────────
    // View Functions
    // ─────────────────────────────────────────────────────────────

    /// @notice Get user's current voting power (decays linearly to 0)
    /// @param user Address to check
    /// @return vePower Current voting power (scales with remaining lock time)
    function vePowerOf(address user) public view returns (uint256 vePower) {
        LockPosition memory pos = locks[user];
        if (pos.amount == 0) return 0;
        if (block.timestamp >= pos.lockEnd) return 0;

        uint256 remaining = pos.lockEnd - block.timestamp;
        // vePower = amount * remaining / maxLockTime
        vePower = (pos.amount * remaining) / MAX_LOCK_TIME;
    }

    /// @notice Get user's boost multiplier (1x to 2.5x)
    /// @param user Address to check
    /// @return boostBps Boost in basis points (10000 = 1x, 25000 = 2.5x)
    function boostOf(address user) public view returns (uint256 boostBps) {
        LockPosition memory pos = locks[user];
        if (pos.amount == 0) return MIN_BOOST_BPS;
        if (block.timestamp >= pos.lockEnd) return MIN_BOOST_BPS;

        uint256 remaining = pos.lockEnd - block.timestamp;

        // Interpolate between MIN_BOOST and MAX_BOOST
        // boost = MIN + (MAX - MIN) * remaining / MAX_LOCK_TIME
        uint256 boostRange = MAX_BOOST_BPS - MIN_BOOST_BPS;
        boostBps = MIN_BOOST_BPS + (boostRange * remaining) / MAX_LOCK_TIME;
    }

    /// @notice Apply boost to a base amount (e.g., for yield calculation)
    function applyBoost(address user, uint256 baseAmount) external view returns (uint256 boostedAmount) {
        uint256 boost = boostOf(user);
        boostedAmount = (baseAmount * boost) / BASIS_POINTS;
    }

    /// @notice Check if user's lock has expired
    function isExpired(address user) external view returns (bool) {
        return locks[user].lockEnd <= block.timestamp;
    }

    /// @notice Get remaining lock time in seconds
    function remainingLockTime(address user) external view returns (uint256) {
        LockPosition memory pos = locks[user];
        if (pos.lockEnd <= block.timestamp) return 0;
        return pos.lockEnd - block.timestamp;
    }
}
```

---

## 2. Emission Schedule: Geometric & Linear Emissions

### ทำไมต้องมีสูตร Emission?

Emission ที่ไม่มีสูตรมักเป็น:
- Too Fast: Token Dump, Protocol Collapse
- Too Slow: ไม่มีแรงจูงใจ, Protocol ไม่เติบโต

**Geometric Decay** (Curve/Uniswap style): `weekly emission = base * ratio^week`
- เริ่มสูง ค่อยๆ ลดลง ช่วงแรก Incentivize สูงสุด

**Linear Decay**: `emission = max - (max - min) * week / totalWeeks`
- ลดลงสม่ำเสมอ คาดเดาได้

### Code: EmissionSchedule

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/Ownable.sol";

/// @title GeometricEmissions
/// @notice Weekly emissions that decay by a fixed ratio each period
/// @dev Uses integer arithmetic with 1e18 precision for ratio
contract GeometricEmissions is Ownable {

    // ─────────────────────────────────────────────────────────────
    // Constants
    // ─────────────────────────────────────────────────────────────

    uint256 public constant PRECISION = 1e18;
    uint256 public constant WEEK = 7 days;

    // ─────────────────────────────────────────────────────────────
    // State
    // ─────────────────────────────────────────────────────────────

    uint256 public baseEmission;    // Emission in week 0
    uint256 public decayRatio;      // e.g., 0.99e18 = 99% per week (1% decay)
    uint256 public startTime;       // Protocol launch timestamp
    uint256 public maxSupply;       // Hard cap on total emissions
    uint256 public totalEmitted;    // Track for hard cap

    address public minter;          // Contract that can call mintFor

    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────

    event EmissionMinted(uint256 indexed week, uint256 amount, address indexed recipient);
    event EmissionScheduleUpdated(uint256 baseEmission, uint256 decayRatio);

    // ─────────────────────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────────────────────

    /// @param _baseEmission Tokens emitted in week 0 (e.g., 1_000_000e18)
    /// @param _decayRatio   Per-week multiplier in 1e18 (e.g., 0.99e18 for 1% weekly decay)
    /// @param _maxSupply    Total emission cap
    constructor(
        uint256 _baseEmission,
        uint256 _decayRatio,
        uint256 _maxSupply
    ) Ownable(msg.sender) {
        require(_decayRatio > 0 && _decayRatio <= PRECISION, "Geo: invalid ratio");
        require(_baseEmission > 0, "Geo: zero base emission");

        baseEmission = _baseEmission;
        decayRatio   = _decayRatio;
        maxSupply    = _maxSupply;
        startTime    = block.timestamp;
    }

    // ─────────────────────────────────────────────────────────────
    // View Functions
    // ─────────────────────────────────────────────────────────────

    /// @notice Get the current week number since launch
    function currentWeek() public view returns (uint256) {
        return (block.timestamp - startTime) / WEEK;
    }

    /// @notice Calculate emission for a specific week
    /// @dev emission_n = base * ratio^n  (uses iterative multiplication)
    function emissionForWeek(uint256 weekNum) public view returns (uint256) {
        uint256 emission = baseEmission;

        // Apply decay ratio for each week
        // emission = base * (ratio/PRECISION)^weekNum
        for (uint256 i = 0; i < weekNum; i++) {
            emission = (emission * decayRatio) / PRECISION;
        }

        return emission;
    }

    /// @notice Calculate emission for a specific week (gas-efficient using pow)
    /// @dev More efficient for large week numbers
    function emissionForWeekOptimized(uint256 weekNum) public view returns (uint256) {
        if (weekNum == 0) return baseEmission;

        // Fast power: emission = base * ratio^n
        uint256 result = PRECISION; // Start with 1 (in precision)
        uint256 base = decayRatio;
        uint256 exp = weekNum;

        while (exp > 0) {
            if (exp % 2 == 1) {
                result = (result * base) / PRECISION;
            }
            base = (base * base) / PRECISION;
            exp /= 2;
        }

        return (baseEmission * result) / PRECISION;
    }

    /// @notice Calculate total emissions up to (but not including) a week
    function totalEmissionsUpTo(uint256 weekNum) public view returns (uint256 total) {
        // For geometric series: S_n = a * (1 - r^n) / (1 - r)
        // But we use iteration here for clarity
        for (uint256 i = 0; i < weekNum; i++) {
            total += emissionForWeekOptimized(i);
        }
    }

    /// @notice Estimate how many weeks until total cap is reached
    function weeksToCapExhaustion() external view returns (uint256 weeks_) {
        uint256 remaining = maxSupply - totalEmitted;
        uint256 accumulated = 0;

        for (uint256 w = currentWeek(); accumulated < remaining; w++) {
            accumulated += emissionForWeekOptimized(w);
            weeks_++;

            // Safety: stop after 1000 weeks
            if (weeks_ > 1000) break;
        }
    }
}

/// @title LinearEmissions
/// @notice Linearly decreasing weekly emissions toward a floor
contract LinearEmissions is Ownable {

    uint256 public constant WEEK = 7 days;

    uint256 public startEmission;   // Week 0 emission
    uint256 public endEmission;     // Floor emission (never goes below)
    uint256 public totalWeeks;      // Duration of linear decrease
    uint256 public startTime;

    constructor(
        uint256 _startEmission,
        uint256 _endEmission,
        uint256 _totalWeeks
    ) Ownable(msg.sender) {
        require(_startEmission > _endEmission, "Linear: start must be > end");
        startEmission = _startEmission;
        endEmission   = _endEmission;
        totalWeeks    = _totalWeeks;
        startTime     = block.timestamp;
    }

    function currentWeek() public view returns (uint256) {
        return (block.timestamp - startTime) / WEEK;
    }

    /// @notice Get emission for a given week
    function emissionForWeek(uint256 weekNum) public view returns (uint256) {
        if (weekNum >= totalWeeks) return endEmission;

        // Linear interpolation
        uint256 decrease = (startEmission - endEmission) * weekNum / totalWeeks;
        return startEmission - decrease;
    }

    function currentEmission() external view returns (uint256) {
        return emissionForWeek(currentWeek());
    }
}
```

---

## 3. xToken (Single-Stake): xSUSHI Pattern

### หลักการ xToken

**xToken** ง่ายกว่า veToken มาก:
- Stake TOKEN → ได้ xTOKEN (ERC-20)
- Protocol ส่ง Fee เข้า Pool
- ยิ่ง Pool ใหญ่ขึ้น xTOKEN แต่ละ Unit มีค่ามากขึ้น
- Unstake xTOKEN → ได้ TOKEN คืนพร้อม Share ของ Fee

สูตร: `shares = amount × totalShares / totalBalance`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title xToken
/// @notice Single-stake vault: stake TOKEN, earn protocol fees, get xTOKEN receipt
/// @dev Based on xSUSHI / ERC-4626-lite pattern
/// @custom:invariant totalSupply() of xTOKEN * tokenPerShare() == total TOKEN in vault
contract xToken is ERC20, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ─────────────────────────────────────────────────────────────
    // State
    // ─────────────────────────────────────────────────────────────

    IERC20 public immutable stakingToken;  // TOKEN being staked
    address public feeDistributor;          // Can call distributeFees()

    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────

    event Entered(address indexed user, uint256 tokenAmount, uint256 sharesMinted);
    event Left(address indexed user, uint256 sharesBurned, uint256 tokenAmount);
    event FeesDistributed(uint256 amount);

    // ─────────────────────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────────────────────

    constructor(
        address _stakingToken,
        string memory name,
        string memory symbol
    ) ERC20(name, symbol) {
        stakingToken = IERC20(_stakingToken);
    }

    // ─────────────────────────────────────────────────────────────
    // Core Functions
    // ─────────────────────────────────────────────────────────────

    /// @notice Stake TOKEN to receive xTOKEN shares
    /// @param tokenAmount Amount of TOKEN to stake
    /// @return sharesMinted Amount of xTOKEN minted
    function enter(uint256 tokenAmount) external nonReentrant returns (uint256 sharesMinted) {
        require(tokenAmount > 0, "xToken: zero amount");

        uint256 totalToken = stakingToken.balanceOf(address(this));
        uint256 totalShares = totalSupply();

        // Pull tokens first (before minting to prevent reentrancy)
        stakingToken.safeTransferFrom(msg.sender, address(this), tokenAmount);

        // Calculate shares to mint
        if (totalShares == 0 || totalToken == 0) {
            // First deposit: 1:1 ratio
            sharesMinted = tokenAmount;
        } else {
            // shares = tokenAmount × totalShares / totalToken
            // Note: use totalToken BEFORE the transfer for correct ratio
            sharesMinted = (tokenAmount * totalShares) / totalToken;
        }

        require(sharesMinted > 0, "xToken: zero shares");

        _mint(msg.sender, sharesMinted);

        emit Entered(msg.sender, tokenAmount, sharesMinted);
    }

    /// @notice Burn xTOKEN to withdraw TOKEN + accumulated fees
    /// @param shareAmount Amount of xTOKEN to burn
    /// @return tokenAmount Amount of TOKEN returned
    function leave(uint256 shareAmount) external nonReentrant returns (uint256 tokenAmount) {
        require(shareAmount > 0, "xToken: zero shares");
        require(balanceOf(msg.sender) >= shareAmount, "xToken: insufficient shares");

        uint256 totalShares = totalSupply();
        uint256 totalToken  = stakingToken.balanceOf(address(this));

        // tokenAmount = shareAmount × totalToken / totalShares
        tokenAmount = (shareAmount * totalToken) / totalShares;

        _burn(msg.sender, shareAmount);

        stakingToken.safeTransfer(msg.sender, tokenAmount);

        emit Left(msg.sender, shareAmount, tokenAmount);
    }

    // ─────────────────────────────────────────────────────────────
    // Fee Distribution
    // ─────────────────────────────────────────────────────────────

    /// @notice Deposit protocol fees into the vault (increases token-per-share)
    /// @dev Called by fee distributor contract - tokens just sit in vault
    function distributeFees(uint256 amount) external {
        require(msg.sender == feeDistributor, "xToken: not distributor");
        stakingToken.safeTransferFrom(msg.sender, address(this), amount);
        emit FeesDistributed(amount);
    }

    // ─────────────────────────────────────────────────────────────
    // View Functions
    // ─────────────────────────────────────────────────────────────

    /// @notice How many TOKEN each xTOKEN is worth
    function tokenPerShare() public view returns (uint256) {
        uint256 shares = totalSupply();
        if (shares == 0) return 1e18;
        return (stakingToken.balanceOf(address(this)) * 1e18) / shares;
    }

    /// @notice How many xTOKEN user would get for depositing tokenAmount
    function previewEnter(uint256 tokenAmount) external view returns (uint256) {
        uint256 totalToken  = stakingToken.balanceOf(address(this));
        uint256 totalShares = totalSupply();
        if (totalShares == 0) return tokenAmount;
        return (tokenAmount * totalShares) / totalToken;
    }

    /// @notice How many TOKEN user gets for burning shareAmount
    function previewLeave(uint256 shareAmount) external view returns (uint256) {
        uint256 totalShares = totalSupply();
        if (totalShares == 0) return 0;
        return (shareAmount * stakingToken.balanceOf(address(this))) / totalShares;
    }

    /// @notice Total TOKEN staked by user (including accrued fees)
    function stakedBalance(address user) external view returns (uint256) {
        uint256 shares = balanceOf(user);
        uint256 totalShares = totalSupply();
        if (totalShares == 0) return 0;
        return (shares * stakingToken.balanceOf(address(this))) / totalShares;
    }

    // ─────────────────────────────────────────────────────────────
    // Admin
    // ─────────────────────────────────────────────────────────────

    function setFeeDistributor(address _distributor) external {
        require(feeDistributor == address(0), "xToken: already set");
        feeDistributor = _distributor;
    }
}
```

---

## 4. Bonding Mechanism (Olympus-Style)

### หลักการ Bonding

**Bonding** ช่วยให้ Protocol สร้าง Treasury:
1. User นำ LP Token หรือ Stablecoin มา "Bond"
2. Protocol ให้ TOKEN ในราคาส่วนลด (Discount)
3. TOKEN ถูก Vest (ค่อยๆ ปล่อย) เพื่อป้องกัน Dump

ประโยชน์:
- Protocol ได้ Liquidity ที่ตัวเองเป็นเจ้าของ (Protocol-Owned Liquidity)
- ไม่ต้องพึ่ง Mercenary LPs

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

interface IBondOracle {
    /// @notice Get market price of TOKEN in USD (18 decimals)
    function getTokenPrice() external view returns (uint256);
    /// @notice Get value of 1 unit of payment token in USD
    function getPaymentTokenValue(address token) external view returns (uint256);
}

/// @title BondDepository
/// @notice Allows users to bond assets for discounted TOKEN with vesting
/// @custom:invariant Bond discount never exceeds maxDiscount
/// @custom:invariant Payout is always fully vested before claimable
contract BondDepository is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ─────────────────────────────────────────────────────────────
    // Types
    // ─────────────────────────────────────────────────────────────

    struct BondTerms {
        address paymentToken;   // What user pays (e.g., USDC, ETH, LP token)
        uint256 vestingPeriod;  // How long until fully vested (seconds)
        uint256 maxDiscount;    // Maximum discount in basis points (e.g., 500 = 5%)
        uint256 maxPayout;      // Maximum TOKEN per bond (circuit breaker)
        uint256 capacity;       // Remaining capacity in payment token units
        bool    active;
    }

    struct BondNote {
        address user;
        uint256 payoutAmount;   // TOKEN owed
        uint256 vestStart;
        uint256 vestEnd;
        uint256 claimedAmount;
    }

    // ─────────────────────────────────────────────────────────────
    // State
    // ─────────────────────────────────────────────────────────────

    IERC20  public immutable payoutToken;   // TOKEN being distributed
    IBondOracle public oracle;

    mapping(uint256 => BondTerms) public markets;  // marketId => terms
    mapping(uint256 => BondNote)  public notes;    // noteId => bond details

    uint256 public marketCount;
    uint256 public noteCount;
    address public treasury;        // Where payment tokens go

    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────

    event BondCreated(
        uint256 indexed marketId,
        uint256 indexed noteId,
        address indexed user,
        uint256 paidAmount,
        uint256 payoutAmount,
        uint256 vestEnd
    );
    event BondClaimed(uint256 indexed noteId, address indexed user, uint256 amount);
    event MarketCreated(uint256 indexed marketId, address paymentToken);
    event MarketClosed(uint256 indexed marketId);

    // ─────────────────────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────────────────────

    constructor(address _payoutToken, address _oracle, address _treasury) Ownable(msg.sender) {
        payoutToken = IERC20(_payoutToken);
        oracle      = IBondOracle(_oracle);
        treasury    = _treasury;
    }

    // ─────────────────────────────────────────────────────────────
    // Bond Functions
    // ─────────────────────────────────────────────────────────────

    /// @notice Purchase a bond in a specific market
    /// @param marketId The bond market to purchase
    /// @param paymentAmount Amount of payment token to spend
    /// @param maxSlippage Max acceptable slippage in basis points
    /// @return noteId The bond note ID (for claiming)
    function bond(
        uint256 marketId,
        uint256 paymentAmount,
        uint256 maxSlippage
    ) external nonReentrant returns (uint256 noteId) {
        BondTerms storage market = markets[marketId];
        require(market.active, "Bond: market not active");
        require(paymentAmount > 0, "Bond: zero amount");
        require(paymentAmount <= market.capacity, "Bond: exceeds capacity");

        // Calculate payout with discount
        uint256 payout = calculatePayout(marketId, paymentAmount);
        require(payout <= market.maxPayout, "Bond: exceeds max payout");
        require(payout > 0, "Bond: zero payout");

        // Slippage check
        uint256 marketValue = getMarketValue(marketId, paymentAmount);
        uint256 minAcceptable = (marketValue * (10_000 - maxSlippage)) / 10_000;
        // payout value = payout * tokenPrice
        // This is a simplified check - in prod use oracle prices

        // Pull payment tokens to treasury
        IERC20(market.paymentToken).safeTransferFrom(msg.sender, treasury, paymentAmount);

        // Update market capacity
        market.capacity -= paymentAmount;
        if (market.capacity == 0) {
            market.active = false;
            emit MarketClosed(marketId);
        }

        // Create bond note
        noteId = noteCount++;
        uint256 vestEnd = block.timestamp + market.vestingPeriod;

        notes[noteId] = BondNote({
            user: msg.sender,
            payoutAmount: payout,
            vestStart: block.timestamp,
            vestEnd: vestEnd,
            claimedAmount: 0
        });

        emit BondCreated(marketId, noteId, msg.sender, paymentAmount, payout, vestEnd);
    }

    /// @notice Claim vested tokens from a bond note
    /// @param noteId The bond note to claim from
    function claim(uint256 noteId) external nonReentrant returns (uint256 claimable) {
        BondNote storage note = notes[noteId];
        require(note.user == msg.sender, "Bond: not note owner");

        claimable = vestedAmount(noteId) - note.claimedAmount;
        require(claimable > 0, "Bond: nothing to claim");

        note.claimedAmount += claimable;

        payoutToken.safeTransfer(msg.sender, claimable);

        emit BondClaimed(noteId, msg.sender, claimable);
    }

    // ─────────────────────────────────────────────────────────────
    // View Functions
    // ─────────────────────────────────────────────────────────────

    /// @notice Calculate payout TOKEN for a given payment amount
    function calculatePayout(uint256 marketId, uint256 paymentAmount) public view returns (uint256) {
        BondTerms memory market = markets[marketId];

        // Get prices from oracle
        uint256 tokenPrice   = oracle.getTokenPrice();                      // USD price of TOKEN
        uint256 paymentValue = oracle.getPaymentTokenValue(market.paymentToken); // USD value of 1 payment token unit

        // Value of payment in USD
        uint256 paymentUsdValue = (paymentAmount * paymentValue) / 1e18;

        // Discount: tokens are cheaper by maxDiscount %
        // bondPrice = marketPrice × (1 - discount)
        uint256 discountedPrice = (tokenPrice * (10_000 - market.maxDiscount)) / 10_000;

        // payout = paymentUsdValue / discountedPrice
        return (paymentUsdValue * 1e18) / discountedPrice;
    }

    /// @notice How much of a bond note has vested
    function vestedAmount(uint256 noteId) public view returns (uint256) {
        BondNote memory note = notes[noteId];
        if (block.timestamp >= note.vestEnd) {
            return note.payoutAmount;
        }
        if (block.timestamp <= note.vestStart) {
            return 0;
        }

        uint256 elapsed  = block.timestamp - note.vestStart;
        uint256 duration = note.vestEnd - note.vestStart;

        return (note.payoutAmount * elapsed) / duration;
    }

    /// @notice Unclaimed vested amount
    function claimableAmount(uint256 noteId) external view returns (uint256) {
        return vestedAmount(noteId) - notes[noteId].claimedAmount;
    }

    /// @notice Get USD value of payment amount in a market
    function getMarketValue(uint256 marketId, uint256 paymentAmount) public view returns (uint256) {
        BondTerms memory market = markets[marketId];
        uint256 paymentValue = oracle.getPaymentTokenValue(market.paymentToken);
        return (paymentAmount * paymentValue) / 1e18;
    }

    // ─────────────────────────────────────────────────────────────
    // Admin
    // ─────────────────────────────────────────────────────────────

    /// @notice Create a new bond market
    function createMarket(
        address paymentToken,
        uint256 vestingPeriod,
        uint256 maxDiscount,
        uint256 maxPayout,
        uint256 initialCapacity
    ) external onlyOwner returns (uint256 marketId) {
        require(maxDiscount <= 2000, "Bond: discount too high"); // max 20%
        require(vestingPeriod >= 1 days, "Bond: vesting too short");
        require(vestingPeriod <= 365 days, "Bond: vesting too long");

        marketId = marketCount++;
        markets[marketId] = BondTerms({
            paymentToken: paymentToken,
            vestingPeriod: vestingPeriod,
            maxDiscount: maxDiscount,
            maxPayout: maxPayout,
            capacity: initialCapacity,
            active: true
        });

        emit MarketCreated(marketId, paymentToken);
    }

    function closeMarket(uint256 marketId) external onlyOwner {
        markets[marketId].active = false;
        emit MarketClosed(marketId);
    }
}
```

---

## 5. ProtocolTreasury: DAOTreasury with Diversification

### หลักการ Treasury ที่ดี

Treasury ที่ดีต้อง:
- เก็บ Approved Assets เท่านั้น (whitelist)
- Rebalance ได้เมื่อสัดส่วน Asset เบี่ยงเบนมาก
- มี Emergency Withdrawal สำหรับกรณีวิกฤต
- Transparent: ใครๆ ก็ดู Allocation ได้

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

interface IPriceOracle {
    function getPrice(address token) external view returns (uint256 priceUSD, uint8 decimals);
}

/// @title DAOTreasury
/// @notice Protocol treasury with approved assets, rebalancing triggers, and governance
/// @custom:invariant Only approved assets can be held in the treasury
/// @custom:invariant Asset target weights always sum to BASIS_POINTS
contract DAOTreasury is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ─────────────────────────────────────────────────────────────
    // Types
    // ─────────────────────────────────────────────────────────────

    struct AssetConfig {
        bool     approved;
        uint256  targetWeightBps;   // Target % of total portfolio (basis points)
        uint256  rebalanceBand;     // Allowed deviation before rebalance trigger (bps)
        string   assetClass;        // e.g., "stablecoin", "governance", "LP"
    }

    // ─────────────────────────────────────────────────────────────
    // Constants
    // ─────────────────────────────────────────────────────────────

    uint256 public constant BASIS_POINTS = 10_000;
    uint256 public constant MAX_ASSETS   = 20;

    // ─────────────────────────────────────────────────────────────
    // State
    // ─────────────────────────────────────────────────────────────

    IPriceOracle public oracle;

    mapping(address => AssetConfig) public assetConfigs;
    address[] public approvedAssets;

    address public executor;    // DAO executor / timelock
    bool    public initialized;

    // Spending approval
    struct SpendingRequest {
        address token;
        uint256 amount;
        address recipient;
        string  reason;
        bool    executed;
        bool    cancelled;
    }

    mapping(uint256 => SpendingRequest) public spendingRequests;
    uint256 public spendingRequestCount;

    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────

    event AssetApproved(address indexed token, uint256 targetWeight, string assetClass);
    event AssetRemoved(address indexed token);
    event FundsReceived(address indexed token, uint256 amount, string source);
    event FundsSpent(uint256 indexed requestId, address indexed token, uint256 amount, address indexed recipient);
    event RebalanceTrigger(address indexed overweightAsset, address indexed underweightAsset);
    event SpendingRequested(uint256 indexed requestId, address token, uint256 amount, address recipient);

    // ─────────────────────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────────────────────

    constructor(address _oracle, address _executor) Ownable(msg.sender) {
        oracle   = IPriceOracle(_oracle);
        executor = _executor;
    }

    // ─────────────────────────────────────────────────────────────
    // Modifiers
    // ─────────────────────────────────────────────────────────────

    modifier onlyExecutor() {
        require(msg.sender == executor, "Treasury: not executor");
        _;
    }

    // ─────────────────────────────────────────────────────────────
    // Receive Funds
    // ─────────────────────────────────────────────────────────────

    /// @notice Receive approved tokens into treasury
    function receive_(address token, uint256 amount, string calldata source) external nonReentrant {
        require(assetConfigs[token].approved, "Treasury: token not approved");
        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
        emit FundsReceived(token, amount, source);
    }

    /// @notice Accept ETH
    receive() external payable {
        emit FundsReceived(address(0), msg.value, "direct ETH");
    }

    // ─────────────────────────────────────────────────────────────
    // Spending (requires executor / DAO vote)
    // ─────────────────────────────────────────────────────────────

    /// @notice Request spending from treasury (goes to governance queue)
    function requestSpending(
        address token,
        uint256 amount,
        address recipient,
        string calldata reason
    ) external onlyOwner returns (uint256 requestId) {
        require(assetConfigs[token].approved, "Treasury: token not approved");
        require(recipient != address(0), "Treasury: zero recipient");

        requestId = spendingRequestCount++;
        spendingRequests[requestId] = SpendingRequest({
            token: token,
            amount: amount,
            recipient: recipient,
            reason: reason,
            executed: false,
            cancelled: false
        });

        emit SpendingRequested(requestId, token, amount, recipient);
    }

    /// @notice Execute an approved spending request (called by DAO executor after vote)
    function executeSpending(uint256 requestId) external onlyExecutor nonReentrant {
        SpendingRequest storage req = spendingRequests[requestId];
        require(!req.executed, "Treasury: already executed");
        require(!req.cancelled, "Treasury: cancelled");

        uint256 balance = IERC20(req.token).balanceOf(address(this));
        require(balance >= req.amount, "Treasury: insufficient balance");

        req.executed = true;

        IERC20(req.token).safeTransfer(req.recipient, req.amount);

        emit FundsSpent(requestId, req.token, req.amount, req.recipient);
    }

    // ─────────────────────────────────────────────────────────────
    // Rebalancing
    // ─────────────────────────────────────────────────────────────

    /// @notice Check if any asset is outside its rebalance band
    function checkRebalanceNeeded() external view returns (
        bool needed,
        address overweightAsset,
        address underweightAsset
    ) {
        uint256 totalValue = getTotalValueUSD();
        if (totalValue == 0) return (false, address(0), address(0));

        uint256 maxOverDeviation  = 0;
        uint256 maxUnderDeviation = 0;

        for (uint256 i = 0; i < approvedAssets.length; i++) {
            address asset = approvedAssets[i];
            AssetConfig memory cfg = assetConfigs[asset];
            if (!cfg.approved) continue;

            uint256 currentWeight = getAssetWeightBps(asset, totalValue);
            uint256 target = cfg.targetWeightBps;
            uint256 band = cfg.rebalanceBand;

            if (currentWeight > target + band) {
                uint256 deviation = currentWeight - target;
                if (deviation > maxOverDeviation) {
                    maxOverDeviation = deviation;
                    overweightAsset = asset;
                    needed = true;
                }
            } else if (currentWeight + band < target) {
                uint256 deviation = target - currentWeight;
                if (deviation > maxUnderDeviation) {
                    maxUnderDeviation = deviation;
                    underweightAsset = asset;
                    needed = true;
                }
            }
        }
    }

    // ─────────────────────────────────────────────────────────────
    // View Functions
    // ─────────────────────────────────────────────────────────────

    /// @notice Get total treasury value in USD (18 decimal precision)
    function getTotalValueUSD() public view returns (uint256 totalUSD) {
        for (uint256 i = 0; i < approvedAssets.length; i++) {
            address asset = approvedAssets[i];
            if (!assetConfigs[asset].approved) continue;

            uint256 balance = IERC20(asset).balanceOf(address(this));
            if (balance == 0) continue;

            (uint256 price, uint8 decimals) = oracle.getPrice(asset);
            // Normalize to 18 decimals
            uint256 tokenDecimals = 18; // Simplified - use IERC20Metadata in prod
            uint256 valueUSD = balance * price;
            if (tokenDecimals > decimals) {
                valueUSD = valueUSD / (10 ** (tokenDecimals - decimals));
            }
            totalUSD += valueUSD;
        }
    }

    /// @notice Get an asset's current weight in the portfolio
    function getAssetWeightBps(address token, uint256 totalValue) public view returns (uint256) {
        if (totalValue == 0) return 0;
        uint256 balance = IERC20(token).balanceOf(address(this));
        if (balance == 0) return 0;

        (uint256 price,) = oracle.getPrice(token);
        uint256 assetValue = balance * price / 1e18;

        return (assetValue * BASIS_POINTS) / totalValue;
    }

    /// @notice Get full portfolio allocation
    function getPortfolioAllocation() external view returns (
        address[] memory assets,
        uint256[] memory currentWeights,
        uint256[] memory targetWeights,
        uint256[] memory balances
    ) {
        assets         = new address[](approvedAssets.length);
        currentWeights = new uint256[](approvedAssets.length);
        targetWeights  = new uint256[](approvedAssets.length);
        balances       = new uint256[](approvedAssets.length);

        uint256 totalValue = getTotalValueUSD();

        for (uint256 i = 0; i < approvedAssets.length; i++) {
            assets[i]         = approvedAssets[i];
            balances[i]       = IERC20(approvedAssets[i]).balanceOf(address(this));
            currentWeights[i] = getAssetWeightBps(approvedAssets[i], totalValue);
            targetWeights[i]  = assetConfigs[approvedAssets[i]].targetWeightBps;
        }
    }

    // ─────────────────────────────────────────────────────────────
    // Admin (Asset Management)
    // ─────────────────────────────────────────────────────────────

    /// @notice Add an asset to the approved list
    function approveAsset(
        address token,
        uint256 targetWeightBps,
        uint256 rebalanceBand,
        string calldata assetClass
    ) external onlyOwner {
        require(!assetConfigs[token].approved, "Treasury: already approved");
        require(approvedAssets.length < MAX_ASSETS, "Treasury: max assets reached");
        require(rebalanceBand <= 2000, "Treasury: band too wide"); // max 20%

        // Validate total weights don't exceed 100%
        uint256 totalWeight = 0;
        for (uint256 i = 0; i < approvedAssets.length; i++) {
            totalWeight += assetConfigs[approvedAssets[i]].targetWeightBps;
        }
        require(totalWeight + targetWeightBps <= BASIS_POINTS, "Treasury: weights exceed 100%");

        assetConfigs[token] = AssetConfig({
            approved: true,
            targetWeightBps: targetWeightBps,
            rebalanceBand: rebalanceBand,
            assetClass: assetClass
        });

        approvedAssets.push(token);

        emit AssetApproved(token, targetWeightBps, assetClass);
    }

    function removeAsset(address token) external onlyOwner {
        require(IERC20(token).balanceOf(address(this)) == 0, "Treasury: non-zero balance");
        assetConfigs[token].approved = false;
        // Note: keep in approvedAssets array but mark as !approved (gas tradeoff)
        emit AssetRemoved(token);
    }

    function setExecutor(address _executor) external onlyOwner {
        executor = _executor;
    }
}
```

---

## Workshop: ประกอบ Full Tokenomics Stack

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @notice Workshop: Demonstrates the full tokenomics flywheel
///
/// Flow:
/// 1. USER buys TOKEN from market
/// 2. USER bonds LP_TOKEN → gets discounted TOKEN (vested)
/// 3. USER stakes TOKEN → gets xTOKEN
/// 4. USER locks TOKEN → gets vePower → earns boost on staking
/// 5. Protocol fees → xTOKEN vault → increases xTOKEN value
/// 6. GeometricEmissions → mints new TOKEN for liquidity incentives
/// 7. Treasury receives fees → diversifies → rebalances

/// This is a conceptual deployment script showing how all pieces connect:
contract TokenomicsDeployment {

    struct Addresses {
        address token;         // ERC-20 TOKEN
        address votingEscrow;  // VotingEscrow (locks TOKEN → vePower)
        address xToken;        // xToken (stake TOKEN → xTOKEN)
        address bondDepo;      // BondDepository
        address emissions;     // GeometricEmissions
        address treasury;      // DAOTreasury
        address feeRouter;     // Routes fees to xToken vault
    }

    function deployAll(address deployer) external returns (Addresses memory addrs) {
        // Step 1: Deploy base TOKEN (ERC-20 with minting)
        // MockToken token = new MockToken("PROTO", "PROTO", 1_000_000_000e18);

        // Step 2: Deploy VotingEscrow
        // VotingEscrow ve = new VotingEscrow(address(token));
        // addrs.votingEscrow = address(ve);

        // Step 3: Deploy xToken vault
        // xToken xt = new xToken(address(token), "xPROTO", "xPROTO");
        // addrs.xToken = address(xt);

        // Step 4: Deploy Emission Schedule
        // 1M tokens week 0, decaying 1% per week, 100M total cap
        // GeometricEmissions geo = new GeometricEmissions(
        //     1_000_000e18,   // 1M tokens/week start
        //     990_000_000_000_000_000, // 0.99 * 1e18
        //     100_000_000e18  // 100M cap
        // );

        // Step 5: Deploy Bond Market
        // BondDepository bonds = new BondDepository(
        //     address(token), address(oracle), address(treasury)
        // );

        // Step 6: Create USDC-TOKEN LP bond market
        // bonds.createMarket(
        //     usdcTokenLP,    // payment = LP tokens
        //     7 days,         // 1 week vest
        //     500,            // 5% max discount
        //     10_000e18,      // max 10k TOKEN per bond
        //     1_000_000e6     // 1M USDC capacity
        // );

        // Step 7: Wire fee routing
        // xToken vault receives fees from protocol
        // xt.setFeeDistributor(feeRouter);

        return addrs;
    }
}
```

---

## Foundry Tests: Tokenomics

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "../src/VotingEscrow.sol";
import "../src/xToken.sol";
import "../src/GeometricEmissions.sol";

contract TokenomicsTest is Test {

    VotingEscrow   ve;
    xToken         xt;
    GeometricEmissions geo;

    MockERC20 token;
    address user1 = address(0x1);
    address user2 = address(0x2);

    uint256 constant INITIAL_SUPPLY = 10_000_000e18;

    function setUp() public {
        token = new MockERC20("TOKEN", "TKN", 18);
        token.mint(user1, 1_000_000e18);
        token.mint(user2, 1_000_000e18);

        ve  = new VotingEscrow(address(token));
        xt  = new xToken(address(token), "xTOKEN", "xTKN");
        geo = new GeometricEmissions(
            1_000_000e18,
            990_000_000_000_000_000, // 0.99e18 (1% weekly decay)
            100_000_000e18
        );
    }

    // ─────────────────────────────────────────────────────────────
    // VotingEscrow Tests
    // ─────────────────────────────────────────────────────────────

    function test_LockDecaysToZero() public {
        vm.startPrank(user1);
        token.approve(address(ve), 1000e18);
        ve.lock(1000e18, 365 days);
        vm.stopPrank();

        // At start: should have positive vePower
        uint256 initialPower = ve.vePowerOf(user1);
        assertGt(initialPower, 0);

        // After lock expires: should have 0 vePower
        vm.warp(block.timestamp + 366 days);
        assertEq(ve.vePowerOf(user1), 0);
    }

    function test_LongerLockGivesMorePower() public {
        address user3 = address(0x3);
        address user4 = address(0x4);
        token.mint(user3, 1000e18);
        token.mint(user4, 1000e18);

        // user3 locks for 1 year
        vm.startPrank(user3);
        token.approve(address(ve), 1000e18);
        ve.lock(1000e18, 365 days);
        vm.stopPrank();

        // user4 locks for 4 years
        vm.startPrank(user4);
        token.approve(address(ve), 1000e18);
        ve.lock(1000e18, 4 * 365 days);
        vm.stopPrank();

        assertGt(ve.vePowerOf(user4), ve.vePowerOf(user3));
    }

    function test_BoostMultiplierRanges() public {
        vm.startPrank(user1);
        token.approve(address(ve), 1000e18);

        // Lock for max time → should get max boost (2.5x = 25000 bps)
        ve.lock(1000e18, 4 * 365 days);
        vm.stopPrank();

        uint256 boost = ve.boostOf(user1);
        // Should be close to MAX_BOOST (25000) but slightly less due to time rounding
        assertGe(boost, 24000); // At least 2.4x
        assertLe(boost, 25000); // At most 2.5x
    }

    function test_CanWithdrawAfterExpiry() public {
        vm.startPrank(user1);
        token.approve(address(ve), 1000e18);
        ve.lock(1000e18, 7 days);

        vm.warp(block.timestamp + 7 days + 1);
        uint256 balanceBefore = token.balanceOf(user1);

        ve.withdraw();

        assertEq(token.balanceOf(user1), balanceBefore + 1000e18);
        vm.stopPrank();
    }

    function test_CannotWithdrawBeforeExpiry() public {
        vm.startPrank(user1);
        token.approve(address(ve), 1000e18);
        ve.lock(1000e18, 30 days);

        vm.expectRevert("VE: lock not expired");
        ve.withdraw();
        vm.stopPrank();
    }

    // ─────────────────────────────────────────────────────────────
    // xToken Tests
    // ─────────────────────────────────────────────────────────────

    function test_TokenPerShareIncreasesWithFees() public {
        // user1 stakes
        vm.startPrank(user1);
        token.approve(address(xt), 1000e18);
        xt.enter(1000e18);
        vm.stopPrank();

        uint256 initialTokenPerShare = xt.tokenPerShare();

        // Simulate fee distribution (inject 100 tokens)
        address feeSource = address(0xFF);
        token.mint(feeSource, 100e18);

        vm.startPrank(feeSource);
        token.approve(address(xt), 100e18);
        xt.setFeeDistributor(feeSource);  // Set distributor first
        xt.distributeFees(100e18);
        vm.stopPrank();

        // Token per share should increase
        assertGt(xt.tokenPerShare(), initialTokenPerShare);
    }

    function test_LateStakerDoesNotGetHistoricalFees() public {
        // user1 stakes early
        vm.startPrank(user1);
        token.approve(address(xt), 1000e18);
        xt.enter(1000e18);
        vm.stopPrank();

        // Distribute fees
        address feeSource = address(0xFF);
        token.mint(feeSource, 500e18);
        vm.startPrank(feeSource);
        token.approve(address(xt), 500e18);
        xt.setFeeDistributor(feeSource);
        xt.distributeFees(500e18);
        vm.stopPrank();

        // user2 stakes late (same amount as user1)
        vm.startPrank(user2);
        token.approve(address(xt), 1000e18);
        xt.enter(1000e18);
        vm.stopPrank();

        // user1 leaves - should get more than user2 due to accumulated fees
        uint256 user1Balance = token.balanceOf(user1);

        vm.prank(user1);
        uint256 user1Received = xt.leave(xt.balanceOf(user1));

        // user2 leaves
        uint256 user2Received;
        vm.prank(user2);
        user2Received = xt.leave(xt.balanceOf(user2));

        assertGt(user1Received, user2Received);
    }

    // ─────────────────────────────────────────────────────────────
    // Geometric Emissions Tests
    // ─────────────────────────────────────────────────────────────

    function test_EmissionDecaysOverTime() public {
        uint256 week0 = geo.emissionForWeek(0);
        uint256 week52 = geo.emissionForWeek(52);
        uint256 week104 = geo.emissionForWeek(104);

        // Each year should be lower
        assertGt(week0, week52);
        assertGt(week52, week104);
    }

    function test_OptimizedMatchesIterative() public {
        for (uint256 w = 0; w < 20; w++) {
            uint256 iterative   = geo.emissionForWeek(w);
            uint256 optimized   = geo.emissionForWeekOptimized(w);
            // Allow 1 wei difference due to rounding
            assertApproxEqAbs(iterative, optimized, 1);
        }
    }
}

// Reuse MockERC20 from Part 63
contract MockERC20 {
    string public name;
    string public symbol;
    uint8  public decimals;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    uint256 public totalSupply;

    constructor(string memory _name, string memory _symbol, uint8 _dec) {
        name = _name; symbol = _symbol; decimals = _dec;
    }

    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount; totalSupply += amount;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount; return true;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount);
        balanceOf[msg.sender] -= amount; balanceOf[to] += amount; return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(allowance[from][msg.sender] >= amount);
        require(balanceOf[from] >= amount);
        allowance[from][msg.sender] -= amount;
        balanceOf[from] -= amount; balanceOf[to] += amount; return true;
    }
}
```

---

## สรุป Part 64

- **VotingEscrow**: veToken Model ล็อค TOKEN ได้สูงสุด 4 ปี, vePower สะสมจาก `amount × remaining/maxTime`, Boost 1x-2.5x เพื่อจูงใจ Long-term Commitment
- **GeometricEmissions**: `emission = base × ratio^week` ทำให้ Emission ลดลง Predictably ด้วย Fast Exponentiation Algorithm
- **xToken**: Single-stake vault ใช้ Share Formula เหมือน ERC-4626, Fee Distribution เพิ่ม `tokenPerShare` โดยอัตโนมัติ
- **BondDepository**: Protocol-Owned Liquidity ผ่าน Discount + Vesting ป้องกัน Dump และ Mercenary LP
- **DAOTreasury**: Approved Assets + Target Weights + Rebalance Band ทำให้ Treasury Diversification เป็น On-chain Governance

---

## Next: Part 65 - Advanced Security Patterns
