# Part 32: Real World Assets (RWA)

## สารบัญ
1. RWA Overview
2. Tokenized Securities
3. Stablecoins Architecture
4. KYC/AML Compliance
5. Workshop: Tokenized Bond

---

## 1. RWA Overview

```
Real World Assets (RWA):
การนำ assets จากโลกจริงมา tokenize บน blockchain
- ทรัพย์สิน (Real estate, commodities)
- Securities (stocks, bonds)
- Receivables (invoices, royalties)
- Currencies (stablecoins)

ทำไม RWA ถึงสำคัญ:
1. เปิด DeFi ให้กับ $500T+ ของ traditional finance
2. ลด counterparty risk ผ่าน smart contracts
3. Fractional ownership (ซื้อ 0.001% ของ building)
4. 24/7 trading, instant settlement
5. Programmable compliance (KYC, AML, transfer restrictions)

Challenges:
- Legal off-chain enforcement
- Oracle for asset valuation
- Regulatory compliance
- Custody of real assets

Leading Projects:
- MakerDAO (RWA vaults)
- Centrifuge (trade finance)
- Ondo Finance (US Treasuries)
- BlackRock BUIDL Fund (tokenized money market)
```

---

## 2. Tokenized Security (ERC-3643)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * ERC-3643 (T-REX): Security Token Standard
 * 
 * Features:
 * - Transfer restrictions (KYC/AML)
 * - Identity registry
 * - Compliance module
 * - Forced transfer for regulatory
 * - Freeze/pause per account
 */
interface IIdentityRegistry {
    function isVerified(address user) external view returns (bool);
    function getCountry(address user) external view returns (uint16);
}

interface ICompliance {
    function canTransfer(address from, address to, uint256 amount) external view returns (bool);
    function transferred(address from, address to, uint256 amount) external;
}

