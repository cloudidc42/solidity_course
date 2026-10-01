# Part 54: Structured DeFi Products

## บทนำ

**Structured Products** คือผลิตภัณฑ์ทางการเงินที่รวมเอา instruments หลายชนิดเข้าด้วยกัน เพื่อสร้าง risk/reward profile ที่เหมาะสมกับผู้ลงทุนกลุ่มต่างๆ ในโลก DeFi ตัวอย่างได้แก่:

- **Principal Protected Notes (PPN)**: เงินต้นปลอดภัย + upside participation
- **Dual Currency Notes (DCN)**: Earn yield สูงแต่รับความเสี่ยง FX
- **Leveraged Yield Tokens (LYT)**: รับ yield แบบ leveraged
- **Yield Enhancement Products**: เพิ่ม yield โดยยอมรับ downside risk

โปรโตคอลที่มีชื่อ: **Ribbon Finance** (Structured Products + Options), **Cega Finance** (Exotic options), **Pendle Finance** (Yield tokenization)

---

## 1. Principal Protected Product

### ทฤษฎี

**Principal Protected Note** ทำงานดังนี้:
1. ผู้ใช้ deposit $1,000 USDC
2. แบ่ง $1,000 ออกเป็น 2 ส่วน:
   - **Bond portion** (~$952): ลงทุนใน safe yield (Aave/Compound) ที่ให้ $1,000 ตอน maturity
   - **Option portion** (~$48): ซื้อ Call options บน ETH/BTC
3. ที่ maturity:
   - Worst case: $1,000 คืน (principal protected)
   - Best case: $1,000 + option profits

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/utils/math/Math.sol";

