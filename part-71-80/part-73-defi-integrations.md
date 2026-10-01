# Part 73: DeFi Protocol Integrations

## บทนำ

ในโลกของ DeFi (Decentralized Finance) โปรโตคอลต่างๆ สามารถทำงานร่วมกันได้อย่างไร้รอยต่อ ผ่านการใช้ interfaces มาตรฐาน บทนี้จะครอบคลุมการ integrate กับโปรโตคอล DeFi ชั้นนำ ได้แก่ Aave V3, Compound V3 (Comet), Uniswap V3, และ Curve Finance พร้อมทั้งแนวทางความปลอดภัยในการ integrate

---

## 1. Aave V3 Integration

### 1.1 ทำความรู้จัก Aave V3

Aave V3 เป็น lending/borrowing protocol ที่ได้รับความนิยมสูงสุดใน DeFi Aave V3 นำเสนอ:
- **Efficiency Mode (eMode)**: ช่วยเพิ่ม capital efficiency สำหรับ correlated assets
- **Isolation Mode**: จำกัดความเสี่ยงสำหรับ assets ใหม่
- **Cross-Chain Portals**: รองรับ cross-chain liquidity
- **Flash Loans**: กู้เงินไม่มีหลักประกันภายใน 1 transaction

### 1.2 IPool Interface

`IPool` คือ interface หลักของ Aave V3 ที่ smart contract อื่นๆ ใช้ในการ interact

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// Interface สำหรับ Aave V3 Pool
interface IPool {
    // ข้อมูลของ Reserve แต่ละ asset
    struct ReserveData {
        ReserveConfigurationMap configuration;
        uint128 liquidityIndex;
        uint128 currentLiquidityRate;
        uint128 variableBorrowIndex;
        uint128 currentVariableBorrowRate;
        uint128 currentStableBorrowRate;
        uint40 lastUpdateTimestamp;
        uint16 id;
        address aTokenAddress;
        address stableDebtTokenAddress;
        address variableDebtTokenAddress;
        address interestRateStrategyAddress;
        uint128 accruedToTreasury;
        uint128 unbacked;
        uint128 isolationModeTotalDebt;
    }

    struct ReserveConfigurationMap {
        uint256 data;
    }

    // Events
    event Supply(
        address indexed reserve,
        address user,
        address indexed onBehalfOf,
        uint256 amount,
        uint16 indexed referralCode
    );

    event Withdraw(
        address indexed reserve,
        address indexed user,
        address indexed to,
        uint256 amount
    );

    event Borrow(
        address indexed reserve,
        address user,
        address indexed onBehalfOf,
        uint256 amount,
        DataTypes.InterestRateMode interestRateMode,
        uint256 borrowRate,
        uint16 indexed referralCode
    );

    event Repay(
        address indexed reserve,
        address indexed user,
        address indexed repayer,
        uint256 amount,
        bool useATokens
    );

    event LiquidationCall(
        address indexed collateralAsset,
        address indexed debtAsset,
        address indexed user,
        uint256 debtToCover,
        uint256 liquidatedCollateralAmount,
        address liquidator,
        bool receiveAToken
    );

    event FlashLoan(
        address indexed target,
        address initiator,
        address indexed asset,
        uint256 amount,
        DataTypes.InterestRateMode interestRateMode,
        uint256 premium,
        uint16 indexed referralCode
    );

    // Functions หลัก
    function supply(
        address asset,
        uint256 amount,
        address onBehalfOf,
        uint16 referralCode
    ) external;

    function withdraw(
        address asset,
        uint256 amount,
        address to
    ) external returns (uint256);

    function borrow(
        address asset,
        uint256 amount,
        uint256 interestRateMode,
        uint16 referralCode,
        address onBehalfOf
    ) external;

    function repay(
        address asset,
        uint256 amount,
        uint256 interestRateMode,
        address onBehalfOf
    ) external returns (uint256);

    function liquidationCall(
        address collateralAsset,
        address debtAsset,
        address user,
        uint256 debtToCover,
        bool receiveAToken
    ) external;

    function flashLoanSimple(
        address receiverAddress,
        address asset,
        uint256 amount,
        bytes calldata params,
        uint16 referralCode
    ) external;

    function getUserAccountData(address user)
        external
        view
        returns (
            uint256 totalCollateralBase,
            uint256 totalDebtBase,
            uint256 availableBorrowsBase,
            uint256 currentLiquidationThreshold,
            uint256 ltv,
            uint256 healthFactor
        );

    function getReserveData(address asset)
        external
        view
        returns (ReserveData memory);
}

library DataTypes {
    enum InterestRateMode {
        NONE,
        STABLE,
        VARIABLE
    }
}
```

### 1.3 Flash Loan Receiver Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// Interface สำหรับรับ Flash Loan จาก Aave V3
interface IFlashLoanSimpleReceiver {
    /**
     * @notice ฟังก์ชัน callback ที่ Aave Pool จะเรียกหลังส่ง flash loan
     * @param asset address ของ asset ที่กู้
     * @param amount จำนวนที่กู้
     * @param premium ค่าธรรมเนียม (0.05% ใน Aave V3)
     * @param initiator address ที่เริ่ม flash loan
     * @param params ข้อมูลเพิ่มเติมที่ encode มา
     * @return true ถ้าทำสำเร็จ
     */
    function executeOperation(
        address asset,
        uint256 amount,
        uint256 premium,
        address initiator,
        bytes calldata params
    ) external returns (bool);

    function ADDRESSES_PROVIDER() external view returns (address);
    function POOL() external view returns (address);
}
```

### 1.4 AaveV3Integration Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title AaveV3Integration
 * @notice ตัวอย่างการ integrate กับ Aave V3 Pool
 * @dev รองรับ supply, borrow, repay, liquidation และ flash loans
 */
