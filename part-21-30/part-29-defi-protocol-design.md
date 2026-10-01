# Part 29: DeFi Protocol Design

## สารบัญ
1. Protocol Architecture
2. Yield Aggregator
3. Options Protocol
4. Perpetuals Basics
5. Workshop: Mini Yield Aggregator

---

## 1. Protocol Architecture

```
DeFi Protocol Design Principles:

1. Modularity: แยก logic ออกเป็น modules
   - Core: business logic
   - Periphery: user-facing wrappers
   - Libraries: shared utilities

2. Composability: ทำงานร่วมกับ protocols อื่น
   - ERC-4626 Vault standard
   - ERC-20 สำหรับ share tokens
   - Flash loans

3. Upgradeability: อัพเดทได้โดยไม่ทำให้ funds ติดค้าง
   - Proxy patterns
   - Migration guides

4. Security:
   - Timelock สำหรับ parameter changes
   - Emergency pause
   - Rate limits
   - Oracle redundancy

5. Fee Structure:
   - Protocol fee
   - Strategist fee
   - Performance fee
   - Management fee (AUM-based)

ERC-4626 Tokenized Vault Standard:
- deposit(assets) → shares
- withdraw(assets) → shares burned
- previewDeposit/previewWithdraw
- totalAssets(): underlying value
```

---

## 2. ERC-4626 Vault

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * ERC-4626: Tokenized Vault Standard
 * 
 * vault token (shares) → represent ownership of underlying assets
 * share price increases as yield accrues
 */
interface IERC4626 {
    event Deposit(address indexed sender, address indexed owner, uint256 assets, uint256 shares);
    event Withdraw(address indexed sender, address indexed receiver, address indexed owner, uint256 assets, uint256 shares);
    
    function asset() external view returns (address);
    function totalAssets() external view returns (uint256);
    function convertToShares(uint256 assets) external view returns (uint256);
    function convertToAssets(uint256 shares) external view returns (uint256);
    function maxDeposit(address) external view returns (uint256);
    function previewDeposit(uint256 assets) external view returns (uint256);
    function deposit(uint256 assets, address receiver) external returns (uint256 shares);
    function maxWithdraw(address owner) external view returns (uint256);
    function previewWithdraw(uint256 assets) external view returns (uint256);
    function withdraw(uint256 assets, address receiver, address owner) external returns (uint256 shares);
    function maxRedeem(address owner) external view returns (uint256);
    function previewRedeem(uint256 shares) external view returns (uint256);
    function redeem(uint256 shares, address receiver, address owner) external returns (uint256 assets);
}