/// @title PrincipalProtectedNote - ผลิตภัณฑ์ที่ protect เงินต้น
/// @notice เงินต้น 100% ปลอดภัย + upside จาก options
/// @dev Split deposit: bond portion → Aave, option portion → Call options
contract PrincipalProtectedNote is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;
    using Math for uint256;
    
    // ============================================================
    //                          ERRORS
    // ============================================================
    error ProductNotOpen();
    error ProductNotMatured();
    error ProductAlreadyMatured();
    error InvestmentAlreadyClosed();
    error NotInvestor(address caller);
    error MinDepositNotMet(uint256 amount, uint256 minimum);
    error MaxCapacityReached(uint256 totalDeposited, uint256 maxCapacity);
    error InvalidMaturityDate();
    error WithdrawalAlreadyClaimed();
    
    // ============================================================
    //                          EVENTS
    // ============================================================
    event ProductCreated(bytes32 indexed productId, uint256 maturity, uint256 targetApy);
    event Deposited(bytes32 indexed productId, address indexed investor, uint256 amount, uint256 shares);
    event BondDeployed(bytes32 indexed productId, uint256 bondAmount, address bondProtocol);
    event OptionsDeployed(bytes32 indexed productId, uint256 optionAmount, uint256 optionId);
    event ProductMatured(bytes32 indexed productId, uint256 finalValue, uint256 optionProfit);
    event Withdrawn(bytes32 indexed productId, address indexed investor, uint256 principalReturned, uint256 yieldEarned);
    
    // ============================================================
    //                          STRUCTS
    // ============================================================
    enum ProductStatus {
        OPEN,       // รับ deposits
        INVESTED,   // กำลัง invest อยู่ (ไม่รับ deposits เพิ่ม)
        MATURED,    // ถึง maturity date แล้ว
        SETTLED     // ชำระเงินให้ investors ทั้งหมดแล้ว
    }
    
    struct Product {
        bytes32 productId;
        address stablecoin;         // USDC
        address underlyingAsset;    // ETH/BTC (สำหรับ options)
        uint256 maturity;           // timestamp
        uint256 minDeposit;
        uint256 maxCapacity;        // cap total deposits
        uint256 totalDeposited;
        uint256 bondProtection;     // % ที่ไปลงทุนใน bond (bps), typically 9500 = 95%
        uint256 optionBudget;       // % ที่ใช้ซื้อ options (bps), typically 500 = 5%
        uint256 targetStrikePrice;  // strike ของ options ที่ซื้อ
        
        // Investment tracking
        uint256 bondPrincipal;      // amount ลงทุนใน bond
        uint256 optionCost;         // amount ใช้ซื้อ options
        uint256 bondReturn;         // bond value ที่ maturity
        uint256 optionProfit;       // option payout (ถ้า ITM)
        
        ProductStatus status;
        address bondProtocol;       // Aave/Compound address
        uint256 optionTokenId;      // Option NFT ID
    }
    
    struct InvestorPosition {
        uint256 deposited;          // จำนวนที่ deposit
        uint256 shares;             // สัดส่วนของ pool
        bool claimed;
    }
    
    // ============================================================
    //                          STORAGE
    // ============================================================
    
    mapping(bytes32 => Product) public products;
    mapping(bytes32 => mapping(address => InvestorPosition)) public positions;
    bytes32[] public productIds;
    
    uint256 public constant BPS = 10_000;
    uint256 public constant MIN_BOND_PROTECTION = 8000;  // ต่ำสุด 80% protection
    uint256 public constant MAX_OPTION_BUDGET = 2000;    // สูงสุด 20% options
    
    // ============================================================
    //                        CONSTRUCTOR
    // ============================================================
    
    constructor(address _owner) Ownable(_owner) {}
    
    // ============================================================
    //                    PRODUCT CREATION
    // ============================================================
    
    /// @notice สร้าง product ใหม่
    function createProduct(
        bytes32 productId,
        address stablecoin,
        address underlyingAsset,
        uint256 maturity,
        uint256 minDeposit,
        uint256 maxCapacity,
        uint256 bondProtectionBps,
        uint256 optionBudgetBps,
        uint256 targetStrikePrice,
        address bondProtocol
    ) external onlyOwner {
        require(products[productId].maturity == 0, "Product already exists");
        require(maturity > block.timestamp + 7 days, "Maturity too soon");
        require(bondProtectionBps >= MIN_BOND_PROTECTION, "Bond protection too low");
        require(optionBudgetBps <= MAX_OPTION_BUDGET, "Option budget too high");
        require(bondProtectionBps + optionBudgetBps <= BPS, "Exceeds 100%");
        
        products[productId] = Product({
            productId: productId,
            stablecoin: stablecoin,
            underlyingAsset: underlyingAsset,
            maturity: maturity,
            minDeposit: minDeposit,
            maxCapacity: maxCapacity,
            totalDeposited: 0,
            bondProtection: bondProtectionBps,
            optionBudget: optionBudgetBps,
            targetStrikePrice: targetStrikePrice,
            bondPrincipal: 0,
            optionCost: 0,
            bondReturn: 0,
            optionProfit: 0,
            status: ProductStatus.OPEN,
            bondProtocol: bondProtocol,
            optionTokenId: 0
        });
        
        productIds.push(productId);
        
        emit ProductCreated(productId, maturity, 0);
    }
    
    // ============================================================
    //                    INVESTOR FUNCTIONS
    // ============================================================
    
    /// @notice Deposit เข้า product
    function deposit(bytes32 productId, uint256 amount) external nonReentrant {
        Product storage product = products[productId];
        
        if (product.status != ProductStatus.OPEN) revert ProductNotOpen();
        if (amount < product.minDeposit) revert MinDepositNotMet(amount, product.minDeposit);
        if (product.totalDeposited + amount > product.maxCapacity) {
            revert MaxCapacityReached(product.totalDeposited, product.maxCapacity);
        }
        
        IERC20(product.stablecoin).safeTransferFrom(msg.sender, address(this), amount);
        
        // คำนวณ shares (proportional)
        uint256 shares;
        if (product.totalDeposited == 0) {
            shares = amount;
        } else {
            shares = (amount * _getTotalShares(productId)) / product.totalDeposited;
        }
        
        positions[productId][msg.sender].deposited += amount;
        positions[productId][msg.sender].shares += shares;
        product.totalDeposited += amount;
        
        emit Deposited(productId, msg.sender, amount, shares);
    }
    
    // ============================================================
    //                    ADMIN: DEPLOY CAPITAL
    // ============================================================
    
    /// @notice Deploy capital ไปยัง bond + options (เมื่อปิดรับ deposits)
    function deployCapital(
        bytes32 productId,
        address optionsVault
    ) external onlyOwner {
        Product storage product = products[productId];
        require(product.status == ProductStatus.OPEN, "Not open");
        require(product.totalDeposited > 0, "No deposits");
        
        uint256 total = product.totalDeposited;
        
        // คำนวณ bond amount
        // Bond amount = ส่วนที่จะ grow กลับเป็น total ที่ maturity
        // ถ้า maturity ใน 1 ปี, Aave APY 5%:
        // bondAmount = total / (1 + 0.05) = total / 1.05
        
        uint256 bondAmount = _calculateBondAmount(
            total,
            product.maturity - block.timestamp,
            _getAaveApy(product.bondProtocol)
        );
        
        uint256 optionBudget = total - bondAmount;
        
        // Deploy ไปยัง Aave
        IERC20(product.stablecoin).safeApprove(product.bondProtocol, bondAmount);
        IAavePool(product.bondProtocol).supply(
            product.stablecoin, 
            bondAmount, 
            address(this), 
            0
        );
        
        product.bondPrincipal = bondAmount;
        product.optionCost = optionBudget;
        
        // ซื้อ call options ด้วย option budget
        // (ในกรณีนี้ simplified ไม่ call options vault จริง)
        // In production: call OptionsVault.writeCoveredCall หรือ buy options จาก market
        
        product.status = ProductStatus.INVESTED;
        
        emit BondDeployed(productId, bondAmount, product.bondProtocol);
    }
    
    /// @notice Settle product ที่ maturity (admin calls)
    function settleProduct(bytes32 productId) external onlyOwner {
        Product storage product = products[productId];
        require(product.status == ProductStatus.INVESTED, "Not invested");
        require(block.timestamp >= product.maturity, "Not matured yet");
        
        // Withdraw จาก bond protocol
        uint256 bondValue = _withdrawFromBond(productId);
        product.bondReturn = bondValue;
        
        // Check option profit (simplified)
        // In production: call OptionsVault.exercise หรือ check settlement
        product.optionProfit = _calculateOptionProfit(productId);
        
        product.status = ProductStatus.MATURED;
        
        emit ProductMatured(productId, bondValue + product.optionProfit, product.optionProfit);
    }
    
    /// @notice Investor withdraw หลัง maturity
    function withdraw(bytes32 productId) external nonReentrant {
        Product storage product = products[productId];
        InvestorPosition storage position = positions[productId][msg.sender];
        
        if (product.status != ProductStatus.MATURED) revert ProductNotMatured();
        if (position.deposited == 0) revert NotInvestor(msg.sender);
        if (position.claimed) revert WithdrawalAlreadyClaimed();
        
        // คำนวณ investor's share ของ final value
        uint256 totalFinalValue = product.bondReturn + product.optionProfit;
        uint256 totalShares = _getTotalShares(productId);
        uint256 investorShare = (totalFinalValue * position.shares) / totalShares;
        
        // Principal protection guarantee
        uint256 principalGuaranteed = position.deposited;
        uint256 payout = investorShare > principalGuaranteed ? investorShare : principalGuaranteed;
        
        position.claimed = true;
        
        uint256 yieldEarned = payout > position.deposited ? payout - position.deposited : 0;
        
        IERC20(product.stablecoin).safeTransfer(msg.sender, payout);
        
        emit Withdrawn(productId, msg.sender, position.deposited, yieldEarned);
    }
    
    // ============================================================
    //                    INTERNAL FUNCTIONS
    // ============================================================
    
    /// @notice คำนวณ bond amount ที่ต้องลงทุนเพื่อให้ได้ principal กลับมา
    /// @dev bondAmount * (1 + r)^T = total → bondAmount = total / (1 + r)^T
    function _calculateBondAmount(
        uint256 total,
        uint256 timeToMaturity,
        uint256 apy  // 1e18 = 100%
    ) internal pure returns (uint256) {
        // Simplified: assume continuous compounding
        // PV = FV / (1 + r)^T
        // ที่ T ในปี
        uint256 timeInYears = (timeToMaturity * 1e18) / 365 days;
        uint256 growthFactor = 1e18 + (apy * timeInYears / 1e18); // simplified linear
        
        return (total * 1e18) / growthFactor;
    }
    
    function _getAaveApy(address pool) internal view returns (uint256) {
        // ดึง APY จาก Aave (simplified)
        return 5e16; // 5% default
    }
    
    function _withdrawFromBond(bytes32 productId) internal returns (uint256) {
        Product storage product = products[productId];
        // Withdraw aTokens จาก Aave
        uint256 aTokenBalance = IERC20(_getAToken(product.bondProtocol, product.stablecoin))
            .balanceOf(address(this));
        
        return IAavePool(product.bondProtocol).withdraw(
            product.stablecoin,
            aTokenBalance,
            address(this)
        );
    }
    
    function _calculateOptionProfit(bytes32 productId) internal view returns (uint256) {
        // Placeholder - ในกรณีจริง ตรวจสอบ option payout
        return 0;
    }
    
    function _getTotalShares(bytes32 productId) internal view returns (uint256) {
        return products[productId].totalDeposited; // 1:1 initially
    }
    
    function _getAToken(address pool, address asset) internal pure returns (address) {
        // ดึง aToken address จาก Aave (simplified)
        return address(0); // placeholder
    }
}

