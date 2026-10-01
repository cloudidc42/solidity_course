# Part 20: Staking และ Rewards

## สารบัญ
1. Staking Basics
2. Reward Distribution (Synthetix Model)
3. Boosted Staking
4. veToken Model
5. Workshop: Full Staking System

---

## 1. Staking Basics

```
Staking = Lock tokens เพื่อรับ rewards

Types:
1. Native Staking: stake token เดียวกัน (สนับสนุน network)
2. LP Staking: stake LP tokens รับ rewards (Liquidity Mining)
3. Single-sided: stake A รับ B
4. Boosted: stake ยิ่งนาน ยิ่งได้ rewards มาก

Core Metrics:
- APR (Annual Percentage Rate): ดอกเบี้ยแบบง่าย ไม่รวม compound
- APY (Annual Percentage Yield): รวม compound ด้วย
- TVL (Total Value Locked): มูลค่ารวมที่ stake อยู่
- Emissions Rate: tokens ออกใหม่ต่อวัน/บล็อก
```

---

## 2. Synthetix Reward Distribution (Standard Model)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Synthetix StakingRewards (Industry Standard)
 * 
 * rewardPerTokenStored: cumulative rewards per token since deployment
 * Increases by rewardRate × dt / totalSupply each second
 *
 * userRewardPerTokenPaid: last snapshot of rewardPerTokenStored for each user
 * rewards[user]: pending rewards not yet claimed
 *
 * Formula:
 * earned = balance × (rewardPerToken - userRewardPerTokenPaid) + rewards
 */