contract AaveV3Integration is IFlashLoanSimpleReceiver, Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ================== Constants ==================
    uint256 public constant INTEREST_RATE_MODE_VARIABLE = 2;
    uint16 public constant REFERRAL_CODE = 0;

    // ================== State Variables ==================
    IPool public immutable pool;
    address public immutable addressesProvider;

    // Track positions
    mapping(address => uint256) public suppliedAmounts;
    mapping(address => uint256) public borrowedAmounts;

    // Flash loan callback validation
    mapping(bytes32 => bool) private pendingFlashLoans;

    // ================== Events ==================
    event Supplied(address indexed asset, uint256 amount, address indexed onBehalfOf);
    event Withdrawn(address indexed asset, uint256 amount);
    event Borrowed(address indexed asset, uint256 amount);
    event Repaid(address indexed asset, uint256 amount);
    event FlashLoanExecuted(address indexed asset, uint256 amount, uint256 premium);
    event LiquidationExecuted(
        address indexed collateral,
        address indexed debt,
        address indexed user,
        uint256 debtCovered
    );

    // ================== Errors ==================
    error UnauthorizedFlashLoan();
    error InvalidInitiator();
    error InsufficientAllowance();
    error HealthFactorTooLow(uint256 healthFactor);
    error SlippageExceeded(uint256 expected, uint256 received);

    // ================== Constructor ==================
    constructor(
        address _pool,
        address _addressesProvider,
        address _owner
    ) Ownable(_owner) {
        pool = IPool(_pool);
        addressesProvider = _addressesProvider;
    }

    // ================== IFlashLoanSimpleReceiver ==================
    function ADDRESSES_PROVIDER() external view override returns (address) {
        return addressesProvider;
    }

    function POOL() external view override returns (address) {
        return address(pool);
    }

    // ================== Supply Functions ==================

    /**
     * @notice ฝาก asset เข้า Aave pool เพื่อรับ aToken และดอกเบี้ย
     * @param asset address ของ token ที่จะฝาก
     * @param amount จำนวนที่จะฝาก
     * @param onBehalfOf address ที่จะรับ aToken
     */
    function supply(
        address asset,
        uint256 amount,
        address onBehalfOf
    ) external nonReentrant {
        // ดึง token จาก user มายัง contract นี้
        IERC20(asset).safeTransferFrom(msg.sender, address(this), amount);

        // Approve pool ให้ใช้ token
        IERC20(asset).forceApprove(address(pool), amount);

        // Supply เข้า Aave
        pool.supply(asset, amount, onBehalfOf, REFERRAL_CODE);

        // Track amount
        suppliedAmounts[asset] += amount;

        emit Supplied(asset, amount, onBehalfOf);
    }

    /**
     * @notice ถอน asset จาก Aave pool
     * @param asset address ของ token ที่จะถอน
     * @param amount จำนวนที่จะถอน (ใช้ type(uint256).max สำหรับถอนทั้งหมด)
     * @param to address ที่จะรับ token
     */
    function withdraw(
        address asset,
        uint256 amount,
        address to
    ) external nonReentrant onlyOwner returns (uint256 withdrawn) {
        withdrawn = pool.withdraw(asset, amount, to);

        // อัพเดท tracking
        if (amount == type(uint256).max) {
            suppliedAmounts[asset] = 0;
        } else {
            suppliedAmounts[asset] -= amount;
        }

        emit Withdrawn(asset, withdrawn);
    }

    // ================== Borrow Functions ==================

    /**
     * @notice กู้ asset จาก Aave pool (ต้องมี collateral เพียงพอ)
     * @param asset address ของ token ที่จะกู้
     * @param amount จำนวนที่จะกู้
     * @param minHealthFactor health factor ขั้นต่ำหลังกู้ (1e18 = 1.0)
     */
    function borrow(
        address asset,
        uint256 amount,
        uint256 minHealthFactor
    ) external nonReentrant onlyOwner {
        // ตรวจสอบ health factor ก่อนกู้
        (,,,,, uint256 healthFactorBefore) = pool.getUserAccountData(address(this));
        
        pool.borrow(
            asset,
            amount,
            INTEREST_RATE_MODE_VARIABLE,
            REFERRAL_CODE,
            address(this)
        );

        // ตรวจสอบ health factor หลังกู้
        (,,,,, uint256 healthFactorAfter) = pool.getUserAccountData(address(this));

        if (healthFactorAfter < minHealthFactor) {
            revert HealthFactorTooLow(healthFactorAfter);
        }

        borrowedAmounts[asset] += amount;

        emit Borrowed(asset, amount);
    }

    /**
     * @notice คืนหนี้ใน Aave pool
     * @param asset address ของ token ที่จะคืน
     * @param amount จำนวนที่จะคืน (ใช้ type(uint256).max สำหรับคืนทั้งหมด)
     */
    function repay(
        address asset,
        uint256 amount
    ) external nonReentrant returns (uint256 repaid) {
        // ดึง token จาก user
        uint256 actualAmount = amount == type(uint256).max
            ? borrowedAmounts[asset]
            : amount;

        IERC20(asset).safeTransferFrom(msg.sender, address(this), actualAmount);
        IERC20(asset).forceApprove(address(pool), actualAmount);

        repaid = pool.repay(
            asset,
            amount,
            INTEREST_RATE_MODE_VARIABLE,
            address(this)
        );

        borrowedAmounts[asset] = borrowedAmounts[asset] > repaid
            ? borrowedAmounts[asset] - repaid
            : 0;

        emit Repaid(asset, repaid);
    }

    // ================== Liquidation ==================

    /**
     * @notice ทำ liquidation บน Aave V3
     * @dev ต้องมี debtAsset เพียงพอสำหรับ cover หนี้
     * @param collateralAsset asset ที่จะรับเป็น collateral
     * @param debtAsset asset ที่จะคืนหนี้แทน user
     * @param user address ของ user ที่จะ liquidate
     * @param debtToCover จำนวนหนี้ที่จะ cover (ใช้ type(uint256).max สำหรับ 50% ของหนี้)
     * @param receiveAToken true = รับ aToken, false = รับ underlying token
     */
    function liquidate(
        address collateralAsset,
        address debtAsset,
        address user,
        uint256 debtToCover,
        bool receiveAToken
    ) external nonReentrant onlyOwner {
        // ดึง debt asset มาเพื่อทำ liquidation
        IERC20(debtAsset).safeTransferFrom(msg.sender, address(this), debtToCover);
        IERC20(debtAsset).forceApprove(address(pool), debtToCover);

        // บันทึก balance ก่อน
        uint256 collateralBefore = IERC20(collateralAsset).balanceOf(address(this));

        pool.liquidationCall(
            collateralAsset,
            debtAsset,
            user,
            debtToCover,
            receiveAToken
        );

        // คำนวณ collateral ที่ได้รับ
        uint256 collateralReceived = IERC20(collateralAsset).balanceOf(address(this)) 
            - collateralBefore;

        emit LiquidationExecuted(collateralAsset, debtAsset, user, debtToCover);
    }

    // ================== Flash Loan ==================

    /**
     * @notice เริ่ม flash loan จาก Aave V3
     * @param asset token ที่จะกู้
     * @param amount จำนวนที่จะกู้
     * @param params ข้อมูลเพิ่มเติมสำหรับ executeOperation
     */
    function executeFlashLoan(
        address asset,
        uint256 amount,
        bytes calldata params
    ) external nonReentrant onlyOwner {
        // บันทึก flash loan ที่รอดำเนินการ
        bytes32 loanId = keccak256(abi.encodePacked(asset, amount, block.timestamp));
        pendingFlashLoans[loanId] = true;

        pool.flashLoanSimple(
            address(this),  // receiver
            asset,
            amount,
            params,
            REFERRAL_CODE
        );

        // ลบ pending loan หลังเสร็จ
        delete pendingFlashLoans[loanId];
    }

    /**
     * @notice Callback จาก Aave Pool เมื่อส่ง flash loan
     * @dev ต้องคืนเงิน + premium ก่อน function นี้จบ
     */
    function executeOperation(
        address asset,
        uint256 amount,
        uint256 premium,
        address initiator,
        bytes calldata params
    ) external override returns (bool) {
        // ตรวจสอบว่า caller คือ Aave Pool จริง
        if (msg.sender != address(pool)) revert UnauthorizedFlashLoan();

        // ตรวจสอบว่า initiator คือ contract นี้
        if (initiator != address(this)) revert InvalidInitiator();

        // =============================================
        // ส่วนนี้คือ logic สำหรับใช้เงิน flash loan
        // ตัวอย่าง: Arbitrage, Liquidation, Collateral Swap
        // =============================================
        _handleFlashLoanLogic(asset, amount, params);

        // คืนเงิน + premium ให้ Pool
        uint256 totalRepay = amount + premium;
        IERC20(asset).forceApprove(address(pool), totalRepay);

        emit FlashLoanExecuted(asset, amount, premium);

        return true;
    }

    /**
     * @dev Internal function สำหรับ flash loan logic
     * Override ใน subcontract เพื่อ implement logic จริง
     */
    function _handleFlashLoanLogic(
        address asset,
        uint256 amount,
        bytes calldata params
    ) internal virtual {
        // Decode params และ execute strategy
        // ตัวอย่าง: arbitrage, collateral swap, etc.
    }

    // ================== View Functions ==================

    /**
     * @notice ดูข้อมูล account ทั้งหมดใน Aave
     */
    function getAccountData() external view returns (
        uint256 totalCollateralBase,
        uint256 totalDebtBase,
        uint256 availableBorrowsBase,
        uint256 currentLiquidationThreshold,
        uint256 ltv,
        uint256 healthFactor
    ) {
        return pool.getUserAccountData(address(this));
    }

    /**
     * @notice ดูข้อมูล reserve ของ asset
     */
    function getReserveData(address asset)
        external
        view
        returns (IPool.ReserveData memory)
    {
        return pool.getReserveData(asset);
    }
}
```

---

## 2. Compound V3 (Comet) Integration

### 2.1 ทำความรู้จัก Compound V3

Compound V3 (หรือ Comet) เป็นการออกแบบใหม่ที่แตกต่างจาก Compound V2 อย่างมาก:
- **Single Borrowable Asset**: แต่ละ Comet instance มีเพียง 1 base asset (เช่น USDC)
- **Multiple Collaterals**: สามารถฝาก collateral หลายชนิดได้
- **Built-in Oracle**: ใช้ Chainlink ภายในตัวสำหรับ price feeds
- **No Interest Rate Tokens**: ไม่มี cToken, ใช้ balance tracking โดยตรง

### 2.2 IComet Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title IComet
 * @notice Interface สำหรับ Compound V3 (Comet)
 */
interface IComet {
    // ข้อมูล asset configuration
    struct AssetInfo {
        uint8 offset;
        address asset;
        address priceFeed;
        uint64 scale;
        uint64 borrowCollateralFactor;
        uint64 liquidateCollateralFactor;
        uint64 liquidationFactor;
        uint128 supplyCap;
    }

    // ข้อมูล user
    struct UserBasic {
        int104 principal;
        uint64 baseTrackingIndex;
        uint64 baseTrackingAccrued;
        uint16 assetsIn;
        uint8 _reserved;
    }

    // ข้อมูล collateral user
    struct UserCollateral {
        uint128 balance;
        uint128 _reserved;
    }

    // Events
    event Supply(address indexed from, address indexed dst, uint256 amount);
    event Withdraw(address indexed src, address indexed to, uint256 amount);
    event SupplyCollateral(
        address indexed from,
        address indexed dst,
        address indexed asset,
        uint256 amount
    );
    event WithdrawCollateral(
        address indexed src,
        address indexed to,
        address indexed asset,
        uint256 amount
    );
    event AbsorbDebt(
        address indexed absorber,
        address indexed borrower,
        uint256 basePaidOut,
        uint256 usdValue
    );

    // Supply/Withdraw base asset
    function supply(address asset, uint256 amount) external;
    function supplyTo(address dst, address asset, uint256 amount) external;
    function withdraw(address asset, uint256 amount) external;
    function withdrawTo(address to, address asset, uint256 amount) external;

    // Supply/Withdraw collateral
    function supplyCollateral(address asset, uint256 amount) external;
    function withdrawCollateral(address asset, uint256 amount) external;

    // Liquidation (absorption)
    function absorb(address absorber, address[] calldata accounts) external;
    function buyCollateral(
        address asset,
        uint256 minAmount,
        uint256 baseAmount,
        address recipient
    ) external;

    // View functions
    function balanceOf(address account) external view returns (uint256);
    function borrowBalanceOf(address account) external view returns (uint256);
    function collateralBalanceOf(address account, address asset)
        external
        view
        returns (uint128);
    function getPrice(address priceFeed) external view returns (uint128);
    function getAssetInfo(uint8 i) external view returns (AssetInfo memory);
    function getAssetInfoByAddress(address asset)
        external
        view
        returns (AssetInfo memory);
    function numAssets() external view returns (uint8);
    function isLiquidatable(address account) external view returns (bool);
    function isBorrowCollateralized(address account) external view returns (bool);
    function baseToken() external view returns (address);
    function baseTokenPriceFeed() external view returns (address);
    function baseScale() external view returns (uint256);
    function baseMinForRewards() external view returns (uint256);
}
```