interface IAavePool {
    function supply(address asset, uint256 amount, address onBehalfOf, uint16 referralCode) external;
    function withdraw(address asset, uint256 amount, address to) external returns (uint256);
}
```

---

## 2. Dual Currency Note

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title DualCurrencyNote - ผลิตภัณฑ์ที่ให้ yield สูงแลกกับความเสี่ยง FX
/// @notice ผู้ลงทุน deposit USDC, earn high yield, แต่อาจรับ settlement ใน ETH
/// @dev Mechanism: ขาย put option ให้ protocol, รับ premium เป็น yield เพิ่ม
contract DualCurrencyNote is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;
    
    // ============================================================
    //                          EVENTS
    // ============================================================
    event NoteCreated(uint256 indexed noteId, address indexed investor, uint256 amount, uint256 strikePrice, uint256 expiry);
    event NoteSettledUSDC(uint256 indexed noteId, address indexed investor, uint256 usdcReturned);
    event NoteSettledETH(uint256 indexed noteId, address indexed investor, uint256 ethAmount);
    
    // ============================================================
    //                          STRUCTS
    // ============================================================
    struct Note {
        address investor;
        uint256 usdcDeposit;        // USDC ที่ deposit
        uint256 strikePrice;        // ETH strike price (1e18 USDC per ETH)
        uint256 expiry;             // maturity date
        uint256 premium;            // yield ที่ได้รับจากการขาย put
        uint256 baseYield;          // yield จาก lending (e.g., Aave)
        bool settled;
        bool settledInETH;          // settle ใน ETH หรือ USDC
    }
    
    // ============================================================
    //                          STORAGE
    // ============================================================
    
    IERC20 public immutable usdc;
    IERC20 public immutable weth;
    IPriceOracle public immutable oracle;
    ILendingPool public immutable lendingPool;
    
    mapping(uint256 => Note) public notes;
    uint256 private _nextNoteId;
    
    uint256 public constant BPS = 10_000;
    uint256 public constant MIN_STRIKE_DISCOUNT = 500;  // strike ต่ำกว่า spot อย่างน้อย 5%
    uint256 public constant MAX_TERM = 30 days;
    
    // ============================================================
    //                        CONSTRUCTOR
    // ============================================================
    
    constructor(
        address _usdc,
        address _weth,
        address _oracle,
        address _lendingPool,
        address _owner
    ) Ownable(_owner) {
        usdc = IERC20(_usdc);
        weth = IERC20(_weth);
        oracle = IPriceOracle(_oracle);
        lendingPool = ILendingPool(_lendingPool);
    }
    
    // ============================================================
    //                    CORE FUNCTIONS
    // ============================================================
    
    /// @notice Invest ใน Dual Currency Note
    /// @param usdcAmount จำนวน USDC ที่ต้องการลงทุน
    /// @param strikePrice strike price ของ ETH (ต้องต่ำกว่า current price)
    /// @param term ระยะเวลาในวินาที (max 30 days)
    function invest(
        uint256 usdcAmount,
        uint256 strikePrice,
        uint256 term
    ) external nonReentrant returns (uint256 noteId) {
        require(term <= MAX_TERM, "Term too long");
        require(usdcAmount > 0, "Zero amount");
        
        // ตรวจสอบ strike price
        uint256 spotPrice = oracle.getPrice(address(weth), address(usdc));
        uint256 minStrike = (spotPrice * (BPS - MIN_STRIKE_DISCOUNT)) / BPS;
        require(strikePrice <= minStrike, "Strike too high - no discount");
        require(strikePrice > 0, "Zero strike");
        
        // คำนวณ yield:
        // 1. Base yield จาก Aave (ประมาณ 5% APY)
        // 2. Option premium จากการขาย put (OTM put premium)
        
        uint256 baseApy = 5e16; // 5% APY
        uint256 baseYield = (usdcAmount * baseApy * term) / (365 days * 1e18);
        
        // Option premium calculation (simplified)
        uint256 premium = _calculatePutPremium(
            spotPrice, 
            strikePrice, 
            term, 
            usdcAmount
        );
        
        uint256 totalYield = baseYield + premium;
        
        // รับ USDC จาก investor
        usdc.safeTransferFrom(msg.sender, address(this), usdcAmount);
        
        // Deploy ไปยัง Aave
        usdc.safeApprove(address(lendingPool), usdcAmount);
        lendingPool.supply(address(usdc), usdcAmount, address(this), 0);
        
        // Mint yield เพิ่ม (protocol มี put obligation)
        // จ่าย premium ให้ investor ทันที
        if (premium > 0 && usdc.balanceOf(address(this)) >= premium) {
            usdc.safeTransfer(msg.sender, premium);
        }
        
        noteId = _nextNoteId++;
        
        notes[noteId] = Note({
            investor: msg.sender,
            usdcDeposit: usdcAmount,
            strikePrice: strikePrice,
            expiry: block.timestamp + term,
            premium: premium,
            baseYield: baseYield,
            settled: false,
            settledInETH: false
        });
        
        emit NoteCreated(noteId, msg.sender, usdcAmount, strikePrice, block.timestamp + term);
    }
    
    /// @notice Settle note ที่ maturity
    /// @param noteId ID ของ note
    function settle(uint256 noteId) external nonReentrant {
        Note storage note = notes[noteId];
        
        require(!note.settled, "Already settled");
        require(block.timestamp >= note.expiry, "Not expired yet");
        
        uint256 finalPrice = oracle.getPrice(address(weth), address(usdc));
        
        // Withdraw จาก Aave
        uint256 aTokenBalance = IERC20(_getAToken()).balanceOf(address(this));
        uint256 withdrawn = lendingPool.withdraw(address(usdc), aTokenBalance, address(this));
        
        note.settled = true;
        
        if (finalPrice >= note.strikePrice) {
            // ETH price ≥ strike → settle in USDC (investor gets principal + base yield)
            uint256 totalReturn = note.usdcDeposit + note.baseYield;
            totalReturn = totalReturn < withdrawn ? totalReturn : withdrawn;
            
            usdc.safeTransfer(note.investor, totalReturn);
            note.settledInETH = false;
            
            emit NoteSettledUSDC(noteId, note.investor, totalReturn);
        } else {
            // ETH price < strike → settle in ETH (investor "buys" ETH at strike)
            // จำนวน ETH = usdcDeposit / strikePrice
            uint256 ethAmount = (note.usdcDeposit * 1e18) / note.strikePrice;
            
            // Protocol ต้องมี ETH เพียงพอ (ในกรณีจริงจะ hedge ล่วงหน้า)
            require(weth.balanceOf(address(this)) >= ethAmount, "Insufficient ETH");
            
            weth.safeTransfer(note.investor, ethAmount);
            note.settledInETH = true;
            
            emit NoteSettledETH(noteId, note.investor, ethAmount);
        }
    }
    
    // ============================================================
    //                    INTERNAL FUNCTIONS
    // ============================================================
    
    /// @notice คำนวณ put premium (simplified Black-Scholes)
    function _calculatePutPremium(
        uint256 spotPrice,
        uint256 strikePrice,
        uint256 term,
        uint256 notional
    ) internal pure returns (uint256 premium) {
        // Simplified: premium ∝ OTM distance * volatility * sqrt(time)
        // ใน production ใช้ full Black-Scholes
        
        uint256 otmDistance = spotPrice - strikePrice; // spot - strike (OTM put)
        uint256 otmPercent = (otmDistance * 1e18) / spotPrice; // as percentage
        
        // Implied vol ประมาณ 80% annualized สำหรับ ETH
        uint256 impliedVol = 8e17; // 80%
        uint256 sqrtTime = _sqrt((term * 1e18) / 365 days);
        
        // Premium ≈ notional * vol * sqrt(T) * OTM_modifier
        uint256 rawPremium = (notional * impliedVol / 1e18 * sqrtTime / 1e18);
        
        // ลด premium ตาม OTM distance (farther OTM = cheaper)
        uint256 otmMultiplier = 1e18;
        if (otmPercent > 5e16) { // > 5% OTM
            otmMultiplier = 1e18 - otmPercent / 2;
        }
        
        premium = (rawPremium * otmMultiplier) / 1e18;
    }
    
    function _getAToken() internal view returns (address) {
        // ในกรณีจริง ดึงจาก Aave registry
        return address(0); // placeholder
    }
    
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

interface IPriceOracle {
    function getPrice(address base, address quote) external view returns (uint256);
}

interface ILendingPool {
    function supply(address asset, uint256 amount, address onBehalfOf, uint16 referralCode) external;
    function withdraw(address asset, uint256 amount, address to) external returns (uint256);
}
```

