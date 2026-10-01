# Part 55: Advanced Lending Protocols

## บทนำ

Lending protocols เป็น backbone ของ DeFi ที่ lock TVL มากที่สุด ในบทนี้เราจะเรียนรู้ architecture ขั้นสูงที่ใช้ใน Morpho, Aave V3, Compound V3 และ Euler Finance:

- **Isolated Margin Lending**: แต่ละคู่ collateral-borrow มี pool แยกกัน ลด systemic risk
- **JumpRate Interest Model**: Interest model ที่ใช้ "kink" เพื่อ incentivize optimal utilization
- **Dutch Auction Liquidation**: ประมูลราคาลดลงตามเวลาเพื่อ efficient liquidation
- **Bad Debt Socialization**: กระจายหนี้เสียให้ LP ทั้งหมดรับผิดชอบ
- **Morpho P2P Matching**: Match lenders และ borrowers โดยตรงเพื่อ better rates

---

## 1. Isolated Margin Lending

### ทฤษฎี

**Isolated Pools** แก้ปัญหาของ shared pool (เช่น Aave V2):
- ใน shared pool: ถ้า asset เดียวถูก exploit → pool ทั้งหมดเสียหาย
- ใน isolated pool: แต่ละคู่ (WETH/USDC, WBTC/USDC, etc.) มี liquidity แยก

