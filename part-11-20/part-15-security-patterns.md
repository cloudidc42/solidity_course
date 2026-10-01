# Part 15: Security Patterns และ Attack Vectors

## สารบัญ
1. Reentrancy Attacks
2. Integer Overflow/Underflow
3. Front-Running
4. Flash Loan Attacks
5. Oracle Manipulation
6. Access Control Flaws
7. Signature Replay
8. Workshop: Secure Vault

---

## 1. Reentrancy Attacks

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// ❌ VULNERABLE: Classic Reentrancy
contract VulnerableBank {
    
    mapping(address => uint256) public balances;
    
    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }
    
    // ❌ vulnerable: state updated AFTER external call
    function withdraw(uint256 amount) external {
        require(balances[msg.sender] >= amount, "Insufficient");
        
        // 1. External call BEFORE state update
        (bool success,) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
        
        // 2. State update AFTER - too late! Already reentered
        balances[msg.sender] -= amount;
    }
}

// Attacker Contract
contract Attacker {
    VulnerableBank public target;
    address public owner;
    
    constructor(address _target) {
        target = VulnerableBank(_target);
        owner = msg.sender;
    }
    
    function attack() external payable {
        require(msg.value >= 1 ether, "Need 1 ETH");
        target.deposit{value: 1 ether}();
        target.withdraw(1 ether);
    }
    
    // Fallback: called when receiving ETH
    receive() external payable {
        if (address(target).balance >= 1 ether) {
            target.withdraw(1 ether); // Reenter!
        }
    }
    
    function steal() external {
        require(msg.sender == owner);
        (bool sent,) = owner.call{value: address(this).balance}("");
        require(sent);
    }
}

// ✅ FIX 1: Checks-Effects-Interactions Pattern
contract SafeBank_CEI {
    
    mapping(address => uint256) public balances;
    
    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }
    
    function withdraw(uint256 amount) external {
        // 1. Checks
        require(balances[msg.sender] >= amount, "Insufficient");
        
        // 2. Effects (state change FIRST)
        balances[msg.sender] -= amount;
        
        // 3. Interactions (external call LAST)
        (bool success,) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}

// ✅ FIX 2: ReentrancyGuard
contract ReentrancyGuard {
    
    uint256 private constant _NOT_ENTERED = 1;
    uint256 private constant _ENTERED = 2;
    uint256 private _status;
    
    error ReentrancyGuardReentrantCall();
    
    constructor() {
        _status = _NOT_ENTERED;
    }
    
    modifier nonReentrant() {
        _nonReentrantBefore();
        _;
        _nonReentrantAfter();
    }
    
    function _nonReentrantBefore() private {
        if (_status == _ENTERED) {
            revert ReentrancyGuardReentrantCall();
        }
        _status = _ENTERED;
    }
    
    function _nonReentrantAfter() private {
        _status = _NOT_ENTERED;
    }
}

contract SafeBank_Guard is ReentrancyGuard {
    
    mapping(address => uint256) public balances;
    
    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }
    
    // ✅ nonReentrant prevents double calls
    function withdraw(uint256 amount) external nonReentrant {
        require(balances[msg.sender] >= amount, "Insufficient");
        balances[msg.sender] -= amount;
        (bool success,) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}