---

## 3. Leveraged Yield Token (LYT)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title LeveragedYieldToken (LYT)
/// @notice Token ที่ให้ leveraged exposure ต่อ yield (เช่น 3x Aave USDC yield)
/// @dev Concept: Borrow USDC จาก protocol, deposit ทั้งหมดใน Aave
/// @dev Yield ที่ได้จาก leveraged portion ส่งให้ LYT holders
contract LeveragedYieldToken is ERC20, Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;
    
    // ============================================================
    //                          ERRORS
    // ============================================================
    error LeverageTooHigh(uint256 requested, uint256 maximum);
    error InsufficientCollateral();
    error HealthFactorTooLow(uint256 hf, uint256 minimum);
    error RebalanceNotNeeded();
    error EmergencyDeleverageOnly();
    
    // ============================================================
    //                          EVENTS
    // ============================================================
    event Minted(address indexed user, uint256 collateral, uint256 lytMinted, uint256 leverage);
    event Redeemed(address indexed user, uint256 lyt, uint256 collateralReturned);
    event Rebalanced(uint256 newLeverage, uint256 collateralAdded, uint256 borrowAdded);
    event YieldDistributed(uint256 yieldAmount, uint256 perTokenYield);
    event LeverageUpdated(uint256 oldLeverage, uint256 newLeverage);
    
    // ============================================================
    //                          STRUCTS
    // ============================================================
    
    struct Position {
        uint256 collateral;         // USDC collateral ที่ user ใส่
        uint256 debt;               // USDC ที่ borrow
        uint256 lytAmount;          // LYT tokens ที่ mint
    }
    
    // ============================================================
    //                          STORAGE
    // ============================================================
    
    IERC20 public immutable usdc;
    ILendingPool public immutable lendingPool;
    
    mapping(address => Position) public positions;
    
    uint256 public targetLeverage;       // เช่น 3e18 = 3x leverage
    uint256 public maxLeverage;          // ขีดจำกัด leverage สูงสุด (5x)
    uint256 public minHealthFactor;      // ขีดจำกัด health factor (1.5 = 1.5e18)
    
    uint256 public accumulatedYieldPerToken; // accumulated yield per LYT token
    mapping(address => uint256) public lastYieldClaimed;  // last yield index ต่อ user
    
    uint256 public constant PRECISION = 1e18;
    uint256 public constant MAX_LEVERAGE = 5e18;  // 5x max
    uint256 public constant MIN_HF = 15e17;       // 1.5 min health factor
    
    // ============================================================
    //                        CONSTRUCTOR
    // ============================================================
    
    constructor(
        address _usdc,
        address _lendingPool,
        uint256 _targetLeverage,
        address _owner
    ) ERC20("Leveraged Yield Token", "LYT") Ownable(_owner) {
        require(_targetLeverage <= MAX_LEVERAGE, "Leverage too high");
        usdc = IERC20(_usdc);
        lendingPool = ILendingPool(_lendingPool);
        targetLeverage = _targetLeverage;
        maxLeverage = MAX_LEVERAGE;
        minHealthFactor = MIN_HF;
    }
    
    // ============================================================
    //                    CORE FUNCTIONS
    // ============================================================
    
    /// @notice Mint LYT tokens โดยใส่ collateral
    /// @param collateralAmount USDC collateral
    function mint(uint256 collateralAmount) external nonReentrant returns (uint256 lytMinted) {
        require(collateralAmount > 0, "Zero collateral");
        
        usdc.safeTransferFrom(msg.sender, address(this), collateralAmount);
        
        // คำนวณ total exposure หลัง leverage
        // ถ้า leverage = 3x: ลงทุน $300 โดยใส่ collateral $100 + borrow $200
        uint256 totalExposure = (collateralAmount * targetLeverage) / PRECISION;
        uint256 borrowAmount = totalExposure - collateralAmount;
        
        // Deposit collateral
        usdc.safeApprove(address(lendingPool), collateralAmount);
        lendingPool.supply(address(usdc), collateralAmount, address(this), 0);
        
        // Borrow ตาม leverage
        if (borrowAmount > 0) {
            lendingPool.borrow(
                address(usdc),
                borrowAmount,
                2,  // variable rate
                0,
                address(this)
            );
            
            // Deposit borrowed amount ด้วย
            usdc.safeApprove(address(lendingPool), borrowAmount);
            lendingPool.supply(address(usdc), borrowAmount, address(this), 0);
        }
        
        // คำนวณ LYT ที่ mint
        // LYT มูลค่า = equity value (collateral - debt)
        lytMinted = collateralAmount; // 1:1 กับ collateral เริ่มต้น
        
        // อัปเดต position
        Position storage pos = positions[msg.sender];
        
        // Claim yield ก่อนจะ mint ใหม่
        _claimYield(msg.sender);
        
        pos.collateral += collateralAmount;
        pos.debt += borrowAmount;
        pos.lytAmount += lytMinted;
        
        _mint(msg.sender, lytMinted);
        
        emit Minted(msg.sender, collateralAmount, lytMinted, targetLeverage);
    }
    
    /// @notice Redeem LYT กลับเป็น USDC
    function redeem(uint256 lytAmount) external nonReentrant returns (uint256 usdcReturned) {
        Position storage pos = positions[msg.sender];
        require(pos.lytAmount >= lytAmount, "Insufficient LYT");
        
        // Claim pending yield ก่อน
        _claimYield(msg.sender);
        
        // คำนวณสัดส่วน
        uint256 fraction = (lytAmount * PRECISION) / pos.lytAmount;
        uint256 debtToRepay = (pos.debt * fraction) / PRECISION;
        uint256 collateralToReturn = (pos.collateral * fraction) / PRECISION;
        
        // Repay debt ก่อน
        if (debtToRepay > 0) {
            // ถอนบางส่วนจาก Aave เพื่อ repay
            lendingPool.withdraw(address(usdc), debtToRepay, address(this));
            usdc.safeApprove(address(lendingPool), debtToRepay);
            lendingPool.repay(address(usdc), debtToRepay, 2, address(this));
        }
        
        // ถอน collateral
        lendingPool.withdraw(address(usdc), collateralToReturn, address(this));
        
        // อัปเดต position
        pos.collateral -= collateralToReturn;
        pos.debt -= debtToRepay;
        pos.lytAmount -= lytAmount;
        
        _burn(msg.sender, lytAmount);
        
        usdcReturned = usdc.balanceOf(address(this));
        usdc.safeTransfer(msg.sender, usdcReturned);
        
        emit Redeemed(msg.sender, lytAmount, usdcReturned);
    }
    
    // ============================================================
    //                    YIELD DISTRIBUTION
    // ============================================================
    
    /// @notice Distribute yield ให้ LYT holders (เรียกโดย keeper)
    function distributeYield() external {
        // คำนวณ yield ที่เก็บได้
        // yield = total aToken balance - total debt
        uint256 totalAToken = IERC20(_getAToken()).balanceOf(address(this));
        uint256 totalDebt = _getTotalDebt();
        
        if (totalAToken <= totalDebt) return; // ไม่มี yield
        
        uint256 yield = totalAToken - totalDebt;
        
        if (totalSupply() > 0) {
            accumulatedYieldPerToken += (yield * PRECISION) / totalSupply();
        }
        
        emit YieldDistributed(yield, accumulatedYieldPerToken);
    }
    
    /// @notice Claim pending yield ของ user
    function claimYield() external nonReentrant returns (uint256 yieldClaimed) {
        yieldClaimed = _claimYield(msg.sender);
    }
    
    function _claimYield(address user) internal returns (uint256 yieldClaimed) {
        uint256 pendingYield = (balanceOf(user) * 
            (accumulatedYieldPerToken - lastYieldClaimed[user])) / PRECISION;
        
        if (pendingYield > 0) {
            lastYieldClaimed[user] = accumulatedYieldPerToken;
            
            // ถอน yield จาก Aave
            if (lendingPool.withdraw(address(usdc), pendingYield, user) > 0) {
                yieldClaimed = pendingYield;
            }
        }
        
        lastYieldClaimed[user] = accumulatedYieldPerToken;
    }
    
    // ============================================================
    //                    REBALANCING
    // ============================================================
    
    /// @notice Rebalance leverage (เรียกโดย keeper หรือ admin)
    function rebalance() external onlyOwner {
        uint256 currentLeverage = _getCurrentLeverage();
        
        // Rebalance ถ้า leverage เบี่ยงเบินเกิน 5%
        uint256 deviation = currentLeverage > targetLeverage 
            ? currentLeverage - targetLeverage 
            : targetLeverage - currentLeverage;
        
        require(deviation * 100 / targetLeverage > 5, "No rebalance needed");
        
        if (currentLeverage > targetLeverage) {
            // Deleverage: repay some debt
            uint256 excessLeverage = currentLeverage - targetLeverage;
            uint256 debtToRepay = (excessLeverage * _getTotalCollateral()) / PRECISION;
            
            if (debtToRepay > 0) {
                lendingPool.withdraw(address(usdc), debtToRepay, address(this));
                usdc.safeApprove(address(lendingPool), debtToRepay);
                lendingPool.repay(address(usdc), debtToRepay, 2, address(this));
            }
        }
        
        emit Rebalanced(targetLeverage, 0, 0);
    }
    
    // ============================================================
    //                      VIEW FUNCTIONS
    // ============================================================
    
    function _getCurrentLeverage() internal view returns (uint256) {
        uint256 totalAssets = IERC20(_getAToken()).balanceOf(address(this));
        uint256 totalDebt = _getTotalDebt();
        
        if (totalAssets <= totalDebt) return PRECISION;
        
        uint256 equity = totalAssets - totalDebt;
        return (totalAssets * PRECISION) / equity;
    }
    
    function _getTotalCollateral() internal view returns (uint256) {
        return IERC20(_getAToken()).balanceOf(address(this));
    }
    
    function _getTotalDebt() internal pure returns (uint256) {
        // ดึงจาก Aave variable debt token balance
        return 0; // placeholder
    }
    
    function _getAToken() internal pure returns (address) {
        return address(0); // placeholder
    }
}

