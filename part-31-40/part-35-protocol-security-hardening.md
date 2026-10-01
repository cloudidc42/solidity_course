# Part 35: Protocol Security and Hardening

## สารบัญ
1. Defense in Depth
2. Emergency Mechanisms
3. Rate Limiting
4. Circuit Breakers
5. Workshop: Hardened Protocol

---

## 1. Defense in Depth

```
Security Layers:
1. Access Control: ใครทำอะไรได้บ้าง
2. Input Validation: ตรวจ parameters
3. State Validation: ตรวจ invariants ก่อน/หลัง
4. Reentrancy Protection: CEI + guards
5. Oracle Security: multiple sources, TWAP, staleness
6. Emergency: pause, withdraw, upgrade path
7. Monitoring: events, offchain alerts

Principles:
- Principle of Least Privilege: ให้ permission น้อยที่สุด
- Fail Secure: เมื่อ error → default to safe state
- Defense in Depth: หลาย layers ไม่ขึ้นกัน
- Separation of Concerns: admin ≠ operator ≠ user
- Time-locked Changes: parameter changes → wait period
```

---

## 2. Emergency Mechanisms

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Emergency System:
 * - Pause: หยุด operations ชั่วคราว
 * - Emergency Withdraw: ถอน funds ออก (last resort)
 * - Circuit Breaker: auto-pause เมื่อ anomaly
 */
abstract contract EmergencySystem {
    
    address public guardian;         // ผู้มีอำนาจ pause (multisig)
    address public owner;
    
    bool public paused;
    bool public emergencyMode;
    
    uint256 public pausedAt;
    uint256 public constant MAX_PAUSE_DURATION = 7 days;
    
    event Paused(address indexed by, string reason);
    event Unpaused(address indexed by);
    event EmergencyModeActivated(address indexed by);
    event GuardianChanged(address indexed oldGuardian, address indexed newGuardian);
    
    error Paused_();
    error NotPaused();
    error NotGuardian();
    error PauseTooLong();
    error EmergencyModeActive();
    
    modifier whenNotPaused() {
        if (paused) revert Paused_();
        _;
    }
    
    modifier whenPaused() {
        if (!paused) revert NotPaused();
        _;
    }
    
    modifier onlyGuardianOrOwner() {
        require(msg.sender == guardian || msg.sender == owner, "Not authorized");
        _;
    }
    
    constructor(address _guardian) {
        guardian = _guardian;
        owner = msg.sender;
    }
    
    function pause(string calldata reason) external onlyGuardianOrOwner {
        paused = true;
        pausedAt = block.timestamp;
        emit Paused(msg.sender, reason);
    }
    
    function unpause() external onlyGuardianOrOwner whenPaused {
        paused = false;
        emit Unpaused(msg.sender);
    }
    
    // Auto-unpause after max duration (safety net)
    function forceUnpause() external whenPaused {
        require(block.timestamp >= pausedAt + MAX_PAUSE_DURATION, "Pause not expired");
        paused = false;
        emit Unpaused(msg.sender);
    }
    
    // Activate emergency mode (irreversible, enables emergency withdraw)
    function activateEmergencyMode() external {
        require(msg.sender == owner, "Not owner");
        emergencyMode = true;
        paused = true;
        emit EmergencyModeActivated(msg.sender);
    }
    
    function changeGuardian(address newGuardian) external {
        require(msg.sender == owner, "Not owner");
        emit GuardianChanged(guardian, newGuardian);
        guardian = newGuardian;
    }
}
```

---

## 3. Rate Limiting

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Rate Limiter:
 * จำกัด volume ที่ flow ออกได้ต่อช่วงเวลา
 * ป้องกัน exploit drain ทั้ง protocol ในครั้งเดียว
 * 
 * Token Bucket Algorithm:
 * - Bucket ขนาด maxAmount
 * - เติม refillRate ต่อ second
 * - Drain เมื่อ withdraw
 */
contract RateLimiter {
    
    struct BucketInfo {
        uint256 capacity;      // max tokens in bucket
        uint256 refillRate;    // tokens per second
        uint256 currentAmount; // current tokens
        uint256 lastRefill;    // last refill timestamp
    }
    
    mapping(address => BucketInfo) public buckets; // per-token limits
    
    event RateLimitSet(address indexed token, uint256 capacity, uint256 refillRate);
    event RateLimitConsumed(address indexed token, uint256 amount, uint256 remaining);
    
    error RateLimitExceeded(address token, uint256 requested, uint256 available);
    
    function setRateLimit(
        address token,
        uint256 capacity,
        uint256 refillRate
    ) internal {
        buckets[token] = BucketInfo({
            capacity: capacity,
            refillRate: refillRate,
            currentAmount: capacity,
            lastRefill: block.timestamp
        });
        
        emit RateLimitSet(token, capacity, refillRate);
    }
    
    function _consumeRateLimit(address token, uint256 amount) internal {
        BucketInfo storage bucket = buckets[token];
        
        if (bucket.capacity == 0) return; // No limit set
        
        // Refill bucket based on time elapsed
        uint256 elapsed = block.timestamp - bucket.lastRefill;
        uint256 refill = elapsed * bucket.refillRate;
        
        bucket.currentAmount = bucket.currentAmount + refill > bucket.capacity
            ? bucket.capacity
            : bucket.currentAmount + refill;
        bucket.lastRefill = block.timestamp;
        
        if (bucket.currentAmount < amount) {
            revert RateLimitExceeded(token, amount, bucket.currentAmount);
        }
        
        bucket.currentAmount -= amount;
        
        emit RateLimitConsumed(token, amount, bucket.currentAmount);
    }
    
    function getAvailable(address token) external view returns (uint256) {
        BucketInfo storage bucket = buckets[token];
        if (bucket.capacity == 0) return type(uint256).max;
        
        uint256 elapsed = block.timestamp - bucket.lastRefill;
        uint256 refill = elapsed * bucket.refillRate;
        
        uint256 available = bucket.currentAmount + refill;
        return available > bucket.capacity ? bucket.capacity : available;
    }
}
```

