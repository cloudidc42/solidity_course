# Part 26: Advanced Token Economics

## สารบัญ
1. Token Design Patterns
2. Bonding Curves
3. Vesting และ Token Distribution
4. Fee Mechanisms
5. Workshop: Protocol Token System

---

## 1. Token Design Patterns

```
Token Economics คือ:
การออกแบบ incentive structure ของ protocol
ให้ทุก stakeholder มี aligned incentives

Key Components:
1. Supply: Total, Circulating, Max
2. Distribution: Team, Investors, Community, Treasury
3. Vesting: ป้องกัน dump
4. Utility: ใช้ทำอะไรได้บ้าง
5. Demand Drivers: ทำไม user ถึงต้องการ token
6. Burn/Deflationary: ลด supply เพื่อเพิ่ม value

Common Models:
- Governance: vote ใน DAO
- Fee Sharing: protocol fees แจก token holders
- Work Token: ต้อง stake ถึงจะให้บริการได้
- Burn: ใช้แล้ว burn (ลด supply)
- Points → Token: สะสม points แล้ว convert
```

---

## 2. Bonding Curves

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Bonding Curve: ราคา token เปลี่ยนตาม supply
 * 
 * Linear: price = a + b × supply
 * Quadratic: price = a × supply^2
 * Bancor: price = reserve × (totalSupply / reserveRatio)
 * 
 * Use cases:
 * - Continuous token models
 * - NFT minting (rarer = more expensive)
 * - Community currencies
 */
contract LinearBondingCurve {
    
    string public name;
    string public symbol;
    uint8 public constant decimals = 18;
    
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    
    uint256 public immutable slope;     // price increase per token
    uint256 public immutable intercept; // base price
    
    // Reserve in ETH
    uint256 public reserve;
    
    event Buy(address indexed buyer, uint256 tokenAmount, uint256 ethPaid);
    event Sell(address indexed seller, uint256 tokenAmount, uint256 ethReceived);
    
    constructor(
        string memory _name,
        string memory _symbol,
        uint256 _slope,    // e.g., 0.001 ether (price increase per token)
        uint256 _intercept // e.g., 0.01 ether (starting price)
    ) {
        name = _name;
        symbol = _symbol;
        slope = _slope;
        intercept = _intercept;
    }
    
    // Price for the next token
    function currentPrice() public view returns (uint256) {
        return intercept + (slope * totalSupply / 1e18);
    }
    
    // Cost to buy n tokens (integral of price curve)
    // cost = n × intercept + slope × (n² + 2n×supply) / 2
    function getBuyPrice(uint256 tokenAmount) public view returns (uint256) {
        // Integral from totalSupply to totalSupply + tokenAmount
        uint256 n = tokenAmount;
        uint256 s = totalSupply;
        
        // Area = intercept × n + slope × (s × n + n²/2) / 1e18
        uint256 area = intercept * n / 1e18;
        area += slope * (s * n / 1e18 + n * n / 2e18) / 1e18;
        
        return area;
    }
    
    // Amount received for selling n tokens
    function getSellPrice(uint256 tokenAmount) public view returns (uint256) {
        require(tokenAmount <= totalSupply, "Exceeds supply");
        
        uint256 n = tokenAmount;
        uint256 s = totalSupply - tokenAmount; // new supply after sell
        
        uint256 area = intercept * n / 1e18;
        area += slope * (s * n / 1e18 + n * n / 2e18) / 1e18;
        
        return area;
    }
    
    function buy(uint256 minTokens) external payable {
        require(msg.value > 0, "Send ETH");
        
        // Binary search / approximation สำหรับ tokens จาก ETH amount
        // (simplified: calculate directly)
        uint256 tokens = _calculateTokensForEth(msg.value);
        
        require(tokens >= minTokens, "Slippage too high");
        
        reserve += msg.value;
        totalSupply += tokens;
        balanceOf[msg.sender] += tokens;
        
        emit Buy(msg.sender, tokens, msg.value);
    }
    
    function sell(uint256 tokenAmount, uint256 minEth) external {
        require(balanceOf[msg.sender] >= tokenAmount, "Insufficient tokens");
        
        uint256 ethAmount = getSellPrice(tokenAmount);
        require(ethAmount >= minEth, "Slippage too high");
        require(reserve >= ethAmount, "Insufficient reserve");
        
        balanceOf[msg.sender] -= tokenAmount;
        totalSupply -= tokenAmount;
        reserve -= ethAmount;
        
        payable(msg.sender).transfer(ethAmount);
        
        emit Sell(msg.sender, tokenAmount, ethAmount);
    }
    
    // Simplified: estimate tokens for given ETH
    function _calculateTokensForEth(uint256 ethIn) internal view returns (uint256) {
        // For linear curve: use quadratic formula
        // ethIn = intercept × t/1e18 + slope × (supply × t + t²/2) / 1e36
        // Simplified iteration (in practice use exact formula)
        
        uint256 price = currentPrice();
        uint256 estimate = (ethIn * 1e18) / price;
        
        // Newton-Raphson refinement
        for (uint256 i = 0; i < 10; i++) {
            uint256 cost = getBuyPrice(estimate);
            if (cost == ethIn) break;
            if (cost > ethIn) {
                estimate = estimate * ethIn / cost;
            } else {
                estimate = estimate * 101 / 100;
            }
        }
        
        return estimate;
    }
}