```
Trade-offs:
Pros: ↓ systemic risk, สามารถ list assets ที่ exotic ได้
Cons: ↑ liquidity fragmentation, ต้องจัดการหลาย pools
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/utils/math/Math.sol";

/// @title IsolatedLendingPool - Pool ที่แยกความเสี่ยงต่อ collateral-borrow คู่
/// @notice แต่ละ pool handle ความสัมพันธ์ระหว่าง asset คู่เดียว
contract IsolatedLendingPool is ReentrancyGuard {
    using SafeERC20 for IERC20;
    using Math for uint256;
    
    // ============================================================
    //                          ERRORS
    // ============================================================
    error InsufficientLiquidity(uint256 available, uint256 requested);
    error InsufficientCollateral(uint256 collateral, uint256 required);
    error PositionHealthy(address borrower, uint256 healthFactor);
    error PositionUnhealthy(address borrower, uint256 healthFactor);
    error MaxLoanToValueExceeded(uint256 ltv, uint256 maxLtv);
    error ZeroAmount();
    error OnlyLiquidator();
    error MarketFrozen();
    error BorrowCapExceeded(uint256 current, uint256 cap);
    error SupplyCapExceeded(uint256 current, uint256 cap);
    
    // ============================================================
    //                          EVENTS
    // ============================================================
    event Supply(address indexed supplier, uint256 amount, uint256 shares);
    event Withdraw(address indexed supplier, uint256 amount, uint256 shares);
    event Borrow(address indexed borrower, uint256 amount);
    event Repay(address indexed borrower, uint256 amount, uint256 remaining);
    event Liquidate(address indexed liquidator, address indexed borrower, uint256 repaid, uint256 collateralSeized);
    event InterestAccrued(uint256 newBorrowIndex, uint256 newSupplyIndex);
    event MarketConfigUpdated(uint256 maxLtv, uint256 liqThreshold, uint256 liqBonus);
    
    // ============================================================
    //                          STRUCTS
    // ============================================================
    
    struct MarketConfig {
        uint256 maxLoanToValue;         // LTV สูงสุด (bps), e.g. 7500 = 75%
        uint256 liquidationThreshold;   // threshold ที่ liquidation เกิดขึ้น (bps), e.g. 8000 = 80%
        uint256 liquidationBonus;       // โบนัส liquidator (bps), e.g. 500 = 5%
        uint256 supplyCap;              // Cap total supply
        uint256 borrowCap;              // Cap total borrows
        bool frozen;                    // ถ้า frozen = ไม่รับ supply/borrow ใหม่
    }
    
    struct SupplyPosition {
        uint256 shares;                 // Shares ใน supply pool
    }
    
    struct BorrowPosition {
        uint256 principal;              // Principal amount borrowed
        uint256 interestIndex;          // Index ตอนที่ borrow (สำหรับ interest calculation)
    }
    
    struct PoolState {
        uint256 totalSupplyAssets;      // Total assets supplied (รวม interest)
        uint256 totalSupplyShares;      // Total shares ออกไป
        uint256 totalBorrowAssets;      // Total assets borrowed (รวม interest)
        uint256 totalBorrowShares;      // Total borrow shares
        uint256 borrowIndex;            // Accumulated borrow interest index (1e18 = 1x)
        uint256 supplyIndex;            // Accumulated supply interest index
        uint256 lastAccrualTimestamp;   // ครั้งล่าสุดที่ accrue interest
    }
    
    // ============================================================
    //                          STORAGE
    // ============================================================
    
    IERC20 public immutable collateralToken;    // Collateral (เช่น WETH)
    IERC20 public immutable borrowToken;         // Borrow asset (เช่น USDC)
    
    IInterestRateModel public interestRateModel;
    IPriceOracle public oracle;
    
    MarketConfig public config;
    PoolState public poolState;
    
    mapping(address => SupplyPosition) public supplyPositions;
    mapping(address => BorrowPosition) public borrowPositions;
    mapping(address => uint256) public collateralBalances;
    
    uint256 public constant PRECISION = 1e18;
    uint256 public constant BPS = 10_000;
    uint256 public constant INITIAL_INDEX = 1e18;
    
    // ============================================================
    //                        CONSTRUCTOR
    // ============================================================
    
    constructor(
        address _collateralToken,
        address _borrowToken,
        address _oracle,
        address _interestRateModel,
        uint256 _maxLtv,
        uint256 _liquidationThreshold,
        uint256 _liquidationBonus
    ) {
        collateralToken = IERC20(_collateralToken);
        borrowToken = IERC20(_borrowToken);
        oracle = IPriceOracle(_oracle);
        interestRateModel = IInterestRateModel(_interestRateModel);
        
        config = MarketConfig({
            maxLoanToValue: _maxLtv,
            liquidationThreshold: _liquidationThreshold,
            liquidationBonus: _liquidationBonus,
            supplyCap: type(uint256).max,
            borrowCap: type(uint256).max,
            frozen: false
        });
        
        poolState = PoolState({
            totalSupplyAssets: 0,
            totalSupplyShares: 0,
            totalBorrowAssets: 0,
            totalBorrowShares: 0,
            borrowIndex: INITIAL_INDEX,
            supplyIndex: INITIAL_INDEX,
            lastAccrualTimestamp: block.timestamp
        });
    }
    
    // ============================================================
    //                    SUPPLY FUNCTIONS
    // ============================================================
    
    /// @notice Supply tokens เป็น liquidity
    function supply(uint256 amount) external nonReentrant returns (uint256 shares) {
        if (config.frozen) revert MarketFrozen();
        if (amount == 0) revert ZeroAmount();
        
        // Accrue interest ก่อนอัปเดต state
        _accrueInterest();
        
        // ตรวจสอบ supply cap
        if (poolState.totalSupplyAssets + amount > config.supplyCap) {
            revert SupplyCapExceeded(poolState.totalSupplyAssets, config.supplyCap);
        }
        
        // คำนวณ shares ที่ได้
        shares = _toSupplyShares(amount, poolState.totalSupplyAssets, poolState.totalSupplyShares);
        
        borrowToken.safeTransferFrom(msg.sender, address(this), amount);
        
        supplyPositions[msg.sender].shares += shares;
        poolState.totalSupplyAssets += amount;
        poolState.totalSupplyShares += shares;
        
        emit Supply(msg.sender, amount, shares);
    }
    
    /// @notice Withdraw assets
    function withdraw(uint256 shares) external nonReentrant returns (uint256 amount) {
        if (shares == 0) revert ZeroAmount();
        
        _accrueInterest();
        
        // คำนวณ amount จาก shares
        amount = _toSupplyAssets(shares, poolState.totalSupplyAssets, poolState.totalSupplyShares);
        
        // ตรวจสอบ liquidity เพียงพอ
        uint256 available = borrowToken.balanceOf(address(this));
        if (available < amount) revert InsufficientLiquidity(available, amount);
        
        supplyPositions[msg.sender].shares -= shares;
        poolState.totalSupplyAssets -= amount;
        poolState.totalSupplyShares -= shares;
        
        borrowToken.safeTransfer(msg.sender, amount);
        
        emit Withdraw(msg.sender, amount, shares);
    }
    
    // ============================================================
    //                  COLLATERAL MANAGEMENT
    // ============================================================
    
    /// @notice ฝาก collateral (ไม่ earn yield)
    function depositCollateral(uint256 amount) external nonReentrant {
        if (amount == 0) revert ZeroAmount();
        
        collateralToken.safeTransferFrom(msg.sender, address(this), amount);
        collateralBalances[msg.sender] += amount;
    }
    
    /// @notice ถอน collateral (ต้อง healthy หลังถอน)
    function withdrawCollateral(uint256 amount) external nonReentrant {
        if (amount == 0) revert ZeroAmount();
        
        _accrueInterest();
        
        uint256 borrowedValue = _getBorrowedValue(msg.sender);
        uint256 remainingCollateral = collateralBalances[msg.sender] - amount;
        uint256 collateralValue = _getCollateralValue(remainingCollateral);
        
        // ตรวจสอบว่า LTV ยัง valid
        if (borrowedValue > 0) {
            uint256 newLtv = (borrowedValue * BPS) / collateralValue;
            if (newLtv > config.maxLoanToValue) {
                revert MaxLoanToValueExceeded(newLtv, config.maxLoanToValue);
            }
        }
        
        collateralBalances[msg.sender] -= amount;
        collateralToken.safeTransfer(msg.sender, amount);
    }
    
    // ============================================================
    //                    BORROW FUNCTIONS
    // ============================================================
    
    /// @notice Borrow assets
    function borrow(uint256 amount) external nonReentrant {
        if (config.frozen) revert MarketFrozen();
        if (amount == 0) revert ZeroAmount();
        
        _accrueInterest();
        
        // ตรวจสอบ borrow cap
        if (poolState.totalBorrowAssets + amount > config.borrowCap) {
            revert BorrowCapExceeded(poolState.totalBorrowAssets, config.borrowCap);
        }
        
        // ตรวจสอบ collateral เพียงพอ
        uint256 newBorrowedValue = _getBorrowedValue(msg.sender) + 
            _toUSDValue(address(borrowToken), amount);
        uint256 collateralValue = _getCollateralValue(collateralBalances[msg.sender]);
        
        uint256 ltv = (newBorrowedValue * BPS) / collateralValue;
        if (ltv > config.maxLoanToValue) {
            revert MaxLoanToValueExceeded(ltv, config.maxLoanToValue);
        }
        
        // อัปเดต borrow position
        BorrowPosition storage pos = borrowPositions[msg.sender];
        
        // Update existing debt to current index
        if (pos.principal > 0) {
            pos.principal = _accruePositionInterest(pos.principal, pos.interestIndex);
        }
        
        pos.principal += amount;
        pos.interestIndex = poolState.borrowIndex;
        
        poolState.totalBorrowAssets += amount;
        
        // ตรวจสอบ liquidity
        uint256 available = borrowToken.balanceOf(address(this));
        if (available < amount) revert InsufficientLiquidity(available, amount);
        
        borrowToken.safeTransfer(msg.sender, amount);
        
        emit Borrow(msg.sender, amount);
    }
    
    /// @notice Repay debt
    function repay(address borrower, uint256 amount) external nonReentrant {
        if (amount == 0) revert ZeroAmount();
        
        _accrueInterest();
        
        BorrowPosition storage pos = borrowPositions[borrower];
        
        // คำนวณ current debt รวม interest
        uint256 currentDebt = _accruePositionInterest(pos.principal, pos.interestIndex);
        uint256 repayAmount = amount < currentDebt ? amount : currentDebt;
        
        borrowToken.safeTransferFrom(msg.sender, address(this), repayAmount);
        
        uint256 remaining = currentDebt - repayAmount;
        pos.principal = remaining;
        pos.interestIndex = poolState.borrowIndex;
        
        poolState.totalBorrowAssets -= repayAmount;
        
        emit Repay(borrower, repayAmount, remaining);
    }
    
    // ============================================================
    //                    LIQUIDATION
    // ============================================================
    
    /// @notice Liquidate unhealthy position
    /// @param borrower address ที่จะ liquidate
    /// @param repayAmount จำนวนที่ liquidator จะ repay
    function liquidate(
        address borrower,
        uint256 repayAmount
    ) external nonReentrant returns (uint256 collateralSeized) {
        _accrueInterest();
        
        // ตรวจสอบว่า position unhealthy
        uint256 healthFactor = getHealthFactor(borrower);
        if (healthFactor >= PRECISION) {
            revert PositionHealthy(borrower, healthFactor);
        }
        
        BorrowPosition storage pos = borrowPositions[borrower];
        uint256 currentDebt = _accruePositionInterest(pos.principal, pos.interestIndex);
        
        uint256 actualRepay = repayAmount < currentDebt ? repayAmount : currentDebt;
        
        // Repay debt
        borrowToken.safeTransferFrom(msg.sender, address(this), actualRepay);
        pos.principal = currentDebt - actualRepay;
        pos.interestIndex = poolState.borrowIndex;
        poolState.totalBorrowAssets -= actualRepay;
        
        // คำนวณ collateral ที่จะยึด
        // collateralSeized = repayValue * (1 + liquidationBonus) / collateralPrice
        uint256 repayValue = _toUSDValue(address(borrowToken), actualRepay);
        uint256 collateralPrice = oracle.getPrice(address(collateralToken), address(borrowToken));
        uint256 seizeValue = (repayValue * (BPS + config.liquidationBonus)) / BPS;
        collateralSeized = (seizeValue * PRECISION) / collateralPrice;
        
        // ไม่ยึดมากกว่าที่มี
        uint256 available = collateralBalances[borrower];
        if (collateralSeized > available) {
            collateralSeized = available;
        }
        
        collateralBalances[borrower] -= collateralSeized;
        collateralToken.safeTransfer(msg.sender, collateralSeized);
        
        emit Liquidate(msg.sender, borrower, actualRepay, collateralSeized);
    }
    
    // ============================================================
    //                    INTEREST ACCRUAL
    // ============================================================
    
    /// @notice Accrue interest บน supply และ borrow
    function _accrueInterest() internal {
        PoolState storage state = poolState;
        uint256 elapsed = block.timestamp - state.lastAccrualTimestamp;
        
        if (elapsed == 0) return;
        if (state.totalBorrowAssets == 0) {
            state.lastAccrualTimestamp = block.timestamp;
            return;
        }
        
        // คำนวณ utilization
        uint256 utilization = _getUtilization();
        
        // ดึง borrow rate จาก interest rate model
        uint256 borrowRatePerSec = interestRateModel.getBorrowRate(
            utilization,
            state.totalSupplyAssets,
            state.totalBorrowAssets
        );
        
        // คำนวณ interest
        // Interest = principal * rate * time
        uint256 interestFactor = borrowRatePerSec * elapsed;
        uint256 interest = (state.totalBorrowAssets * interestFactor) / PRECISION;
        
        // อัปเดต indexes
        if (state.totalBorrowAssets > 0) {
            state.borrowIndex += (state.borrowIndex * interestFactor) / PRECISION;
        }
        
        // Supply interest = borrow interest * (1 - reserve factor)
        uint256 reserveFactor = 1000; // 10% ไปที่ protocol
        uint256 supplyInterest = (interest * (BPS - reserveFactor)) / BPS;
        
        state.totalBorrowAssets += interest;
        state.totalSupplyAssets += supplyInterest;
        
        if (state.totalSupplyAssets > 0) {
            state.supplyIndex += (state.supplyIndex * supplyInterest) / state.totalSupplyAssets;
        }
        
        state.lastAccrualTimestamp = block.timestamp;
        
        emit InterestAccrued(state.borrowIndex, state.supplyIndex);
    }
    
    // ============================================================
    //                      VIEW FUNCTIONS
    // ============================================================
    
    /// @notice คำนวณ Health Factor
    /// @return hf Health Factor (1e18 = 1.0, < 1e18 = undercollateralized)
    function getHealthFactor(address borrower) public view returns (uint256 hf) {
        uint256 borrowedValue = _getBorrowedValue(borrower);
        if (borrowedValue == 0) return type(uint256).max;
        
        uint256 collateralValue = _getCollateralValue(collateralBalances[borrower]);
        uint256 adjustedCollateral = (collateralValue * config.liquidationThreshold) / BPS;
        
        hf = (adjustedCollateral * PRECISION) / borrowedValue;
    }
    
    /// @notice APY สำหรับ supply (annualized)
    function getSupplyApy() external view returns (uint256) {
        uint256 utilization = _getUtilization();
        uint256 borrowRate = interestRateModel.getBorrowRate(
            utilization,
            poolState.totalSupplyAssets,
            poolState.totalBorrowAssets
        );
        
        // Supply APY = Borrow APY * Utilization * (1 - Reserve Factor)
        return (borrowRate * 365 days * utilization / PRECISION) * 9000 / BPS;
    }
    
    function _getUtilization() internal view returns (uint256) {
        if (poolState.totalSupplyAssets == 0) return 0;
        return (poolState.totalBorrowAssets * PRECISION) / poolState.totalSupplyAssets;
    }
    
    function _getBorrowedValue(address borrower) internal view returns (uint256) {
        BorrowPosition memory pos = borrowPositions[borrower];
        if (pos.principal == 0) return 0;
        
        uint256 currentDebt = _accruePositionInterestView(pos.principal, pos.interestIndex);
        return _toUSDValue(address(borrowToken), currentDebt);
    }
    
    function _getCollateralValue(uint256 amount) internal view returns (uint256) {
        return _toUSDValue(address(collateralToken), amount);
    }
    
    function _toUSDValue(address token, uint256 amount) internal view returns (uint256) {
        uint256 price = oracle.getPrice(token, address(borrowToken));
        return (amount * price) / PRECISION;
    }
    
    function _accruePositionInterest(
        uint256 principal,
        uint256 positionIndex
    ) internal view returns (uint256) {
        if (positionIndex == 0) return principal;
        return (principal * poolState.borrowIndex) / positionIndex;
    }
    
    function _accruePositionInterestView(
        uint256 principal,
        uint256 positionIndex
    ) internal view returns (uint256) {
        if (positionIndex == 0) return principal;
        return (principal * poolState.borrowIndex) / positionIndex;
    }
    
    function _toSupplyShares(
        uint256 assets,
        uint256 totalAssets,
        uint256 totalShares
    ) internal pure returns (uint256) {
        if (totalAssets == 0 || totalShares == 0) return assets;
        return (assets * totalShares) / totalAssets;
    }
    
    function _toSupplyAssets(
        uint256 shares,
        uint256 totalAssets,
        uint256 totalShares
    ) internal pure returns (uint256) {
        if (totalShares == 0) return shares;
        return (shares * totalAssets) / totalShares;
    }
}

// Interfaces
interface IInterestRateModel {
    function getBorrowRate(uint256 utilization, uint256 totalSupply, uint256 totalBorrow) 
        external view returns (uint256);
}

interface IPriceOracle {
    function getPrice(address token, address quoteToken) external view returns (uint256);
}
```

