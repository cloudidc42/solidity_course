# Part 74: Stablecoin Protocol Design

## บทนำ

Stablecoin เป็นหัวใจสำคัญของ DeFi ecosystem การออกแบบ stablecoin protocol ต้องพิจารณาหลายมิติ ทั้งกลไก collateral, การ liquidation, การรักษา peg, และ emergency procedures บทนี้จะพาสร้าง stablecoin protocol ที่สมบูรณ์พร้อม CDP (Collateralized Debt Position) mechanics

---

## 1. CDP Mechanics

### 1.1 ทำความรู้จัก CDP

CDP (Collateralized Debt Position) เป็นกลไกหลักของ stablecoin แบบ crypto-backed:
- **Collateral**: ผู้ใช้ฝาก crypto asset เป็นหลักประกัน
- **Minting**: ได้รับ stablecoin ตามสัดส่วน collateral (overcollateralized)
- **Health Factor**: อัตราส่วน collateral value / debt value
- **Liquidation**: เมื่อ health factor ต่ำกว่าเกณฑ์ สถานะถูก liquidate

### 1.2 Stablecoin Token Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {ERC20Burnable} from "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import {AccessControl} from "@openzeppelin/contracts/access/AccessControl.sol";

/**
 * @title StableCoin
 * @notice Decentralized stablecoin token ที่ mint/burn ได้โดย CDPVault
 * @dev ERC20 ที่มี role-based access สำหรับ mint/burn
 */