contract StakingRewards {
    
    IERC20 public immutable rewardsToken;
    IERC20 public immutable stakingToken;
    
    uint256 public periodFinish;           // when current period ends
    uint256 public rewardRate;             // rewards per second
    uint256 public rewardsDuration = 7 days;
    uint256 public lastUpdateTime;
    uint256 public rewardPerTokenStored;   // cumulative
    
    mapping(address => uint256) public userRewardPerTokenPaid;
    mapping(address => uint256) public rewards;
    
    uint256 private _totalSupply;
    mapping(address => uint256) private _balances;
    
    address public owner;
    
    event RewardAdded(uint256 reward);
    event Staked(address indexed user, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);
    event RewardPaid(address indexed user, uint256 reward);
    event RewardsDurationUpdated(uint256 newDuration);
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    modifier updateReward(address account) {
        rewardPerTokenStored = rewardPerToken();
        lastUpdateTime = lastTimeRewardApplicable();
        
        if (account != address(0)) {
            rewards[account] = earned(account);
            userRewardPerTokenPaid[account] = rewardPerTokenStored;
        }
        _;
    }
    
    constructor(address _rewardsToken, address _stakingToken) {
        rewardsToken = IERC20(_rewardsToken);
        stakingToken = IERC20(_stakingToken);
        owner = msg.sender;
    }
    
    function totalSupply() external view returns (uint256) {
        return _totalSupply;
    }
    
    function balanceOf(address account) external view returns (uint256) {
        return _balances[account];
    }
    
    function lastTimeRewardApplicable() public view returns (uint256) {
        return block.timestamp < periodFinish ? block.timestamp : periodFinish;
    }
    
    function rewardPerToken() public view returns (uint256) {
        if (_totalSupply == 0) {
            return rewardPerTokenStored;
        }
        return rewardPerTokenStored + (
            (lastTimeRewardApplicable() - lastUpdateTime) * rewardRate * 1e18 / _totalSupply
        );
    }
    
    function earned(address account) public view returns (uint256) {
        return (
            _balances[account] * (rewardPerToken() - userRewardPerTokenPaid[account]) / 1e18
        ) + rewards[account];
    }
    
    function getRewardForDuration() external view returns (uint256) {
        return rewardRate * rewardsDuration;
    }
    
    function stake(uint256 amount) external updateReward(msg.sender) {
        require(amount > 0, "Cannot stake 0");
        _totalSupply += amount;
        _balances[msg.sender] += amount;
        stakingToken.transferFrom(msg.sender, address(this), amount);
        emit Staked(msg.sender, amount);
    }
    
    function withdraw(uint256 amount) public updateReward(msg.sender) {
        require(amount > 0, "Cannot withdraw 0");
        require(_balances[msg.sender] >= amount, "Insufficient");
        _totalSupply -= amount;
        _balances[msg.sender] -= amount;
        stakingToken.transfer(msg.sender, amount);
        emit Withdrawn(msg.sender, amount);
    }
    
    function getReward() public updateReward(msg.sender) {
        uint256 reward = rewards[msg.sender];
        if (reward > 0) {
            rewards[msg.sender] = 0;
            rewardsToken.transfer(msg.sender, reward);
            emit RewardPaid(msg.sender, reward);
        }
    }
    
    function exit() external {
        withdraw(_balances[msg.sender]);
        getReward();
    }
    
    // Admin: notify new rewards
    function notifyRewardAmount(uint256 reward) external onlyOwner updateReward(address(0)) {
        if (block.timestamp >= periodFinish) {
            rewardRate = reward / rewardsDuration;
        } else {
            uint256 remaining = periodFinish - block.timestamp;
            uint256 leftover = remaining * rewardRate;
            rewardRate = (reward + leftover) / rewardsDuration;
        }
        
        require(
            rewardRate <= rewardsToken.balanceOf(address(this)) / rewardsDuration,
            "Reward rate too high"
        );
        
        lastUpdateTime = block.timestamp;
        periodFinish = block.timestamp + rewardsDuration;
        
        emit RewardAdded(reward);
    }
    
    function setRewardsDuration(uint256 _rewardsDuration) external onlyOwner {
        require(block.timestamp > periodFinish, "Period not ended");
        rewardsDuration = _rewardsDuration;
        emit RewardsDurationUpdated(_rewardsDuration);
    }
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
    function balanceOf(address) external view returns (uint256);
}
```

---

## 3. veToken Model (Vote-Escrow)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * veToken (Vote-Escrow) - Curve Style
 * 
 * Lock tokens for 1 week to 4 years
 * Longer lock = more voting power = more rewards boost
 * 
 * Voting Power decays linearly over time:
 * veBAL = BAL × (lock_time_remaining / max_lock_time)
 * 
 * Benefits:
 * - Align long-term incentives
 * - Reduce sell pressure
 * - Give loyal users more influence
 */
contract VoteEscrow {
    
    struct LockedBalance {
        int128 amount;  // locked amount
        uint256 end;    // lock end time
    }
    
    IERC20 public immutable token;
    
    uint256 public constant WEEK = 7 * 24 * 3600;
    uint256 public constant MAX_LOCK_TIME = 4 * 365 * 24 * 3600; // 4 years
    
    mapping(address => LockedBalance) public locked;
    
    uint256 public supply; // total locked
    
    event Deposit(address indexed provider, uint256 value, uint256 indexed locktime, uint256 ts);
    event Withdraw(address indexed provider, uint256 value, uint256 ts);
    
    error ZeroValue();
    error LockExpired();
    error NoLockFound();
    error LockTooLong();
    error LockTooShort();
    error AlreadyLocked();
    error LockNotExpired();
    
    constructor(address _token) {
        token = IERC20(_token);
    }
    
    // Create new lock
    function createLock(uint256 value, uint256 unlockTime) external {
        if (value == 0) revert ZeroValue();
        if (locked[msg.sender].amount != 0) revert AlreadyLocked();
        
        // Round down to week
        uint256 roundedUnlock = (unlockTime / WEEK) * WEEK;
        
        if (roundedUnlock <= block.timestamp) revert LockTooShort();
        if (roundedUnlock > block.timestamp + MAX_LOCK_TIME) revert LockTooLong();
        
        locked[msg.sender] = LockedBalance({
            amount: int128(int256(value)),
            end: roundedUnlock
        });
        supply += value;
        
        token.transferFrom(msg.sender, address(this), value);
        
        emit Deposit(msg.sender, value, roundedUnlock, block.timestamp);
    }
    
    // Add to existing lock (more tokens)
    function increaseAmount(uint256 value) external {
        LockedBalance storage lock = locked[msg.sender];
        if (lock.amount == 0) revert NoLockFound();
        if (lock.end <= block.timestamp) revert LockExpired();
        if (value == 0) revert ZeroValue();
        
        lock.amount += int128(int256(value));
        supply += value;
        
        token.transferFrom(msg.sender, address(this), value);
        
        emit Deposit(msg.sender, value, lock.end, block.timestamp);
    }
    
    // Extend lock time
    function increaseUnlockTime(uint256 unlockTime) external {
        LockedBalance storage lock = locked[msg.sender];
        if (lock.amount == 0) revert NoLockFound();
        if (lock.end <= block.timestamp) revert LockExpired();
        
        uint256 roundedUnlock = (unlockTime / WEEK) * WEEK;
        if (roundedUnlock <= lock.end) revert LockTooShort();
        if (roundedUnlock > block.timestamp + MAX_LOCK_TIME) revert LockTooLong();
        
        lock.end = roundedUnlock;
        
        emit Deposit(msg.sender, 0, roundedUnlock, block.timestamp);
    }
    
    // Withdraw after lock expires
    function withdraw() external {
        LockedBalance storage lock = locked[msg.sender];
        if (lock.end > block.timestamp) revert LockNotExpired();
        
        uint256 value = uint256(int256(lock.amount));
        
        supply -= value;
        lock.amount = 0;
        lock.end = 0;
        
        token.transfer(msg.sender, value);
        
        emit Withdraw(msg.sender, value, block.timestamp);
    }
    
    // Voting power (decays over time)
    function balanceOf(address addr) external view returns (uint256) {
        return _balanceOf(addr, block.timestamp);
    }
    
    function balanceOfAt(address addr, uint256 timestamp) external view returns (uint256) {
        return _balanceOf(addr, timestamp);
    }
    
    function _balanceOf(address addr, uint256 timestamp) internal view returns (uint256) {
        LockedBalance storage lock = locked[addr];
        if (lock.end <= timestamp) return 0;
        
        // veTokens = amount × (remaining_time / max_lock_time)
        uint256 remaining = lock.end - timestamp;
        return (uint256(int256(lock.amount)) * remaining) / MAX_LOCK_TIME;
    }
}
```