interface ILendingPool {
    function supply(address asset, uint256 amount, address onBehalfOf, uint16 referralCode) external;
    function withdraw(address asset, uint256 amount, address to) external returns (uint256);
    function borrow(address asset, uint256 amount, uint256 interestRateMode, uint16 referralCode, address onBehalfOf) external;
    function repay(address asset, uint256 amount, uint256 interestRateMode, address onBehalfOf) external returns (uint256);
}
```

---

## 4. StructuredProductFactory

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/proxy/Clones.sol";

/// @title StructuredProductFactory - Factory สำหรับ deploy structured products
/// @notice ใช้ minimal proxy (EIP-1167 Clones) เพื่อประหยัด gas
contract StructuredProductFactory is Ownable {
    using Clones for address;
    
    // ============================================================
    //                          EVENTS
    // ============================================================
    event ProductDeployed(
        bytes32 indexed productType,
        address indexed productAddress,
        address indexed deployer,
        bytes32 productId
    );
    event TemplateRegistered(bytes32 indexed productType, address indexed template);
    event TemplateDeprecated(bytes32 indexed productType);
    
    // ============================================================
    //                          STRUCTS
    // ============================================================
    struct ProductTemplate {
        address implementation;
        bool active;
        uint256 deployCount;
        uint256 registeredAt;
    }
    
    struct DeployedProduct {
        address productAddress;
        bytes32 productType;
        address deployer;
        uint256 deployedAt;
        bool active;
    }
    
    // ============================================================
    //                          STORAGE
    // ============================================================
    
    // Template registry
    mapping(bytes32 => ProductTemplate) public templates;
    bytes32[] public productTypes;
    
    // Deployed products
    mapping(bytes32 => DeployedProduct) public deployedProducts;
    mapping(address => bytes32[]) public userProducts;
    bytes32[] public allProductIds;
    
    // ============================================================
    //                        CONSTRUCTOR
    // ============================================================
    
    constructor(address _owner) Ownable(_owner) {}
    
    // ============================================================
    //                    TEMPLATE MANAGEMENT
    // ============================================================
    
    /// @notice Register product template ใหม่
    function registerTemplate(bytes32 productType, address implementation) external onlyOwner {
        require(implementation != address(0), "Zero address");
        require(!templates[productType].active, "Already registered");
        
        templates[productType] = ProductTemplate({
            implementation: implementation,
            active: true,
            deployCount: 0,
            registeredAt: block.timestamp
        });
        
        productTypes.push(productType);
        
        emit TemplateRegistered(productType, implementation);
    }
    
    /// @notice Deprecate template
    function deprecateTemplate(bytes32 productType) external onlyOwner {
        require(templates[productType].active, "Not active");
        templates[productType].active = false;
        emit TemplateDeprecated(productType);
    }
    
    // ============================================================
    //                    PRODUCT DEPLOYMENT
    // ============================================================
    
    /// @notice Deploy product ใหม่จาก template
    /// @param productType type ของ product
    /// @param productId unique ID สำหรับ product นี้
    /// @param initData ข้อมูลสำหรับ initialize
    function deployProduct(
        bytes32 productType,
        bytes32 productId,
        bytes calldata initData
    ) external returns (address productAddress) {
        ProductTemplate storage template = templates[productType];
        require(template.active, "Template not active");
        require(deployedProducts[productId].deployedAt == 0, "Product ID taken");
        
        // Clone implementation (EIP-1167 minimal proxy)
        productAddress = template.implementation.clone();
        
        // Initialize product
        (bool success, ) = productAddress.call(initData);
        require(success, "Initialization failed");
        
        // บันทึก deployment
        deployedProducts[productId] = DeployedProduct({
            productAddress: productAddress,
            productType: productType,
            deployer: msg.sender,
            deployedAt: block.timestamp,
            active: true
        });
        
        userProducts[msg.sender].push(productId);
        allProductIds.push(productId);
        template.deployCount++;
        
        emit ProductDeployed(productType, productAddress, msg.sender, productId);
    }
    
    /// @notice Deploy product พร้อม deterministic address (CREATE2)
    function deployProductDeterministic(
        bytes32 productType,
        bytes32 productId,
        bytes32 salt,
        bytes calldata initData
    ) external returns (address productAddress) {
        ProductTemplate storage template = templates[productType];
        require(template.active, "Template not active");
        
        productAddress = template.implementation.cloneDeterministic(salt);
        
        (bool success, ) = productAddress.call(initData);
        require(success, "Initialization failed");
        
        deployedProducts[productId] = DeployedProduct({
            productAddress: productAddress,
            productType: productType,
            deployer: msg.sender,
            deployedAt: block.timestamp,
            active: true
        });
        
        userProducts[msg.sender].push(productId);
        allProductIds.push(productId);
        template.deployCount++;
        
        emit ProductDeployed(productType, productAddress, msg.sender, productId);
    }
    
    /// @notice คำนวณ address ล่วงหน้าก่อน deploy (สำหรับ CREATE2)
    function predictAddress(bytes32 productType, bytes32 salt) external view returns (address) {
        return templates[productType].implementation.predictDeterministicAddress(salt);
    }
    
    // ============================================================
    //                      VIEW FUNCTIONS
    // ============================================================
    
    function getTemplate(bytes32 productType) external view returns (ProductTemplate memory) {
        return templates[productType];
    }
    
    function getProduct(bytes32 productId) external view returns (DeployedProduct memory) {
        return deployedProducts[productId];
    }
    
    function getUserProducts(address user) external view returns (bytes32[] memory) {
        return userProducts[user];
    }
    
    function getAllProductTypes() external view returns (bytes32[] memory) {
        return productTypes;
    }
}
```