contract SecurityToken {
    
    string public name;
    string public symbol;
    uint8 public constant decimals = 0; // Indivisible security tokens
    
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    mapping(address => bool) public frozen; // Frozen accounts
    bool public paused;
    
    IIdentityRegistry public identityRegistry;
    ICompliance public compliance;
    address public owner;
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event TokensFrozen(address indexed account, bool isFrozen);
    event RecoverySuccess(address indexed lostWallet, address indexed newWallet, uint256 amount);
    
    error NotVerified(address account);
    error AccountFrozen(address account);
    error ComplianceRestricted(address from, address to);
    error ContractPaused();
    
    modifier notPaused() {
        if (paused) revert ContractPaused();
        _;
    }
    
    modifier notFrozen(address account) {
        if (frozen[account]) revert AccountFrozen(account);
        _;
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    constructor(
        string memory _name,
        string memory _symbol,
        address _identityRegistry,
        address _compliance
    ) {
        name = _name;
        symbol = _symbol;
        identityRegistry = IIdentityRegistry(_identityRegistry);
        compliance = ICompliance(_compliance);
        owner = msg.sender;
    }
    
    function transfer(address to, uint256 amount)
        external
        notPaused
        notFrozen(msg.sender)
        notFrozen(to)
        returns (bool)
    {
        _validateTransfer(msg.sender, to, amount);
        _transfer(msg.sender, to, amount);
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount)
        external
        notPaused
        notFrozen(from)
        notFrozen(to)
        returns (bool)
    {
        allowance[from][msg.sender] -= amount;
        _validateTransfer(from, to, amount);
        _transfer(from, to, amount);
        return true;
    }
    
    function _validateTransfer(address from, address to, uint256 amount) internal view {
        // Check KYC for both parties
        if (!identityRegistry.isVerified(from)) revert NotVerified(from);
        if (!identityRegistry.isVerified(to)) revert NotVerified(to);
        
        // Check compliance (jurisdiction, transfer limits, etc.)
        if (!compliance.canTransfer(from, to, amount)) {
            revert ComplianceRestricted(from, to);
        }
    }
    
    function _transfer(address from, address to, uint256 amount) internal {
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        compliance.transferred(from, to, amount);
        emit Transfer(from, to, amount);
    }
    
    // Mint to verified addresses only
    function mint(address to, uint256 amount) external onlyOwner {
        if (!identityRegistry.isVerified(to)) revert NotVerified(to);
        
        totalSupply += amount;
        balanceOf[to] += amount;
        emit Transfer(address(0), to, amount);
    }
    
    // Forced transfer (court order, regulatory)
    function forcedTransfer(
        address from,
        address to,
        uint256 amount
    ) external onlyOwner returns (bool) {
        require(balanceOf[from] >= amount, "Insufficient");
        _transfer(from, to, amount);
        return true;
    }
    
    // Recover lost wallet (identity proof off-chain)
    function recoveryAddress(
        address lostWallet,
        address newWallet,
        address investorOnchainID
    ) external onlyOwner {
        require(identityRegistry.isVerified(newWallet), "New wallet not verified");
        
        uint256 investorTokens = balanceOf[lostWallet];
        balanceOf[lostWallet] = 0;
        balanceOf[newWallet] += investorTokens;
        
        emit RecoverySuccess(lostWallet, newWallet, investorTokens);
        emit Transfer(lostWallet, newWallet, investorTokens);
    }
    
    // Admin: freeze account
    function setAddressFrozen(address account, bool isFrozen) external onlyOwner {
        frozen[account] = isFrozen;
        emit TokensFrozen(account, isFrozen);
    }
    
    function setPaused(bool _paused) external onlyOwner {
        paused = _paused;
    }
    
    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }
}
```

---

## 3. Algorithmic Stablecoin

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Over-collateralized Stablecoin (MakerDAO style)
 * 
 * Deposit ETH → mint stablecoin (1:1 USD peg)
 * Maintain >150% collateral ratio
 * Liquidate if ratio drops below 130%
 */
contract CollateralizedStablecoin {
    
    string public constant name = "USD Stablecoin";
    string public constant symbol = "USDS";
    uint8 public constant decimals = 18;
    
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    
    IOracle public oracle;
    
    uint256 public constant MIN_COLLATERAL_RATIO = 15000; // 150%
    uint256 public constant LIQUIDATION_RATIO = 13000;    // 130%
    uint256 public constant LIQUIDATION_BONUS = 500;      // 5%
    uint256 public constant BASIS = 10000;
    
    struct Vault {
        uint256 collateral; // ETH deposited (wei)
        uint256 debt;       // USDS minted
    }
    
    mapping(address => Vault) public vaults;
    uint256 public totalDebt;
    
    uint256 public stabilityFee = 200; // 2% annual fee
    uint256 public lastFeeAccrual;
    uint256 public feeAccumulator = 1e18; // starts at 1.0
    
    event Deposit(address indexed user, uint256 ethAmount);
    event Withdraw(address indexed user, uint256 ethAmount);
    event Mint(address indexed user, uint256 stableAmount);
    event Burn(address indexed user, uint256 stableAmount);
    event Liquidated(address indexed user, address indexed liquidator, uint256 collateralSeized);
    
    error CollateralRatioTooLow(uint256 ratio);
    error NotLiquidatable(uint256 ratio);
    
    constructor(address _oracle) {
        oracle = IOracle(_oracle);
        lastFeeAccrual = block.timestamp;
    }
    
    // Deposit ETH as collateral
    function deposit() external payable {
        vaults[msg.sender].collateral += msg.value;
        emit Deposit(msg.sender, msg.value);
    }
    
    // Withdraw ETH (must maintain collateral ratio)
    function withdraw(uint256 amount) external {
        vaults[msg.sender].collateral -= amount;
        
        uint256 ratio = getCollateralRatio(msg.sender);
        if (ratio < MIN_COLLATERAL_RATIO) revert CollateralRatioTooLow(ratio);
        
        payable(msg.sender).transfer(amount);
        emit Withdraw(msg.sender, amount);
    }
    
    // Mint stablecoins against collateral
    function mint(uint256 amount) external {
        accrueInterest();
        
        vaults[msg.sender].debt += amount;
        totalDebt += amount;
        totalSupply += amount;
        balanceOf[msg.sender] += amount;
        
        uint256 ratio = getCollateralRatio(msg.sender);
        if (ratio < MIN_COLLATERAL_RATIO) revert CollateralRatioTooLow(ratio);
        
        emit Mint(msg.sender, amount);
    }
    
    // Burn stablecoins to reduce debt
    function burn(uint256 amount) external {
        accrueInterest();
        
        vaults[msg.sender].debt -= amount;
        totalDebt -= amount;
        balanceOf[msg.sender] -= amount;
        totalSupply -= amount;
        
        emit Burn(msg.sender, amount);
    }
    
    function getCollateralRatio(address user) public view returns (uint256) {
        Vault storage vault = vaults[user];
        if (vault.debt == 0) return type(uint256).max;
        
        uint256 ethPrice = oracle.getMarkPrice(); // USD price
        uint256 collateralValue = (vault.collateral * ethPrice) / 1e18;
        
        return (collateralValue * BASIS) / vault.debt;
    }
    
    function liquidate(address user) external {
        accrueInterest();
        
        uint256 ratio = getCollateralRatio(user);
        if (ratio >= LIQUIDATION_RATIO) revert NotLiquidatable(ratio);
        
        Vault storage vault = vaults[user];
        uint256 debt = vault.debt;
        
        // Liquidator burns user's debt
        require(balanceOf[msg.sender] >= debt, "Insufficient USDS");
        balanceOf[msg.sender] -= debt;
        totalSupply -= debt;
        
        uint256 ethPrice = oracle.getMarkPrice();
        uint256 debtInEth = (debt * 1e18) / ethPrice;
        uint256 bonus = (debtInEth * LIQUIDATION_BONUS) / BASIS;
        uint256 collateralToSeize = debtInEth + bonus;
        
        if (collateralToSeize > vault.collateral) {
            collateralToSeize = vault.collateral;
        }
        
        vault.debt = 0;
        vault.collateral -= collateralToSeize;
        totalDebt -= debt;
        
        payable(msg.sender).transfer(collateralToSeize);
        
        emit Liquidated(user, msg.sender, collateralToSeize);
    }
    
    function accrueInterest() public {
        uint256 elapsed = block.timestamp - lastFeeAccrual;
        if (elapsed == 0) return;
        
        // Compound annually
        uint256 feePerSecond = stabilityFee * 1e18 / (365 days * BASIS);
        feeAccumulator = feeAccumulator + (feeAccumulator * feePerSecond * elapsed / 1e18);
        
        lastFeeAccrual = block.timestamp;
    }
    
    function transfer(address to, uint256 amount) external returns (bool) {
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }
}

interface IOracle {
    function getMarkPrice() external view returns (uint256);
}
```

---

## สรุป Part 32

RWA ที่เรียนรู้:
- ✅ RWA overview (securities, bonds, stablecoins)
- ✅ ERC-3643 security token (KYC, compliance)
- ✅ Forced transfer + account recovery
- ✅ Over-collateralized stablecoin (MakerDAO style)
- ✅ Stability fee + liquidation mechanism

## Quiz

1. ทำไม Security Token ถึงต้องมี compliance module?
2. Collateral ratio 150% หมายความว่าอะไร?
3. Stability fee ใน MakerDAO ทำงานอย่างไร?
4. Forced transfer มีประโยชน์ในกรณีไหน?

---

## Next: Part 33 - Account Abstraction (EIP-4337)