/**
 * Sigmoid Bonding Curve
 * ราคาเพิ่มช้าตอนต้น เร็วตอนกลาง ช้าตอนท้าย
 * เหมาะสำหรับ token ที่ต้องการ fair launch
 */
contract SigmoidBondingCurve {
    
    // Price = maxPrice / (1 + e^(-k × (supply - midpoint)))
    // Approximation using integers:
    // Price = maxPrice × supply^2 / (supply^2 + midpoint^2)
    
    uint256 public immutable maxPrice;
    uint256 public immutable midpoint; // supply at max growth rate
    
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    
    constructor(uint256 _maxPrice, uint256 _midpoint) {
        maxPrice = _maxPrice;
        midpoint = _midpoint;
    }
    
    function currentPrice() public view returns (uint256) {
        uint256 s = totalSupply;
        uint256 m = midpoint;
        return maxPrice * s * s / (s * s + m * m + 1);
    }
}
```

---

## 3. Token Vesting

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Token Vesting Contract
 * 
 * Cliff: ระยะเวลาก่อนที่ tokens จะเริ่ม vest
 * Vesting: tokens unlock ทีละนิดตามเวลา
 * 
 * Example:
 * Cliff: 6 months (ไม่ได้รับอะไรเลย)
 * Vesting: 2 years linear (หลัง cliff)
 * Total: 2.5 years ถึงจะได้ครบ
 */
contract VestingWallet {
    
    IERC20 public immutable token;
    
    address public immutable beneficiary;
    uint256 public immutable start;
    uint256 public immutable cliff;
    uint256 public immutable duration;
    
    uint256 public released;
    bool public revoked;
    
    address public immutable owner;
    
    event TokensReleased(uint256 amount);
    event VestingRevoked(uint256 refund);
    
    error Cliff();
    error AlreadyRevoked();
    error NotOwner();
    
    constructor(
        address _token,
        address _beneficiary,
        uint256 _start,        // vesting start timestamp
        uint256 _cliff,        // cliff duration in seconds
        uint256 _duration,     // total vesting duration
        address _owner
    ) {
        require(_duration > 0, "Duration must be > 0");
        require(_cliff <= _duration, "Cliff must be <= duration");
        
        token = IERC20(_token);
        beneficiary = _beneficiary;
        start = _start;
        cliff = _cliff;
        duration = _duration;
        owner = _owner;
    }
    
    // Amount vested so far
    function vestedAmount() public view returns (uint256) {
        return _vestingSchedule(token.balanceOf(address(this)) + released, block.timestamp);
    }
    
    // Amount claimable (vested - already released)
    function releasableAmount() public view returns (uint256) {
        return vestedAmount() - released;
    }
    
    // Claim vested tokens
    function release() external {
        uint256 releasable = releasableAmount();
        require(releasable > 0, "Nothing to release");
        
        released += releasable;
        token.transfer(beneficiary, releasable);
        
        emit TokensReleased(releasable);
    }
    
    // Owner can revoke (for employee tokens that leave)
    function revoke() external {
        if (msg.sender != owner) revert NotOwner();
        if (revoked) revert AlreadyRevoked();
        
        revoked = true;
        
        uint256 vested = vestedAmount();
        uint256 totalBalance = token.balanceOf(address(this)) + released;
        uint256 refund = totalBalance - vested;
        
        if (refund > 0) {
            token.transfer(owner, refund);
        }
        
        emit VestingRevoked(refund);
    }
    
    function _vestingSchedule(uint256 totalAllocation, uint256 timestamp)
        internal view returns (uint256)
    {
        if (timestamp < start + cliff) {
            return 0; // In cliff period
        } else if (timestamp >= start + duration) {
            return totalAllocation; // Fully vested
        } else {
            return (totalAllocation * (timestamp - start)) / duration;
        }
    }
}

/**
 * Multi-beneficiary Vesting Factory
 */
contract VestingFactory {
    
    IERC20 public immutable token;
    address public owner;
    
    address[] public vestingContracts;
    mapping(address => address) public beneficiaryToVesting;
    
    event VestingCreated(address indexed beneficiary, address vestingContract, uint256 amount);
    
    constructor(address _token) {
        token = IERC20(_token);
        owner = msg.sender;
    }
    
    function createVesting(
        address beneficiary,
        uint256 amount,
        uint256 startTime,
        uint256 cliff,
        uint256 duration
    ) external returns (address) {
        require(msg.sender == owner, "Not owner");
        require(beneficiaryToVesting[beneficiary] == address(0), "Vesting exists");
        
        VestingWallet vesting = new VestingWallet(
            address(token),
            beneficiary,
            startTime,
            cliff,
            duration,
            owner
        );
        
        address vestingAddr = address(vesting);
        vestingContracts.push(vestingAddr);
        beneficiaryToVesting[beneficiary] = vestingAddr;
        
        token.transferFrom(msg.sender, vestingAddr, amount);
        
        emit VestingCreated(beneficiary, vestingAddr, amount);
        
        return vestingAddr;
    }
    
    // Batch create vestings
    function batchCreateVesting(
        address[] calldata beneficiaries,
        uint256[] calldata amounts,
        uint256 startTime,
        uint256 cliff,
        uint256 duration
    ) external {
        require(msg.sender == owner, "Not owner");
        require(beneficiaries.length == amounts.length, "Length mismatch");
        
        for (uint256 i = 0; i < beneficiaries.length; i++) {
            this.createVesting(beneficiaries[i], amounts[i], startTime, cliff, duration);
        }
    }
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
    function balanceOf(address) external view returns (uint256);
}
```

