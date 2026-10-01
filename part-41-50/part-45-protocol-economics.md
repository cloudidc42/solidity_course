# Part 45: Protocol Economics Design

## สารบัญ
1. Token Economics Fundamentals
2. Fee Models
3. Liquidity Mining Design
4. Sustainable Protocol Revenue
5. Workshop: Protocol Token System

---

## 1. Token Economics Fundamentals

```
Token Economics (Tokenomics):
การออกแบบ economic incentives ของ protocol

Key Questions:
1. ทำไม token ถึงมีมูลค่า?
   - Utility: ใช้จ่าย fees, governance
   - Cash flow: แชร์ protocol revenue
   - Collateral: ใช้ใน DeFi
   - Access: gate premium features

2. ใคร hold token?
   - Long-term: governance, revenue share
   - Short-term: speculation, farming
   - Protocol: treasury, liquidity

3. Supply schedule:
   - Fixed: Bitcoin (21M cap)
   - Inflationary: Ethereum (rewards validators)
   - Deflationary: buyback and burn
   - Elastic: algorithmic supply

Vesting Schedule Design:
- Team: 4-year vest, 1-year cliff
- Investors: 2-year vest, 6-month cliff
- Community: emissions over 4 years
- Treasury: unlock via governance

Common Mistakes:
- Too much inflation → dump
- Too concentrated (team/VC) → bad optics
- No real utility → ponzinomics
- Unlocks cause price crashes
```

---

## 2. Fee Models

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Protocol Fee System:
 * แยกประเภท fees และ routing ไปยังผู้รับ
 */