contract ERC4626Vault is IERC4626 {
    
    IERC20 public immutable asset_;
    
    string public name;
    string public symbol;
    uint8 public constant decimals = 18;
    
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    // Performance fee: 20% of profits
    uint256 public constant PERFORMANCE_FEE = 2000; // 20%
    uint256 public constant FEE_DENOMINATOR = 10000;
    
    address public feeRecipient;
    uint256 public totalFees;
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    
    constructor(address _asset, string memory _name, string memory _symbol, address _feeRecipient) {
        asset_ = IERC20(_asset);
        name = _name;
        symbol = _symbol;
        feeRecipient = _feeRecipient;
    }
    
    function asset() external view override returns (address) {
        return address(asset_);
    }
    
    // Total assets managed (including yield)
    function totalAssets() public view override returns (uint256) {
        return asset_.balanceOf(address(this)) - totalFees;
    }
    
    // Convert assets to shares (based on exchange rate)
    function convertToShares(uint256 assets) public view override returns (uint256) {
        uint256 supply = totalSupply;
        return supply == 0 ? assets : (assets * supply) / totalAssets();
    }
    
    // Convert shares to assets
    function convertToAssets(uint256 shares) public view override returns (uint256) {
        uint256 supply = totalSupply;
        return supply == 0 ? shares : (shares * totalAssets()) / supply;
    }
    
    function maxDeposit(address) external pure override returns (uint256) {
        return type(uint256).max;
    }
    
    function previewDeposit(uint256 assets) public view override returns (uint256) {
        return convertToShares(assets);
    }
    
    function deposit(uint256 assets, address receiver) external override returns (uint256 shares) {
        shares = previewDeposit(assets);
        require(shares > 0, "Zero shares");
        
        asset_.transferFrom(msg.sender, address(this), assets);
        
        totalSupply += shares;
        balanceOf[receiver] += shares;
        
        emit Transfer(address(0), receiver, shares);
        emit Deposit(msg.sender, receiver, assets, shares);
    }
    
    function maxWithdraw(address owner) external view override returns (uint256) {
        return convertToAssets(balanceOf[owner]);
    }
    
    function previewWithdraw(uint256 assets) public view override returns (uint256) {
        return convertToShares(assets);
    }
    
    function withdraw(
        uint256 assets,
        address receiver,
        address owner
    ) external override returns (uint256 shares) {
        shares = previewWithdraw(assets);
        
        if (msg.sender != owner) {
            allowance[owner][msg.sender] -= shares;
        }
        
        balanceOf[owner] -= shares;
        totalSupply -= shares;
        
        asset_.transfer(receiver, assets);
        
        emit Transfer(owner, address(0), shares);
        emit Withdraw(msg.sender, receiver, owner, assets, shares);
    }
    
    function maxRedeem(address owner) external view override returns (uint256) {
        return balanceOf[owner];
    }
    
    function previewRedeem(uint256 shares) public view override returns (uint256) {
        return convertToAssets(shares);
    }
    
    function redeem(
        uint256 shares,
        address receiver,
        address owner
    ) external override returns (uint256 assets) {
        if (msg.sender != owner) {
            allowance[owner][msg.sender] -= shares;
        }
        
        assets = previewRedeem(shares);
        
        balanceOf[owner] -= shares;
        totalSupply -= shares;
        
        asset_.transfer(receiver, assets);
        
        emit Transfer(owner, address(0), shares);
        emit Withdraw(msg.sender, receiver, owner, assets, shares);
    }
    
    // Strategy: earn yield (simplified - just adds mock yield)
    function reportYield(uint256 gain) external {
        // In real protocol: this comes from strategy harvest
        require(gain > 0);
        
        // Take performance fee
        uint256 fee = (gain * PERFORMANCE_FEE) / FEE_DENOMINATOR;
        totalFees += fee;
        
        // gain stays in vault, increasing share price for depositors
    }
    
    function collectFees() external {
        uint256 amount = totalFees;
        totalFees = 0;
        asset_.transfer(feeRecipient, amount);
    }
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
    function balanceOf(address) external view returns (uint256);
}
```

---

## 3. Yield Aggregator (Yearn-style)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Yield Aggregator:
 * - Vault รับ deposits
 * - แบ่ง funds ให้ strategies หลายๆ ตัว
 * - Rebalance automatically
 * - Harvest yields
 */
contract YieldAggregator {
    
    IERC20 public immutable token;
    
    struct Strategy {
        address strategyAddress;
        uint256 allocation;   // basis points (10000 = 100%)
        uint256 totalDeposited;
        bool active;
    }
    
    Strategy[] public strategies;
    uint256 public totalAllocation;
    
    uint256 public totalDeposits;
    mapping(address => uint256) public userDeposits;
    mapping(address => uint256) public shares;
    uint256 public totalShares;
    
    address public keeper; // automated harvester
    
    event Deposited(address indexed user, uint256 amount, uint256 sharesReceived);
    event Withdrawn(address indexed user, uint256 amount, uint256 sharesBurned);
    event Harvested(uint256 yield, uint256 timestamp);
    event StrategyAdded(address strategy, uint256 allocation);
    
    error InsufficientFunds();
    error InvalidAllocation();
    error NotKeeper();
    
    modifier onlyKeeper() {
        if (msg.sender != keeper) revert NotKeeper();
        _;
    }
    
    constructor(address _token, address _keeper) {
        token = IERC20(_token);
        keeper = _keeper;
    }
    
    function addStrategy(address strategy, uint256 allocation) external {
        if (totalAllocation + allocation > 10000) revert InvalidAllocation();
        
        strategies.push(Strategy({
            strategyAddress: strategy,
            allocation: allocation,
            totalDeposited: 0,
            active: true
        }));
        
        totalAllocation += allocation;
        
        emit StrategyAdded(strategy, allocation);
    }
    
    function deposit(uint256 amount) external {
        require(amount > 0, "Zero amount");
        
        token.transferFrom(msg.sender, address(this), amount);
        
        // Calculate shares
        uint256 sharesAmount;
        if (totalShares == 0) {
            sharesAmount = amount;
        } else {
            uint256 totalValue = _totalValue();
            sharesAmount = (amount * totalShares) / totalValue;
        }
        
        shares[msg.sender] += sharesAmount;
        totalShares += sharesAmount;
        totalDeposits += amount;
        userDeposits[msg.sender] += amount;
        
        // Deploy to strategies
        _deployToStrategies(amount);
        
        emit Deposited(msg.sender, amount, sharesAmount);
    }
    
    function withdraw(uint256 shareAmount) external {
        require(shares[msg.sender] >= shareAmount, "Insufficient shares");
        
        uint256 totalValue = _totalValue();
        uint256 amount = (shareAmount * totalValue) / totalShares;
        
        shares[msg.sender] -= shareAmount;
        totalShares -= shareAmount;
        
        // Withdraw from strategies if needed
        uint256 available = token.balanceOf(address(this));
        if (available < amount) {
            _withdrawFromStrategies(amount - available);
        }
        
        token.transfer(msg.sender, amount);
        
        emit Withdrawn(msg.sender, amount, shareAmount);
    }
    
    // Harvest yield from all strategies
    function harvest() external onlyKeeper {
        uint256 before = token.balanceOf(address(this));
        
        for (uint256 i = 0; i < strategies.length; i++) {
            if (strategies[i].active) {
                IStrategy(strategies[i].strategyAddress).harvest();
            }
        }
        
        uint256 yield = token.balanceOf(address(this)) - before;
        
        if (yield > 0) {
            // Redeploy yield
            _deployToStrategies(yield);
            emit Harvested(yield, block.timestamp);
        }
    }
    
    function _totalValue() internal view returns (uint256) {
        uint256 total = token.balanceOf(address(this));
        for (uint256 i = 0; i < strategies.length; i++) {
            if (strategies[i].active) {
                total += IStrategy(strategies[i].strategyAddress).totalValue();
            }
        }
        return total;
    }
    
    function _deployToStrategies(uint256 amount) internal {
        for (uint256 i = 0; i < strategies.length; i++) {
            if (!strategies[i].active) continue;
            
            uint256 stratAmount = (amount * strategies[i].allocation) / totalAllocation;
            if (stratAmount == 0) continue;
            
            token.approve(strategies[i].strategyAddress, stratAmount);
            IStrategy(strategies[i].strategyAddress).deposit(stratAmount);
            strategies[i].totalDeposited += stratAmount;
        }
    }
    
    function _withdrawFromStrategies(uint256 needed) internal {
        for (uint256 i = 0; i < strategies.length; i++) {
            if (!strategies[i].active) continue;
            if (needed == 0) break;
            
            uint256 stratValue = IStrategy(strategies[i].strategyAddress).totalValue();
            uint256 toWithdraw = stratValue > needed ? needed : stratValue;
            
            IStrategy(strategies[i].strategyAddress).withdraw(toWithdraw);
            needed = token.balanceOf(address(this)) >= needed ? 0 : needed - toWithdraw;
        }
    }
    
    function getSharePrice() external view returns (uint256) {
        if (totalShares == 0) return 1e18;
        return (_totalValue() * 1e18) / totalShares;
    }
}

interface IStrategy {
    function deposit(uint256 amount) external;
    function withdraw(uint256 amount) external returns (uint256);
    function harvest() external returns (uint256);
    function totalValue() external view returns (uint256);
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
    function approve(address, uint256) external returns (bool);
    function balanceOf(address) external view returns (uint256);
}
```

---

## สรุป Part 29

DeFi Protocol Design ที่เรียนรู้:
- ✅ Protocol architecture principles
- ✅ ERC-4626 Tokenized Vault standard
- ✅ Yield Aggregator (Yearn-style)
- ✅ Strategy pattern
- ✅ Performance fees

## Quiz

1. ERC-4626 ช่วยอะไรใน DeFi ecosystem?
2. Share price เพิ่มขึ้นอย่างไรเมื่อมี yield?
3. ทำไม Yield Aggregator ถึงดีกว่าการ deposit เข้า protocol โดยตรง?
4. Performance fee vs Management fee ต่างกันอย่างไร?

---

## Next: Part 30 - NFT Marketplace