### 2.3 CompoundV3Integration Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title CompoundV3Integration
 * @notice ตัวอย่างการ integrate กับ Compound V3 (Comet)
 * @dev รองรับ supply, borrow, withdraw, และดึงราคาจาก built-in oracle
 */
contract CompoundV3Integration is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ================== State Variables ==================
    IComet public immutable comet;
    address public immutable baseToken;

    // ================== Events ==================
    event BaseSupplied(address indexed supplier, uint256 amount);
    event BaseWithdrawn(address indexed recipient, uint256 amount);
    event Borrowed(uint256 amount);
    event CollateralSupplied(address indexed asset, uint256 amount);
    event CollateralWithdrawn(address indexed asset, uint256 amount);
    event LiquidationTriggered(address indexed borrower);

    // ================== Errors ==================
    error NotLiquidatable(address account);
    error InsufficientBalance(uint256 available, uint256 required);
    error StalePrice(address priceFeed, uint256 price);

    // ================== Constructor ==================
    constructor(address _comet, address _owner) Ownable(_owner) {
        comet = IComet(_comet);
        baseToken = IComet(_comet).baseToken();
    }

    // ================== Supply/Lending Functions ==================

    /**
     * @notice ฝาก base asset (เช่น USDC) เข้า Compound V3
     * @dev เมื่อฝากแล้วจะได้รับดอกเบี้ยโดยอัตโนมัติผ่าน balance tracking
     * @param amount จำนวนที่จะฝาก
     */
    function supplyBase(uint256 amount) external nonReentrant {
        IERC20(baseToken).safeTransferFrom(msg.sender, address(this), amount);
        IERC20(baseToken).forceApprove(address(comet), amount);

        comet.supply(baseToken, amount);

        emit BaseSupplied(msg.sender, amount);
    }

    /**
     * @notice ถอน base asset จาก Compound V3
     * @param amount จำนวนที่จะถอน
     * @param to address ที่จะรับ
     */
    function withdrawBase(uint256 amount, address to) 
        external 
        nonReentrant 
        onlyOwner 
    {
        uint256 available = comet.balanceOf(address(this));
        if (available < amount) revert InsufficientBalance(available, amount);

        comet.withdrawTo(to, baseToken, amount);

        emit BaseWithdrawn(to, amount);
    }

    // ================== Collateral Functions ==================

    /**
     * @notice ฝาก collateral asset (เช่น ETH, WBTC)
     * @param asset address ของ collateral token
     * @param amount จำนวนที่จะฝาก
     */
    function supplyCollateral(address asset, uint256 amount) 
        external 
        nonReentrant 
    {
        IERC20(asset).safeTransferFrom(msg.sender, address(this), amount);
        IERC20(asset).forceApprove(address(comet), amount);

        comet.supplyCollateral(asset, amount);

        emit CollateralSupplied(asset, amount);
    }

    /**
     * @notice ถอน collateral asset
     * @param asset address ของ collateral token
     * @param amount จำนวนที่จะถอน
     * @param to address ที่จะรับ
     */
    function withdrawCollateral(address asset, uint256 amount, address to)
        external
        nonReentrant
        onlyOwner
    {
        comet.withdrawCollateral(asset, amount);
        IERC20(asset).safeTransfer(to, amount);

        emit CollateralWithdrawn(asset, amount);
    }

    // ================== Borrow Functions ==================

    /**
     * @notice กู้ base asset จาก Compound V3
     * @dev ใช้ withdraw function พร้อม negative balance
     * @param amount จำนวนที่จะกู้
     */
    function borrow(uint256 amount) external nonReentrant onlyOwner {
        // ใน Comet, การกู้คือการ withdraw มากกว่า balance
        comet.withdraw(baseToken, amount);

        emit Borrowed(amount);
    }

    // ================== Price Oracle Functions ==================

    /**
     * @notice ดึงราคา asset จาก built-in oracle ของ Compound V3
     * @param asset address ของ asset
     * @return price ราคาใน USD (8 decimals)
     */
    function getAssetPrice(address asset) external view returns (uint128) {
        IComet.AssetInfo memory info = comet.getAssetInfoByAddress(asset);
        uint128 price = comet.getPrice(info.priceFeed);
        
        // ตรวจสอบ price ว่าไม่เป็น 0 (stale/invalid)
        if (price == 0) revert StalePrice(info.priceFeed, price);

        return price;
    }

    /**
     * @notice ดึงราคา base asset
     * @return price ราคาใน USD (8 decimals)
     */
    function getBasePrice() external view returns (uint128) {
        address priceFeed = comet.baseTokenPriceFeed();
        uint128 price = comet.getPrice(priceFeed);

        if (price == 0) revert StalePrice(priceFeed, price);

        return price;
    }

    /**
     * @notice คำนวณ USD value ของ collateral
     * @param asset address ของ collateral
     * @param amount จำนวน collateral
     * @return usdValue มูลค่าใน USD (8 decimals)
     */
    function getCollateralUSDValue(address asset, uint256 amount)
        external
        view
        returns (uint256 usdValue)
    {
        IComet.AssetInfo memory info = comet.getAssetInfoByAddress(asset);
        uint128 price = comet.getPrice(info.priceFeed);
        
        // usdValue = amount * price / scale
        usdValue = (amount * uint256(price)) / uint256(info.scale);
    }

    // ================== Liquidation Functions ==================

    /**
     * @notice ทำ liquidation บน Compound V3
     * @param borrower address ของ borrower ที่จะ liquidate
     */
    function liquidate(address borrower) external nonReentrant {
        if (!comet.isLiquidatable(borrower)) revert NotLiquidatable(borrower);

        address[] memory accounts = new address[](1);
        accounts[0] = borrower;

        comet.absorb(address(this), accounts);

        emit LiquidationTriggered(borrower);
    }

    // ================== View Functions ==================

    /**
     * @notice ดู lending balance (รวมดอกเบี้ยสะสม)
     */
    function getLendingBalance() external view returns (uint256) {
        return comet.balanceOf(address(this));
    }

    /**
     * @notice ดู borrow balance (รวมดอกเบี้ยสะสม)
     */
    function getBorrowBalance() external view returns (uint256) {
        return comet.borrowBalanceOf(address(this));
    }

    /**
     * @notice ดู collateral balance ของ asset ใดๆ
     */
    function getCollateralBalance(address asset) 
        external 
        view 
        returns (uint128) 
    {
        return comet.collateralBalanceOf(address(this), asset);
    }

    /**
     * @notice ตรวจสอบว่า position ปลอดภัยหรือไม่
     */
    function isPositionSafe() external view returns (bool) {
        return comet.isBorrowCollateralized(address(this));
    }
}
```

---

## 3. Uniswap V3 Integration

### 3.1 ทำความรู้จัก Uniswap V3

Uniswap V3 เป็น AMM (Automated Market Maker) ที่นำเสนอ:
- **Concentrated Liquidity**: Liquidity providers เลือก price range ได้
- **Multiple Fee Tiers**: 0.01%, 0.05%, 0.3%, 1%
- **TWAP Oracle**: Time-Weighted Average Price oracle ในตัว
- **NFT Positions**: แต่ละ LP position เป็น NFT

### 3.2 ISwapRouter Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title ISwapRouter
 * @notice Interface สำหรับ Uniswap V3 Swap Router
 */
interface ISwapRouter {
    struct ExactInputSingleParams {
        address tokenIn;            // token ที่จะ swap จาก
        address tokenOut;           // token ที่จะ swap ไป
        uint24 fee;                 // fee tier (100, 500, 3000, 10000)
        address recipient;          // ผู้รับ token
        uint256 deadline;           // deadline สำหรับ transaction
        uint256 amountIn;           // จำนวน tokenIn ที่จะ swap
        uint256 amountOutMinimum;   // จำนวน tokenOut ขั้นต่ำ (slippage protection)
        uint160 sqrtPriceLimitX96;  // ราคา limit (0 = ไม่มี limit)
    }

    struct ExactInputParams {
        bytes path;                 // encoded path: tokenA + fee + tokenB + fee + tokenC
        address recipient;
        uint256 deadline;
        uint256 amountIn;
        uint256 amountOutMinimum;
    }

    struct ExactOutputSingleParams {
        address tokenIn;
        address tokenOut;
        uint24 fee;
        address recipient;
        uint256 deadline;
        uint256 amountOut;          // จำนวน tokenOut ที่ต้องการ
        uint256 amountInMaximum;    // จำนวน tokenIn สูงสุด
        uint160 sqrtPriceLimitX96;
    }

    struct ExactOutputParams {
        bytes path;
        address recipient;
        uint256 deadline;
        uint256 amountOut;
        uint256 amountInMaximum;
    }

    /**
     * @notice Swap exact input สำหรับ output ผ่าน single pool
     */
    function exactInputSingle(ExactInputSingleParams calldata params)
        external
        payable
        returns (uint256 amountOut);

    /**
     * @notice Swap exact input สำหรับ output ผ่าน multiple pools (multi-hop)
     */
    function exactInput(ExactInputParams calldata params)
        external
        payable
        returns (uint256 amountOut);

    /**
     * @notice Swap input สำหรับ exact output ผ่าน single pool
     */
    function exactOutputSingle(ExactOutputSingleParams calldata params)
        external
        payable
        returns (uint256 amountIn);

    /**
     * @notice Swap input สำหรับ exact output ผ่าน multiple pools
     */
    function exactOutput(ExactOutputParams calldata params)
        external
        payable
        returns (uint256 amountIn);
}

/**
 * @title IQuoterV2
 * @notice Interface สำหรับ simulate swap โดยไม่ต้องทำ swap จริง
 */
interface IQuoterV2 {
    struct QuoteExactInputSingleParams {
        address tokenIn;
        address tokenOut;
        uint256 amountIn;
        uint24 fee;
        uint160 sqrtPriceLimitX96;
    }

    struct QuoteExactInputSingleResult {
        uint256 amountOut;
        uint160 sqrtPriceX96After;
        uint32 initializedTicksCrossed;
        uint256 gasEstimate;
    }

    struct QuoteExactOutputSingleParams {
        address tokenIn;
        address tokenOut;
        uint256 amount;
        uint24 fee;
        uint160 sqrtPriceLimitX96;
    }

    function quoteExactInputSingle(QuoteExactInputSingleParams memory params)
        external
        returns (QuoteExactInputSingleResult memory result);

    function quoteExactInput(bytes memory path, uint256 amountIn)
        external
        returns (
            uint256 amountOut,
            uint160[] memory sqrtPriceX96AfterList,
            uint32[] memory initializedTicksCrossedList,
            uint256 gasEstimate
        );
}

/**
 * @title IUniswapV3SwapCallback
 * @notice Interface สำหรับ callback เมื่อ Uniswap Pool ทำ swap
 * @dev ใช้เมื่อ interact กับ Pool โดยตรง (ไม่ผ่าน Router)
 */
interface IUniswapV3SwapCallback {
    /**
     * @notice Callback หลัง swap
     * @param amount0Delta จำนวน token0 ที่เปลี่ยน (บวก = ต้องส่ง, ลบ = ได้รับ)
     * @param amount1Delta จำนวน token1 ที่เปลี่ยน
     * @param data ข้อมูลเพิ่มเติม
     */
    function uniswapV3SwapCallback(
        int256 amount0Delta,
        int256 amount1Delta,
        bytes calldata data
    ) external;
}
```