contract StableCoin is ERC20, ERC20Burnable, AccessControl {
    // ================== Roles ==================
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant BURNER_ROLE = keccak256("BURNER_ROLE");

    // ================== State Variables ==================
    uint256 public totalMinted;
    bool public emergencyStopped;

    // ================== Events ==================
    event EmergencyStop(address indexed triggeredBy);
    event EmergencyResume(address indexed resumedBy);

    // ================== Errors ==================
    error EmergencyStopActive();
    error NotMinter(address caller);

    // ================== Constructor ==================
    constructor(string memory name, string memory symbol)
        ERC20(name, symbol)
    {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }

    // ================== Modifiers ==================
    modifier whenNotStopped() {
        if (emergencyStopped) revert EmergencyStopActive();
        _;
    }

    // ================== Mint/Burn ==================

    /**
     * @notice Mint stablecoin ให้กับ recipient
     * @dev เรียกได้เฉพาะ CDPVault ที่มี MINTER_ROLE
     */
    function mint(address to, uint256 amount)
        external
        onlyRole(MINTER_ROLE)
        whenNotStopped
    {
        totalMinted += amount;
        _mint(to, amount);
    }

    /**
     * @notice Burn stablecoin จาก account
     * @dev เรียกได้เฉพาะ CDPVault ที่มี BURNER_ROLE
     */
    function burnFrom(address from, uint256 amount)
        public
        override
        onlyRole(BURNER_ROLE)
    {
        _burn(from, amount);
        if (totalMinted >= amount) {
            totalMinted -= amount;
        }
    }

    // ================== Emergency ==================

    function triggerEmergencyStop() external onlyRole(DEFAULT_ADMIN_ROLE) {
        emergencyStopped = true;
        emit EmergencyStop(msg.sender);
    }

    function resumeFromEmergency() external onlyRole(DEFAULT_ADMIN_ROLE) {
        emergencyStopped = false;
        emit EmergencyResume(msg.sender);
    }
}
```

### 1.3 CDPVault Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";

/**
 * @title CDPVault
 * @notice Collateralized Debt Position Vault สำหรับ mint stablecoin
 * @dev ผู้ใช้ deposit collateral และ mint stablecoin โดยต้องรักษา health factor >= 150%
 */
contract CDPVault is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;
    using Math for uint256;

    // ================== Constants ==================
    uint256 public constant PRECISION = 1e18;
    uint256 public constant BASIS_POINTS = 10000;

    // Health factor thresholds
    uint256 public constant MINIMUM_HEALTH_FACTOR = 150;  // 150% collateralization
    uint256 public constant LIQUIDATION_THRESHOLD = 110;   // Liquidation ที่ 110%

    // Liquidation bonus 5%
    uint256 public constant LIQUIDATION_BONUS = 500; // 500 bps = 5%

    // Stability fee (interest rate) per year: 2%
    uint256 public constant STABILITY_FEE_RATE = 200; // 200 bps = 2%
    uint256 public constant SECONDS_PER_YEAR = 365 days;

    // ================== Types ==================
    struct Vault {
        uint256 collateral;     // จำนวน collateral token
        uint256 debt;           // จำนวน stablecoin ที่ mint (หนี้)
        uint256 lastUpdate;     // timestamp ล่าสุดที่อัพเดท interest
        bool isOpen;            // vault ยังเปิดอยู่หรือไม่
    }

    // ================== State Variables ==================
    StableCoin public immutable stablecoin;
    IERC20 public immutable collateralToken;
    IPriceFeed public immutable priceFeed;

    // Vault data
    mapping(address => Vault) public vaults;
    mapping(address => uint256) public vaultIds;

    // Global stats
    uint256 public totalCollateral;
    uint256 public totalDebt;
    uint256 public totalVaults;

    // Risk parameters
    uint256 public debtCeiling;       // debt ceiling สูงสุด
    uint256 public minimumVaultDebt;  // หนี้ขั้นต่ำต่อ vault (ป้องกัน dust)
    uint256 public collateralDecimals;

    // Emergency state
    bool public caged;              // emergency shutdown ถูก trigger
    uint256 public cagePrice;       // ราคาที่ fix ตอน cage
    uint256 public cageTimestamp;

    // ================== Events ==================
    event VaultOpened(address indexed owner, uint256 indexed vaultId);
    event CollateralDeposited(address indexed owner, uint256 amount, uint256 total);
    event CollateralWithdrawn(address indexed owner, uint256 amount, uint256 total);
    event StablecoinMinted(address indexed owner, uint256 amount, uint256 totalDebt);
    event StablecoinRepaid(address indexed owner, uint256 amount, uint256 totalDebt);
    event VaultClosed(address indexed owner, uint256 collateralReturned);
    event VaultLiquidated(
        address indexed owner,
        address indexed liquidator,
        uint256 debtCovered,
        uint256 collateralSeized,
        uint256 bonus
    );
    event Caged(uint256 price, uint256 timestamp);
    event VaultRedeemed(address indexed owner, uint256 collateralReturned);

    // ================== Errors ==================
    error VaultAlreadyOpen(address owner);
    error VaultNotOpen(address owner);
    error InsufficientCollateral(uint256 available, uint256 required);
    error HealthFactorTooLow(uint256 healthFactor, uint256 minimum);
    error DebtCeilingExceeded(uint256 current, uint256 ceiling);
    error MinimumDebtNotMet(uint256 debt, uint256 minimum);
    error VaultIsHealthy(address owner, uint256 healthFactor);
    error ProtocolCaged();
    error ProtocolNotCaged();

    // ================== Modifiers ==================
    modifier vaultExists(address owner) {
        if (!vaults[owner].isOpen) revert VaultNotOpen(owner);
        _;
    }

    modifier whenNotCaged() {
        if (caged) revert ProtocolCaged();
        _;
    }

    modifier whenCaged() {
        if (!caged) revert ProtocolNotCaged();
        _;
    }

    // ================== Constructor ==================
    constructor(
        address _stablecoin,
        address _collateralToken,
        address _priceFeed,
        uint256 _debtCeiling,
        uint256 _minimumVaultDebt,
        uint256 _collateralDecimals,
        address _owner
    ) Ownable(_owner) {
        stablecoin = StableCoin(_stablecoin);
        collateralToken = IERC20(_collateralToken);
        priceFeed = IPriceFeed(_priceFeed);
        debtCeiling = _debtCeiling;
        minimumVaultDebt = _minimumVaultDebt;
        collateralDecimals = _collateralDecimals;
    }

    // ================== Vault Operations ==================

    /**
     * @notice เปิด vault ใหม่
     * @dev แต่ละ address มีได้ 1 vault เท่านั้น
     */
    function openVault() external whenNotCaged {
        if (vaults[msg.sender].isOpen) revert VaultAlreadyOpen(msg.sender);

        totalVaults++;
        vaultIds[msg.sender] = totalVaults;

        vaults[msg.sender] = Vault({
            collateral: 0,
            debt: 0,
            lastUpdate: block.timestamp,
            isOpen: true
        });

        emit VaultOpened(msg.sender, totalVaults);
    }

    /**
     * @notice ฝาก collateral เข้า vault
     * @param amount จำนวน collateral ที่จะฝาก
     */
    function depositCollateral(uint256 amount)
        external
        nonReentrant
        whenNotCaged
        vaultExists(msg.sender)
    {
        require(amount > 0, "Zero amount");

        // อัพเดท accrued interest ก่อน
        _accrueInterest(msg.sender);

        collateralToken.safeTransferFrom(msg.sender, address(this), amount);

        vaults[msg.sender].collateral += amount;
        totalCollateral += amount;

        emit CollateralDeposited(
            msg.sender,
            amount,
            vaults[msg.sender].collateral
        );
    }

    /**
     * @notice ถอน collateral จาก vault
     * @param amount จำนวน collateral ที่จะถอน
     */
    function withdrawCollateral(uint256 amount)
        external
        nonReentrant
        whenNotCaged
        vaultExists(msg.sender)
    {
        Vault storage vault = vaults[msg.sender];
        if (vault.collateral < amount) {
            revert InsufficientCollateral(vault.collateral, amount);
        }

        _accrueInterest(msg.sender);

        vault.collateral -= amount;
        totalCollateral -= amount;

        // ตรวจสอบ health factor หลังถอน
        if (vault.debt > 0) {
            uint256 healthFactor = _calculateHealthFactor(
                vault.collateral,
                vault.debt
            );
            if (healthFactor < MINIMUM_HEALTH_FACTOR) {
                revert HealthFactorTooLow(healthFactor, MINIMUM_HEALTH_FACTOR);
            }
        }

        collateralToken.safeTransfer(msg.sender, amount);

        emit CollateralWithdrawn(
            msg.sender,
            amount,
            vault.collateral
        );
    }

    /**
     * @notice Mint stablecoin โดยใช้ collateral เป็นหลักประกัน
     * @param amount จำนวน stablecoin ที่ต้องการ mint
     */
    function mintStablecoin(uint256 amount)
        external
        nonReentrant
        whenNotCaged
        vaultExists(msg.sender)
    {
        require(amount > 0, "Zero amount");

        _accrueInterest(msg.sender);

        Vault storage vault = vaults[msg.sender];
        uint256 newDebt = vault.debt + amount;

        // ตรวจสอบ debt ceiling
        if (totalDebt + amount > debtCeiling) {
            revert DebtCeilingExceeded(totalDebt + amount, debtCeiling);
        }

        // ตรวจสอบ minimum vault debt
        if (newDebt < minimumVaultDebt) {
            revert MinimumDebtNotMet(newDebt, minimumVaultDebt);
        }

        // ตรวจสอบ health factor
        uint256 healthFactor = _calculateHealthFactor(vault.collateral, newDebt);
        if (healthFactor < MINIMUM_HEALTH_FACTOR) {
            revert HealthFactorTooLow(healthFactor, MINIMUM_HEALTH_FACTOR);
        }

        vault.debt = newDebt;
        totalDebt += amount;

        stablecoin.mint(msg.sender, amount);

        emit StablecoinMinted(msg.sender, amount, vault.debt);
    }

    /**
     * @notice คืนหนี้ stablecoin
     * @param amount จำนวน stablecoin ที่จะคืน (type(uint256).max = คืนทั้งหมด)
     */
    function repay(uint256 amount)
        external
        nonReentrant
        vaultExists(msg.sender)
    {
        _accrueInterest(msg.sender);

        Vault storage vault = vaults[msg.sender];
        uint256 actualRepay = amount == type(uint256).max
            ? vault.debt
            : amount;

        require(actualRepay <= vault.debt, "Repay exceeds debt");

        // Burn stablecoin
        stablecoin.burnFrom(msg.sender, actualRepay);

        vault.debt -= actualRepay;
        totalDebt -= actualRepay;

        emit StablecoinRepaid(msg.sender, actualRepay, vault.debt);
    }

    /**
     * @notice ปิด vault และรับ collateral คืน
     * @dev ต้องคืนหนี้ทั้งหมดก่อน
     */
    function closeVault()
        external
        nonReentrant
        vaultExists(msg.sender)
    {
        _accrueInterest(msg.sender);

        Vault storage vault = vaults[msg.sender];

        // ต้องไม่มีหนี้คงค้าง
        require(vault.debt == 0, "Must repay all debt first");

        uint256 collateralToReturn = vault.collateral;
        totalCollateral -= collateralToReturn;

        // ลบ vault
        delete vaults[msg.sender];

        // คืน collateral
        if (collateralToReturn > 0) {
            collateralToken.safeTransfer(msg.sender, collateralToReturn);
        }

        emit VaultClosed(msg.sender, collateralToReturn);
    }

    // ================== Interest Accrual ==================

    /**
     * @notice คำนวณและบวก stability fee (interest) ที่สะสม
     * @dev Interest = principal * rate * time
     */
    function _accrueInterest(address owner) internal {
        Vault storage vault = vaults[owner];
        if (vault.debt == 0 || vault.lastUpdate == block.timestamp) return;

        uint256 elapsed = block.timestamp - vault.lastUpdate;
        uint256 interest = (vault.debt * STABILITY_FEE_RATE * elapsed)
            / (BASIS_POINTS * SECONDS_PER_YEAR);

        if (interest > 0) {
            vault.debt += interest;
            totalDebt += interest;
        }

        vault.lastUpdate = block.timestamp;
    }

    // ================== Health Factor ==================

    /**
     * @notice คำนวณ health factor
     * @dev health factor = (collateral_value / debt_value) * 100
     * ถ้า >= 150 ปลอดภัย, < 110 ถูก liquidate
     */
    function _calculateHealthFactor(
        uint256 collateralAmount,
        uint256 debtAmount
    ) internal view returns (uint256) {
        if (debtAmount == 0) return type(uint256).max;

        // ดึงราคา collateral
        uint256 collateralPrice = priceFeed.getPrice();

        // คำนวณ collateral value ใน stablecoin units
        uint256 collateralValue = (collateralAmount * collateralPrice)
            / (10 ** collateralDecimals);

        // health factor = collateral_value * 100 / debt
        return (collateralValue * 100) / debtAmount;
    }

    /**
     * @notice ดู health factor ของ vault
     * @param owner address ของเจ้าของ vault
     */
    function getHealthFactor(address owner)
        external
        view
        returns (uint256)
    {
        Vault memory vault = vaults[owner];
        return _calculateHealthFactor(vault.collateral, vault.debt);
    }

    // ================== View Functions ==================

    /**
     * @notice ดูข้อมูล vault
     */
    function getVault(address owner)
        external
        view
        returns (
            uint256 collateral,
            uint256 debt,
            uint256 healthFactor,
            uint256 maxMintable,
            bool liquidatable
        )
    {
        Vault memory vault = vaults[owner];
        collateral = vault.collateral;
        debt = vault.debt;
        healthFactor = _calculateHealthFactor(vault.collateral, vault.debt);

        // คำนวณ max mintable (ที่ health factor = 150%)
        uint256 collateralPrice = priceFeed.getPrice();
        uint256 collateralValue = (vault.collateral * collateralPrice)
            / (10 ** collateralDecimals);
        uint256 maxDebt = (collateralValue * 100) / MINIMUM_HEALTH_FACTOR;
        maxMintable = maxDebt > vault.debt ? maxDebt - vault.debt : 0;

        liquidatable = healthFactor < LIQUIDATION_THRESHOLD;
    }
}

// Price feed interface
interface IPriceFeed {
    function getPrice() external view returns (uint256);
}
```