---

## 2. JumpRate Interest Model

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title JumpRateModel - Interest Rate Model แบบ Compound V2 Kink-based
/// @notice Rate ปรับตาม utilization: ก่อน kink จะต่ำ หลัง kink จะสูงขึ้นชัน
/// @dev ใช้หน่วย per-second rate (สำหรับ precision สูงสุด)
contract JumpRateModel is IInterestRateModel {
    // ============================================================
    //                          EVENTS
    // ============================================================
    event NewInterestParams(
        uint256 baseRatePerYear,
        uint256 multiplierPerYear,
        uint256 jumpMultiplierPerYear,
        uint256 kink
    );
    
    // ============================================================
    //                          STORAGE
    // ============================================================
    
    /// @notice ผู้มีอำนาจ update parameters
    address public owner;
    
    /// @notice Seconds ในหนึ่งปี (approx)
    uint256 public constant SECONDS_PER_YEAR = 31_536_000;
    
    uint256 public constant PRECISION = 1e18;
    
    /// @notice Base rate per second (ต่ำสุดที่ utilization = 0)
    uint256 public baseRatePerSecond;
    
    /// @notice Rate ที่เพิ่มต่อ unit ของ utilization ก่อน kink
    uint256 public multiplierPerSecond;
    
    /// @notice Rate ที่เพิ่มชันมากหลัง kink
    uint256 public jumpMultiplierPerSecond;
    
    /// @notice Kink point: utilization ที่ rate เริ่ม "jump"
    /// @dev 0.8e18 = 80% utilization
    uint256 public kink;
    
    // ============================================================
    //                        CONSTRUCTOR
    // ============================================================
    
    /// @param baseRatePerYear Base APR (1e18 = 100%)
    /// @param multiplierPerYear Rate multiplier ก่อน kink
    /// @param jumpMultiplierPerYear Rate multiplier หลัง kink
    /// @param kink_ Utilization ที่ jump เริ่มต้น (0-1e18)
    constructor(
        uint256 baseRatePerYear,
        uint256 multiplierPerYear,
        uint256 jumpMultiplierPerYear,
        uint256 kink_
    ) {
        owner = msg.sender;
        _updateJumpRateModel(
            baseRatePerYear,
            multiplierPerYear,
            jumpMultiplierPerYear,
            kink_
        );
    }
    
    // ============================================================
    //                    RATE CALCULATIONS
    // ============================================================
    
    /// @notice คำนวณ Borrow Rate Per Second
    /// @param utilization Utilization ratio (0-1e18)
    /// @param totalSupply ไม่ใช้ใน JumpRate แต่ต้องมีตาม interface
    /// @param totalBorrow ไม่ใช้ใน JumpRate
    /// @return rate Borrow rate per second (1e18 = 100% per second)
    function getBorrowRate(
        uint256 utilization,
        uint256 totalSupply,
        uint256 totalBorrow
    ) external view override returns (uint256 rate) {
        return _getBorrowRate(utilization);
    }
    
    /// @notice คำนวณ Borrow APR (annualized)
    function getBorrowApr(uint256 utilization) external view returns (uint256) {
        return _getBorrowRate(utilization) * SECONDS_PER_YEAR;
    }
    
    /// @notice คำนวณ Supply APR
    function getSupplyApr(uint256 utilization) external view returns (uint256) {
        uint256 borrowApr = _getBorrowRate(utilization) * SECONDS_PER_YEAR;
        // Supply APR = Borrow APR * Utilization (simple model ไม่มี reserve factor)
        return (borrowApr * utilization) / PRECISION;
    }
    
    // ============================================================
    //                    INTERNAL FUNCTIONS
    // ============================================================
    
    /// @notice Core rate calculation
    /// @dev 
    ///   ถ้า utilization ≤ kink:
    ///     rate = baseRate + multiplier * utilization
    ///   ถ้า utilization > kink:
    ///     rate = normalRate(kink) + jumpMultiplier * (utilization - kink)
    function _getBorrowRate(uint256 utilization) internal view returns (uint256) {
        if (utilization <= kink) {
            // Linear section: ต่ำกว่า kink
            return baseRatePerSecond + (multiplierPerSecond * utilization) / PRECISION;
        } else {
            // Jump section: สูงกว่า kink
            uint256 normalRate = baseRatePerSecond + (multiplierPerSecond * kink) / PRECISION;
            uint256 excessUtil = utilization - kink;
            return normalRate + (jumpMultiplierPerSecond * excessUtil) / PRECISION;
        }
    }
    
    // ============================================================
    //                    ADMIN FUNCTIONS
    // ============================================================
    
    /// @notice Update rate model parameters
    function updateJumpRateModel(
        uint256 baseRatePerYear,
        uint256 multiplierPerYear,
        uint256 jumpMultiplierPerYear,
        uint256 kink_
    ) external {
        require(msg.sender == owner, "Not owner");
        _updateJumpRateModel(baseRatePerYear, multiplierPerYear, jumpMultiplierPerYear, kink_);
    }
    
    function _updateJumpRateModel(
        uint256 baseRatePerYear,
        uint256 multiplierPerYear,
        uint256 jumpMultiplierPerYear,
        uint256 kink_
    ) internal {
        require(kink_ <= PRECISION, "Kink must be <= 1");
        
        baseRatePerSecond = baseRatePerYear / SECONDS_PER_YEAR;
        multiplierPerSecond = multiplierPerYear / SECONDS_PER_YEAR;
        jumpMultiplierPerSecond = jumpMultiplierPerYear / SECONDS_PER_YEAR;
        kink = kink_;
        
        emit NewInterestParams(baseRatePerYear, multiplierPerYear, jumpMultiplierPerYear, kink_);
    }
    
    // ============================================================
    //                    UTIL FUNCTIONS
    // ============================================================
    
    /// @notice Visualize rate curve สำหรับ utilization points ต่างๆ
    function getRateCurve() external view returns (
        uint256[] memory utilizations,
        uint256[] memory borrowAprs,
        uint256[] memory supplyAprs
    ) {
        uint256 points = 11;
        utilizations = new uint256[](points);
        borrowAprs = new uint256[](points);
        supplyAprs = new uint256[](points);
        
        for (uint256 i = 0; i < points; i++) {
            uint256 util = (i * PRECISION) / (points - 1);
            utilizations[i] = util;
            uint256 borrowRate = _getBorrowRate(util);
            borrowAprs[i] = borrowRate * SECONDS_PER_YEAR;
            supplyAprs[i] = (borrowAprs[i] * util) / PRECISION;
        }
    }
}
```

---

## 3. Dutch Auction Liquidation Engine

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title DutchAuctionLiquidationEngine - Liquidation แบบ Dutch Auction
/// @notice ราคา collateral เริ่มสูง แล้วลดลงตามเวลา จนมีคนมาซื้อ
/// @dev เป็น mechanism ที่ดีกว่า fixed-bonus liquidation เพราะ:
///   - เปิดโอกาสให้ liquidators ที่มี capital น้อยเข้าร่วม
///   - ลด bad debt ที่เกิดจาก over-liquidation
///   - Market determines fair price
contract DutchAuctionLiquidationEngine is ReentrancyGuard {
    using SafeERC20 for IERC20;
    
    // ============================================================
    //                          ERRORS
    // ============================================================
    error AuctionNotFound(uint256 auctionId);
    error AuctionAlreadySettled(uint256 auctionId);
    error AuctionExpired(uint256 auctionId);
    error AuctionStillActive(uint256 auctionId);
    error PositionStillHealthy(address borrower, uint256 healthFactor);
    error InsufficientBid(uint256 bid, uint256 currentPrice);
    error InvalidAuction();
    
    // ============================================================
    //                          EVENTS
    // ============================================================
    event AuctionStarted(
        uint256 indexed auctionId,
        address indexed borrower,
        address collateralToken,
        address debtToken,
        uint256 collateralAmount,
        uint256 debtAmount,
        uint256 startPrice,
        uint256 endPrice,
        uint256 auctionEnd
    );
    event AuctionSettled(
        uint256 indexed auctionId,
        address indexed buyer,
        uint256 collateralReceived,
        uint256 debtPaid,
        uint256 discount
    );
    event AuctionExpiredResolved(uint256 indexed auctionId, uint256 badDebt);
    
    // ============================================================
    //                          STRUCTS
    // ============================================================
    
    struct Auction {
        address borrower;
        address collateralToken;
        address debtToken;
        address lendingPool;        // Pool ที่ trigger auction
        
        uint256 collateralAmount;   // Collateral ที่ auction
        uint256 debtAmount;         // Debt ที่ต้อง repay
        
        uint256 startPrice;         // เริ่มต้นที่ราคาพรีเมียม (เช่น 110% ของมูลค่าจริง)
        uint256 endPrice;           // ราคาต่ำสุดที่ยอมรับได้ (เช่น 85%)
        
        uint256 startTime;          // เวลาเริ่ม auction
        uint256 endTime;            // เวลาสิ้นสุด (ถ้าไม่มีคนซื้อ = bad debt)
        
        bool settled;               // auction จบแล้วหรือยัง
    }
    
    // ============================================================
    //                          STORAGE
    // ============================================================
    
    mapping(uint256 => Auction) public auctions;
    uint256 private _nextAuctionId;
    
    IPriceOracle public oracle;
    ILendingPoolHub public lendingHub;  // Hub ที่รู้จัก pools ทั้งหมด
    
    /// @notice ระยะเวลา auction (เช่น 1 hour)
    uint256 public constant AUCTION_DURATION = 1 hours;
    
    /// @notice เริ่ม auction ที่ premium นี้ (bps above market)
    uint256 public constant START_PREMIUM = 1000;  // 10% above market
    
    /// @notice สิ้นสุด auction ที่ discount นี้ (bps below market)
    uint256 public constant END_DISCOUNT = 2000;   // 20% below market
    
    uint256 public constant BPS = 10_000;
    uint256 public constant PRECISION = 1e18;
    
    // ============================================================
    //                        CONSTRUCTOR
    // ============================================================
    
    constructor(address _oracle, address _lendingHub) {
        oracle = IPriceOracle(_oracle);
        lendingHub = ILendingPoolHub(_lendingHub);
    }
    
    // ============================================================
    //                    AUCTION MANAGEMENT
    // ============================================================
    
    /// @notice Start Dutch Auction สำหรับ unhealthy position
    function startAuction(
        address borrower,
        address lendingPool
    ) external returns (uint256 auctionId) {
        // ตรวจสอบ health factor
        uint256 hf = IIsolatedLendingPool(lendingPool).getHealthFactor(borrower);
        if (hf >= PRECISION) revert PositionStillHealthy(borrower, hf);
        
        // ดึง position info จาก pool
        (
            address collateral,
            address debt,
            uint256 collateralAmt,
            uint256 debtAmt
        ) = IIsolatedLendingPool(lendingPool).getPosition(borrower);
        
        uint256 collateralValue = oracle.getPrice(collateral, debt) * collateralAmt / PRECISION;
        
        // คำนวณ start/end price
        uint256 startPrice = (debtAmt * (BPS + START_PREMIUM)) / BPS;  // 110% of debt
        uint256 endPrice = (debtAmt * (BPS - END_DISCOUNT)) / BPS;     // 80% of debt (min price)
        
        // ถ้า collateral value < end price → immediate bad debt
        if (collateralValue < endPrice) {
            endPrice = collateralValue;
        }
        
        auctionId = _nextAuctionId++;
        
        auctions[auctionId] = Auction({
            borrower: borrower,
            collateralToken: collateral,
            debtToken: debt,
            lendingPool: lendingPool,
            collateralAmount: collateralAmt,
            debtAmount: debtAmt,
            startPrice: startPrice,
            endPrice: endPrice,
            startTime: block.timestamp,
            endTime: block.timestamp + AUCTION_DURATION,
            settled: false
        });
        
        emit AuctionStarted(
            auctionId,
            borrower,
            collateral,
            debt,
            collateralAmt,
            debtAmt,
            startPrice,
            endPrice,
            block.timestamp + AUCTION_DURATION
        );
    }
    
    /// @notice Bid ใน Dutch Auction
    /// @dev Buyer จ่าย current price ใน debt token และรับ collateral ทั้งหมด
    function bid(uint256 auctionId) external nonReentrant {
        Auction storage auction = auctions[auctionId];
        
        if (auction.startTime == 0) revert AuctionNotFound(auctionId);
        if (auction.settled) revert AuctionAlreadySettled(auctionId);
        if (block.timestamp > auction.endTime) revert AuctionExpired(auctionId);
        
        // คำนวณ current price
        uint256 currentPrice = getCurrentPrice(auctionId);
        
        // Buyer จ่าย current price ของ debt
        IERC20(auction.debtToken).safeTransferFrom(msg.sender, address(this), currentPrice);
        
        // Repay debt ผ่าน lending pool
        IERC20(auction.debtToken).safeApprove(auction.lendingPool, auction.debtAmount);
        
        IIsolatedLendingPool(auction.lendingPool).repay(auction.borrower, auction.debtAmount);
        
        // ส่ง collateral ให้ buyer
        IIsolatedLendingPool(auction.lendingPool).seizeCollateral(
            auction.borrower,
            msg.sender,
            auction.collateralAmount
        );
        
        auction.settled = true;
        
        uint256 discount = auction.debtAmount > currentPrice 
            ? ((auction.debtAmount - currentPrice) * BPS) / auction.debtAmount
            : 0;
        
        emit AuctionSettled(
            auctionId,
            msg.sender,
            auction.collateralAmount,
            currentPrice,
            discount
        );
    }
    
    /// @notice Resolve expired auction (bad debt)
    function resolveExpiredAuction(uint256 auctionId) external {
        Auction storage auction = auctions[auctionId];
        
        if (auction.startTime == 0) revert AuctionNotFound(auctionId);
        if (auction.settled) revert AuctionAlreadySettled(auctionId);
        if (block.timestamp <= auction.endTime) revert AuctionStillActive(auctionId);
        
        // Auction expired โดยไม่มีคนซื้อ → Bad Debt
        uint256 collateralValue = oracle.getPrice(
            auction.collateralToken, 
            auction.debtToken
        ) * auction.collateralAmount / PRECISION;
        
        uint256 badDebt = auction.debtAmount > collateralValue 
            ? auction.debtAmount - collateralValue 
            : 0;
        
        // บอก lending pool ให้ socialize bad debt
        IIsolatedLendingPool(auction.lendingPool).socializeBadDebt(
            auction.borrower,
            badDebt,
            auction.collateralAmount
        );
        
        auction.settled = true;
        
        emit AuctionExpiredResolved(auctionId, badDebt);
    }
    
    // ============================================================
    //                      VIEW FUNCTIONS
    // ============================================================
    
    /// @notice คำนวณราคา ณ ปัจจุบันของ auction
    /// @dev ราคาลดจาก startPrice ลงเป็น endPrice แบบ linear ตาม time
    function getCurrentPrice(uint256 auctionId) public view returns (uint256) {
        Auction storage auction = auctions[auctionId];
        
        if (block.timestamp >= auction.endTime) return auction.endPrice;
        if (block.timestamp <= auction.startTime) return auction.startPrice;
        
        uint256 elapsed = block.timestamp - auction.startTime;
        uint256 totalDuration = auction.endTime - auction.startTime;
        
        // Linear interpolation
        uint256 priceDrop = auction.startPrice - auction.endPrice;
        uint256 currentDrop = (priceDrop * elapsed) / totalDuration;
        
        return auction.startPrice - currentDrop;
    }
}

// Extended interfaces
interface IIsolatedLendingPool {
    function getHealthFactor(address borrower) external view returns (uint256);
    function getPosition(address borrower) external view returns (
        address collateral, address debt, uint256 collateralAmt, uint256 debtAmt
    );
    function repay(address borrower, uint256 amount) external;
    function seizeCollateral(address borrower, address recipient, uint256 amount) external;
    function socializeBadDebt(address borrower, uint256 badDebt, uint256 collateral) external;
}

interface ILendingPoolHub {
    function isPool(address pool) external view returns (bool);
}
```

