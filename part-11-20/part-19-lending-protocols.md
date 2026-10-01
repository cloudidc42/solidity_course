# Part 19: Lending Protocols

## สารบัญ
1. Lending Protocol Basics
2. Interest Rate Models
3. Collateralization
4. Liquidations
5. Aave-style Protocol
6. Workshop: Simple Lending

---

## 1. Lending Protocol Basics

```
Lending Protocol:
- Lenders: deposit assets → earn interest
- Borrowers: provide collateral → borrow assets
- Protocol: manages risk, collects fees

Key Concepts:
- Supply Rate: ดอกเบี้ยที่ lender ได้รับ
- Borrow Rate: ดอกเบี้ยที่ borrower จ่าย
- Utilization: borrowed / total_supplied
- LTV (Loan-to-Value): borrowed / collateral_value
- Health Factor: collateral_value / borrowed_value
- Liquidation: when HF < threshold, anyone can liquidate

Interest Rate Formula (Compound-style):
- Borrow Rate = BaseRate + (Utilization × Multiplier)
- Supply Rate = BorrowRate × Utilization × (1 - ReserveFactor)
```

---

## 2. Interest Rate Model

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Jump Rate Model (Compound V2 style)
 * Normal utilization: low rates
 * Above kink: rates jump sharply
 */