---

## 2. Liquidation Mechanism

### 2.1 หลักการ Liquidation

Liquidation เกิดขึ้นเมื่อ health factor ของ vault ต่ำกว่า threshold กลไก:
1. Liquidator คืนหนี้บางส่วนหรือทั้งหมดแทน borrower
2. Liquidator ได้รับ collateral + bonus
3. Bonus ชดเชย gas cost และ incentivize การ liquidate

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title LiquidationEngine
 * @notice Engine สำหรับ liquidate vaults ที่ unhealthy
 * @dev รองรับทั้ง direct liquidation และ collateral auction
 */
contract LiquidationEngine is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ================== Constants ==================
    uint256 public constant LIQUIDATION_BONUS = 500;       // 5% bonus
    uint256 public constant MAX_LIQUIDATION_RATIO = 5000;  // Max 50% ของหนี้ต่อครั้ง
    uint256 public constant BASIS_POINTS = 10000;

    // Auction parameters
    uint256 public constant AUCTION_DURATION = 6 hours;
    uint256 public constant AUCTION_PRICE_DROP_RATE = 10; // 10% ต่อชั่วโมง
    uint256 public constant AUCTION_MIN_BID = 1; // ราคาขั้นต่ำ

    // ================== Types ==================
    struct Auction {
        address vaultOwner;
        address collateralToken;
        uint256 collateralAmount;
        uint256 debtAmount;
        uint256 startPrice;     // ราคาเริ่มต้นของ collateral
        uint256 startTime;      // เวลาเริ่ม auction
        bool settled;           // auction เสร็จสิ้นหรือไม่
    }

    // ================== State Variables ==================
    CDPVault public immutable vault;
    StableCoin public immutable stablecoin;
    IERC20 public immutable collateralToken;

    mapping(uint256 => Auction) public auctions;
    uint256 public auctionCount;

    // Statistics
    uint256 public totalLiquidations;
    uint256 public totalDebtLiquidated;
    uint256 public totalCollateralSeized;

    // ================== Events ==================
    event DirectLiquidation(
        address indexed vaultOwner,
        address indexed liquidator,
        uint256 debtCovered,
        uint256 collateralSeized,
        uint256 bonus
    );
    event AuctionStarted(
        uint256 indexed auctionId,
        address indexed vaultOwner,
        uint256 collateralAmount,
        uint256 debtAmount
    );
    event AuctionBid(
        uint256 indexed auctionId,
        address indexed bidder,
        uint256 collateralBought,
        uint256 debtPaid
    );
    event AuctionSettled(uint256 indexed auctionId, address indexed winner);

    // ================== Errors ==================
    error VaultIsHealthy(address owner, uint256 healthFactor);
    error DebtCoverExceedsLimit(uint256 requested, uint256 maximum);
    error AuctionExpired(uint256 auctionId);
    error AuctionAlreadySettled(uint256 auctionId);
    error InsufficientBid(uint256 bid, uint256 required);

    // ================== Constructor ==================
    constructor(
        address _vault,
        address _stablecoin,
        address _collateralToken,
        address _owner
    ) Ownable(_owner) {
        vault = CDPVault(_vault);
        stablecoin = StableCoin(_stablecoin);
        collateralToken = IERC20(_collateralToken);
    }

    // ================== Direct Liquidation ==================

    /**
     * @notice Liquidate vault โดยตรง (single transaction)
     * @dev Liquidator คืนหนี้และรับ collateral + bonus ทันที
     * @param vaultOwner address ของเจ้าของ vault ที่จะ liquidate
     * @param debtToCover จำนวนหนี้ที่จะ cover
     */
    function liquidate(
        address vaultOwner,
        uint256 debtToCover
    ) external nonReentrant {
        // ตรวจสอบ health factor
        uint256 healthFactor = vault.getHealthFactor(vaultOwner);
        if (healthFactor >= vault.LIQUIDATION_THRESHOLD()) {
            revert VaultIsHealthy(vaultOwner, healthFactor);
        }

        // ดึงข้อมูล vault
        (
            uint256 collateralAmount,
            uint256 debtAmount,
            ,
            ,

        ) = vault.getVault(vaultOwner);

        // จำกัด debtToCover ไม่เกิน 50% ของหนี้ทั้งหมด
        uint256 maxCoverable = (debtAmount * MAX_LIQUIDATION_RATIO) / BASIS_POINTS;
        if (debtToCover > maxCoverable) {
            revert DebtCoverExceedsLimit(debtToCover, maxCoverable);
        }

        // คำนวณ collateral ที่จะรับ (proportional + bonus)
        uint256 collateralRatio = (debtToCover * collateralAmount) / debtAmount;
        uint256 bonus = (collateralRatio * LIQUIDATION_BONUS) / BASIS_POINTS;
        uint256 totalCollateralToSeize = collateralRatio + bonus;

        // ตรวจสอบว่าไม่เกิน collateral ที่มี
        if (totalCollateralToSeize > collateralAmount) {
            totalCollateralToSeize = collateralAmount;
            debtToCover = debtAmount; // liquidate ทั้งหมดถ้า collateral ไม่พอ
        }

        // Burn stablecoin จาก liquidator
        stablecoin.burnFrom(msg.sender, debtToCover);

        // ลด debt และ collateral ของ vault (ผ่าน vault contract)
        // ในตัวอย่างนี้ vault มี function สำหรับ liquidation engine
        vault.executeLiquidation(vaultOwner, debtToCover, totalCollateralToSeize);

        // ส่ง collateral ให้ liquidator
        collateralToken.safeTransfer(msg.sender, totalCollateralToSeize);

        // อัพเดทสถิติ
        totalLiquidations++;
        totalDebtLiquidated += debtToCover;
        totalCollateralSeized += totalCollateralToSeize;

        emit DirectLiquidation(
            vaultOwner,
            msg.sender,
            debtToCover,
            totalCollateralToSeize,
            bonus
        );
    }

    // ================== Collateral Auction ==================

    /**
     * @notice เริ่ม collateral auction สำหรับ vault ที่ underwater
     * @dev Dutch auction: ราคาเริ่มสูงและลดลงตามเวลา
     */
    function startAuction(address vaultOwner) external nonReentrant {
        uint256 healthFactor = vault.getHealthFactor(vaultOwner);
        if (healthFactor >= vault.LIQUIDATION_THRESHOLD()) {
            revert VaultIsHealthy(vaultOwner, healthFactor);
        }

        (
            uint256 collateralAmount,
            uint256 debtAmount,
            ,
            ,

        ) = vault.getVault(vaultOwner);

        // ยึด collateral จาก vault
        vault.seizeCollateral(vaultOwner, collateralAmount);

        // ดึงราคาปัจจุบัน
        uint256 currentPrice = IPriceFeed(vault.priceFeed()).getPrice();

        auctionCount++;
        auctions[auctionCount] = Auction({
            vaultOwner: vaultOwner,
            collateralToken: address(collateralToken),
            collateralAmount: collateralAmount,
            debtAmount: debtAmount,
            startPrice: currentPrice * 110 / 100, // เริ่มที่ 110% ของ market price
            startTime: block.timestamp,
            settled: false
        });

        emit AuctionStarted(
            auctionCount,
            vaultOwner,
            collateralAmount,
            debtAmount
        );
    }

    /**
     * @notice Bid ใน collateral auction
     * @dev Dutch auction ราคาลดลงตามเวลา incentivize การ bid เร็ว
     * @param auctionId ID ของ auction
     * @param collateralToBuy จำนวน collateral ที่ต้องการ
     */
    function bid(
        uint256 auctionId,
        uint256 collateralToBuy
    ) external nonReentrant {
        Auction storage auction = auctions[auctionId];

        if (auction.settled) revert AuctionAlreadySettled(auctionId);
        if (block.timestamp > auction.startTime + AUCTION_DURATION) {
            revert AuctionExpired(auctionId);
        }

        // คำนวณราคาปัจจุบัน (Dutch auction - ลดลงตามเวลา)
        uint256 currentPrice = _getCurrentAuctionPrice(auctionId);

        // คำนวณ stablecoin ที่ต้องจ่าย
        uint256 debtRequired = (collateralToBuy * currentPrice)
            / (10 ** 18); // normalize ตาม decimals

        if (collateralToBuy > auction.collateralAmount) {
            collateralToBuy = auction.collateralAmount;
            debtRequired = auction.debtAmount;
        }

        // Burn stablecoin จาก bidder
        stablecoin.burnFrom(msg.sender, debtRequired);

        // ส่ง collateral ให้ bidder
        collateralToken.safeTransfer(msg.sender, collateralToBuy);

        // อัพเดท auction
        auction.collateralAmount -= collateralToBuy;
        auction.debtAmount -= debtRequired;

        if (auction.collateralAmount == 0 || auction.debtAmount == 0) {
            auction.settled = true;

            // คืน collateral ที่เหลือ (ถ้าหนี้หมดแล้วแต่ collateral ยังเหลือ) ให้เจ้าของ vault
            if (auction.collateralAmount > 0) {
                collateralToken.safeTransfer(
                    auction.vaultOwner,
                    auction.collateralAmount
                );
            }

            emit AuctionSettled(auctionId, msg.sender);
        }

        emit AuctionBid(auctionId, msg.sender, collateralToBuy, debtRequired);
    }

    /**
     * @notice คำนวณราคาปัจจุบันของ auction (Dutch auction)
     */
    function _getCurrentAuctionPrice(uint256 auctionId)
        internal
        view
        returns (uint256)
    {
        Auction memory auction = auctions[auctionId];
        uint256 elapsed = block.timestamp - auction.startTime;

        // ราคาลดลง 10% ต่อชั่วโมง
        uint256 hoursElapsed = elapsed / 3600;
        uint256 priceDropBps = hoursElapsed * AUCTION_PRICE_DROP_RATE * 100;

        if (priceDropBps >= BASIS_POINTS) {
            return AUCTION_MIN_BID;
        }

        return auction.startPrice * (BASIS_POINTS - priceDropBps) / BASIS_POINTS;
    }

    function getCurrentAuctionPrice(uint256 auctionId)
        external
        view
        returns (uint256)
    {
        return _getCurrentAuctionPrice(auctionId);
    }
}
```

---

## 3. Peg Stability Module (PSM)

### 3.1 ทำความรู้จัก PSM

PSM (Peg Stability Module) ช่วยรักษา peg ของ stablecoin:
- **1:1 Swap**: แลก stablecoin กับ USDC ในอัตรา 1:1 (ลบ fees)
- **Debt Ceiling**: จำกัดปริมาณ USDC ที่รับได้
- **Fees**: เก็บ fee ต่ำมาก (0.01-0.1%)
- **Arbitrage**: ราคา stablecoin เบี่ยงจาก $1 → arbitrageur ใช้ PSM

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title PegStabilityModule
 * @notice Peg Stability Module สำหรับรักษา 1:1 peg กับ USDC
 * @dev ผู้ใช้สามารถแลก stablecoin <-> USDC ในอัตรา 1:1 ลบ fee
 */
contract PegStabilityModule is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ================== Constants ==================
    uint256 public constant PRECISION = 1e18;
    uint256 public constant FEE_DENOMINATOR = 10000;

    // ================== State Variables ==================
    StableCoin public immutable stablecoin;
    IERC20 public immutable usdc;

    // PSM parameters
    uint256 public debtCeiling;      // USDC ceiling ใน PSM
    uint256 public currentDebt;      // USDC ที่รับมาแล้ว
    uint256 public buyFee;           // fee เมื่อซื้อ stablecoin (USDC -> stablecoin) bps
    uint256 public sellFee;          // fee เมื่อขาย stablecoin (stablecoin -> USDC) bps

    // Fee accumulation
    uint256 public accumulatedFees;

    // USDC scaling (USDC มี 6 decimals, stablecoin มี 18)
    uint256 public constant USDC_DECIMALS = 6;
    uint256 public constant STABLECOIN_DECIMALS = 18;
    uint256 public constant DECIMAL_DIFF = STABLECOIN_DECIMALS - USDC_DECIMALS; // 12

    // ================== Events ==================
    event BoughtStablecoin(
        address indexed buyer,
        uint256 usdcIn,
        uint256 stablecoinOut,
        uint256 fee
    );
    event SoldStablecoin(
        address indexed seller,
        uint256 stablecoinIn,
        uint256 usdcOut,
        uint256 fee
    );
    event DebtCeilingUpdated(uint256 oldCeiling, uint256 newCeiling);
    event FeeUpdated(uint256 buyFee, uint256 sellFee);
    event FeesWithdrawn(address indexed to, uint256 amount);

    // ================== Errors ==================
    error DebtCeilingExceeded(uint256 current, uint256 ceiling);
    error InsufficientUSDC(uint256 available, uint256 required);
    error ZeroAmount();
    error PSMPaused();

    // ================== State ==================
    bool public paused;

    modifier whenNotPaused() {
        if (paused) revert PSMPaused();
        _;
    }

    // ================== Constructor ==================
    constructor(
        address _stablecoin,
        address _usdc,
        uint256 _debtCeiling,
        uint256 _buyFee,
        uint256 _sellFee,
        address _owner
    ) Ownable(_owner) {
        stablecoin = StableCoin(_stablecoin);
        usdc = IERC20(_usdc);
        debtCeiling = _debtCeiling;
        buyFee = _buyFee;
        sellFee = _sellFee;
    }

    // ================== Core Functions ==================

    /**
     * @notice ซื้อ stablecoin ด้วย USDC (USDC → stablecoin)
     * @dev Swap USDC → stablecoin ที่ rate 1:1 ลบ buyFee
     * @param usdcAmount จำนวน USDC ที่จะใช้ (6 decimals)
     * @return stablecoinOut จำนวน stablecoin ที่ได้รับ (18 decimals)
     */
    function buyStablecoin(uint256 usdcAmount)
        external
        nonReentrant
        whenNotPaused
        returns (uint256 stablecoinOut)
    {
        if (usdcAmount == 0) revert ZeroAmount();

        // ตรวจสอบ debt ceiling
        if (currentDebt + usdcAmount > debtCeiling) {
            revert DebtCeilingExceeded(currentDebt + usdcAmount, debtCeiling);
        }

        // คำนวณ fee
        uint256 fee = (usdcAmount * buyFee) / FEE_DENOMINATOR;
        uint256 netUsdc = usdcAmount - fee;

        // Convert USDC (6 decimals) → stablecoin (18 decimals)
        stablecoinOut = netUsdc * (10 ** DECIMAL_DIFF);

        // รับ USDC
        usdc.safeTransferFrom(msg.sender, address(this), usdcAmount);

        // สะสม fee (ใน USDC)
        accumulatedFees += fee;
        currentDebt += usdcAmount;

        // Mint stablecoin ให้ผู้ซื้อ
        stablecoin.mint(msg.sender, stablecoinOut);

        emit BoughtStablecoin(msg.sender, usdcAmount, stablecoinOut, fee);
    }

    /**
     * @notice ขาย stablecoin เพื่อรับ USDC (stablecoin → USDC)
     * @dev Swap stablecoin → USDC ที่ rate 1:1 ลบ sellFee
     * @param stablecoinAmount จำนวน stablecoin ที่จะขาย (18 decimals)
     * @return usdcOut จำนวน USDC ที่ได้รับ (6 decimals)
     */
    function sellStablecoin(uint256 stablecoinAmount)
        external
        nonReentrant
        whenNotPaused
        returns (uint256 usdcOut)
    {
        if (stablecoinAmount == 0) revert ZeroAmount();

        // Convert stablecoin (18 decimals) → USDC (6 decimals)
        uint256 usdcEquivalent = stablecoinAmount / (10 ** DECIMAL_DIFF);

        // ตรวจสอบ USDC ที่มีใน PSM
        uint256 usdcBalance = usdc.balanceOf(address(this));
        if (usdcBalance < usdcEquivalent) {
            revert InsufficientUSDC(usdcBalance, usdcEquivalent);
        }

        // คำนวณ fee
        uint256 fee = (usdcEquivalent * sellFee) / FEE_DENOMINATOR;
        usdcOut = usdcEquivalent - fee;

        // Burn stablecoin
        stablecoin.burnFrom(msg.sender, stablecoinAmount);

        // อัพเดท debt tracking
        currentDebt = currentDebt >= usdcEquivalent 
            ? currentDebt - usdcEquivalent 
            : 0;

        // สะสม fee
        accumulatedFees += fee;

        // ส่ง USDC
        usdc.safeTransfer(msg.sender, usdcOut);

        emit SoldStablecoin(msg.sender, stablecoinAmount, usdcOut, fee);
    }

    // ================== Admin Functions ==================

    function setDebtCeiling(uint256 _debtCeiling) external onlyOwner {
        emit DebtCeilingUpdated(debtCeiling, _debtCeiling);
        debtCeiling = _debtCeiling;
    }

    function setFees(uint256 _buyFee, uint256 _sellFee) external onlyOwner {
        require(_buyFee <= 100, "Buy fee too high"); // max 1%
        require(_sellFee <= 100, "Sell fee too high"); // max 1%
        buyFee = _buyFee;
        sellFee = _sellFee;
        emit FeeUpdated(_buyFee, _sellFee);
    }

    function withdrawFees(address to) external onlyOwner {
        uint256 fees = accumulatedFees;
        accumulatedFees = 0;
        usdc.safeTransfer(to, fees);
        emit FeesWithdrawn(to, fees);
    }

    function setPaused(bool _paused) external onlyOwner {
        paused = _paused;
    }

    // ================== View Functions ==================

    function getAvailableToSell() external view returns (uint256) {
        return usdc.balanceOf(address(this)) - accumulatedFees;
    }

    function getAvailableToBuy() external view returns (uint256) {
        return debtCeiling > currentDebt ? debtCeiling - currentDebt : 0;
    }
}
```