### 3.3 UniswapV3Integration Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title UniswapV3Integration
 * @notice ตัวอย่างการ integrate กับ Uniswap V3
 * @dev รองรับ single-hop และ multi-hop swaps พร้อม slippage protection
 */
contract UniswapV3Integration is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ================== Constants ==================
    uint24 public constant FEE_LOW = 100;       // 0.01% - สำหรับ stablecoins
    uint24 public constant FEE_MEDIUM = 500;    // 0.05% - สำหรับ correlated pairs
    uint24 public constant FEE_HIGH = 3000;     // 0.3% - สำหรับ standard pairs
    uint24 public constant FEE_VERY_HIGH = 10000; // 1% - สำหรับ exotic pairs

    uint256 public constant SLIPPAGE_DENOMINATOR = 10000;

    // ================== State Variables ==================
    ISwapRouter public immutable swapRouter;
    IQuoterV2 public immutable quoter;

    // Maximum slippage (basis points, e.g. 50 = 0.5%)
    uint256 public maxSlippageBps;

    // ================== Events ==================
    event SwapExecuted(
        address indexed tokenIn,
        address indexed tokenOut,
        uint256 amountIn,
        uint256 amountOut
    );
    event MultiHopSwapExecuted(
        bytes path,
        uint256 amountIn,
        uint256 amountOut
    );
    event MaxSlippageUpdated(uint256 oldSlippage, uint256 newSlippage);

    // ================== Errors ==================
    error DeadlineExpired(uint256 deadline, uint256 currentTime);
    error SlippageTooHigh(uint256 expected, uint256 minimum);
    error InvalidPath();
    error ZeroAmount();

    // ================== Constructor ==================
    constructor(
        address _swapRouter,
        address _quoter,
        uint256 _maxSlippageBps,
        address _owner
    ) Ownable(_owner) {
        swapRouter = ISwapRouter(_swapRouter);
        quoter = IQuoterV2(_quoter);
        maxSlippageBps = _maxSlippageBps;
    }

    // ================== Single-Hop Swap ==================

    /**
     * @notice Swap token ผ่าน single pool
     * @param tokenIn token ที่จะ swap จาก
     * @param tokenOut token ที่จะ swap ไป
     * @param fee fee tier ของ pool
     * @param amountIn จำนวน tokenIn ที่จะ swap
     * @param minAmountOut จำนวน tokenOut ขั้นต่ำ (0 = คำนวณจาก maxSlippageBps)
     * @return amountOut จำนวน tokenOut ที่ได้รับ
     */
    function swapExactInputSingle(
        address tokenIn,
        address tokenOut,
        uint24 fee,
        uint256 amountIn,
        uint256 minAmountOut
    ) external nonReentrant returns (uint256 amountOut) {
        if (amountIn == 0) revert ZeroAmount();

        // ดึง token จาก user
        IERC20(tokenIn).safeTransferFrom(msg.sender, address(this), amountIn);
        IERC20(tokenIn).forceApprove(address(swapRouter), amountIn);

        // คำนวณ minimum output ถ้าไม่ได้ระบุ
        if (minAmountOut == 0) {
            // Get quote แล้วคำนวณ minimum จาก slippage
            (uint256 quotedAmount,,,) = quoter.quoteExactInput(
                abi.encodePacked(tokenIn, fee, tokenOut),
                amountIn
            );
            minAmountOut = quotedAmount * (SLIPPAGE_DENOMINATOR - maxSlippageBps) 
                / SLIPPAGE_DENOMINATOR;
        }

        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: tokenIn,
                tokenOut: tokenOut,
                fee: fee,
                recipient: msg.sender,
                deadline: block.timestamp + 30 minutes,
                amountIn: amountIn,
                amountOutMinimum: minAmountOut,
                sqrtPriceLimitX96: 0
            });

        amountOut = swapRouter.exactInputSingle(params);

        emit SwapExecuted(tokenIn, tokenOut, amountIn, amountOut);
    }

    // ================== Multi-Hop Swap ==================

    /**
     * @notice Swap token ผ่านหลาย pool (multi-hop)
     * @dev path encoding: tokenA + fee + tokenB + fee + tokenC
     * @param path encoded path สำหรับ multi-hop swap
     * @param amountIn จำนวน tokenIn ที่จะ swap
     * @param minAmountOut จำนวน tokenOut ขั้นต่ำ
     * @return amountOut จำนวน tokenOut ที่ได้รับ
     */
    function swapExactInputMultiHop(
        bytes calldata path,
        uint256 amountIn,
        uint256 minAmountOut
    ) external nonReentrant returns (uint256 amountOut) {
        if (amountIn == 0) revert ZeroAmount();
        if (path.length < 43) revert InvalidPath(); // ขั้นต่ำ 2 tokens + 1 fee

        // ดึง tokenIn จาก path (20 bytes แรก)
        address tokenIn = address(bytes20(path[:20]));

        // ดึง token จาก user
        IERC20(tokenIn).safeTransferFrom(msg.sender, address(this), amountIn);
        IERC20(tokenIn).forceApprove(address(swapRouter), amountIn);

        ISwapRouter.ExactInputParams memory params = ISwapRouter.ExactInputParams({
            path: path,
            recipient: msg.sender,
            deadline: block.timestamp + 30 minutes,
            amountIn: amountIn,
            amountOutMinimum: minAmountOut
        });

        amountOut = swapRouter.exactInput(params);

        emit MultiHopSwapExecuted(path, amountIn, amountOut);
    }

    // ================== Quote Functions ==================

    /**
     * @notice ดูราคาโดยประมาณสำหรับ single-hop swap
     * @dev ใช้ IQuoterV2 ซึ่งเป็น view-like แต่จริงๆ ต้อง call ใน off-chain
     * @param tokenIn token ที่จะ swap จาก
     * @param tokenOut token ที่จะ swap ไป
     * @param fee fee tier
     * @param amountIn จำนวน tokenIn
     * @return amountOut จำนวน tokenOut โดยประมาณ
     */
    function getQuoteSingle(
        address tokenIn,
        address tokenOut,
        uint24 fee,
        uint256 amountIn
    ) external returns (uint256 amountOut) {
        IQuoterV2.QuoteExactInputSingleResult memory result = quoter
            .quoteExactInputSingle(
                IQuoterV2.QuoteExactInputSingleParams({
                    tokenIn: tokenIn,
                    tokenOut: tokenOut,
                    amountIn: amountIn,
                    fee: fee,
                    sqrtPriceLimitX96: 0
                })
            );

        return result.amountOut;
    }

    /**
     * @notice ดูราคาโดยประมาณสำหรับ multi-hop swap
     * @param path encoded path
     * @param amountIn จำนวน tokenIn
     * @return amountOut จำนวน tokenOut โดยประมาณ
     */
    function getQuoteMultiHop(
        bytes calldata path,
        uint256 amountIn
    ) external returns (uint256 amountOut) {
        (amountOut,,,) = quoter.quoteExactInput(path, amountIn);
    }

    // ================== Path Builder Helper ==================

    /**
     * @notice สร้าง path สำหรับ multi-hop swap
     * @param tokens array ของ tokens ในลำดับ
     * @param fees array ของ fee tiers
     * @return path encoded path
     */
    function buildPath(
        address[] calldata tokens,
        uint24[] calldata fees
    ) external pure returns (bytes memory path) {
        require(tokens.length >= 2, "Need at least 2 tokens");
        require(fees.length == tokens.length - 1, "Fee count mismatch");

        path = abi.encodePacked(tokens[0]);

        for (uint256 i = 0; i < fees.length; i++) {
            path = abi.encodePacked(path, fees[i], tokens[i + 1]);
        }
    }

    // ================== Admin Functions ==================

    function setMaxSlippage(uint256 _maxSlippageBps) external onlyOwner {
        require(_maxSlippageBps <= 1000, "Max slippage too high"); // max 10%
        emit MaxSlippageUpdated(maxSlippageBps, _maxSlippageBps);
        maxSlippageBps = _maxSlippageBps;
    }

    function rescueTokens(address token, uint256 amount) external onlyOwner {
        IERC20(token).safeTransfer(owner(), amount);
    }
}
```

---

## 4. Curve Finance Integration

### 4.1 ทำความรู้จัก Curve Finance

Curve Finance เป็น AMM ที่ออกแบบมาเฉพาะสำหรับ stablecoin trading:
- **StableSwap Algorithm**: ลด slippage สำหรับ pegged assets
- **Liquidity Pools**: รองรับหลาย token ใน pool เดียว
- **CRV Rewards**: ผู้ให้ liquidity ได้รับ CRV token
- **Gauge System**: boost rewards ด้วย veCRV
- **Virtual Price**: ติดตาม pool value per LP token

### 4.2 ICurvePool Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title ICurvePool
 * @notice Interface สำหรับ Curve Finance Stable Pool
 * @dev Interface นี้ cover ทั้ง 2-token และ 3-token pools
 */
interface ICurvePool {
    // ================== Trading ==================

    /**
     * @notice Exchange token i สำหรับ token j
     * @param i index ของ token ที่จะส่ง
     * @param j index ของ token ที่จะรับ
     * @param dx จำนวน token i ที่จะส่ง
     * @param min_dy จำนวน token j ขั้นต่ำที่ต้องการรับ
     * @return dy จำนวน token j ที่ได้รับจริง
     */
    function exchange(
        int128 i,
        int128 j,
        uint256 dx,
        uint256 min_dy
    ) external returns (uint256 dy);

    /**
     * @notice Exchange สำหรับ pools ที่ใช้ uint256 indices
     */
    function exchange(
        uint256 i,
        uint256 j,
        uint256 dx,
        uint256 min_dy
    ) external returns (uint256 dy);

    /**
     * @notice ดูราคา exchange ก่อนทำ (read-only)
     */
    function get_dy(int128 i, int128 j, uint256 dx) external view returns (uint256);

    // ================== Liquidity ==================

    /**
     * @notice เพิ่ม liquidity เข้า pool
     * @param amounts จำนวน token แต่ละตัวที่จะเพิ่ม (array ขนาดเท่า N_COINS)
     * @param min_mint_amount จำนวน LP token ขั้นต่ำที่ต้องการรับ
     * @return LP token ที่ได้รับ
     */
    function add_liquidity(uint256[2] calldata amounts, uint256 min_mint_amount)
        external
        returns (uint256);

    function add_liquidity(uint256[3] calldata amounts, uint256 min_mint_amount)
        external
        returns (uint256);

    /**
     * @notice ลบ liquidity ออกจาก pool (สัดส่วนเท่าๆ กัน)
     * @param amount จำนวน LP token ที่จะ burn
     * @param min_amounts จำนวน token ขั้นต่ำที่ต้องการรับ
     */
    function remove_liquidity(
        uint256 amount,
        uint256[2] calldata min_amounts
    ) external;

    function remove_liquidity(
        uint256 amount,
        uint256[3] calldata min_amounts
    ) external;

    /**
     * @notice ลบ liquidity ออกในรูปแบบ token เดียว
     * @param token_amount จำนวน LP token ที่จะ burn
     * @param i index ของ token ที่ต้องการรับ
     * @param min_amount จำนวน token ขั้นต่ำ
     * @return จำนวน token ที่ได้รับ
     */
    function remove_liquidity_one_coin(
        uint256 token_amount,
        int128 i,
        uint256 min_amount
    ) external returns (uint256);

    function remove_liquidity_one_coin(
        uint256 token_amount,
        uint256 i,
        uint256 min_amount
    ) external returns (uint256);

    /**
     * @notice ดูจำนวน token ที่จะได้รับเมื่อ remove liquidity เป็น single token
     */
    function calc_withdraw_one_coin(uint256 token_amount, int128 i)
        external
        view
        returns (uint256);

    /**
     * @notice ดูจำนวน LP token ที่จะได้รับเมื่อ add liquidity
     */
    function calc_token_amount(uint256[2] calldata amounts, bool is_deposit)
        external
        view
        returns (uint256);

    function calc_token_amount(uint256[3] calldata amounts, bool is_deposit)
        external
        view
        returns (uint256);

    // ================== Pool Info ==================

    /**
     * @notice ดู virtual price ของ LP token
     * @dev virtual price คือมูลค่า per LP token ใน USD equivalent
     * ค่านี้ควรเพิ่มขึ้นเรื่อยๆ เมื่อ trading fees สะสม
     * @return virtual_price ราคา LP token (1e18 = $1)
     */
    function get_virtual_price() external view returns (uint256);

    /**
     * @notice ดู balance ของ token ใน pool
     */
    function balances(uint256 i) external view returns (uint256);

    /**
     * @notice ดู fee ของ pool
     */
    function fee() external view returns (uint256);

    /**
     * @notice ดู A parameter (amplification coefficient)
     */
    function A() external view returns (uint256);

    /**
     * @notice ดู address ของ token
     */
    function coins(uint256 i) external view returns (address);
}
```