---

## 4. Fee Distribution

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Fee Distribution to Token Holders (xToken Model)
 * 
 * Stake token → รับ xToken
 * Protocol fees → convert to token → distribute to xToken holders
 * Exchange rate xToken:Token เพิ่มขึ้นเรื่อยๆ
 */
contract FeeDistributor {
    
    IERC20 public immutable token;
    
    string public constant name = "Staked Token";
    string public constant symbol = "xTKN";
    uint8 public constant decimals = 18;
    
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    
    uint256 public totalToken; // total underlying token
    
    event Enter(address indexed user, uint256 tokenAmount, uint256 xTokenAmount);
    event Leave(address indexed user, uint256 xTokenAmount, uint256 tokenAmount);
    event FeeAdded(uint256 amount);
    
    constructor(address _token) {
        token = IERC20(_token);
    }
    
    // Exchange rate: tokens per xToken
    function exchangeRate() public view returns (uint256) {
        if (totalSupply == 0) return 1e18;
        return (totalToken * 1e18) / totalSupply;
    }
    
    // Stake tokens → get xTokens
    function enter(uint256 tokenAmount) external {
        uint256 rate = exchangeRate();
        uint256 xTokenAmount = (tokenAmount * 1e18) / rate;
        
        token.transferFrom(msg.sender, address(this), tokenAmount);
        totalToken += tokenAmount;
        
        totalSupply += xTokenAmount;
        balanceOf[msg.sender] += xTokenAmount;
        
        emit Enter(msg.sender, tokenAmount, xTokenAmount);
    }
    
    // Unstake xTokens → get tokens + accumulated fees
    function leave(uint256 xTokenAmount) external {
        uint256 rate = exchangeRate();
        uint256 tokenAmount = (xTokenAmount * rate) / 1e18;
        
        balanceOf[msg.sender] -= xTokenAmount;
        totalSupply -= xTokenAmount;
        totalToken -= tokenAmount;
        
        token.transfer(msg.sender, tokenAmount);
        
        emit Leave(msg.sender, xTokenAmount, tokenAmount);
    }
    
    // Protocol distributes fees (increases exchange rate for all)
    function distributeFees(uint256 amount) external {
        token.transferFrom(msg.sender, address(this), amount);
        totalToken += amount;
        
        emit FeeAdded(amount);
    }
    
    // Helper: how much token you'd get for xToken
    function xTokenToToken(uint256 xAmount) external view returns (uint256) {
        return (xAmount * exchangeRate()) / 1e18;
    }
    
    // Helper: how much xToken you'd get for token
    function tokenToXToken(uint256 amount) external view returns (uint256) {
        return (amount * 1e18) / exchangeRate();
    }
}
```

---

## สรุป Part 26

Token Economics ที่เรียนรู้:
- ✅ Token design patterns
- ✅ Linear bonding curve
- ✅ Sigmoid bonding curve
- ✅ Cliff + linear vesting
- ✅ Fee distribution (xToken model)

## Quiz

1. Bonding curve ต่างจาก Fixed supply token อย่างไร?
2. Cliff period ช่วยป้องกันอะไร?
3. xToken exchange rate เพิ่มขึ้นอย่างไร?
4. เพราะอะไร team token ถึงต้องมี vesting?

---

## Next: Part 27 - Smart Contract Auditing