---

## 4. Emergency Shutdown

### 4.1 ทำความรู้จัก Emergency Shutdown

Emergency Shutdown เป็น last resort สำหรับโปรโตคอล stablecoin:
- **Cage**: หยุดระบบและ fix ราคา ณ ขณะนั้น
- **Skim**: ผู้ที่มี overcollateralized vault สามารถดึง excess collateral ออก
- **Settle**: ผู้ถือ stablecoin redeem collateral ได้ในอัตรา fixed price

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title EmergencyShutdown
 * @notice Emergency Shutdown Module สำหรับ CDP Protocol
 * @dev Cage → Skim → Settle workflow
 */
contract EmergencyShutdown is Ownable {
    using SafeERC20 for IERC20;

    // ================== Types ==================
    struct ShutdownState {
        bool caged;                  // ระบบถูก cage หรือไม่
        uint256 cagePrice;           // ราคา collateral ที่ fix ตอน cage
        uint256 cageTimestamp;       // เวลาที่ cage
        uint256 totalDebtAtCage;     // total debt ณ เวลา cage
        uint256 totalCollateralAtCage; // total collateral ณ เวลา cage
        uint256 collateralPerStablecoin; // อัตราแลก หลัง skim
    }

    // ================== State Variables ==================
    CDPVault public immutable vault;
    StableCoin public immutable stablecoin;
    IERC20 public immutable collateralToken;
    IPriceFeed public immutable priceFeed;

    ShutdownState public shutdownState;

    // การ track ว่า vault ไหน skim แล้ว
    mapping(address => bool) public skimmed;
    // ใช้ stablecoin แล้วหรือยัง
    mapping(address => uint256) public settledAmount;

    // ================== Events ==================
    event SystemCaged(uint256 price, uint256 timestamp, uint256 totalDebt);
    event VaultSkimmed(
        address indexed owner,
        uint256 excessCollateral,
        uint256 remainingDebt
    );
    event StablecoinSettled(
        address indexed holder,
        uint256 stablecoinBurned,
        uint256 collateralReceived
    );

    // ================== Errors ==================
    error NotCaged();
    error AlreadyCaged();
    error VaultAlreadySkimmed(address owner);
    error SettlementTooEarly(uint256 cageTime, uint256 settleDelay);
    error NoCollateralRate();

    // ================== Constants ==================
    uint256 public constant SETTLE_DELAY = 3 days; // รอ 3 วันก่อน settle

    // ================== Constructor ==================
    constructor(
        address _vault,
        address _stablecoin,
        address _collateralToken,
        address _priceFeed,
        address _owner
    ) Ownable(_owner) {
        vault = CDPVault(_vault);
        stablecoin = StableCoin(_stablecoin);
        collateralToken = IERC20(_collateralToken);
        priceFeed = IPriceFeed(_priceFeed);
    }

    // ================== Step 1: Cage ==================

    /**
     * @notice Trigger Emergency Shutdown (Cage)
     * @dev หยุดการ mint/borrow ใหม่ และ fix ราคา collateral
     * @dev เฉพาะ governance/owner เรียกได้
     */
    function cage() external onlyOwner {
        if (shutdownState.caged) revert AlreadyCaged();

        // Fix ราคา ณ ปัจจุบัน
        uint256 currentPrice = priceFeed.getPrice();

        // หยุด vault operations
        stablecoin.triggerEmergencyStop();
        // vault.pause() ใน implementation จริง

        uint256 totalDebt = vault.totalDebt();
        uint256 totalCollateral = vault.totalCollateral();

        shutdownState = ShutdownState({
            caged: true,
            cagePrice: currentPrice,
            cageTimestamp: block.timestamp,
            totalDebtAtCage: totalDebt,
            totalCollateralAtCage: totalCollateral,
            collateralPerStablecoin: 0 // จะคำนวณหลัง skim phase
        });

        emit SystemCaged(currentPrice, block.timestamp, totalDebt);
    }

    // ================== Step 2: Skim ==================

    /**
     * @notice ดึง excess collateral ออกจาก overcollateralized vaults
     * @dev เจ้าของ vault ที่มี collateral มากกว่าหนี้ สามารถดึง excess ออกได้
     * @param vaultOwner address ของเจ้าของ vault
     */
    function skim(address vaultOwner) external {
        if (!shutdownState.caged) revert NotCaged();
        if (skimmed[vaultOwner]) revert VaultAlreadySkimmed(vaultOwner);

        (
            uint256 collateral,
            uint256 debt,
            ,
            ,

        ) = vault.getVault(vaultOwner);

        if (collateral == 0) return;

        // คำนวณ collateral value ณ cage price
        // collateralValue = collateral * cagePrice / decimals
        uint256 collateralDecimals = vault.collateralDecimals();
        uint256 collateralValue = (collateral * shutdownState.cagePrice)
            / (10 ** collateralDecimals);

        uint256 excessCollateral;
        uint256 debtCollateralEquiv;

        if (collateralValue > debt) {
            // มี excess collateral → คืนให้เจ้าของ
            uint256 excessValue = collateralValue - debt;
            excessCollateral = (excessValue * (10 ** collateralDecimals))
                / shutdownState.cagePrice;

            // ดึง excess collateral ออกจาก vault
            vault.seizeCollateral(vaultOwner, excessCollateral);
            collateralToken.safeTransfer(vaultOwner, excessCollateral);

            debtCollateralEquiv = collateral - excessCollateral;
        } else {
            // undercollateralized: ยึด collateral ทั้งหมด
            vault.seizeCollateral(vaultOwner, collateral);
            debtCollateralEquiv = collateral;
        }

        skimmed[vaultOwner] = true;

        emit VaultSkimmed(vaultOwner, excessCollateral, debt);
    }

    /**
     * @notice คำนวณ collateralPerStablecoin หลังจาก skim ครบทุก vault
     * @dev เรียกหลัง skim ทุก vault แล้ว
     */
    function calculateSettlementRate() external onlyOwner {
        if (!shutdownState.caged) revert NotCaged();

        // collateral ที่เหลือหลัง skim หารด้วย total debt
        uint256 remainingCollateral = collateralToken.balanceOf(address(this));
        uint256 totalSupply = stablecoin.totalSupply();

        if (totalSupply == 0) return;

        // อัตราแลก: กี่ collateral ต่อ stablecoin 1 หน่วย
        shutdownState.collateralPerStablecoin = (remainingCollateral * 1e18)
            / totalSupply;
    }

    // ================== Step 3: Settle ==================

    /**
     * @notice Redeem stablecoin เป็น collateral หลัง settlement
     * @dev เรียกได้หลัง SETTLE_DELAY นับจาก cage
     * @param stablecoinAmount จำนวน stablecoin ที่จะ redeem
     */
    function settle(uint256 stablecoinAmount) external {
        if (!shutdownState.caged) revert NotCaged();

        // ต้องรอ settle delay
        if (block.timestamp < shutdownState.cageTimestamp + SETTLE_DELAY) {
            revert SettlementTooEarly(
                shutdownState.cageTimestamp,
                shutdownState.cageTimestamp + SETTLE_DELAY
            );
        }

        if (shutdownState.collateralPerStablecoin == 0) {
            revert NoCollateralRate();
        }

        // Burn stablecoin
        stablecoin.burnFrom(msg.sender, stablecoinAmount);

        // คำนวณ collateral ที่จะได้รับ
        uint256 collateralOut = (stablecoinAmount * shutdownState.collateralPerStablecoin)
            / 1e18;

        settledAmount[msg.sender] += stablecoinAmount;

        // ส่ง collateral
        collateralToken.safeTransfer(msg.sender, collateralOut);

        emit StablecoinSettled(msg.sender, stablecoinAmount, collateralOut);
    }

    // ================== View Functions ==================

    function getSettlementRate() external view returns (uint256) {
        return shutdownState.collateralPerStablecoin;
    }

    function isCaged() external view returns (bool) {
        return shutdownState.caged;
    }

    function getCagePrice() external view returns (uint256) {
        return shutdownState.cagePrice;
    }
}
```

---

## 5. Architecture Comparison

### 5.1 เปรียบเทียบ DAI, FRAX, LUSD

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @notice ตัวอย่าง Architecture ของ Stablecoin ต่างๆ
 *
 * DAI (MakerDAO) - Multi-Collateral CDP:
 * - Overcollateralized 150%+ ด้วย crypto assets หลายชนิด
 * - Governance ผ่าน MKR token
 * - PSM สำหรับ peg stability กับ USDC
 * - DSR (Dai Savings Rate) สำหรับ demand-side
 *
 * FRAX - Partially Algorithmic:
 * - Fractional reserve: บางส่วน USDC-backed, บางส่วน algorithmic
 * - FXS (Frax Shares) เป็น governance และ volatility absorber
 * - Collateral Ratio ปรับอัตโนมัติตาม market conditions
 * - AMO (Algorithmic Market Operations) Controllers
 *
 * LUSD (Liquity) - Immutable, ETH-only:
 * - Immutable contracts: ไม่มี governance, ไม่สามารถ upgrade
 * - ETH เท่านั้น (ก่อน LUSD v2)
 * - Minimum collateral ratio 110% (aggressive)
 * - Stability Pool: LUSD holders provide insurance
 * - Redemption: LUSD → ETH ที่ face value
 */

/**
 * @title LiquityStyleVault
 * @notice ตัวอย่าง simplified Liquity-style vault
 * @dev Highlight: immutability, low collateral ratio, stability pool
 */
contract LiquityStyleVault {
    // ================== Constants (Immutable by design) ==================
    uint256 public constant MCR = 110;          // Minimum Collateral Ratio 110%
    uint256 public constant CCR = 150;          // Critical Collateral Ratio 150%
    uint256 public constant LUSD_GAS_COMP = 200e18;  // 200 LUSD gas compensation
    uint256 public constant PERCENT_DIVISOR = 200;   // Half percent for liquidation

    // ================== No Governance ==================
    // ไม่มี owner
    // ไม่มี pause
    // ไม่สามารถ upgrade
    // parameters ทั้งหมด fixed ณ deploy time

    // ================== Stability Pool Integration ==================

    /**
     * @notice Stability Pool สำหรับ absorb liquidations
     * @dev LUSD holders deposit เพื่อรับ ETH จาก liquidations
     */
    struct StabilityPoolDeposit {
        uint256 lusdAmount;     // LUSD ที่ deposit
        uint256 snapshotP;      // Product snapshot ณ เวลา deposit
        uint256 snapshotScale;  // Scale snapshot
        uint256 snapshotEpoch;  // Epoch snapshot
    }

    mapping(address => StabilityPoolDeposit) public deposits;
    uint256 public totalDeposits;
    uint256 public totalETHGains;

    // Product accumulator สำหรับคำนวณ gains
    uint256 public P = 1e18; // เริ่มที่ 1.0
    uint256 public currentScale;
    uint256 public currentEpoch;

    // Scale → Epoch → ETH/LUSD ratio
    mapping(uint256 => mapping(uint256 => uint256)) public epochToScaleToSum;

    /**
     * @notice Deposit LUSD เข้า Stability Pool
     */
    function depositToStabilityPool(uint256 amount) external {
        // บันทึก current P, scale, epoch
        deposits[msg.sender] = StabilityPoolDeposit({
            lusdAmount: amount,
            snapshotP: P,
            snapshotScale: currentScale,
            snapshotEpoch: currentEpoch
        });
        totalDeposits += amount;
        // Burn LUSD จาก depositor
    }

    /**
     * @notice Withdraw จาก Stability Pool พร้อม ETH gains
     */
    function withdrawFromStabilityPool(uint256 amount) external {
        StabilityPoolDeposit memory dep = deposits[msg.sender];

        // คำนวณ ETH gain
        uint256 ethGain = _computeETHGain(dep);

        // อัพเดท deposit
        deposits[msg.sender].lusdAmount -= amount;
        totalDeposits -= amount;

        // ส่ง LUSD กลับ + ETH gain
        // (mint LUSD กลับ + transfer ETH)
    }

    function _computeETHGain(StabilityPoolDeposit memory dep)
        internal
        view
        returns (uint256 ethGain)
    {
        // Simplified computation
        if (dep.snapshotEpoch < currentEpoch) {
            return (dep.lusdAmount * epochToScaleToSum[dep.snapshotEpoch][dep.snapshotScale])
                / 1e18;
        }

        uint256 firstPortion = epochToScaleToSum[currentEpoch][dep.snapshotScale]
            - epochToScaleToSum[dep.snapshotEpoch][dep.snapshotScale];
        uint256 secondPortion = epochToScaleToSum[currentEpoch][dep.snapshotScale + 1] / 1e9;

        ethGain = dep.lusdAmount * (firstPortion + secondPortion) / dep.snapshotP;
    }

    // ================== Redemption (LUSD → ETH) ==================

    /**
     * @notice Redeem LUSD เป็น ETH ที่ face value
     * @dev ราคา redemption = 1 LUSD = $1 worth of ETH
     * @dev Sorted troves ที่ ICR ต่ำสุดถูก redeem ก่อน
     */
    function redeemCollateral(
        uint256 lusdAmount,
        address firstRedemptionHint,
        address upperPartialRedemptionHint,
        address lowerPartialRedemptionHint,
        uint256 partialRedemptionHintNICR,
        uint256 maxIterations,
        uint256 maxFeePercentage
    ) external {
        // 1. ตรวจสอบ TCR > MCR
        // 2. ค้นหา troves ที่มี ICR ต่ำสุด
        // 3. Redeem LUSD จาก troves เหล่านั้น
        // 4. ส่ง ETH ให้ redeemer
        // 5. เก็บ redemption fee (0.5% - 5%)
    }
}

/**
 * @title FRAXStyle
 * @notice ตัวอย่าง Fractional-Algorithmic Stablecoin concept
 */
contract FRAXStyleConcept {
    // ================== Collateral Ratio ==================

    uint256 public collateralRatio = 850000; // 85% collateral backed
    uint256 public constant CR_STEP = 2500;  // ปรับทีละ 0.25%
    uint256 public constant REFRESH_COOLDOWN = 3600; // ปรับได้ทุก 1 ชั่วโมง

    uint256 public lastRefreshTime;

    // Price feeds
    uint256 public fraxPrice;  // ราคา FRAX ปัจจุบัน
    uint256 public fxsPrice;   // ราคา FXS ปัจจุบัน

    /**
     * @notice ปรับ Collateral Ratio อัตโนมัติตาม FRAX price
     * @dev ถ้า FRAX > $1 → ลด CR (ใช้ algorithmic มากขึ้น)
     *      ถ้า FRAX < $1 → เพิ่ม CR (ใช้ collateral มากขึ้น)
     */
    function refreshCollateralRatio() external {
        require(
            block.timestamp - lastRefreshTime >= REFRESH_COOLDOWN,
            "Too frequent"
        );

        // สมมติ getPrice() return ราคาใน 6 decimals
        uint256 currentFraxPrice = _getFraxPrice();

        if (currentFraxPrice > 1e6) {
            // FRAX trading above $1 → decrease CR
            collateralRatio = collateralRatio > CR_STEP
                ? collateralRatio - CR_STEP
                : 0;
        } else if (currentFraxPrice < 1e6) {
            // FRAX trading below $1 → increase CR
            collateralRatio = collateralRatio + CR_STEP <= 1e6
                ? collateralRatio + CR_STEP
                : 1e6;
        }

        lastRefreshTime = block.timestamp;
    }

    /**
     * @notice Mint FRAX ด้วย USDC + FXS
     * @dev ตาม CR ปัจจุบัน เช่น CR=85%: ใช้ USDC 85% + FXS 15% ของมูลค่า
     */
    function mintFrax(uint256 fraxAmount) external {
        uint256 usdcNeeded = (fraxAmount * collateralRatio) / 1e6;
        uint256 fxsNeeded = (fraxAmount * (1e6 - collateralRatio)) / fxsPrice;

        // Pull USDC และ FXS จาก user
        // Burn FXS
        // Mint FRAX
    }

    /**
     * @notice Redeem FRAX เป็น USDC + FXS
     */
    function redeemFrax(uint256 fraxAmount) external {
        uint256 usdcOut = (fraxAmount * collateralRatio) / 1e6;
        uint256 fxsOut = (fraxAmount * (1e6 - collateralRatio)) / fxsPrice;

        // Burn FRAX
        // ส่ง USDC กลับ
        // Mint FXS
    }

    function _getFraxPrice() internal view returns (uint256) {
        return fraxPrice; // ควรดึงจาก oracle จริงๆ
    }
}
```