### 4.3 CurveIntegration Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title CurveIntegration
 * @notice ตัวอย่างการ integrate กับ Curve Finance
 * @dev รองรับ exchange, add/remove liquidity, และ virtual price monitoring
 */
contract CurveIntegration is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ================== Constants ==================
    uint256 public constant PRECISION = 1e18;
    uint256 public constant BASIS_POINTS = 10000;

    // ================== State Variables ==================
    ICurvePool public immutable pool;
    address public immutable lpToken;

    // Token addresses ใน pool
    address[] public tokens;

    // Virtual price tracking สำหรับตรวจจับ manipulation
    uint256 public lastVirtualPrice;
    uint256 public lastVirtualPriceTimestamp;

    // Maximum virtual price change per check (0.1% = 10 bps)
    uint256 public maxVirtualPriceChangeBps;

    // ================== Events ==================
    event Exchange(
        address indexed tokenIn,
        address indexed tokenOut,
        uint256 amountIn,
        uint256 amountOut
    );
    event LiquidityAdded(uint256[] amounts, uint256 lpTokens);
    event LiquidityRemoved(uint256 lpTokens, address token, uint256 amount);
    event VirtualPriceUpdated(uint256 oldPrice, uint256 newPrice);

    // ================== Errors ==================
    error InvalidTokenIndex(int128 index);
    error VirtualPriceManipulated(uint256 expected, uint256 actual);
    error SlippageExceeded(uint256 expected, uint256 minimum);
    error TokenCountMismatch(uint256 expected, uint256 actual);

    // ================== Constructor ==================
    constructor(
        address _pool,
        address _lpToken,
        address[] memory _tokens,
        uint256 _maxVirtualPriceChangeBps,
        address _owner
    ) Ownable(_owner) {
        pool = ICurvePool(_pool);
        lpToken = _lpToken;
        tokens = _tokens;
        maxVirtualPriceChangeBps = _maxVirtualPriceChangeBps;

        // บันทึก virtual price เริ่มต้น
        lastVirtualPrice = ICurvePool(_pool).get_virtual_price();
        lastVirtualPriceTimestamp = block.timestamp;
    }

    // ================== Exchange Functions ==================

    /**
     * @notice Exchange token ผ่าน Curve pool
     * @param fromIdx index ของ token ที่จะส่ง
     * @param toIdx index ของ token ที่จะรับ
     * @param amount จำนวนที่จะ exchange
     * @param slippageBps slippage tolerance (basis points)
     * @return received จำนวน token ที่ได้รับ
     */
    function exchange(
        int128 fromIdx,
        int128 toIdx,
        uint256 amount,
        uint256 slippageBps
    ) external nonReentrant returns (uint256 received) {
        require(fromIdx >= 0 && fromIdx < int128(int256(tokens.length)), "Invalid from index");
        require(toIdx >= 0 && toIdx < int128(int256(tokens.length)), "Invalid to index");
        require(fromIdx != toIdx, "Same token");

        address tokenIn = tokens[uint256(int256(fromIdx))];

        // ตรวจสอบ virtual price ก่อน
        _checkVirtualPrice();

        // Get quote ก่อน exchange
        uint256 expectedOut = pool.get_dy(fromIdx, toIdx, amount);
        uint256 minOut = expectedOut * (BASIS_POINTS - slippageBps) / BASIS_POINTS;

        // ดึง token จาก user
        IERC20(tokenIn).safeTransferFrom(msg.sender, address(this), amount);
        IERC20(tokenIn).forceApprove(address(pool), amount);

        // Execute exchange
        received = pool.exchange(fromIdx, toIdx, amount, minOut);

        // ส่ง token ที่ได้รับให้ user
        address tokenOut = tokens[uint256(int256(toIdx))];
        IERC20(tokenOut).safeTransfer(msg.sender, received);

        emit Exchange(tokenIn, tokenOut, amount, received);
    }

    // ================== Liquidity Functions ==================

    /**
     * @notice เพิ่ม liquidity ใน Curve 2-token pool
     * @param amount0 จำนวน token[0]
     * @param amount1 จำนวน token[1]
     * @param minLpTokens จำนวน LP token ขั้นต่ำที่ต้องการ
     * @return lpReceived จำนวน LP token ที่ได้รับ
     */
    function addLiquidity2(
        uint256 amount0,
        uint256 amount1,
        uint256 minLpTokens
    ) external nonReentrant returns (uint256 lpReceived) {
        if (tokens.length != 2) revert TokenCountMismatch(2, tokens.length);

        _checkVirtualPrice();

        // Approve tokens
        if (amount0 > 0) {
            IERC20(tokens[0]).safeTransferFrom(msg.sender, address(this), amount0);
            IERC20(tokens[0]).forceApprove(address(pool), amount0);
        }
        if (amount1 > 0) {
            IERC20(tokens[1]).safeTransferFrom(msg.sender, address(this), amount1);
            IERC20(tokens[1]).forceApprove(address(pool), amount1);
        }

        uint256[2] memory amounts = [amount0, amount1];
        lpReceived = pool.add_liquidity(amounts, minLpTokens);

        // ส่ง LP token ให้ user
        IERC20(lpToken).safeTransfer(msg.sender, lpReceived);

        uint256[] memory amountsArr = new uint256[](2);
        amountsArr[0] = amount0;
        amountsArr[1] = amount1;
        emit LiquidityAdded(amountsArr, lpReceived);
    }

    /**
     * @notice เพิ่ม liquidity ใน Curve 3-token pool
     * @param amount0 จำนวน token[0]
     * @param amount1 จำนวน token[1]
     * @param amount2 จำนวน token[2]
     * @param minLpTokens จำนวน LP token ขั้นต่ำ
     * @return lpReceived จำนวน LP token ที่ได้รับ
     */
    function addLiquidity3(
        uint256 amount0,
        uint256 amount1,
        uint256 amount2,
        uint256 minLpTokens
    ) external nonReentrant returns (uint256 lpReceived) {
        if (tokens.length != 3) revert TokenCountMismatch(3, tokens.length);

        _checkVirtualPrice();

        // Approve tokens
        _transferAndApprove(tokens[0], amount0);
        _transferAndApprove(tokens[1], amount1);
        _transferAndApprove(tokens[2], amount2);

        uint256[3] memory amounts = [amount0, amount1, amount2];
        lpReceived = pool.add_liquidity(amounts, minLpTokens);

        IERC20(lpToken).safeTransfer(msg.sender, lpReceived);

        uint256[] memory amountsArr = new uint256[](3);
        amountsArr[0] = amount0;
        amountsArr[1] = amount1;
        amountsArr[2] = amount2;
        emit LiquidityAdded(amountsArr, lpReceived);
    }

    /**
     * @notice ลบ liquidity ออกในรูปแบบ single token
     * @param lpAmount จำนวน LP token ที่จะ burn
     * @param tokenIdx index ของ token ที่ต้องการรับ
     * @param minAmount จำนวน token ขั้นต่ำที่ต้องการรับ
     * @return received จำนวน token ที่ได้รับ
     */
    function removeLiquidityOneCoin(
        uint256 lpAmount,
        int128 tokenIdx,
        uint256 minAmount
    ) external nonReentrant returns (uint256 received) {
        require(tokenIdx >= 0 && tokenIdx < int128(int256(tokens.length)), "Invalid index");

        // ดึง LP token จาก user
        IERC20(lpToken).safeTransferFrom(msg.sender, address(this), lpAmount);
        IERC20(lpToken).forceApprove(address(pool), lpAmount);

        received = pool.remove_liquidity_one_coin(lpAmount, tokenIdx, minAmount);

        address tokenOut = tokens[uint256(int256(tokenIdx))];
        IERC20(tokenOut).safeTransfer(msg.sender, received);

        emit LiquidityRemoved(lpAmount, tokenOut, received);
    }

    // ================== Virtual Price Security ==================

    /**
     * @notice ตรวจสอบ virtual price ว่าไม่ถูก manipulate
     * @dev Virtual price manipulation เป็น attack vector สำคัญใน DeFi
     */
    function _checkVirtualPrice() internal {
        uint256 currentVirtualPrice = pool.get_virtual_price();

        // อนุญาตให้ virtual price เพิ่มขึ้นได้ตามปกติ
        if (currentVirtualPrice > lastVirtualPrice) {
            uint256 increase = ((currentVirtualPrice - lastVirtualPrice) * BASIS_POINTS) 
                / lastVirtualPrice;
            
            // ถ้าเพิ่มขึ้นมากผิดปกติ อาจเป็น manipulation
            if (increase > maxVirtualPriceChangeBps) {
                revert VirtualPriceManipulated(lastVirtualPrice, currentVirtualPrice);
            }
        }

        emit VirtualPriceUpdated(lastVirtualPrice, currentVirtualPrice);
        lastVirtualPrice = currentVirtualPrice;
        lastVirtualPriceTimestamp = block.timestamp;
    }

    // ================== Internal Helpers ==================

    function _transferAndApprove(address token, uint256 amount) internal {
        if (amount > 0) {
            IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
            IERC20(token).forceApprove(address(pool), amount);
        }
    }

    // ================== View Functions ==================

    /**
     * @notice ดู virtual price ปัจจุบัน
     */
    function getVirtualPrice() external view returns (uint256) {
        return pool.get_virtual_price();
    }

    /**
     * @notice ดู balance ของ token ใน pool
     */
    function getPoolBalance(uint256 tokenIdx) external view returns (uint256) {
        return pool.balances(tokenIdx);
    }

    /**
     * @notice คำนวณ LP token ที่จะได้รับเมื่อ add liquidity
     */
    function calcLpTokensFor2Coins(
        uint256 amount0,
        uint256 amount1,
        bool isDeposit
    ) external view returns (uint256) {
        uint256[2] memory amounts = [amount0, amount1];
        return pool.calc_token_amount(amounts, isDeposit);
    }

    /**
     * @notice คำนวณ token ที่จะได้รับเมื่อ remove one coin
     */
    function calcWithdrawOneCoin(
        uint256 lpAmount,
        int128 tokenIdx
    ) external view returns (uint256) {
        return pool.calc_withdraw_one_coin(lpAmount, tokenIdx);
    }
}
```

---

## 5. Integration Security Best Practices

### 5.1 Interface-Only Imports

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @notice หลักการ: ใช้เฉพาะ interfaces ไม่ใช้ full implementation
 * 
 * ทำไมถึงสำคัญ:
 * 1. ลดความเสี่ยงจาก storage collision
 * 2. ลด bytecode size
 * 3. ชัดเจนว่าอะไรที่ใช้จริงๆ
 * 4. ป้องกัน accidental state changes
 */

// ✅ GOOD: ใช้เฉพาะ interface
interface IExternalProtocol {
    function doSomething(uint256 amount) external returns (uint256);
}

// ❌ BAD: import full contract
// import {ExternalProtocol} from "@external/ExternalProtocol.sol";

contract SecureIntegration {
    IExternalProtocol public immutable protocol;

    constructor(address _protocol) {
        protocol = IExternalProtocol(_protocol);
    }
}
```