---

## 4. Bad Debt Socialization + Morpho P2P Matching

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title MorphoStyleLendingOptimizer - P2P Matching สำหรับ better rates
/// @notice Match lenders กับ borrowers โดยตรง bypass pool spread
/// @dev Inspired by Morpho Blue architecture
contract MorphoStyleLendingOptimizer is ReentrancyGuard {
    using SafeERC20 for IERC20;
    
    // ============================================================
    //                          ERRORS
    // ============================================================
    error NoMatchFound();
    error InvalidAmount();
    error MatchAlreadyExists();
    error MatchNotFound();
    error P2PRateNotBetter(uint256 p2pRate, uint256 poolRate);
    error Unauthorized();
    
    // ============================================================
    //                          EVENTS
    // ============================================================
    event P2PMatchCreated(
        uint256 indexed matchId,
        address indexed lender,
        address indexed borrower,
        uint256 amount,
        uint256 p2pRate
    );
    event P2PMatchClosed(uint256 indexed matchId, uint256 elapsed, uint256 interest);
    event BadDebtSocialized(address indexed pool, uint256 amount, uint256 perShareLoss);
    event SupplyQueueUpdated(address[] newQueue);
    
    // ============================================================
    //                          STRUCTS
    // ============================================================
    
    struct P2PMatch {
        address lender;
        address borrower;
        address asset;
        uint256 principal;
        uint256 p2pRate;        // agreed rate (per second, 1e18)
        uint256 startTime;
        uint256 lastAccrual;
        uint256 borrowerCollateral;
        bool active;
    }
    
    struct PoolMetrics {
        uint256 supplyRate;     // Pool supply APR
        uint256 borrowRate;     // Pool borrow APR
        address poolContract;
    }
    
    /// @notice Bad debt accounting
    struct BadDebtState {
        uint256 totalBadDebt;           // รวม bad debt ที่ต้องกระจาย
        uint256 badDebtPerShare;        // bad debt ต่อ share (accumulated)
        mapping(address => uint256) lastBadDebtIndex;  // index ที่ user รับผิดชอบล่าสุด
    }
    
    // ============================================================
    //                          STORAGE
    // ============================================================
    
    mapping(uint256 => P2PMatch) public p2pMatches;
    uint256 private _nextMatchId;
    
    mapping(address => PoolMetrics) public poolMetrics;
    mapping(address => BadDebtState) private _badDebtStates;
    
    /// @notice Supply queue: sorted list ของ lenders ที่รอ match
    mapping(address => address[]) public supplyQueues;  // asset => lenders
    mapping(address => uint256) public pendingLenderAmounts;  // lender => unmatched amount
    
    /// @notice Borrow queue: sorted list ของ borrowers ที่รอ match
    mapping(address => address[]) public borrowQueues;  // asset => borrowers
    mapping(address => uint256) public pendingBorrowerAmounts;
    
    uint256 public constant PRECISION = 1e18;
    uint256 public constant BPS = 10_000;
    
    // ============================================================
    //                    P2P MATCHING ENGINE
    // ============================================================
    
    /// @notice ผู้ให้กู้ register ใน supply queue
    function supplyP2P(address asset, uint256 amount) external nonReentrant {
        IERC20(asset).safeTransferFrom(msg.sender, address(this), amount);
        
        pendingLenderAmounts[msg.sender] += amount;
        supplyQueues[asset].push(msg.sender);
        
        // ลอง match ทันที
        _tryMatch(asset);
    }
    
    /// @notice ผู้กู้ request กู้ผ่าน P2P
    function borrowP2P(
        address asset,
        uint256 amount,
        address collateralToken,
        uint256 collateralAmount
    ) external nonReentrant {
        // ตรวจสอบ collateral
        require(collateralAmount > 0, "No collateral");
        
        IERC20(collateralToken).safeTransferFrom(msg.sender, address(this), collateralAmount);
        
        pendingBorrowerAmounts[msg.sender] += amount;
        borrowQueues[asset].push(msg.sender);
        
        // ลอง match ทันที
        _tryMatch(asset);
    }
    
    /// @notice Engine matching: จับคู่ lenders กับ borrowers
    function _tryMatch(address asset) internal {
        PoolMetrics memory metrics = poolMetrics[asset];
        
        address[] storage lenders = supplyQueues[asset];
        address[] storage borrowers = borrowQueues[asset];
        
        if (lenders.length == 0 || borrowers.length == 0) return;
        
        // คำนวณ P2P rate = midpoint ระหว่าง supply rate และ borrow rate
        // Lender ได้มากกว่า pool supply rate
        // Borrower จ่ายน้อยกว่า pool borrow rate
        uint256 p2pRate = (metrics.supplyRate + metrics.borrowRate) / 2;
        
        // ตรวจสอบว่า P2P rate ดีกว่า pool สำหรับทั้งสองฝ่าย
        require(p2pRate > metrics.supplyRate, "P2P not better for lenders");
        require(p2pRate < metrics.borrowRate, "P2P not better for borrowers");
        
        // Match lender คนแรกกับ borrower คนแรก (FIFO)
        address lender = lenders[0];
        address borrower = borrowers[0];
        
        uint256 lenderAmount = pendingLenderAmounts[lender];
        uint256 borrowerAmount = pendingBorrowerAmounts[borrower];
        
        uint256 matchAmount = lenderAmount < borrowerAmount ? lenderAmount : borrowerAmount;
        
        if (matchAmount == 0) return;
        
        // สร้าง P2P match
        uint256 matchId = _nextMatchId++;
        
        p2pMatches[matchId] = P2PMatch({
            lender: lender,
            borrower: borrower,
            asset: asset,
            principal: matchAmount,
            p2pRate: p2pRate,
            startTime: block.timestamp,
            lastAccrual: block.timestamp,
            borrowerCollateral: 0, // ดึงจาก collateral balance
            active: true
        });
        
        // อัปเดต pending amounts
        pendingLenderAmounts[lender] -= matchAmount;
        pendingBorrowerAmounts[borrower] -= matchAmount;
        
        // ส่ง asset ให้ borrower
        IERC20(asset).safeTransfer(borrower, matchAmount);
        
        // ลบออกจาก queues ถ้า fully matched
        if (pendingLenderAmounts[lender] == 0) {
            _removeFromQueue(lenders, lender);
        }
        if (pendingBorrowerAmounts[borrower] == 0) {
            _removeFromQueue(borrowers, borrower);
        }
        
        emit P2PMatchCreated(matchId, lender, borrower, matchAmount, p2pRate);
    }
    
    // ============================================================
    //                    BAD DEBT SOCIALIZATION
    // ============================================================
    
    /// @notice Socialize bad debt - กระจายให้ suppliers ทั้งหมดรับผิดชอบ
    /// @dev เรียกโดย liquidation engine หลัง auction expired
    function socializeBadDebt(
        address asset,
        uint256 badDebtAmount,
        uint256 totalSupplyShares
    ) external {
        // ในกรณีจริง: เรียกโดย authorized liquidation engine เท่านั้น
        
        if (totalSupplyShares == 0 || badDebtAmount == 0) return;
        
        BadDebtState storage state = _badDebtStates[asset];
        
        // คำนวณ bad debt per share
        uint256 lossPerShare = (badDebtAmount * PRECISION) / totalSupplyShares;
        
        state.totalBadDebt += badDebtAmount;
        state.badDebtPerShare += lossPerShare;
        
        emit BadDebtSocialized(asset, badDebtAmount, lossPerShare);
    }
    
    /// @notice คำนวณ loss ของ supplier จาก bad debt
    function getUserBadDebtLoss(
        address user,
        address asset,
        uint256 userShares
    ) external view returns (uint256 loss) {
        BadDebtState storage state = _badDebtStates[asset];
        
        uint256 userLastIndex = state.lastBadDebtIndex[user];
        uint256 currentIndex = state.badDebtPerShare;
        
        if (currentIndex <= userLastIndex) return 0;
        
        uint256 deltaIndex = currentIndex - userLastIndex;
        loss = (userShares * deltaIndex) / PRECISION;
    }
    
    /// @notice Claim bad debt loss (ลด shares ของ user)
    function acknowledgeBadDebt(address user, address asset) external {
        BadDebtState storage state = _badDebtStates[asset];
        state.lastBadDebtIndex[user] = state.badDebtPerShare;
    }
    
    // ============================================================
    //                      INTERNAL HELPERS
    // ============================================================
    
    function _removeFromQueue(address[] storage queue, address target) internal {
        for (uint256 i = 0; i < queue.length; i++) {
            if (queue[i] == target) {
                queue[i] = queue[queue.length - 1];
                queue.pop();
                return;
            }
        }
    }
    
    // ============================================================
    //                    POOL MANAGEMENT
    // ============================================================
    
    function updatePoolMetrics(
        address asset,
        address pool,
        uint256 supplyRate,
        uint256 borrowRate
    ) external {
        poolMetrics[asset] = PoolMetrics({
            supplyRate: supplyRate,
            borrowRate: borrowRate,
            poolContract: pool
        });
    }
}
```

---

## 5. Complete Integration - Advanced Lending System

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title LendingSystemFactory - สร้างและจัดการ isolated pools
/// @notice Central factory สำหรับ isolated lending system
contract LendingSystemFactory is Ownable {
    // ============================================================
    //                          EVENTS
    // ============================================================
    event PoolCreated(
        address indexed pool,
        address indexed collateral,
        address indexed borrow,
        bytes32 poolId
    );
    event PoolPaused(bytes32 indexed poolId);
    event PoolResumed(bytes32 indexed poolId);
    
    // ============================================================
    //                          STORAGE
    // ============================================================
    
    struct PoolRecord {
        address pool;
        address collateralToken;
        address borrowToken;
        bool active;
        uint256 createdAt;
    }
    
    mapping(bytes32 => PoolRecord) public pools;
    bytes32[] public poolIds;
    
    address public immutable rateModelTemplate;
    address public immutable poolTemplate;
    address public oracle;
    
    uint256 public defaultMaxLtv = 7500;          // 75%
    uint256 public defaultLiqThreshold = 8000;    // 80%
    uint256 public defaultLiqBonus = 500;         // 5%
    
    constructor(
        address _oracle,
        address _rateModelTemplate,
        address _poolTemplate,
        address _owner
    ) Ownable(_owner) {
        oracle = _oracle;
        rateModelTemplate = _rateModelTemplate;
        poolTemplate = _poolTemplate;
    }
    
    /// @notice สร้าง isolated pool ใหม่สำหรับ collateral-borrow pair
    function createPool(
        address collateral,
        address borrow,
        uint256 baseRatePerYear,
        uint256 multiplierPerYear,
        uint256 jumpMultiplierPerYear,
        uint256 kink
    ) external onlyOwner returns (bytes32 poolId, address poolAddress) {
        poolId = keccak256(abi.encodePacked(collateral, borrow));
        require(pools[poolId].pool == address(0), "Pool exists");
        
        // Deploy rate model
        address rateModel = address(new JumpRateModel(
            baseRatePerYear,
            multiplierPerYear,
            jumpMultiplierPerYear,
            kink
        ));
        
        // Deploy isolated pool
        poolAddress = address(new IsolatedLendingPool(
            collateral,
            borrow,
            oracle,
            rateModel,
            defaultMaxLtv,
            defaultLiqThreshold,
            defaultLiqBonus
        ));
        
        pools[poolId] = PoolRecord({
            pool: poolAddress,
            collateralToken: collateral,
            borrowToken: borrow,
            active: true,
            createdAt: block.timestamp
        });
        
        poolIds.push(poolId);
        
        emit PoolCreated(poolAddress, collateral, borrow, poolId);
    }
    
    function getPool(address collateral, address borrow) external view returns (address) {
        bytes32 poolId = keccak256(abi.encodePacked(collateral, borrow));
        return pools[poolId].pool;
    }
    
    function getAllPools() external view returns (bytes32[] memory) {
        return poolIds;
    }
    
    function pausePool(bytes32 poolId) external onlyOwner {
        pools[poolId].active = false;
        emit PoolPaused(poolId);
    }
}
```