---

## 4. Workshop: Full Staking System

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title BoostableStaking
 * @dev Staking ที่รองรับ boost ด้วย veToken
 * - Base rewards สำหรับทุกคน
 * - Boosted rewards สำหรับ veToken holders
 * - Max boost: 2.5x
 */
contract BoostableStaking is StakingRewards {
    
    VoteEscrow public immutable veToken;
    
    uint256 public constant MAX_BOOST = 25000; // 2.5x in basis points / 10000
    uint256 public constant BASE_BOOST = 10000; // 1x base
    uint256 public constant BOOST_WEIGHT = 4000; // 40% of boost from veToken
    
    constructor(
        address _rewardsToken,
        address _stakingToken,
        address _veToken
    ) StakingRewards(_rewardsToken, _stakingToken) {
        veToken = VoteEscrow(_veToken);
    }
    
    // Calculate boost for user
    function getBoost(address user) public view returns (uint256) {
        uint256 stakeBalance = _balances[user];
        if (stakeBalance == 0) return BASE_BOOST;
        
        uint256 veBalance = veToken.balanceOf(user);
        uint256 veTotalSupply = token.balanceOf(address(veToken)); // simplified
        
        if (veBalance == 0 || veTotalSupply == 0) return BASE_BOOST;
        
        // Boost formula (Curve-style):
        // boost = (stake × 0.4) + (totalStake × veBalance / veTotalSupply × 0.6)
        uint256 minBalance = (stakeBalance * BOOST_WEIGHT) / 10000;
        uint256 maxBalance = (stakeBalance * (10000 - BOOST_WEIGHT)) / 10000 
            + (_totalSupply * veBalance / veTotalSupply * BOOST_WEIGHT / 10000);
        
        uint256 boostedBalance = minBalance + maxBalance;
        
        uint256 boost = (boostedBalance * 10000) / stakeBalance;
        
        return boost > MAX_BOOST ? MAX_BOOST : boost;
    }
    
    // Boosted earned
    function earned(address account) public view override returns (uint256) {
        uint256 baseEarned = super.earned(account);
        uint256 boost = getBoost(account);
        return (baseEarned * boost) / BASE_BOOST;
    }
    
    // Allow token reference
    IERC20Ext public token;
    
    mapping(address => uint256) internal _balances;
    uint256 internal _totalSupply;
}