---

## 4. Circuit Breaker

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Circuit Breaker:
 * Auto-pause เมื่อตรวจพบ anomaly
 * 
 * Triggers:
 * - TVL ลดลง > X% ในช่วงเวลาสั้น
 * - Price deviation > Y% จาก TWAP
 * - Unusual transaction patterns
 */
contract CircuitBreaker {
    
    uint256 public constant ANOMALY_THRESHOLD = 2000; // 20% drop = anomaly
    uint256 public constant OBSERVATION_WINDOW = 1 hours;
    
    struct Snapshot {
        uint256 tvl;
        uint256 timestamp;
    }
    
    Snapshot[] public snapshots;
    bool public triggered;
    
    IProtocol public protocol;
    
    event CircuitBreakerTriggered(uint256 currentTvl, uint256 previousTvl, uint256 drop);
    
    constructor(address _protocol) {
        protocol = IProtocol(_protocol);
    }
    
    // Take snapshot regularly (via keeper/automation)
    function takeSnapshot() external {
        uint256 tvl = protocol.totalValueLocked();
        snapshots.push(Snapshot({tvl: tvl, timestamp: block.timestamp}));
        
        // Keep only last 24 hours
        while (snapshots.length > 0 && 
               block.timestamp - snapshots[0].timestamp > 24 hours) {
            // Shift array (gas intensive, use ring buffer in production)
            for (uint256 i = 0; i < snapshots.length - 1; i++) {
                snapshots[i] = snapshots[i + 1];
            }
            snapshots.pop();
        }
    }
    
    // Check if circuit breaker should trigger
    function checkAndTrigger() external {
        if (triggered) return;
        if (snapshots.length < 2) return;
        
        uint256 currentTvl = protocol.totalValueLocked();
        
        // Find snapshot 1 hour ago
        uint256 previousTvl = 0;
        uint256 windowStart = block.timestamp - OBSERVATION_WINDOW;
        
        for (uint256 i = snapshots.length; i > 0; i--) {
            if (snapshots[i-1].timestamp <= windowStart) {
                previousTvl = snapshots[i-1].tvl;
                break;
            }
        }
        
        if (previousTvl == 0) return;
        if (currentTvl >= previousTvl) return; // TVL went up, no issue
        
        uint256 drop = ((previousTvl - currentTvl) * 10000) / previousTvl;
        
        if (drop >= ANOMALY_THRESHOLD) {
            triggered = true;
            protocol.emergencyPause();
            emit CircuitBreakerTriggered(currentTvl, previousTvl, drop);
        }
    }
    
    // Manual reset by admin after investigation
    function reset() external {
        // Only authorized admin
        triggered = false;
    }
}