contract JumpRateModel {
    
    uint256 public constant BLOCKS_PER_YEAR = 2_628_000; // ~12s per block
    uint256 public constant BASE = 1e18;
    
    uint256 public immutable baseRatePerBlock;   // e.g., 0
    uint256 public immutable multiplierPerBlock;  // e.g., 0.05 per year → per block
    uint256 public immutable jumpMultiplierPerBlock; // after kink
    uint256 public immutable kink;                // utilization kink (e.g., 80%)
    
    constructor(
        uint256 baseRatePerYear,    // e.g., 0
        uint256 multiplierPerYear,  // e.g., 0.05e18 = 5% at 100% util
        uint256 jumpMultiplierPerYear, // e.g., 1.09e18 = 109% above kink
        uint256 kink_               // e.g., 0.8e18 = 80%
    ) {
        baseRatePerBlock = baseRatePerYear / BLOCKS_PER_YEAR;
        multiplierPerBlock = (multiplierPerYear * BASE) / (BLOCKS_PER_YEAR * kink_);
        jumpMultiplierPerBlock = jumpMultiplierPerYear / BLOCKS_PER_YEAR;
        kink = kink_;
    }
    
    function utilizationRate(
        uint256 cash,
        uint256 borrows,
        uint256 reserves
    ) public pure returns (uint256) {
        if (borrows == 0) return 0;
        return (borrows * BASE) / (cash + borrows - reserves);
    }
    
    function getBorrowRate(
        uint256 cash,
        uint256 borrows,
        uint256 reserves
    ) public view returns (uint256) {
        uint256 util = utilizationRate(cash, borrows, reserves);
        
        if (util <= kink) {
            return ((util * multiplierPerBlock) / BASE) + baseRatePerBlock;
        } else {
            uint256 normalRate = ((kink * multiplierPerBlock) / BASE) + baseRatePerBlock;
            uint256 excessUtil = util - kink;
            return normalRate + ((excessUtil * jumpMultiplierPerBlock) / BASE);
        }
    }
    
    function getSupplyRate(
        uint256 cash,
        uint256 borrows,
        uint256 reserves,
        uint256 reserveFactorMantissa
    ) external view returns (uint256) {
        uint256 oneMinusReserveFactor = BASE - reserveFactorMantissa;
        uint256 borrowRate = getBorrowRate(cash, borrows, reserves);
        uint256 rateToPool = (borrowRate * oneMinusReserveFactor) / BASE;
        return (utilizationRate(cash, borrows, reserves) * rateToPool) / BASE;
    }
}
```

---

## 3. cToken (Compound Token)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * cToken: interest-bearing token
 * เมื่อ deposit USDC → ได้ cUSDC
 * cUSDC ราคาขึ้นเรื่อยๆ ตามดอกเบี้ย
 * เมื่อ redeem cUSDC → ได้ USDC + ดอกเบี้ย
 */
contract CToken {
    
    IERC20 public immutable underlying;
    JumpRateModel public immutable interestRateModel;
    
    string public name;
    string public symbol;
    uint8 public constant decimals = 8;
    
    uint256 public totalSupply;     // cToken supply
    uint256 public totalCash;       // underlying in contract
    uint256 public totalBorrows;    // total outstanding borrows
    uint256 public totalReserves;   // protocol reserves
    
    uint256 public reserveFactorMantissa = 0.1e18; // 10%
    
    uint256 public accrualBlockNumber;
    uint256 public borrowIndex = 1e18; // starts at 1
    
    uint256 public constant initialExchangeRate = 0.02e18; // 1 cToken = 0.02 underlying
    
    mapping(address => uint256) public balanceOf;    // cToken balance
    mapping(address => BorrowSnapshot) public borrowBalance;
    
    struct BorrowSnapshot {
        uint256 principal;
        uint256 interestIndex;
    }
    
    event AccrueInterest(uint256 cashPrior, uint256 interestAccumulated, uint256 borrowIndex, uint256 totalBorrows);
    event Mint(address minter, uint256 mintAmount, uint256 mintTokens);
    event Redeem(address redeemer, uint256 redeemAmount, uint256 redeemTokens);
    event Borrow(address borrower, uint256 borrowAmount, uint256 accountBorrows, uint256 totalBorrows);
    event RepayBorrow(address payer, address borrower, uint256 repayAmount, uint256 accountBorrows, uint256 totalBorrows);
    
    constructor(address _underlying, address _interestRateModel, string memory _name, string memory _symbol) {
        underlying = IERC20(_underlying);
        interestRateModel = JumpRateModel(_interestRateModel);
        name = _name;
        symbol = _symbol;
        accrualBlockNumber = block.number;
    }
    
    // Accrue interest (call before any state change)
    function accrueInterest() public {
        uint256 currentBlockNumber = block.number;
        uint256 accrualBlockNumberPrior = accrualBlockNumber;
        
        if (currentBlockNumber == accrualBlockNumberPrior) return;
        
        uint256 cashPrior = totalCash;
        uint256 borrowsPrior = totalBorrows;
        uint256 reservesPrior = totalReserves;
        uint256 borrowIndexPrior = borrowIndex;
        
        uint256 borrowRateMantissa = interestRateModel.getBorrowRate(cashPrior, borrowsPrior, reservesPrior);
        
        uint256 blockDelta = currentBlockNumber - accrualBlockNumberPrior;
        uint256 simpleInterestFactor = borrowRateMantissa * blockDelta;
        uint256 interestAccumulated = (simpleInterestFactor * borrowsPrior) / 1e18;
        
        uint256 totalBorrowsNew = borrowsPrior + interestAccumulated;
        uint256 totalReservesNew = reservesPrior + (interestAccumulated * reserveFactorMantissa / 1e18);
        uint256 borrowIndexNew = borrowIndexPrior + (simpleInterestFactor * borrowIndexPrior / 1e18);
        
        accrualBlockNumber = currentBlockNumber;
        borrowIndex = borrowIndexNew;
        totalBorrows = totalBorrowsNew;
        totalReserves = totalReservesNew;
        
        emit AccrueInterest(cashPrior, interestAccumulated, borrowIndexNew, totalBorrowsNew);
    }
    
    // Exchange Rate: (totalCash + totalBorrows - totalReserves) / totalSupply
    function exchangeRateCurrent() public returns (uint256) {
        accrueInterest();
        return _exchangeRateStored();
    }
    
    function _exchangeRateStored() internal view returns (uint256) {
        if (totalSupply == 0) return initialExchangeRate;
        return ((totalCash + totalBorrows - totalReserves) * 1e18) / totalSupply;
    }
    
    // Deposit underlying → get cTokens
    function mint(uint256 mintAmount) external returns (uint256) {
        accrueInterest();
        
        uint256 exchangeRate = _exchangeRateStored();
        
        underlying.transferFrom(msg.sender, address(this), mintAmount);
        totalCash += mintAmount;
        
        uint256 mintTokens = (mintAmount * 1e18) / exchangeRate;
        
        totalSupply += mintTokens;
        balanceOf[msg.sender] += mintTokens;
        
        emit Mint(msg.sender, mintAmount, mintTokens);
        return mintTokens;
    }
    
    // Burn cTokens → get underlying + interest
    function redeem(uint256 redeemTokens) external returns (uint256) {
        accrueInterest();
        
        uint256 exchangeRate = _exchangeRateStored();
        uint256 redeemAmount = (redeemTokens * exchangeRate) / 1e18;
        
        require(totalCash >= redeemAmount, "Insufficient cash");
        
        totalSupply -= redeemTokens;
        balanceOf[msg.sender] -= redeemTokens;
        totalCash -= redeemAmount;
        
        underlying.transfer(msg.sender, redeemAmount);
        
        emit Redeem(msg.sender, redeemAmount, redeemTokens);
        return redeemAmount;
    }
    
    // Borrow underlying
    function borrow(uint256 borrowAmount) external {
        accrueInterest();
        
        require(totalCash >= borrowAmount, "Insufficient liquidity");
        
        BorrowSnapshot storage snapshot = borrowBalance[msg.sender];
        
        uint256 accountBorrows = _borrowBalanceStored(msg.sender);
        uint256 accountBorrowsNew = accountBorrows + borrowAmount;
        
        snapshot.principal = accountBorrowsNew;
        snapshot.interestIndex = borrowIndex;
        
        totalBorrows += borrowAmount;
        totalCash -= borrowAmount;
        
        underlying.transfer(msg.sender, borrowAmount);
        
        emit Borrow(msg.sender, borrowAmount, accountBorrowsNew, totalBorrows);
    }
    
    // Repay borrow
    function repayBorrow(uint256 repayAmount) external returns (uint256) {
        accrueInterest();
        
        uint256 accountBorrows = _borrowBalanceStored(msg.sender);
        uint256 actualRepayAmount = repayAmount == type(uint256).max ? accountBorrows : repayAmount;
        
        underlying.transferFrom(msg.sender, address(this), actualRepayAmount);
        
        uint256 accountBorrowsNew = accountBorrows - actualRepayAmount;
        BorrowSnapshot storage snapshot = borrowBalance[msg.sender];
        snapshot.principal = accountBorrowsNew;
        snapshot.interestIndex = borrowIndex;
        
        totalBorrows -= actualRepayAmount;
        totalCash += actualRepayAmount;
        
        emit RepayBorrow(msg.sender, msg.sender, actualRepayAmount, accountBorrowsNew, totalBorrows);
        return actualRepayAmount;
    }
    
    function _borrowBalanceStored(address account) internal view returns (uint256) {
        BorrowSnapshot storage snapshot = borrowBalance[account];
        if (snapshot.principal == 0) return 0;
        return (snapshot.principal * borrowIndex) / snapshot.interestIndex;
    }
    
    function borrowBalanceCurrent(address account) external returns (uint256) {
        accrueInterest();
        return _borrowBalanceStored(account);
    }
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
    function balanceOf(address) external view returns (uint256);
}
```