// ✅ FIX 3: Pull over Push pattern
contract SafeBank_Pull {
    
    mapping(address => uint256) public balances;
    mapping(address => uint256) public pendingWithdrawals;
    
    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }
    
    function requestWithdraw(uint256 amount) external {
        require(balances[msg.sender] >= amount, "Insufficient");
        balances[msg.sender] -= amount;
        pendingWithdrawals[msg.sender] += amount;
    }
    
    // User pulls their own funds (no reentrancy risk)
    function withdrawPending() external nonReentrant {
        uint256 amount = pendingWithdrawals[msg.sender];
        require(amount > 0, "Nothing to withdraw");
        
        pendingWithdrawals[msg.sender] = 0;
        
        (bool success,) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

---

## 2. Integer Overflow (Pre-0.8)

```solidity
// Solidity ^0.8.0 มี overflow protection built-in
// แต่ตรงที่ใช้ unchecked{} ต้องระวัง!

contract OverflowExample {
    
    // ✅ 0.8.0+ - auto reverts on overflow
    function safeAdd(uint256 a, uint256 b) public pure returns (uint256) {
        return a + b; // Reverts if overflow
    }
    
    // ❌ ถ้าใช้ unchecked ต้องตรวจเอง
    function unsafeAdd(uint256 a, uint256 b) public pure returns (uint256) {
        unchecked {
            return a + b; // WRAPS AROUND if overflow!
        }
    }
    
    // ✅ Safe unchecked usage (เมื่อรู้ว่าไม่ overflow)
    function safeLoopCounter() public pure returns (uint256 sum) {
        // ปลอดภัย: i < 100 ดังนั้น i++ ไม่ overflow
        for (uint256 i = 0; i < 100;) {
            sum += i;
            unchecked { ++i; }  // gas optimization
        }
    }
    
    // ❌ Type casting ระวัง!
    function dangerousCast(int256 n) public pure returns (uint256) {
        // ถ้า n = -1, result = type(uint256).max !!
        return uint256(n);
    }
    
    // ✅ Safe casting
    function safeCast(int256 n) public pure returns (uint256) {
        require(n >= 0, "Negative");
        return uint256(n);
    }
}
```

---

## 3. Front-Running Protection

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// ❌ VULNERABLE: Front-runnable
contract VulnerableAuction {
    
    address public highestBidder;
    uint256 public highestBid;
    
    function bid() external payable {
        require(msg.value > highestBid, "Bid too low");
        
        if (highestBidder != address(0)) {
            payable(highestBidder).transfer(highestBid);
        }
        
        highestBidder = msg.sender;
        highestBid = msg.value;
        // Attacker can see this tx, frontrun with higher bid
    }
}

// ✅ Commit-Reveal Scheme
contract CommitRevealAuction {
    
    enum State { Commit, Reveal, Ended }
    
    struct Bid {
        bytes32 commitment; // keccak256(amount, salt)
        uint256 deposit;
        bool revealed;
    }
    
    State public state;
    uint256 public commitEnd;
    uint256 public revealEnd;
    
    address public highestBidder;
    uint256 public highestBid;
    
    mapping(address => Bid) public bids;
    mapping(address => uint256) public refunds;
    
    event BidCommitted(address indexed bidder);
    event BidRevealed(address indexed bidder, uint256 amount);
    event AuctionEnded(address winner, uint256 amount);
    
    constructor(uint256 commitDuration, uint256 revealDuration) {
        commitEnd = block.timestamp + commitDuration;
        revealEnd = commitEnd + revealDuration;
        state = State.Commit;
    }
    
    // Phase 1: Commit (bidder hides actual amount)
    function commit(bytes32 commitment) external payable {
        require(state == State.Commit || block.timestamp < commitEnd, "Commit phase over");
        require(bids[msg.sender].commitment == 0, "Already committed");
        
        bids[msg.sender] = Bid({
            commitment: commitment,
            deposit: msg.value,
            revealed: false
        });
        
        emit BidCommitted(msg.sender);
    }
    
    // Phase 2: Reveal (bidder reveals actual amount)
    function reveal(uint256 amount, bytes32 salt) external {
        require(block.timestamp >= commitEnd && block.timestamp < revealEnd, "Not reveal phase");
        
        Bid storage bid = bids[msg.sender];
        require(!bid.revealed, "Already revealed");
        
        // Verify commitment
        require(
            keccak256(abi.encodePacked(amount, salt)) == bid.commitment,
            "Invalid commitment"
        );
        
        bid.revealed = true;
        
        // Check if deposit covers bid
        if (bid.deposit >= amount) {
            if (amount > highestBid) {
                if (highestBidder != address(0)) {
                    refunds[highestBidder] += highestBid;
                }
                highestBidder = msg.sender;
                highestBid = amount;
                // Refund excess deposit
                uint256 excess = bid.deposit - amount;
                if (excess > 0) refunds[msg.sender] += excess;
            } else {
                refunds[msg.sender] += bid.deposit;
            }
        } else {
            // Deposit too low - forfeit deposit
            refunds[address(0)] += bid.deposit;
        }
        
        emit BidRevealed(msg.sender, amount);
    }
    
    function endAuction() external {
        require(block.timestamp >= revealEnd, "Reveal not over");
        require(state != State.Ended, "Already ended");
        state = State.Ended;
        emit AuctionEnded(highestBidder, highestBid);
    }
    
    function withdraw() external {
        uint256 amount = refunds[msg.sender];
        require(amount > 0, "No refund");
        refunds[msg.sender] = 0;
        (bool sent,) = msg.sender.call{value: amount}("");
        require(sent, "Transfer failed");
    }
    
    // Helper: generate commitment off-chain
    function generateCommitment(uint256 amount, bytes32 salt) public pure returns (bytes32) {
        return keccak256(abi.encodePacked(amount, salt));
    }
}
```

---

## 4. Oracle Manipulation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface AggregatorV3Interface {
    function latestRoundData() external view returns (
        uint80 roundId,
        int256 answer,
        uint256 startedAt,
        uint256 updatedAt,
        uint80 answeredInRound
    );
    function decimals() external view returns (uint8);
}

// ✅ Safe Chainlink Oracle Usage
contract SafeOracleConsumer {
    
    AggregatorV3Interface public immutable priceFeed;
    
    uint256 public constant STALENESS_THRESHOLD = 1 hours;
    int256 public constant MIN_VALID_ANSWER = 1;
    
    error StalePrice(uint256 updatedAt, uint256 threshold);
    error InvalidPrice(int256 price);
    error RoundNotComplete(uint80 roundId, uint80 answeredInRound);
    
    constructor(address _priceFeed) {
        priceFeed = AggregatorV3Interface(_priceFeed);
    }
    
    function getPrice() public view returns (uint256 price, uint8 decimals) {
        (
            uint80 roundId,
            int256 answer,
            ,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = priceFeed.latestRoundData();
        
        // 1. Check round completeness
        if (answeredInRound < roundId) {
            revert RoundNotComplete(roundId, answeredInRound);
        }
        
        // 2. Check staleness
        if (block.timestamp - updatedAt > STALENESS_THRESHOLD) {
            revert StalePrice(updatedAt, STALENESS_THRESHOLD);
        }
        
        // 3. Check valid price
        if (answer <= MIN_VALID_ANSWER) {
            revert InvalidPrice(answer);
        }
        
        return (uint256(answer), priceFeed.decimals());
    }
    
    function getValueInUSD(uint256 tokenAmount, uint8 tokenDecimals) external view returns (uint256) {
        (uint256 price, uint8 priceDecimals) = getPrice();
        
        // Normalize to 18 decimals
        uint256 value = tokenAmount * price;
        
        uint256 scale = tokenDecimals + priceDecimals;
        if (scale > 18) {
            value /= 10 ** (scale - 18);
        } else if (scale < 18) {
            value *= 10 ** (18 - scale);
        }
        
        return value;
    }
}

// TWAP Oracle (Time-Weighted Average Price) - manipulation resistant
contract TWAPOracle {
    
    struct Observation {
        uint256 timestamp;
        uint256 price0Cumulative;
        uint256 price1Cumulative;
    }
    
    Observation[] public observations;
    uint256 public constant WINDOW = 30 minutes;
    
    function update(uint256 price0Cumulative, uint256 price1Cumulative) external {
        observations.push(Observation({
            timestamp: block.timestamp,
            price0Cumulative: price0Cumulative,
            price1Cumulative: price1Cumulative
        }));
    }
    
    function getTWAP() external view returns (uint256 twap0, uint256 twap1) {
        require(observations.length >= 2, "Insufficient data");
        
        Observation memory oldest;
        Observation memory newest = observations[observations.length - 1];
        
        // Find oldest observation within window
        for (uint256 i = observations.length - 1; i > 0; --i) {
            if (block.timestamp - observations[i - 1].timestamp >= WINDOW) {
                oldest = observations[i - 1];
                break;
            }
        }
        
        require(oldest.timestamp > 0, "Window too short");
        
        uint256 timeElapsed = newest.timestamp - oldest.timestamp;
        
        twap0 = (newest.price0Cumulative - oldest.price0Cumulative) / timeElapsed;
        twap1 = (newest.price1Cumulative - oldest.price1Cumulative) / timeElapsed;
    }
}
```

---

## 5. Signature Replay Attack

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// ❌ VULNERABLE: No replay protection
contract VulnerableSigContract {
    function transfer(address to, uint256 amount, bytes calldata sig) external {
        bytes32 hash = keccak256(abi.encodePacked(to, amount));
        address signer = recoverSigner(hash, sig);
        // Process transfer for signer
        // Problem: same sig can be replayed!
    }
    
    function recoverSigner(bytes32 hash, bytes memory sig) internal pure returns (address) {
        bytes32 prefixedHash = keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", hash));
        (bytes32 r, bytes32 s, uint8 v) = splitSig(sig);
        return ecrecover(prefixedHash, v, r, s);
    }
    
    function splitSig(bytes memory sig) internal pure returns (bytes32 r, bytes32 s, uint8 v) {
        assembly {
            r := mload(add(sig, 32))
            s := mload(add(sig, 64))
            v := byte(0, mload(add(sig, 96)))
        }
    }
}

// ✅ SAFE: Nonce + Deadline + ChainId
contract SafeSigContract {
    
    mapping(address => uint256) public nonces;
    
    bytes32 public immutable DOMAIN_SEPARATOR;
    bytes32 public constant TRANSFER_TYPEHASH = 
        keccak256("Transfer(address from,address to,uint256 amount,uint256 nonce,uint256 deadline)");
    
    error InvalidSignature();
    error ExpiredSignature();
    error InvalidNonce();
    
    constructor() {
        DOMAIN_SEPARATOR = keccak256(abi.encode(
            keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
            keccak256("SafeSigContract"),
            keccak256("1"),
            block.chainid,
            address(this)
        ));
    }
    
    function transfer(
        address from,
        address to,
        uint256 amount,
        uint256 deadline,
        bytes calldata sig
    ) external {
        // 1. Check deadline
        if (block.timestamp > deadline) revert ExpiredSignature();
        
        // 2. Build struct hash (includes nonce)
        uint256 nonce = nonces[from];
        bytes32 structHash = keccak256(abi.encode(
            TRANSFER_TYPEHASH,
            from, to, amount, nonce, deadline
        ));
        
        // 3. EIP-712 typed data hash
        bytes32 hash = keccak256(abi.encodePacked("\x19\x01", DOMAIN_SEPARATOR, structHash));
        
        // 4. Recover signer
        (bytes32 r, bytes32 s, uint8 v) = _splitSig(sig);
        address signer = ecrecover(hash, v, r, s);
        
        if (signer == address(0) || signer != from) revert InvalidSignature();
        
        // 5. Increment nonce (prevents replay)
        nonces[from]++;
        
        // Execute transfer
        _doTransfer(from, to, amount);
    }
    
    function _splitSig(bytes calldata sig) internal pure returns (bytes32 r, bytes32 s, uint8 v) {
        require(sig.length == 65, "Invalid sig length");
        assembly {
            r := calldataload(sig.offset)
            s := calldataload(add(sig.offset, 32))
            v := byte(0, calldataload(add(sig.offset, 64)))
        }
    }
    
    function _doTransfer(address from, address to, uint256 amount) internal virtual {}
}
```

---

## 6. Workshop: Secure Vault

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title SecureVault
 * @dev Vault ที่ป้องกัน: reentrancy, access control,
 *      integer overflow, signature replay, oracle manipulation
 */
contract SecureVault is ReentrancyGuard, Ownable2Step {
    
    using SafeERC20 for IERC20;
    
    struct Position {
        uint256 depositAmount;
        uint256 depositTime;
        uint256 lastRewardClaim;
    }
    
    IERC20 public immutable depositToken;
    AggregatorV3Interface public immutable priceOracle;
    
    mapping(address => Position) public positions;
    mapping(address => uint256) public nonces;
    
    uint256 public totalDeposited;
    uint256 public rewardRate = 100; // 1% per day in basis points / 10000
    uint256 public withdrawalDelay = 1 days;
    uint256 public maxDepositUSD = 100_000 * 1e18; // $100k max
    
    bytes32 public immutable DOMAIN_SEPARATOR;
    bytes32 public constant WITHDRAW_TYPEHASH = 
        keccak256("Withdraw(address user,uint256 amount,uint256 nonce,uint256 deadline)");
    
    event Deposited(address indexed user, uint256 amount, uint256 usdValue);
    event Withdrawn(address indexed user, uint256 amount);
    event RewardClaimed(address indexed user, uint256 amount);
    
    error DepositExceedsLimit(uint256 amount, uint256 limit);
    error InsufficientBalance(uint256 available, uint256 requested);
    error TooEarly(uint256 unlockTime, uint256 current);
    error InvalidSignature();
    error ExpiredDeadline();
    
    constructor(address _token, address _oracle) Ownable2Step(msg.sender) {
        depositToken = IERC20(_token);
        priceOracle = AggregatorV3Interface(_oracle);
        
        DOMAIN_SEPARATOR = keccak256(abi.encode(
            keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
            keccak256("SecureVault"),
            keccak256("1"),
            block.chainid,
            address(this)
        ));
    }
    
    // ✅ Checks-Effects-Interactions + amount limit
    function deposit(uint256 amount) external nonReentrant {
        require(amount > 0, "Zero amount");
        
        // Check USD limit using oracle
        uint256 usdValue = _getUSDValue(amount);
        if (usdValue > maxDepositUSD) {
            revert DepositExceedsLimit(usdValue, maxDepositUSD);
        }
        
        // Effects first
        Position storage pos = positions[msg.sender];
        _claimReward(msg.sender); // harvest before modifying
        
        pos.depositAmount += amount;
        pos.depositTime = block.timestamp;
        totalDeposited += amount;
        
        // Interaction last
        depositToken.safeTransferFrom(msg.sender, address(this), amount);
        
        emit Deposited(msg.sender, amount, usdValue);
    }
    
    // ✅ nonReentrant + delay + state update before transfer
    function withdraw(uint256 amount) external nonReentrant {
        Position storage pos = positions[msg.sender];
        
        // Checks
        if (pos.depositAmount < amount) {
            revert InsufficientBalance(pos.depositAmount, amount);
        }
        
        uint256 unlockTime = pos.depositTime + withdrawalDelay;
        if (block.timestamp < unlockTime) {
            revert TooEarly(unlockTime, block.timestamp);
        }
        
        // Effects (harvest + update before transfer)
        _claimReward(msg.sender);
        pos.depositAmount -= amount;
        totalDeposited -= amount;
        
        // Interaction
        depositToken.safeTransfer(msg.sender, amount);
        
        emit Withdrawn(msg.sender, amount);
    }
    
    // ✅ Signature-based withdrawal with nonce + deadline
    function withdrawWithSignature(
        address user,
        uint256 amount,
        uint256 deadline,
        bytes calldata sig
    ) external nonReentrant {
        if (block.timestamp > deadline) revert ExpiredDeadline();
        
        // Verify signature
        uint256 nonce = nonces[user];
        bytes32 structHash = keccak256(abi.encode(
            WITHDRAW_TYPEHASH, user, amount, nonce, deadline
        ));
        bytes32 hash = keccak256(abi.encodePacked("\x19\x01", DOMAIN_SEPARATOR, structHash));
        
        (bytes32 r, bytes32 s, uint8 v) = _splitSig(sig);
        address signer = ecrecover(hash, v, r, s);
        
        if (signer == address(0) || signer != user) revert InvalidSignature();
        
        // Increment nonce before external call
        nonces[user]++;
        
        // Check balance
        Position storage pos = positions[user];
        if (pos.depositAmount < amount) {
            revert InsufficientBalance(pos.depositAmount, amount);
        }
        
        // Effects
        _claimReward(user);
        pos.depositAmount -= amount;
        totalDeposited -= amount;
        
        // Interaction
        depositToken.safeTransfer(user, amount);
        
        emit Withdrawn(user, amount);
    }
    
    function claimReward() external nonReentrant {
        _claimReward(msg.sender);
    }
    
    function _claimReward(address user) internal {
        Position storage pos = positions[user];
        
        if (pos.depositAmount == 0 || pos.lastRewardClaim == 0) {
            pos.lastRewardClaim = block.timestamp;
            return;
        }
        
        uint256 elapsed = block.timestamp - pos.lastRewardClaim;
        uint256 reward = (pos.depositAmount * rewardRate * elapsed) / (10000 * 1 days);
        
        pos.lastRewardClaim = block.timestamp;
        
        if (reward > 0) {
            // Mint or transfer rewards
            emit RewardClaimed(user, reward);
        }
    }
    
    function _getUSDValue(uint256 amount) internal view returns (uint256) {
        (
            uint80 roundId,
            int256 price,
            ,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = priceOracle.latestRoundData();
        
        require(answeredInRound >= roundId, "Stale price");
        require(block.timestamp - updatedAt <= 1 hours, "Price too old");
        require(price > 0, "Invalid price");
        
        uint8 decimals = priceOracle.decimals();
        return (amount * uint256(price)) / (10 ** decimals);
    }
    
    function _splitSig(bytes calldata sig) internal pure returns (bytes32 r, bytes32 s, uint8 v) {
        require(sig.length == 65, "Bad sig length");
        assembly {
            r := calldataload(sig.offset)
            s := calldataload(add(sig.offset, 32))
            v := byte(0, calldataload(add(sig.offset, 64)))
        }
    }
    
    // Admin
    function setMaxDeposit(uint256 newMax) external onlyOwner {
        maxDepositUSD = newMax;
    }
    
    function setWithdrawalDelay(uint256 newDelay) external onlyOwner {
        require(newDelay <= 7 days, "Delay too long");
        withdrawalDelay = newDelay;
    }
    
    function owner() public view returns (address) { return _owner; }
    address private _owner;
}

interface SafeERC20 {
    function safeTransferFrom(IERC20, address, address, uint256) external;
    function safeTransfer(IERC20, address, uint256) external;
}
```

---

## Security Checklist

```
Security Audit ก่อน Deploy:

[ ] Reentrancy
    - ใช้ CEI pattern (Checks → Effects → Interactions)
    - ใช้ nonReentrant modifier
    - ตรวจสอบ state update ก่อน external call

[ ] Access Control  
    - ทุก sensitive function มี modifier
    - onlyOwner, onlyRole, onlyAdmin
    - ตรวจสอบ constructor ตั้ง role ถูกต้อง

[ ] Integer Arithmetic
    - หลีกเลี่ยง unchecked ถ้าไม่จำเป็น
    - ระวัง type casting (int -> uint)
    - ใช้ SafeMath หรือ 0.8+

[ ] Oracle Manipulation
    - ตรวจ staleness
    - ตรวจ valid price range
    - ใช้ TWAP สำหรับ DeFi
    - ใช้หลาย oracle

[ ] Front-Running
    - Commit-reveal สำหรับ auctions
    - Slippage protection ใน DEX
    - Deadline parameter

[ ] Signature Security
    - Include nonce
    - Include deadline
    - Include chainId (domain separator)
    - ใช้ EIP-712

[ ] Flash Loan
    - ไม่ query price ใน same block ที่ manipulate
    - ใช้ TWAP oracle
    - Check before/after balance

[ ] Logic Errors
    - Off-by-one errors
    - Wrong comparison operator
    - Missing validation
```

---

## สรุป Part 15

Security Patterns ที่เรียนรู้:
- ✅ Reentrancy attacks และ 3 วิธีป้องกัน
- ✅ Integer overflow/underflow
- ✅ Front-running (Commit-Reveal)
- ✅ Oracle manipulation (Chainlink best practices)
- ✅ Signature replay attack
- ✅ Secure Vault combining all protections

## Quiz

1. CEI pattern คืออะไร ย่อมาจากอะไร?
2. ทำไม `block.timestamp` ใช้เป็น random source ไม่ได้?
3. Flash Loan attack ทำงานอย่างไร?
4. EIP-712 ช่วยป้องกัน replay attack อย่างไร?

---

## Next: Part 16 - Gas Optimization