---

## Workshop / แบบฝึกหัด

### แบบฝึกหัดที่ 1: Multi-Collateral Pool

**โจทย์**: ปรับ `IsolatedLendingPool` ให้รองรับ collateral หลายชนิด:
1. Borrower สามารถ deposit WETH + WBTC เป็น collateral
2. LTV คำนวณจาก weighted sum
3. ถ้า ETH ราคาตก แต่ BTC ราคาขึ้น → อาจยัง healthy

### แบบฝึกหัดที่ 2: Risk Oracle Integration

**โจทย์**: สร้าง `RiskOracle` ที่:
1. ดึงราคาจาก Chainlink + Uniswap TWAP
2. ใช้ min ของทั้งสอง (conservative)
3. Circuit breaker ถ้า deviation > 5% ระหว่าง sources

```solidity
contract RiskOracle {
    IChainlinkFeed public chainlink;
    IUniswapTWAP public uniswap;
    
    uint256 public constant MAX_DEVIATION = 500; // 5%
    uint256 public constant STALENESS_THRESHOLD = 1 hours;
    
    error PriceStale(address token, uint256 lastUpdate);
    error PriceDeviation(uint256 chainlinkPrice, uint256 twapPrice, uint256 deviation);
    
    // TODO: Implement getPrice(address token) 
    // TODO: ตรวจสอบ staleness
    // TODO: ตรวจสอบ deviation
    // TODO: Return min price (conservative)
}
```