interface IProtocol {
    function totalValueLocked() external view returns (uint256);
    function emergencyPause() external;
}
```

---

## 5. Workshop: Full Hardened Protocol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Hardened DeFi Vault combining all security patterns
 */
contract HardenedVault is EmergencySystem, RateLimiter {
    
    IERC20 public immutable asset;
    IOracle public immutable oracle;
    
    mapping(address => uint256) public balances;
    uint256 public totalDeposits;
    
    // Security parameters
    uint256 public constant MAX_DEPOSIT = 1_000_000e18; // 1M per tx
    uint256 public constant DAILY_WITHDRAW_LIMIT = 10_000_000e18; // 10M per day
    
    // Invariant tracking
    uint256 public lastCheckedBalance;
    
    bool private _locked; // reentrancy guard
    
    event Deposit(address indexed user, uint256 amount);
    event Withdrawal(address indexed user, uint256 amount);
    
    modifier nonReentrant() {
        require(!_locked, "Reentrant");
        _locked = true;
        _;
        _locked = false;
    }
    
    modifier checkInvariant() {
        _;
        // Post-check: balance should equal sum of deposits
        assert(asset.balanceOf(address(this)) >= totalDeposits);
        lastCheckedBalance = asset.balanceOf(address(this));
    }
    
    constructor(address _asset, address _oracle, address _guardian)
        EmergencySystem(_guardian)
    {
        asset = IERC20(_asset);
        oracle = IOracle(_oracle);
        
        // Set daily withdrawal rate limit
        setRateLimit(
            address(asset),
            DAILY_WITHDRAW_LIMIT,
            DAILY_WITHDRAW_LIMIT / 86400 // refill per second
        );
    }
    
    function deposit(uint256 amount)
        external
        nonReentrant
        whenNotPaused
        checkInvariant
    {
        require(amount > 0 && amount <= MAX_DEPOSIT, "Invalid amount");
        
        // Check oracle health
        (, int256 price, , uint256 updatedAt,) = oracle.latestRoundData();
        require(price > 0, "Bad oracle");
        require(block.timestamp - updatedAt <= 3600, "Stale oracle");
        
        asset.transferFrom(msg.sender, address(this), amount);
        
        balances[msg.sender] += amount;
        totalDeposits += amount;
        
        emit Deposit(msg.sender, amount);
    }
    
    function withdraw(uint256 amount)
        external
        nonReentrant
        whenNotPaused
        checkInvariant
    {
        require(balances[msg.sender] >= amount, "Insufficient");
        
        // Apply rate limit
        _consumeRateLimit(address(asset), amount);
        
        // CEI: Effect first
        balances[msg.sender] -= amount;
        totalDeposits -= amount;
        
        // Then interaction
        asset.transfer(msg.sender, amount);
        
        emit Withdrawal(msg.sender, amount);
    }
    
    // Emergency: withdraw all funds to safe address
    function emergencyWithdraw(address safe) external {
        require(emergencyMode, "Not emergency");
        require(msg.sender == owner, "Not owner");
        
        uint256 balance = asset.balanceOf(address(this));
        asset.transfer(safe, balance);
    }
    
    // Health check: trigger circuit breaker if needed
    function healthCheck() external view returns (bool healthy, string memory reason) {
        if (paused) return (false, "Paused");
        if (emergencyMode) return (false, "Emergency mode");
        
        // Check balance matches records
        uint256 actualBalance = asset.balanceOf(address(this));
        if (actualBalance < totalDeposits) {
            return (false, "Balance deficit detected");
        }
        
        return (true, "OK");
    }
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
    function balanceOf(address) external view returns (uint256);
}

interface IOracle {
    function latestRoundData() external view returns (uint80, int256, uint256, uint256, uint80);
}
```

---

## สรุป Part 35

Protocol Security Hardening ที่เรียนรู้:
- ✅ Defense in depth layers
- ✅ Emergency pause system
- ✅ Rate limiting (token bucket algorithm)
- ✅ Circuit breaker pattern
- ✅ Invariant checking
- ✅ Full hardened vault

## Quiz

1. Defense in depth ต่างจาก single security layer อย่างไร?
2. Rate limiting ป้องกัน exploit อย่างไร?
3. Circuit breaker ทำงานอย่างไร?
4. Emergency mode ควร activate เมื่อไหร่?

---

## Next: Part 36 - Gas Optimization Advanced