---

## Workshop: สร้าง Complete Stablecoin System

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title CompleteStablecoinSystem
 * @notice Workshop: ระบบ stablecoin ครบวงจร
 * @dev รวม CDP + PSM + Liquidation + Emergency Shutdown
 */
contract CompleteStablecoinSystem is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ================== Immutable Protocol Parameters ==================
    uint256 public constant COLLATERAL_RATIO = 150;      // 150% minimum
    uint256 public constant LIQUIDATION_RATIO = 110;     // 110% liquidation
    uint256 public constant LIQUIDATION_BONUS = 500;     // 5% bonus
    uint256 public constant STABILITY_FEE = 200;         // 2% annual
    uint256 public constant PSM_BUY_FEE = 10;            // 0.1% PSM buy fee
    uint256 public constant PSM_SELL_FEE = 10;           // 0.1% PSM sell fee
    uint256 public constant BASIS = 10000;
    uint256 public constant SECONDS_PER_YEAR = 365 days;

    // ================== State ==================
    StableCoin public immutable stablecoin;
    IERC20 public immutable collateralToken;
    IERC20 public immutable usdc;
    IPriceFeed public immutable oracle;

    // CDP storage
    mapping(address => uint256) public collateral;
    mapping(address => uint256) public debt;
    mapping(address => uint256) public lastInterestUpdate;

    // PSM storage
    uint256 public psmDebtCeiling;
    uint256 public psmCurrentDebt;

    // Emergency
    bool public caged;
    uint256 public fixedPrice;

    // Globals
    uint256 public totalCollateral;
    uint256 public totalDebt;
    uint256 public collateralDecimals;

    // ================== Events ==================
    event Deposit(address indexed user, uint256 amount);
    event Withdraw(address indexed user, uint256 amount);
    event Mint(address indexed user, uint256 amount);
    event Repay(address indexed user, uint256 amount);
    event Liquidated(address indexed user, address indexed liquidator, uint256 debt, uint256 collateral);
    event PSMBuy(address indexed user, uint256 usdc, uint256 stablecoin);
    event PSMSell(address indexed user, uint256 stablecoin, uint256 usdc);
    event EmergencyCage(uint256 price);

    // ================== Constructor ==================
    constructor(
        address _stablecoin,
        address _collateral,
        address _usdc,
        address _oracle,
        uint256 _psmDebtCeiling,
        uint256 _collateralDecimals,
        address _owner
    ) Ownable(_owner) {
        stablecoin = StableCoin(_stablecoin);
        collateralToken = IERC20(_collateral);
        usdc = IERC20(_usdc);
        oracle = IPriceFeed(_oracle);
        psmDebtCeiling = _psmDebtCeiling;
        collateralDecimals = _collateralDecimals;
    }

    // ================== CDP Operations ==================

    function deposit(uint256 amount) external nonReentrant {
        require(!caged, "System caged");
        _accrueInterest(msg.sender);
        collateralToken.safeTransferFrom(msg.sender, address(this), amount);
        collateral[msg.sender] += amount;
        totalCollateral += amount;
        emit Deposit(msg.sender, amount);
    }

    function withdrawCollateral(uint256 amount) external nonReentrant {
        require(!caged, "System caged");
        _accrueInterest(msg.sender);
        require(collateral[msg.sender] >= amount, "Insufficient collateral");

        uint256 newCollateral = collateral[msg.sender] - amount;
        require(
            _healthFactor(newCollateral, debt[msg.sender]) >= COLLATERAL_RATIO,
            "Would breach minimum ratio"
        );

        collateral[msg.sender] = newCollateral;
        totalCollateral -= amount;
        collateralToken.safeTransfer(msg.sender, amount);
        emit Withdraw(msg.sender, amount);
    }

    function mintStablecoin(uint256 amount) external nonReentrant {
        require(!caged, "System caged");
        _accrueInterest(msg.sender);
        uint256 newDebt = debt[msg.sender] + amount;
        require(
            _healthFactor(collateral[msg.sender], newDebt) >= COLLATERAL_RATIO,
            "Insufficient collateral"
        );
        debt[msg.sender] = newDebt;
        totalDebt += amount;
        stablecoin.mint(msg.sender, amount);
        emit Mint(msg.sender, amount);
    }

    function repay(uint256 amount) external nonReentrant {
        _accrueInterest(msg.sender);
        uint256 actualAmount = amount > debt[msg.sender] ? debt[msg.sender] : amount;
        stablecoin.burnFrom(msg.sender, actualAmount);
        debt[msg.sender] -= actualAmount;
        totalDebt -= actualAmount;
        emit Repay(msg.sender, actualAmount);
    }

    // ================== Liquidation ==================

    function liquidate(address user, uint256 debtAmount) external nonReentrant {
        _accrueInterest(user);
        require(
            _healthFactor(collateral[user], debt[user]) < LIQUIDATION_RATIO,
            "Not liquidatable"
        );

        uint256 maxDebt = debt[user] / 2; // max 50%
        uint256 actualDebt = debtAmount > maxDebt ? maxDebt : debtAmount;

        // คำนวณ collateral seized
        uint256 price = oracle.getPrice();
        uint256 debtValue = actualDebt;
        uint256 collateralNeeded = (debtValue * (10 ** collateralDecimals)) / price;
        uint256 bonus = (collateralNeeded * LIQUIDATION_BONUS) / BASIS;
        uint256 totalSeized = collateralNeeded + bonus;

        if (totalSeized > collateral[user]) {
            totalSeized = collateral[user];
            actualDebt = debt[user];
        }

        stablecoin.burnFrom(msg.sender, actualDebt);
        debt[user] -= actualDebt;
        collateral[user] -= totalSeized;
        totalDebt -= actualDebt;
        totalCollateral -= totalSeized;

        collateralToken.safeTransfer(msg.sender, totalSeized);

        emit Liquidated(user, msg.sender, actualDebt, totalSeized);
    }

    // ================== PSM ==================

    function psmBuy(uint256 usdcAmount) external nonReentrant {
        require(!caged, "System caged");
        require(psmCurrentDebt + usdcAmount <= psmDebtCeiling, "PSM ceiling");

        uint256 fee = (usdcAmount * PSM_BUY_FEE) / BASIS;
        uint256 stablecoinOut = (usdcAmount - fee) * 1e12; // 6→18 decimals

        usdc.safeTransferFrom(msg.sender, address(this), usdcAmount);
        psmCurrentDebt += usdcAmount;
        stablecoin.mint(msg.sender, stablecoinOut);

        emit PSMBuy(msg.sender, usdcAmount, stablecoinOut);
    }

    function psmSell(uint256 stablecoinAmount) external nonReentrant {
        uint256 usdcEquivalent = stablecoinAmount / 1e12;
        uint256 fee = (usdcEquivalent * PSM_SELL_FEE) / BASIS;
        uint256 usdcOut = usdcEquivalent - fee;

        require(usdc.balanceOf(address(this)) >= usdcOut, "Insufficient USDC");

        stablecoin.burnFrom(msg.sender, stablecoinAmount);
        psmCurrentDebt = psmCurrentDebt >= usdcEquivalent ? psmCurrentDebt - usdcEquivalent : 0;
        usdc.safeTransfer(msg.sender, usdcOut);

        emit PSMSell(msg.sender, stablecoinAmount, usdcOut);
    }

    // ================== Emergency ==================

    function cage() external onlyOwner {
        require(!caged, "Already caged");
        caged = true;
        fixedPrice = oracle.getPrice();
        stablecoin.triggerEmergencyStop();
        emit EmergencyCage(fixedPrice);
    }

    function settle(uint256 stablecoinAmount) external nonReentrant {
        require(caged, "Not caged");
        require(fixedPrice > 0, "Price not fixed");

        uint256 collateralOut = (stablecoinAmount * (10 ** collateralDecimals)) / fixedPrice;

        stablecoin.burnFrom(msg.sender, stablecoinAmount);
        collateralToken.safeTransfer(msg.sender, collateralOut);
    }

    // ================== Internal ==================

    function _accrueInterest(address user) internal {
        if (debt[user] == 0 || lastInterestUpdate[user] == block.timestamp) return;
        uint256 elapsed = block.timestamp - lastInterestUpdate[user];
        uint256 interest = (debt[user] * STABILITY_FEE * elapsed) / (BASIS * SECONDS_PER_YEAR);
        debt[user] += interest;
        totalDebt += interest;
        lastInterestUpdate[user] = block.timestamp;
    }

    function _healthFactor(uint256 col, uint256 dbt) internal view returns (uint256) {
        if (dbt == 0) return type(uint256).max;
        uint256 price = oracle.getPrice();
        uint256 colValue = (col * price) / (10 ** collateralDecimals);
        return (colValue * 100) / dbt;
    }

    // ================== View ==================

    function healthFactor(address user) external view returns (uint256) {
        return _healthFactor(collateral[user], debt[user]);
    }

    function getSystemStats() external view returns (
        uint256 _totalCollateral,
        uint256 _totalDebt,
        uint256 _collateralRatio,
        bool _caged
    ) {
        _totalCollateral = totalCollateral;
        _totalDebt = totalDebt;
        uint256 price = caged ? fixedPrice : oracle.getPrice();
        uint256 totalColValue = (totalCollateral * price) / (10 ** collateralDecimals);
        _collateralRatio = totalDebt > 0 ? (totalColValue * 100) / totalDebt : 0;
        _caged = caged;
    }
}
```

---

## สรุป Part 74

- **CDP Mechanics**: Vault ต้องรักษา health factor >= 150%; คำนวณจาก `(collateral * price / debt) >= 150`; มี stability fee เพิ่มทบต้นตาม block.timestamp
- **Liquidation**: เมื่อ health factor < 110% vault ถูก liquidate; liquidator ได้รับ bonus 5%; Dutch auction เป็น fallback สำหรับ bad debt
- **PSM**: แลก stablecoin ↔ USDC ที่ rate 1:1 ลบ fee เล็กน้อย; มี debt ceiling ป้องกัน overexposure ต่อ USDC
- **Emergency Shutdown**: Cage → fix price → skim overcollateral → settle; stablecoin holders redeem collateral ที่ fixed rate หลัง 3 วัน
- **Architecture**: DAI ใช้ multi-collateral CDP + PSM + governance; FRAX ใช้ fractional reserve ปรับ CR อัตโนมัติ; LUSD ใช้ immutable ETH-only + stability pool

## Next: Part 75 - DeFi Risk Management