### แบบฝึกหัดที่ 3: Keeper Network Integration

**โจทย์**: สร้าง `LiquidationKeeper` ที่:
1. Monitor unhealthy positions
2. Calculate profitability ก่อน liquidate
3. Flash loan เพื่อ liquidate โดยไม่ต้องมี capital
4. ส่ง profit ให้ caller

```solidity
contract LiquidationKeeper {
    DutchAuctionLiquidationEngine public engine;
    IFlashLender public flashLender;
    
    function checkLiquidation(
        address borrower,
        address pool
    ) external view returns (bool shouldLiquidate, uint256 expectedProfit) {
        // TODO: ดึง health factor
        // TODO: คำนวณ auction price ปัจจุบัน
        // TODO: คำนวณ profit = collateral value - debt repaid - gas cost
    }
    
    function executeLiquidation(
        address borrower,
        address pool
    ) external {
        // TODO: Flash borrow debt token
        // TODO: Bid ใน auction
        // TODO: Sell collateral บน DEX
        // TODO: Repay flash loan
        // TODO: ส่งกำไรให้ caller
    }
}
```

---

## สรุป Part 55

- **Isolated Pools** แยกความเสี่ยงต่อ collateral pair ลด systemic risk แต่ fragmentation สูงขึ้น
- **Borrow Index** ใช้ track interest ทั่วทั้ง pool โดยไม่ต้อง update ทุก position ทุก block
- **JumpRateModel** ใช้ kink เพื่อ incentivize optimal utilization (~80%) โดยทำให้ rate พุ่งสูงมากเกินกว่า kink
- **Dutch Auction Liquidation** แก้ปัญหา MEV/front-running ใน liquidation โดยเปิดโอกาสให้ market หา fair price
- ราคา Dutch auction ลดลง linear จาก startPrice → endPrice ตาม duration
- **Bad Debt Socialization** กระจายขาดทุนให้ suppliers ทั้งหมดตาม shares เมื่อ collateral ไม่พอ cover debt
- **Morpho P2P Matching** bypass pool spread โดย match lenders-borrowers โดยตรงในอัตราที่ดีกว่า
- P2P rate = midpoint ระหว่าง supply rate และ borrow rate → lender ได้ yield สูงกว่า, borrower จ่ายน้อยกว่า
- Supply/Borrow queues ต้องจัดการ efficiently (sorted by amount, FIFO/priority-based)

## Next: Part 56 - Cross-Chain Protocols & Bridges