interface IERC20Ext is IERC20 {
    function balanceOf(address) external view returns (uint256);
}

// Minimal IERC20
interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
    function balanceOf(address) external view returns (uint256);
}
```

---

## 5. TypeScript Tests สำหรับ StakingRewards

```typescript
// test/StakingRewards.test.ts
import { expect } from "chai";
import { ethers } from "hardhat";
import { StakingRewards, ERC20Mock } from "../typechain-types";
import { Signer } from "ethers";
import { time } from "@nomicfoundation/hardhat-network-helpers";

describe("StakingRewards", function () {
  let staking: StakingRewards;
  let rewardToken: ERC20Mock;
  let stakingToken: ERC20Mock;
  let owner: Signer;
  let alice: Signer;
  let bob: Signer;
  
  const REWARD_DURATION = 7 * 24 * 3600; // 7 days
  const REWARD_AMOUNT = ethers.parseEther("700000"); // 700k tokens
  const STAKE_AMOUNT = ethers.parseEther("1000");
  
  beforeEach(async function () {
    [owner, alice, bob] = await ethers.getSigners();
    
    // Deploy mock tokens
    const ERC20Mock = await ethers.getContractFactory("ERC20Mock");
    rewardToken = await ERC20Mock.deploy("Reward", "RWD", 18);
    stakingToken = await ERC20Mock.deploy("Staking", "STK", 18);
    
    // Deploy staking contract
    const StakingRewards = await ethers.getContractFactory("StakingRewards");
    staking = await StakingRewards.deploy(
      await rewardToken.getAddress(),
      await stakingToken.getAddress()
    );
    
    // Mint tokens
    await rewardToken.mint(await owner.getAddress(), REWARD_AMOUNT * 10n);
    await stakingToken.mint(await alice.getAddress(), STAKE_AMOUNT * 10n);
    await stakingToken.mint(await bob.getAddress(), STAKE_AMOUNT * 10n);
    
    // Approve
    await stakingToken.connect(alice).approve(await staking.getAddress(), ethers.MaxUint256);
    await stakingToken.connect(bob).approve(await staking.getAddress(), ethers.MaxUint256);
    await rewardToken.approve(await staking.getAddress(), ethers.MaxUint256);
    
    // Fund rewards
    await rewardToken.transfer(await staking.getAddress(), REWARD_AMOUNT);
    await staking.notifyRewardAmount(REWARD_AMOUNT);
  });
  
  describe("Staking", function () {
    it("should stake and track balance", async function () {
      await staking.connect(alice).stake(STAKE_AMOUNT);
      
      expect(await staking.balanceOf(await alice.getAddress())).to.equal(STAKE_AMOUNT);
      expect(await staking.totalSupply()).to.equal(STAKE_AMOUNT);
    });
    
    it("should earn rewards proportionally", async function () {
      // Alice stakes 1000, Bob stakes 1000 (50/50 split)
      await staking.connect(alice).stake(STAKE_AMOUNT);
      await staking.connect(bob).stake(STAKE_AMOUNT);
      
      // Advance time by 1 day
      await time.increase(24 * 3600);
      
      const aliceEarned = await staking.earned(await alice.getAddress());
      const bobEarned = await staking.earned(await bob.getAddress());
      
      // Should be roughly equal (50/50)
      const diff = aliceEarned > bobEarned ? aliceEarned - bobEarned : bobEarned - aliceEarned;
      const maxDiff = aliceEarned / 100n; // 1% tolerance
      
      expect(diff).to.be.lessThanOrEqual(maxDiff);
    });
    
    it("should earn more with larger stake", async function () {
      // Alice stakes 2000, Bob stakes 1000 (2:1 ratio)
      await staking.connect(alice).stake(STAKE_AMOUNT * 2n);
      await staking.connect(bob).stake(STAKE_AMOUNT);
      
      await time.increase(24 * 3600);
      
      const aliceEarned = await staking.earned(await alice.getAddress());
      const bobEarned = await staking.earned(await bob.getAddress());
      
      // Alice should earn ~2x Bob
      const ratio = (aliceEarned * 100n) / bobEarned;
      expect(ratio).to.be.within(195n, 205n); // ~200 (2x)
    });
    
    it("should allow withdraw and claim", async function () {
      await staking.connect(alice).stake(STAKE_AMOUNT);
      
      await time.increase(24 * 3600);
      
      const earnedBefore = await staking.earned(await alice.getAddress());
      expect(earnedBefore).to.be.gt(0);
      
      const rewardBalanceBefore = await rewardToken.balanceOf(await alice.getAddress());
      
      await staking.connect(alice).getReward();
      
      const rewardBalanceAfter = await rewardToken.balanceOf(await alice.getAddress());
      expect(rewardBalanceAfter - rewardBalanceBefore).to.be.closeTo(earnedBefore, earnedBefore / 100n);
    });
    
    it("should exit (withdraw + claim)", async function () {
      await staking.connect(alice).stake(STAKE_AMOUNT);
      await time.increase(24 * 3600);
      
      const stakingBalanceBefore = await stakingToken.balanceOf(await alice.getAddress());
      
      await staking.connect(alice).exit();
      
      expect(await staking.balanceOf(await alice.getAddress())).to.equal(0);
      
      const stakingBalanceAfter = await stakingToken.balanceOf(await alice.getAddress());
      expect(stakingBalanceAfter - stakingBalanceBefore).to.equal(STAKE_AMOUNT);
    });
  });
  
  describe("Reward Rate", function () {
    it("should distribute all rewards over duration", async function () {
      await staking.connect(alice).stake(STAKE_AMOUNT);
      
      // Advance to end of period
      await time.increase(REWARD_DURATION + 1);
      
      const earned = await staking.earned(await alice.getAddress());
      
      // Should earn ~100% of rewards (single staker)
      const tolerance = REWARD_AMOUNT / 100n; // 1%
      expect(earned).to.be.closeTo(REWARD_AMOUNT, tolerance);
    });
  });
});
```

---

## สรุป Part 20

Staking และ Rewards ที่เรียนรู้:
- ✅ Synthetix StakingRewards (industry standard)
- ✅ rewardPerToken cumulative formula
- ✅ Vote-Escrow (veToken) mechanism
- ✅ Boosted staking (Curve-style)
- ✅ APR/APY calculations
- ✅ Complete test suite

## Quiz

1. `rewardPerTokenStored` ทำงานอย่างไร เปลี่ยนแปลงเมื่อไหร่?
2. ทำไม VeToken ถึงช่วยลด sell pressure?
3. `updateReward` modifier ต้องทำงานก่อน state change เสมอ ทำไม?
4. APR vs APY ต่างกันอย่างไร?

---

## Next: Part 21 - DAO และ Governance