contract FeeManager {
    
    uint256 public constant BASIS = 10000;
    
    // Fee configuration
    struct FeeConfig {
        uint256 protocolFee;  // to treasury (bps)
        uint256 lpFee;        // to liquidity providers (bps)
        uint256 referralFee;  // to referrers (bps)
        uint256 burnFee;      // burned (bps)
    }
    
    FeeConfig public fees = FeeConfig({
        protocolFee: 20,   // 0.2%
        lpFee: 25,         // 0.25%
        referralFee: 5,    // 0.05%
        burnFee: 0         // 0%
    });
    // Total: 0.5% per trade
    
    address public treasury;
    address public feeToken; // governance token to burn
    
    mapping(address => uint256) public referralEarnings;
    mapping(address => uint256) public lpFeeAccumulator; // per-LP
    
    uint256 public totalProtocolFees;
    uint256 public totalBurned;
    
    event FeesCollected(
        address indexed payer,
        uint256 protocolAmount,
        uint256 lpAmount,
        uint256 referralAmount
    );
    event FeesDistributed(uint256 toTreasury, uint256 burned);
    
    constructor(address _treasury, address _feeToken) {
        treasury = _treasury;
        feeToken = _feeToken;
    }
    
    /**
     * Collect and route fees from a swap
     */
    function collectSwapFees(
        address payer,
        address referrer,
        uint256 swapAmount,
        address lpAddress
    ) external returns (uint256 amountAfterFees) {
        uint256 totalFeeRate = fees.protocolFee + fees.lpFee + fees.referralFee + fees.burnFee;
        uint256 totalFee = swapAmount * totalFeeRate / BASIS;
        
        uint256 protocolAmount = swapAmount * fees.protocolFee / BASIS;
        uint256 lpAmount = swapAmount * fees.lpFee / BASIS;
        uint256 referralAmount = referrer != address(0) 
            ? swapAmount * fees.referralFee / BASIS 
            : 0;
        uint256 burnAmount = swapAmount * fees.burnFee / BASIS;
        
        // Adjust protocol fee if referral is active
        if (referrer == address(0)) {
            protocolAmount += swapAmount * fees.referralFee / BASIS;
        }
        
        // Accrue earnings
        totalProtocolFees += protocolAmount;
        if (referrer != address(0)) {
            referralEarnings[referrer] += referralAmount;
        }
        lpFeeAccumulator[lpAddress] += lpAmount;
        
        // Burn if configured
        if (burnAmount > 0) {
            _burn(burnAmount);
            totalBurned += burnAmount;
        }
        
        emit FeesCollected(payer, protocolAmount, lpAmount, referralAmount);
        
        return swapAmount - totalFee;
    }
    
    /**
     * Buyback and burn using protocol fees
     */
    function buybackAndBurn(uint256 ethAmount) external {
        require(msg.sender == treasury, "Not treasury");
        
        // Use ETH to buy governance token on DEX
        // Then burn it
        // (Implementation depends on DEX interface)
        
        emit FeesDistributed(ethAmount, 0);
    }
    
    function _burn(uint256 amount) internal {
        // Call burn on feeToken
        (bool success,) = feeToken.call(
            abi.encodeWithSignature("burn(uint256)", amount)
        );
        require(success, "Burn failed");
    }
    
    function claimReferralFees() external {
        uint256 amount = referralEarnings[msg.sender];
        require(amount > 0, "Nothing to claim");
        
        referralEarnings[msg.sender] = 0;
        // Transfer tokens to referrer
    }
}
```

---

## 3. Liquidity Mining

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Liquidity Mining with Decay:
 * Rewards สูงช่วงแรก → ลดลงตามเวลา
 * Bootstrap liquidity ในช่วงเริ่มต้น
 */
contract LiquidityMining {
    
    IERC20 public immutable rewardToken;
    IERC20 public immutable lpToken;
    
    uint256 public startTime;
    uint256 public constant INITIAL_EMISSION = 1000e18; // per day
    uint256 public constant DECAY_RATE = 9500;  // 95% per epoch
    uint256 public constant EPOCH_LENGTH = 7 days;
    uint256 public constant BASIS = 10000;
    
    uint256 public rewardPerTokenStored;
    uint256 public lastUpdateTime;
    uint256 public totalStaked;
    
    mapping(address => uint256) public userRewardPerTokenPaid;
    mapping(address => uint256) public rewards;
    mapping(address => uint256) public balances;
    
    event Staked(address indexed user, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);
    event RewardClaimed(address indexed user, uint256 reward);
    
    constructor(address _rewardToken, address _lpToken) {
        rewardToken = IERC20(_rewardToken);
        lpToken = IERC20(_lpToken);
        startTime = block.timestamp;
    }
    
    /**
     * Current emission rate (decays by epoch)
     */
    function currentEmissionRate() public view returns (uint256) {
        uint256 elapsed = block.timestamp - startTime;
        uint256 epochs = elapsed / EPOCH_LENGTH;
        
        uint256 rate = INITIAL_EMISSION;
        for (uint256 i; i < epochs && i < 20; i++) { // Cap at 20 epochs
            rate = rate * DECAY_RATE / BASIS;
        }
        
        return rate / 1 days; // Per second
    }
    
    /**
     * Reward per token accumulated (integral of rate/supply)
     */
    function rewardPerToken() public view returns (uint256) {
        if (totalStaked == 0) return rewardPerTokenStored;
        
        uint256 elapsed = block.timestamp - lastUpdateTime;
        uint256 emissionPerSecond = currentEmissionRate();
        
        return rewardPerTokenStored + (elapsed * emissionPerSecond * 1e18 / totalStaked);
    }
    
    function earned(address account) public view returns (uint256) {
        return (balances[account] * (rewardPerToken() - userRewardPerTokenPaid[account]) / 1e18)
               + rewards[account];
    }
    
    modifier updateReward(address account) {
        rewardPerTokenStored = rewardPerToken();
        lastUpdateTime = block.timestamp;
        
        if (account != address(0)) {
            rewards[account] = earned(account);
            userRewardPerTokenPaid[account] = rewardPerTokenStored;
        }
        _;
    }
    
    function stake(uint256 amount) external updateReward(msg.sender) {
        require(amount > 0, "Cannot stake 0");
        
        lpToken.transferFrom(msg.sender, address(this), amount);
        balances[msg.sender] += amount;
        totalStaked += amount;
        
        emit Staked(msg.sender, amount);
    }
    
    function withdraw(uint256 amount) external updateReward(msg.sender) {
        require(amount > 0, "Cannot withdraw 0");
        require(balances[msg.sender] >= amount, "Insufficient");
        
        balances[msg.sender] -= amount;
        totalStaked -= amount;
        lpToken.transfer(msg.sender, amount);
        
        emit Withdrawn(msg.sender, amount);
    }
    
    function claimReward() external updateReward(msg.sender) {
        uint256 reward = rewards[msg.sender];
        if (reward > 0) {
            rewards[msg.sender] = 0;
            rewardToken.transfer(msg.sender, reward);
            emit RewardClaimed(msg.sender, reward);
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

## 4. Protocol Revenue Model

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Revenue Sharing:
 * Protocol revenue → distribute to stakers/holders
 * 
 * Models:
 * 1. Direct: Revenue → stakers immediately
 * 2. Buyback & Burn: Revenue → buy token → burn
 * 3. veToken: Revenue → ve-locked holders
 * 4. Treasury: Accumulate → governance decides
 */
contract RevenueDistributor {
    
    IERC20 public immutable govToken;
    
    // Revenue epochs: weekly distribution
    uint256 public constant EPOCH = 7 days;
    uint256 public currentEpoch;
    
    struct EpochReward {
        uint256 totalRevenue;
        uint256 totalVotes;   // snapshot of voting power
        bool distributed;
    }
    
    mapping(uint256 => EpochReward) public epochs;
    mapping(address => mapping(uint256 => bool)) public claimed;
    
    event RevenueDeposited(uint256 indexed epoch, uint256 amount);
    event RewardClaimed(address indexed user, uint256 indexed epoch, uint256 amount);
    
    constructor(address _govToken) {
        govToken = IERC20(_govToken);
    }
    
    // Protocol fee deposits call this
    function depositRevenue() external payable {
        uint256 epoch = block.timestamp / EPOCH;
        epochs[epoch].totalRevenue += msg.value;
        
        emit RevenueDeposited(epoch, msg.value);
    }
    
    // Stakers claim their share
    function claimEpochRevenue(uint256 epoch) external {
        require(epoch < block.timestamp / EPOCH, "Epoch not ended");
        require(!claimed[msg.sender][epoch], "Already claimed");
        
        EpochReward storage e = epochs[epoch];
        
        // Get user's vote weight at epoch start
        uint256 epochTimestamp = epoch * EPOCH;
        uint256 userVotes = IVotes(address(govToken)).getPastVotes(
            msg.sender,
            epochTimestamp
        );
        
        if (userVotes == 0) return;
        
        if (e.totalVotes == 0) {
            e.totalVotes = IVotes(address(govToken)).getPastTotalSupply(epochTimestamp);
        }
        
        uint256 userShare = e.totalRevenue * userVotes / e.totalVotes;
        
        claimed[msg.sender][epoch] = true;
        
        (bool success,) = payable(msg.sender).call{value: userShare}("");
        require(success, "Transfer failed");
        
        emit RewardClaimed(msg.sender, epoch, userShare);
    }
}

interface IVotes {
    function getPastVotes(address, uint256) external view returns (uint256);
    function getPastTotalSupply(uint256) external view returns (uint256);
}
```

---

## สรุป Part 45

Protocol Economics ที่เรียนรู้:
- ✅ Tokenomics fundamentals
- ✅ Fee routing system
- ✅ Liquidity mining with decay
- ✅ Revenue sharing (epoch-based)
- ✅ Buyback and burn mechanics

## Next: Part 46 - Chainlink Advanced Integration