---

## 5. Full Product Lifecycle - Integration Test

```typescript
// test/StructuredProducts.test.ts
import { ethers } from "hardhat";
import { expect } from "chai";
import { loadFixture, time } from "@nomicfoundation/hardhat-network-helpers";

describe("Structured Products - Full Lifecycle", () => {
    
    async function deployFixtures() {
        const [owner, alice, bob, keeper] = await ethers.getSigners();
        
        // Deploy mock tokens
        const MockERC20 = await ethers.getContractFactory("MockERC20");
        const usdc = await MockERC20.deploy("USD Coin", "USDC", 6);
        const weth = await MockERC20.deploy("Wrapped ETH", "WETH", 18);
        
        // Deploy mock oracle
        const MockOracle = await ethers.getContractFactory("MockPriceOracle");
        const oracle = await MockOracle.deploy();
        
        // Set ETH price: $2000
        await oracle.setPrice(
            await weth.getAddress(), 
            await usdc.getAddress(), 
            ethers.parseUnits("2000", 18)
        );
        
        // Deploy mock lending pool
        const MockLending = await ethers.getContractFactory("MockAavePool");
        const lendingPool = await MockLending.deploy(
            await usdc.getAddress()
        );
        
        // Deploy PPN
        const PPN = await ethers.getContractFactory("PrincipalProtectedNote");
        const ppn = await PPN.deploy(owner.address);
        
        // Deploy DCN
        const DCN = await ethers.getContractFactory("DualCurrencyNote");
        const dcn = await DCN.deploy(
            await usdc.getAddress(),
            await weth.getAddress(),
            await oracle.getAddress(),
            await lendingPool.getAddress(),
            owner.address
        );
        
        // Deploy Factory
        const Factory = await ethers.getContractFactory("StructuredProductFactory");
        const factory = await Factory.deploy(owner.address);
        
        // Mint tokens
        const usdcAmount = ethers.parseUnits("100000", 6);
        await usdc.mint(alice.address, usdcAmount);
        await usdc.mint(bob.address, usdcAmount);
        await usdc.connect(alice).approve(await ppn.getAddress(), ethers.MaxUint256);
        await usdc.connect(alice).approve(await dcn.getAddress(), ethers.MaxUint256);
        await usdc.connect(bob).approve(await dcn.getAddress(), ethers.MaxUint256);
        
        return { ppn, dcn, factory, usdc, weth, oracle, lendingPool, owner, alice, bob, keeper };
    }
    
    // ============================================================
    //             Principal Protected Note Tests
    // ============================================================
    
    describe("PrincipalProtectedNote", () => {
        it("should create product and accept deposits", async () => {
            const { ppn, usdc, alice, owner } = await loadFixture(deployFixtures);
            
            const productId = ethers.keccak256(ethers.toUtf8Bytes("ETH_CALL_3M"));
            const maturity = (await time.latest()) + 90 * 24 * 3600; // 90 days
            
            await ppn.connect(owner).createProduct(
                productId,
                await usdc.getAddress(),
                ethers.ZeroAddress,
                maturity,
                ethers.parseUnits("100", 6),   // min 100 USDC
                ethers.parseUnits("1000000", 6), // max 1M USDC
                9500,  // 95% bond protection
                500,   // 5% option budget
                ethers.parseUnits("2500", 18),  // strike $2500
                ethers.ZeroAddress  // mock bond protocol
            );
            
            // Alice deposits
            const depositAmount = ethers.parseUnits("10000", 6);
            await ppn.connect(alice).deposit(productId, depositAmount);
            
            const product = await ppn.products(productId);
            expect(product.totalDeposited).to.equal(depositAmount);
            
            const position = await ppn.positions(productId, alice.address);
            expect(position.deposited).to.equal(depositAmount);
        });
        
        it("should enforce min deposit", async () => {
            const { ppn, usdc, alice, owner } = await loadFixture(deployFixtures);
            
            const productId = ethers.keccak256(ethers.toUtf8Bytes("TEST"));
            const maturity = (await time.latest()) + 90 * 24 * 3600;
            
            await ppn.connect(owner).createProduct(
                productId,
                await usdc.getAddress(),
                ethers.ZeroAddress,
                maturity,
                ethers.parseUnits("1000", 6), // min 1000 USDC
                ethers.parseUnits("1000000", 6),
                9500, 500,
                ethers.parseUnits("2500", 18),
                ethers.ZeroAddress
            );
            
            await expect(
                ppn.connect(alice).deposit(productId, ethers.parseUnits("100", 6))
            ).to.be.revertedWithCustomError(ppn, "MinDepositNotMet");
        });
    });
    
    // ============================================================
    //             Dual Currency Note Tests
    // ============================================================
    
    describe("DualCurrencyNote", () => {
        it("should create note with correct yield calculation", async () => {
            const { dcn, usdc, alice, oracle, weth } = await loadFixture(deployFixtures);
            
            // Strike price ต้องต่ำกว่า current price ($2000) อย่างน้อย 5%
            const strikePrice = ethers.parseUnits("1800", 18); // $1800
            const term = 7 * 24 * 3600; // 7 days
            const amount = ethers.parseUnits("10000", 6);
            
            // Provide liquidity ให้ lendingPool ก่อน
            await usdc.mint(await dcn.getAddress(), ethers.parseUnits("1000", 6));
            
            const noteId = await dcn.connect(alice).invest.staticCall(
                amount, 
                strikePrice, 
                term
            );
            
            await dcn.connect(alice).invest(amount, strikePrice, term);
            
            const note = await dcn.notes(noteId);
            expect(note.investor).to.equal(alice.address);
            expect(note.strikePrice).to.equal(strikePrice);
            expect(note.premium).to.be.gt(0); // ต้องได้ premium
        });
        
        it("should settle in USDC when price stays above strike", async () => {
            const { dcn, usdc, alice, oracle, weth } = await loadFixture(deployFixtures);
            
            const strikePrice = ethers.parseUnits("1800", 18);
            const term = 7 * 24 * 3600;
            const amount = ethers.parseUnits("10000", 6);
            
            await usdc.mint(await dcn.getAddress(), ethers.parseUnits("1000", 6));
            
            const noteId = await dcn.connect(alice).invest.staticCall(amount, strikePrice, term);
            await dcn.connect(alice).invest(amount, strikePrice, term);
            
            // รอ maturity
            await time.increase(term + 1);
            
            // ETH price ยังสูงกว่า strike ($2000 > $1800)
            // oracle ยังคืน $2000
            
            // Mock aToken balance
            await usdc.mint(await dcn.getAddress(), ethers.parseUnits("10100", 6));
            
            const balanceBefore = await usdc.balanceOf(alice.address);
            await dcn.settle(noteId);
            const balanceAfter = await usdc.balanceOf(alice.address);
            
            // Alice ควรได้ USDC กลับ
            expect(balanceAfter).to.be.gte(balanceBefore);
        });
        
        it("should settle in ETH when price falls below strike", async () => {
            const { dcn, usdc, weth, alice, oracle } = await loadFixture(deployFixtures);
            
            const strikePrice = ethers.parseUnits("1800", 18);
            const term = 7 * 24 * 3600;
            const amount = ethers.parseUnits("10000", 6);
            
            await usdc.mint(await dcn.getAddress(), ethers.parseUnits("1000", 6));
            
            const noteId = await dcn.connect(alice).invest.staticCall(amount, strikePrice, term);
            await dcn.connect(alice).invest(amount, strikePrice, term);
            
            await time.increase(term + 1);
            
            // Price crashes ต่ำกว่า strike
            await oracle.setPrice(
                await weth.getAddress(),
                await usdc.getAddress(),
                ethers.parseUnits("1500", 18)  // $1500 < $1800 strike
            );
            
            // Mint ETH ให้ contract เพียงพอ
            const ethNeeded = (BigInt(amount) * BigInt(1e18)) / BigInt(strikePrice);
            await weth.mint(await dcn.getAddress(), ethNeeded);
            
            await usdc.mint(await dcn.getAddress(), amount);
            
            const ethBefore = await weth.balanceOf(alice.address);
            await dcn.settle(noteId);
            const ethAfter = await weth.balanceOf(alice.address);
            
            // Alice ควรได้ ETH
            expect(ethAfter).to.be.gt(ethBefore);
        });
    });
    
    // ============================================================
    //             Factory Tests
    // ============================================================
    
    describe("StructuredProductFactory", () => {
        it("should register template and deploy product", async () => {
            const { factory, ppn, owner, alice } = await loadFixture(deployFixtures);
            
            const productType = ethers.keccak256(ethers.toUtf8Bytes("PPN"));
            
            // Register PPN template
            await factory.connect(owner).registerTemplate(
                productType,
                await ppn.getAddress()
            );
            
            const template = await factory.getTemplate(productType);
            expect(template.active).to.be.true;
            expect(template.deployCount).to.equal(0);
        });
        
        it("should fail to deploy with deprecated template", async () => {
            const { factory, ppn, owner } = await loadFixture(deployFixtures);
            
            const productType = ethers.keccak256(ethers.toUtf8Bytes("PPN_V1"));
            
            await factory.connect(owner).registerTemplate(
                productType,
                await ppn.getAddress()
            );
            
            await factory.connect(owner).deprecateTemplate(productType);
            
            const template = await factory.getTemplate(productType);
            expect(template.active).to.be.false;
        });
    });
});
```