---

## 4. Liquidation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract LiquidationEngine {
    
    struct UserAccount {
        address collateralAsset;
        uint256 collateralAmount;
        address borrowedAsset;
        uint256 borrowedAmount;
        uint256 borrowedAtPrice; // USD price when borrowed (18 decimals)
    }
    
    mapping(address => UserAccount) public accounts;
    mapping(address => uint256) public assetPrices; // USD price in 18 decimals
    
    uint256 public constant LTV_RATIO = 7500; // 75% max LTV
    uint256 public constant LIQUIDATION_THRESHOLD = 8000; // 80% - liquidate if below
    uint256 public constant LIQUIDATION_BONUS = 500; // 5% bonus for liquidators
    uint256 public constant BASIS_POINTS = 10000;
    
    event Liquidated(
        address indexed borrower,
        address indexed liquidator,
        uint256 collateralSeized,
        uint256 debtRepaid
    );
    
    error PositionHealthy(address user, uint256 healthFactor);
    error InsufficientRepayAmount();
    
    function getHealthFactor(address user) public view returns (uint256) {
        UserAccount storage acc = accounts[user];
        if (acc.borrowedAmount == 0) return type(uint256).max;
        
        uint256 collateralValueUSD = (acc.collateralAmount * assetPrices[acc.collateralAsset]) / 1e18;
        uint256 borrowedValueUSD = (acc.borrowedAmount * assetPrices[acc.borrowedAsset]) / 1e18;
        
        // Health Factor = (collateral × liquidation threshold) / borrowed
        return (collateralValueUSD * LIQUIDATION_THRESHOLD) / borrowedValueUSD;
    }
    
    function isLiquidatable(address user) public view returns (bool) {
        uint256 hf = getHealthFactor(user);
        return hf < BASIS_POINTS; // < 100%
    }
    
    function liquidate(
        address borrower,
        uint256 repayAmount
    ) external {
        uint256 healthFactor = getHealthFactor(borrower);
        if (healthFactor >= BASIS_POINTS) {
            revert PositionHealthy(borrower, healthFactor);
        }
        
        UserAccount storage acc = accounts[borrower];
        
        uint256 maxRepay = acc.borrowedAmount / 2; // Close factor: max 50% per liquidation
        uint256 actualRepay = repayAmount > maxRepay ? maxRepay : repayAmount;
        
        // Calculate collateral to seize (+ bonus)
        uint256 repayUSD = (actualRepay * assetPrices[acc.borrowedAsset]) / 1e18;
        uint256 collateralPrice = assetPrices[acc.collateralAsset];
        uint256 collateralToSeize = (repayUSD * (BASIS_POINTS + LIQUIDATION_BONUS)) 
            / BASIS_POINTS 
            * 1e18 
            / collateralPrice;
        
        require(collateralToSeize <= acc.collateralAmount, "Seize exceeds collateral");
        
        // Update state
        acc.borrowedAmount -= actualRepay;
        acc.collateralAmount -= collateralToSeize;
        
        // Transfer: liquidator pays debt, receives collateral + bonus
        IERC20(acc.borrowedAsset).transferFrom(msg.sender, address(this), actualRepay);
        IERC20(acc.collateralAsset).transfer(msg.sender, collateralToSeize);
        
        emit Liquidated(borrower, msg.sender, collateralToSeize, actualRepay);
    }
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
}
```

---

## สรุป Part 19

Lending Protocols ที่เรียนรู้:
- ✅ Jump Rate Interest Model
- ✅ cToken (interest-bearing tokens)
- ✅ Exchange rate calculation
- ✅ Borrow index for interest tracking
- ✅ Liquidation engine
- ✅ Health factor

## Quiz

1. Exchange Rate ใน cToken คืออะไร เปลี่ยนแปลงอย่างไร?
2. ทำไม Liquidation Bonus ถึงสำคัญ?
3. Jump Rate Model ต่างจาก Linear Rate อย่างไร?
4. Close Factor คืออะไร ทำไมต้องมี?

---

## Next: Part 20 - Staking และ Rewards