### 5.2 Version Pinning และ Security Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

/**
 * @title DeFiSecurityFramework
 * @notice Framework สำหรับ secure DeFi integration
 * @dev รวม security patterns ที่สำคัญทั้งหมด
 */
contract DeFiSecurityFramework {
    using SafeERC20 for IERC20;

    // ================== Version Pinning ==================

    // Protocol addresses ควร immutable และ verify ที่ constructor
    address public immutable aavePool;      // Aave V3 Pool
    address public immutable uniswapRouter; // Uniswap V3 Router
    address public immutable curvePool;     // Curve Pool

    // Chainlink Price Feed addresses
    address public immutable ethUsdFeed;
    address public immutable btcUsdFeed;

    // ================== Constants ==================
    uint256 private constant PRICE_STALENESS_THRESHOLD = 3600; // 1 hour
    uint256 private constant SLIPPAGE_DENOMINATOR = 10000;
    uint256 private constant MAX_SLIPPAGE_BPS = 200; // 2% maximum

    // ================== Oracle Security ==================

    /**
     * @notice ดึงราคาจาก Chainlink พร้อม staleness check
     * @param priceFeed address ของ Chainlink price feed
     * @return price ราคาใน 8 decimals
     */
    function _getSafePrice(address priceFeed)
        internal
        view
        returns (uint256 price)
    {
        AggregatorV3Interface feed = AggregatorV3Interface(priceFeed);
        (
            uint80 roundId,
            int256 answer,
            uint256 startedAt,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = feed.latestRoundData();

        // ตรวจสอบ round completion
        require(answeredInRound >= roundId, "Stale price: incomplete round");

        // ตรวจสอบ timestamp
        require(
            block.timestamp - updatedAt <= PRICE_STALENESS_THRESHOLD,
            "Stale price: too old"
        );

        // ตรวจสอบ price ไม่ติดลบ
        require(answer > 0, "Invalid price: zero or negative");

        price = uint256(answer);
    }

    /**
     * @notice ดึงราคาพร้อม fallback oracle
     * @dev ถ้า primary oracle stale ให้ใช้ secondary
     */
    function _getSafePriceWithFallback(
        address primaryFeed,
        address fallbackFeed
    ) internal view returns (uint256 price, bool usedFallback) {
        try this._getSafePrice(primaryFeed) returns (uint256 primaryPrice) {
            return (primaryPrice, false);
        } catch {
            // Primary oracle stale หรือ error, ใช้ fallback
            price = this._getSafePrice(fallbackFeed);
            usedFallback = true;
        }
    }

    // ================== Slippage Protection ==================

    /**
     * @notice คำนวณ minimum output พร้อม slippage protection
     * @param expected จำนวนที่คาดว่าจะได้รับ
     * @param slippageBps slippage tolerance (basis points)
     */
    function _calcMinOutput(
        uint256 expected,
        uint256 slippageBps
    ) internal pure returns (uint256 minOutput) {
        require(slippageBps <= MAX_SLIPPAGE_BPS, "Slippage too high");
        minOutput = expected * (SLIPPAGE_DENOMINATOR - slippageBps) / SLIPPAGE_DENOMINATOR;
    }

    // ================== Constructor ==================
    constructor(
        address _aavePool,
        address _uniswapRouter,
        address _curvePool,
        address _ethUsdFeed,
        address _btcUsdFeed
    ) {
        // Verify contracts have code (ไม่ใช่ EOA)
        require(_hasCode(_aavePool), "Aave pool not deployed");
        require(_hasCode(_uniswapRouter), "Uniswap router not deployed");
        require(_hasCode(_curvePool), "Curve pool not deployed");
        require(_hasCode(_ethUsdFeed), "ETH feed not deployed");
        require(_hasCode(_btcUsdFeed), "BTC feed not deployed");

        aavePool = _aavePool;
        uniswapRouter = _uniswapRouter;
        curvePool = _curvePool;
        ethUsdFeed = _ethUsdFeed;
        btcUsdFeed = _btcUsdFeed;
    }

    function _hasCode(address addr) internal view returns (bool) {
        uint256 size;
        assembly {
            size := extcodesize(addr)
        }
        return size > 0;
    }
}

// Helper interface
interface AggregatorV3Interface {
    function latestRoundData()
        external
        view
        returns (
            uint80 roundId,
            int256 answer,
            uint256 startedAt,
            uint256 updatedAt,
            uint80 answeredInRound
        );
}
```

---

## Workshop: สร้าง Multi-Protocol DeFi Strategy

### Workshop เป้าหมาย:
สร้าง contract ที่:
1. Borrow USDC จาก Aave V3 โดยใช้ ETH เป็น collateral
2. Swap USDC บางส่วนเป็น WBTC ผ่าน Uniswap V3
3. เพิ่ม liquidity ใน Curve 3pool (USDC/USDT/DAI)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title MultiProtocolStrategy
 * @notice ตัวอย่าง strategy ที่ใช้หลาย DeFi protocol พร้อมกัน
 * @dev Workshop solution: Aave borrow -> Uniswap swap -> Curve LP
 */
contract MultiProtocolStrategy is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ================== Protocol References ==================
    IPool public immutable aavePool;
    ISwapRouter public immutable uniswapRouter;
    ICurvePool public immutable curve3Pool;

    // Token addresses
    address public immutable WETH;
    address public immutable USDC;
    address public immutable USDT;
    address public immutable DAI;
    address public immutable WBTC;
    address public immutable curve3PoolToken; // 3CRV

    // Constants
    uint256 private constant BORROW_LTV = 5000;  // 50% LTV
    uint256 private constant SLIPPAGE_BPS = 100;  // 1% slippage
    uint24 private constant USDC_WBTC_FEE = 3000; // 0.3% fee

    // ================== Events ==================
    event StrategyExecuted(
        uint256 ethSupplied,
        uint256 usdcBorrowed,
        uint256 wbtcReceived,
        uint256 curveLpReceived
    );

    event StrategyUnwound(
        uint256 curveLpBurned,
        uint256 usdcRepaid,
        uint256 ethWithdrawn
    );

    // ================== Constructor ==================
    constructor(
        address _aavePool,
        address _uniswapRouter,
        address _curve3Pool,
        address _weth,
        address _usdc,
        address _usdt,
        address _dai,
        address _wbtc,
        address _curve3PoolToken,
        address _owner
    ) Ownable(_owner) {
        aavePool = IPool(_aavePool);
        uniswapRouter = ISwapRouter(_uniswapRouter);
        curve3Pool = ICurvePool(_curve3Pool);
        WETH = _weth;
        USDC = _usdc;
        USDT = _usdt;
        DAI = _dai;
        WBTC = _wbtc;
        curve3PoolToken = _curve3PoolToken;
    }

    // ================== Strategy Execution ==================

    /**
     * @notice Execute multi-protocol strategy
     * @param wethAmount จำนวน WETH ที่จะ supply เป็น collateral
     * @param borrowPercent เปอร์เซ็นต์ของ available borrow ที่จะ borrow (50-80%)
     * @param swapPercent เปอร์เซ็นต์ของ USDC ที่จะ swap เป็น WBTC (0-100%)
     */
    function executeStrategy(
        uint256 wethAmount,
        uint256 borrowPercent,
        uint256 swapPercent
    ) external nonReentrant onlyOwner {
        require(borrowPercent <= 80, "Borrow percent too high");
        require(swapPercent <= 100, "Swap percent too high");

        // Step 1: Supply WETH เข้า Aave V3
        IERC20(WETH).safeTransferFrom(msg.sender, address(this), wethAmount);
        IERC20(WETH).forceApprove(address(aavePool), wethAmount);
        aavePool.supply(WETH, wethAmount, address(this), 0);

        // Step 2: Borrow USDC จาก Aave
        (,, uint256 availableBorrows,,,) = aavePool.getUserAccountData(address(this));
        uint256 usdcToBorrow = (availableBorrows * borrowPercent) / 100;
        aavePool.borrow(USDC, usdcToBorrow, 2, 0, address(this));

        // Step 3: Swap บางส่วนเป็น WBTC ผ่าน Uniswap
        uint256 wbtcReceived;
        if (swapPercent > 0) {
            uint256 usdcToSwap = (usdcToBorrow * swapPercent) / 100;
            IERC20(USDC).forceApprove(address(uniswapRouter), usdcToSwap);

            ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
                .ExactInputSingleParams({
                    tokenIn: USDC,
                    tokenOut: WBTC,
                    fee: USDC_WBTC_FEE,
                    recipient: address(this),
                    deadline: block.timestamp + 30 minutes,
                    amountIn: usdcToSwap,
                    amountOutMinimum: 0, // ควรคำนวณจริงๆ ด้วย quoter
                    sqrtPriceLimitX96: 0
                });
            wbtcReceived = uniswapRouter.exactInputSingle(params);
        }

        // Step 4: เพิ่ม remaining USDC เข้า Curve 3pool
        uint256 remainingUsdc = IERC20(USDC).balanceOf(address(this));
        uint256 curveLpReceived;
        if (remainingUsdc > 0) {
            IERC20(USDC).forceApprove(address(curve3Pool), remainingUsdc);
            uint256[3] memory amounts = [remainingUsdc, 0, 0]; // USDC เป็น index 1 ใน 3pool
            curveLpReceived = curve3Pool.add_liquidity(amounts, 0);
        }

        emit StrategyExecuted(wethAmount, usdcToBorrow, wbtcReceived, curveLpReceived);
    }

    /**
     * @notice Unwind strategy: ถอน Curve LP -> คืนหนี้ -> ถอน collateral
     */
    function unwindStrategy() external nonReentrant onlyOwner {
        // Step 1: ถอน liquidity จาก Curve
        uint256 lpBalance = IERC20(curve3PoolToken).balanceOf(address(this));
        if (lpBalance > 0) {
            IERC20(curve3PoolToken).forceApprove(address(curve3Pool), lpBalance);
            curve3Pool.remove_liquidity_one_coin(lpBalance, 1, 0); // ถอนเป็น USDC
        }

        // Step 2: Swap WBTC กลับเป็น USDC
        uint256 wbtcBalance = IERC20(WBTC).balanceOf(address(this));
        if (wbtcBalance > 0) {
            IERC20(WBTC).forceApprove(address(uniswapRouter), wbtcBalance);
            ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
                .ExactInputSingleParams({
                    tokenIn: WBTC,
                    tokenOut: USDC,
                    fee: USDC_WBTC_FEE,
                    recipient: address(this),
                    deadline: block.timestamp + 30 minutes,
                    amountIn: wbtcBalance,
                    amountOutMinimum: 0,
                    sqrtPriceLimitX96: 0
                });
            uniswapRouter.exactInputSingle(params);
        }

        // Step 3: Repay USDC debt ใน Aave
        uint256 usdcBalance = IERC20(USDC).balanceOf(address(this));
        if (usdcBalance > 0) {
            IERC20(USDC).forceApprove(address(aavePool), usdcBalance);
            aavePool.repay(USDC, type(uint256).max, 2, address(this));
        }

        // Step 4: ถอน WETH collateral
        uint256 ethWithdrawn = aavePool.withdraw(WETH, type(uint256).max, msg.sender);

        emit StrategyUnwound(lpBalance, usdcBalance, ethWithdrawn);
    }
}
```

---

## สรุป Part 73

- **Aave V3**: Interface หลักคือ `IPool` รองรับ supply/borrow/repay/liquidationCall และ flash loans ผ่าน `IFlashLoanSimpleReceiver.executeOperation()`
- **Compound V3**: ออกแบบใหม่ด้วย single borrowable asset, built-in oracle ผ่าน `getPrice()`, ใช้ `absorb()` สำหรับ liquidation
- **Uniswap V3**: `ISwapRouter.exactInputSingle()` สำหรับ single-hop, `exactInput()` สำหรับ multi-hop ด้วย encoded path, `IQuoterV2` สำหรับ price simulation
- **Curve Finance**: `exchange()` สำหรับ stablecoin swap, `add_liquidity()`/`remove_liquidity_one_coin()` สำหรับ LP, `get_virtual_price()` สำหรับ LP value
- **Integration Security**: ใช้ interface-only imports, pin addresses ใน immutable variables, ตรวจสอบ oracle staleness, ป้องกัน slippage, ตรวจสอบ virtual price manipulation

## Next: Part 74 - Stablecoin Protocol Design