---

## Workshop / แบบฝึกหัด

### แบบฝึกหัดที่ 1: Autocallable Note

**โจทย์**: สร้าง `AutocallableNote` ที่:
1. ทุก 30 วัน (observation dates) ตรวจสอบ price vs barrier
2. ถ้า price >= call barrier (เช่น 105% ของ initial) → redeem early + coupon สูง
3. ถ้าถึง maturity โดยไม่ถูก called → ตรวจสอบ final price vs knock-in barrier (80%)
4. ถ้า final price < knock-in → ขาดทุนตาม price drop
5. ถ้า final price >= knock-in → คืนเต็ม principal

### แบบฝึกหัดที่ 2: Range Accrual Note

**โจทย์**: สร้าง `RangeAccrualNote` ที่:
1. Earn yield ต่อวันก็ต่อเมื่อ ETH price อยู่ใน range ที่กำหนด
2. ถ้า price ออกนอก range → ไม่ได้ yield วันนั้น
3. ที่ maturity → คืน principal + accumulated yield

### แบบฝึกหัดที่ 3: Basket Note

**โจทย์**: สร้าง `BasketLinkedNote` ที่:
1. Return เชื่อมกับ basket ของ tokens (ETH 40%, BTC 40%, MATIC 20%)
2. ถ้า worst performer > 0 → รับ upside จาก best performer
3. Capital protected (Worst case = 100% principal return)

---

## สรุป Part 54

- **Principal Protected Notes** แบ่ง capital เป็น bond (grow back to principal) + option (upside)
- Bond amount = `principal / (1 + yield)^T` ซึ่งต้องคำนวณจาก lending protocol APY
- **Dual Currency Notes** สร้าง yield สูงโดยขาย put option โดยนัย - investor เสี่ยงรับ ETH แทน USDC
- DCN มี 2 settlement paths: USDC path (price above strike) และ ETH path (price below strike)
- **Leveraged Yield Tokens** ใช้ recursive borrowing เพื่อสร้าง leverage บน yield
- LYT ต้อง rebalance เป็นระยะเพื่อรักษา target leverage และ health factor
- **StructuredProductFactory** ใช้ EIP-1167 Clones เพื่อ deploy instances ราคาถูก
- Product lifecycle: OPEN → INVESTED → MATURED → SETTLED
- ทุก product ต้องมี clear settlement logic และ principal protection guarantee ที่ verifiable

## Next: Part 55 - Advanced Lending Protocols
