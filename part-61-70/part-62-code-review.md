# Part 62: Code Review Best Practices

## บทนำ

Code Review เป็นกระบวนการสำคัญที่สุดอย่างหนึ่งในการพัฒนา Smart Contract การ Review ที่ดีช่วย:
- ตรวจจับ Security Vulnerabilities ก่อน Deploy
- ปรับปรุง Code Quality และ Maintainability
- แชร์ความรู้ในทีม
- บังคับใช้ Coding Standards

## 1. Code Review Checklist สำหรับ Solidity PRs

### 1.1 Security Checklist

```markdown
## Security Review Checklist

### Reentrancy
- [ ] ทุก function ที่ transfer ETH/Token ใช้ CEI pattern (Checks-Effects-Interactions)
- [ ] ใช้ ReentrancyGuard ใน functions ที่มีความเสี่ยง
- [ ] ไม่มีการเรียก external contracts ระหว่าง state changes

### Access Control
- [ ] ทุก privileged function มี modifier ที่เหมาะสม
- [ ] ไม่มี function ที่ควรเป็น internal/private แต่เป็น public/external
- [ ] Owner/Admin keys ถูก protect ด้วย multisig หรือ timelock

### Integer Arithmetic
- [ ] ไม่มี overflow/underflow ที่ไม่ได้ตั้งใจ (Solidity 0.8+ ช่วย)
- [ ] Division ปัดทิศทางถูกต้อง (favor protocol not user)
- [ ] ไม่มี phantom overflow ใน intermediate calculations

### Oracle / External Calls
- [ ] Oracle ถูก validate (freshness, bounds checking)
- [ ] External calls handle failure correctly
- [ ] Flash loan attacks ถูกพิจารณา

### Input Validation
- [ ] ทุก parameter ถูก validate (zero address, zero amount, bounds)
- [ ] Array lengths ถูก validate ป้องกัน out-of-gas

### Signature/Permit
- [ ] ใช้ EIP-712 typed data
- [ ] Nonces ป้องกัน replay attacks
- [ ] Deadline validation

### Upgradeability
- [ ] Storage layout ไม่ขัดแย้งกับ proxy
- [ ] Initializer ถูก guard ด้วย initializer modifier
- [ ] ไม่มี constructor logic ใน upgradeable contracts
```

### 1.2 Gas Optimization Checklist

```markdown
## Gas Optimization Checklist

### Storage
- [ ] ใช้ packed storage (multiple variables ใน 1 slot)
- [ ] Immutables/Constants แทน storage variables ที่ไม่เปลี่ยน
- [ ] ลด storage reads ด้วย caching ใน local variables
- [ ] ใช้ uint128/uint64 เมื่อ data fit

### Loops
- [ ] ไม่มี unbounded loops (DoS potential)
- [ ] Caching array.length ใน loop variable
- [ ] ใช้ unchecked {} สำหรับ arithmetic ที่รู้ว่าไม่ overflow

### Calldata vs Memory
- [ ] Function parameters ใช้ calldata แทน memory เมื่อเป็นไปได้
- [ ] Return values ที่ใหญ่ใช้ memory reference

### Events
- [ ] ข้อมูลที่ไม่ต้องค้นหาไม่ควร index
- [ ] ไม่มีข้อมูลซ้ำซ้อนใน events

### Custom Errors vs Require Strings
- [ ] ใช้ custom errors แทน require strings
```

### 1.3 Code Style Checklist

```markdown
## Code Style Checklist

### Naming
- [ ] Contract names: PascalCase
- [ ] Function names: camelCase
- [ ] Constants: SCREAMING_SNAKE_CASE
- [ ] Private/internal: _prefix
- [ ] Events: PascalCase (past tense สำหรับ actions)

### Documentation
- [ ] ทุก public/external function มี @notice
- [ ] ทุก parameter มี @param
- [ ] ทุก non-trivial logic มี inline comments
- [ ] Complex algorithms มี reference/formula

### Organization
- [ ] Solidity layout order: pragma → imports → errors → interfaces → libraries → contracts
- [ ] Contract layout order: type declarations → state variables → events → errors → modifiers → constructor → functions
- [ ] Functions order: external → public → internal → private, view/pure last

### Testing
- [ ] Test file exists สำหรับทุก contract
- [ ] Happy path tests
- [ ] Revert tests (ทุก require/custom error)
- [ ] Edge cases (zero values, max values)
- [ ] Integration tests กับ contracts อื่น
```

## 2. Before/After Examples: Bad Code → Reviewed/Improved Code

### 2.1 Reentrancy Vulnerability

```solidity
// ❌ BEFORE: Reentrancy vulnerability
// Review comment: "HIGH SEVERITY: Reentrancy attack possible.
//                 Transfer ETH before updating state."
contract VulnerableWithdraw {
    mapping(address => uint256) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    // ❌ BAD: State updated AFTER external call
    function withdraw() external {
        uint256 amount = balances[msg.sender];
        require(amount > 0, "No balance");
        
        // ❌ External call BEFORE state update
        (bool success,) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
        
        // ❌ State updated AFTER - allows reentrancy!
        balances[msg.sender] = 0;
    }
}

// ✅ AFTER: Fixed with CEI pattern
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract SecureWithdraw is ReentrancyGuard {
    mapping(address => uint256) public balances;

    event Deposited(address indexed user, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);

    error InsufficientBalance(uint256 requested, uint256 available);
    error TransferFailed();

    function deposit() external payable {
        balances[msg.sender] += msg.value;
        emit Deposited(msg.sender, msg.value);
    }

    // ✅ GOOD: CEI pattern + ReentrancyGuard
    function withdraw() external nonReentrant {
        uint256 amount = balances[msg.sender];
        if (amount == 0) revert InsufficientBalance(0, 0);

        // ✅ CHECKS: validate state
        // ✅ EFFECTS: update state BEFORE external call
        balances[msg.sender] = 0;

        // ✅ INTERACTIONS: external call LAST
        (bool success,) = msg.sender.call{value: amount}("");
        if (!success) revert TransferFailed();

        emit Withdrawn(msg.sender, amount);
    }
}
```

### 2.2 Access Control Issues

```solidity
// ❌ BEFORE: Missing access control
// Review comment: "CRITICAL: Anyone can call setPrice and drain protocol.
//                 Add proper access control."
contract VulnerableOracle {
    uint256 public price;
    address public owner;

    constructor() {
        owner = msg.sender;
        price = 1e18;
    }

    // ❌ NO ACCESS CONTROL - anyone can set price!
    function setPrice(uint256 newPrice) external {
        price = newPrice;
    }

    // ❌ Wrong pattern: tx.origin instead of msg.sender
    function adminFunction() external {
        require(tx.origin == owner, "Not owner"); // ❌ phishing attack possible
        // admin logic
    }
}

// ✅ AFTER: Proper access control
import {Ownable2Step} from "@openzeppelin/contracts/access/Ownable2Step.sol";
import {AccessControl} from "@openzeppelin/contracts/access/AccessControl.sol";

contract SecureOracle is AccessControl {
    uint256 public price;
    uint256 public lastUpdateTime;

    // ✅ Defined roles
    bytes32 public constant PRICE_UPDATER_ROLE = keccak256("PRICE_UPDATER_ROLE");
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");

    // ✅ Bounds for price validation
    uint256 public constant MIN_PRICE = 1e14; // $0.0001
    uint256 public constant MAX_PRICE = 1e24; // $1,000,000

    // ✅ Staleness threshold
    uint256 public constant MAX_PRICE_AGE = 1 hours;

    error PriceOutOfBounds(uint256 price, uint256 min, uint256 max);
    error PriceStale(uint256 lastUpdate, uint256 maxAge);

    event PriceUpdated(uint256 oldPrice, uint256 newPrice, uint256 timestamp);

    constructor(address admin) {
        _grantRole(DEFAULT_ADMIN_ROLE, admin);
        _grantRole(ADMIN_ROLE, admin);
        price = 1e18;
        lastUpdateTime = block.timestamp;
    }

    // ✅ Proper access control with msg.sender
    function setPrice(uint256 newPrice) external onlyRole(PRICE_UPDATER_ROLE) {
        if (newPrice < MIN_PRICE || newPrice > MAX_PRICE) {
            revert PriceOutOfBounds(newPrice, MIN_PRICE, MAX_PRICE);
        }

        uint256 oldPrice = price;
        price = newPrice;
        lastUpdateTime = block.timestamp;

        emit PriceUpdated(oldPrice, newPrice, block.timestamp);
    }

    // ✅ Staleness check
    function getValidPrice() external view returns (uint256) {
        if (block.timestamp - lastUpdateTime > MAX_PRICE_AGE) {
            revert PriceStale(lastUpdateTime, MAX_PRICE_AGE);
        }
        return price;
    }
}
```

### 2.3 Gas Inefficiency

```solidity
// ❌ BEFORE: Gas inefficient
// Review comment: "GAS: Multiple storage reads, unbounded loop.
//                 Cache storage reads, add pagination."
contract GasInefficient {
    struct User {
        uint256 balance;
        uint256 rewards;
        uint256 lastClaim;
        bool active;
    }

    mapping(address => User) public users;
    address[] public userList;

    // ❌ Multiple storage reads สำหรับ users[msg.sender]
    function claimReward() external {
        require(users[msg.sender].active, "Not active");
        require(block.timestamp > users[msg.sender].lastClaim + 1 days, "Too soon");
        
        uint256 reward = users[msg.sender].balance * 100 / 10000; // 1%
        users[msg.sender].rewards += reward;
        users[msg.sender].lastClaim = block.timestamp;
        users[msg.sender].balance += reward;
    }

    // ❌ Unbounded loop - can run out of gas
    function totalBalance() external view returns (uint256 total) {
        for (uint256 i = 0; i < userList.length; i++) { // ❌ reads length every iteration
            total += users[userList[i]].balance;
        }
    }

    // ❌ Memory parameter that should be calldata
    function processAddresses(address[] memory addresses) external {
        // process...
    }
}

// ✅ AFTER: Gas optimized
contract GasEfficient {
    // ✅ Packed struct (fits in 2 slots instead of 4)
    struct User {
        uint128 balance;    // slot 1: 16 bytes
        uint64 lastClaim;   // slot 1: 8 bytes
        bool active;        // slot 1: 1 byte
        // slot 1 total: 25 bytes (fits in 32 byte slot)
        uint256 rewards;    // slot 2: 32 bytes
    }

    mapping(address => User) public users;

    // ✅ Keep aggregate to avoid loop
    uint256 public totalBalance;

    error NotActive();
    error ClaimTooSoon(uint256 nextClaimTime);

    function claimReward() external {
        // ✅ Cache storage read
        User storage user = users[msg.sender];

        if (!user.active) revert NotActive();
        
        uint256 nextClaim = uint256(user.lastClaim) + 1 days;
        if (block.timestamp <= nextClaim) revert ClaimTooSoon(nextClaim);

        // ✅ Single storage access pattern
        uint256 balance = user.balance;
        uint256 reward = balance * 100 / 10000;
        
        unchecked {
            // ✅ Safe: reward < balance, balance < uint128 max
            user.balance = uint128(balance + reward);
            user.rewards += reward;
            totalBalance += reward;
        }
        user.lastClaim = uint64(block.timestamp);
    }

    // ✅ Paginated view function
    function totalBalancePaginated(
        address[] calldata userAddresses // ✅ calldata not memory
    ) external view returns (uint256 total) {
        uint256 len = userAddresses.length;
        for (uint256 i; i < len;) { // ✅ cache length, start from 0 uninitialized
            total += users[userAddresses[i]].balance;
            unchecked { ++i; } // ✅ unchecked increment
        }
    }
}
```

### 2.4 Integer Precision Issues

```solidity
// ❌ BEFORE: Precision loss
// Review comment: "MATH: Division before multiplication loses precision.
//                 Reorder operations."
contract PrecisionLoss {
    uint256 public constant RATE = 3; // 3%
    uint256 public constant BASIS = 100;

    // ❌ Division before multiplication = precision loss
    function calculateReward(uint256 amount, uint256 duration) external pure returns (uint256) {
        return amount / BASIS * RATE * duration / 365; // ❌ loses precision
    }

    // ❌ No rounding direction consideration
    function convertShares(uint256 assets, uint256 totalAssets, uint256 totalShares) 
        external pure returns (uint256) 
    {
        return assets * totalShares / totalAssets; // may round down or up randomly
    }
}

// ✅ AFTER: Proper precision handling
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";

contract PrecisionCorrect {
    using Math for uint256;

    uint256 public constant RATE_BPS = 300; // 3% in basis points
    uint256 public constant MAX_BPS = 10_000;
    uint256 public constant PRECISION = 1e18;

    // ✅ Multiply before divide
    function calculateReward(uint256 amount, uint256 duration) external pure returns (uint256) {
        // ✅ amount * RATE_BPS * duration first, then divide
        return amount * RATE_BPS * duration / (MAX_BPS * 365);
    }

    // ✅ Explicit rounding direction
    function convertSharesToDeposit(
        uint256 assets, 
        uint256 totalAssets, 
        uint256 totalShares
    ) external pure returns (uint256) {
        // ✅ Round DOWN when depositing (user gets fewer shares = safer for protocol)
        return assets.mulDiv(totalShares, totalAssets, Math.Rounding.Floor);
    }

    function convertSharesToWithdraw(
        uint256 shares,
        uint256 totalAssets,
        uint256 totalShares
    ) external pure returns (uint256) {
        // ✅ Round DOWN when withdrawing (user gets fewer assets = safer for protocol)
        return shares.mulDiv(totalAssets, totalShares, Math.Rounding.Floor);
    }
}
```

### 2.5 Missing Events / Incorrect Event Design

```solidity
// ❌ BEFORE: Missing or poorly designed events
// Review comment: "EVENTS: Missing events for state changes.
//                 Index important fields for filtering."
contract PoorEvents {
    mapping(address => uint256) public stakes;

    // ❌ No event emitted
    function stake(uint256 amount) external {
        stakes[msg.sender] += amount;
        // ❌ MISSING: emit Staked(msg.sender, amount)
    }

    // ❌ Event with wrong indexing
    event Transfer(
        address from,    // ❌ should be indexed
        address to,      // ❌ should be indexed  
        uint256 amount,  // ❌ don't need to index
        bytes32 data     // ❌ indexed bytes32 is fine but wasteful if not used for filtering
    );

    // ❌ Event emitted before state change (misleading if tx reverts)
    function transfer(address to, uint256 amount) external {
        emit Transfer(msg.sender, to, amount, bytes32(0)); // ❌ emitted before validation
        require(stakes[msg.sender] >= amount, "Insufficient");
        stakes[msg.sender] -= amount;
        stakes[to] += amount;
    }
}

// ✅ AFTER: Well-designed events
contract GoodEvents {
    mapping(address => uint256) public stakes;
    uint256 public totalStaked;

    // ✅ Indexed fields for filtering, emit after state change
    event Staked(address indexed user, uint256 amount, uint256 newTotal);
    event Unstaked(address indexed user, uint256 amount, uint256 newTotal);
    event Transfer(
        address indexed from,   // ✅ indexed for filtering by sender
        address indexed to,     // ✅ indexed for filtering by recipient
        uint256 amount          // not indexed (continuous value = bad for indexing)
    );

    error InsufficientStake(uint256 requested, uint256 available);
    error ZeroAmount();

    // ✅ Emit after state change, include useful context
    function stake(uint256 amount) external {
        if (amount == 0) revert ZeroAmount();
        
        stakes[msg.sender] += amount;
        totalStaked += amount;
        
        // ✅ Emit AFTER state change with context
        emit Staked(msg.sender, amount, stakes[msg.sender]);
    }

    // ✅ Validation before emit
    function transfer(address to, uint256 amount) external {
        if (amount == 0) revert ZeroAmount();
        if (stakes[msg.sender] < amount) {
            revert InsufficientStake(amount, stakes[msg.sender]);
        }

        // ✅ State change before emit
        stakes[msg.sender] -= amount;
        stakes[to] += amount;
        
        // ✅ Emit after successful state change
        emit Transfer(msg.sender, to, amount);
    }
}
```

## 3. PR Review Workflow กับ GitHub Actions CI

### 3.1 GitHub Actions Workflow

```yaml
# .github/workflows/pr-review.yml
name: Smart Contract PR Review

on:
  pull_request:
    branches: [main, develop]
    paths:
      - 'src/**/*.sol'
      - 'test/**/*.sol'
      - 'foundry.toml'

jobs:
  # ─────────────────────────────────────────────
  # Formatting Check
  # ─────────────────────────────────────────────
  format:
    name: Check Formatting
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1
        with:
          version: nightly
      
      - name: Check Forge Format
        run: forge fmt --check
        
  # ─────────────────────────────────────────────
  # Build & Test
  # ─────────────────────────────────────────────
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive
      
      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1
        with:
          version: nightly
      
      - name: Build
        run: forge build --sizes
      
      - name: Run Tests
        run: forge test -vvv
      
      - name: Coverage Report
        run: |
          forge coverage --report lcov
          genhtml lcov.info -o coverage-report
      
      - name: Upload Coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage-report/

  # ─────────────────────────────────────────────
  # Static Analysis
  # ─────────────────────────────────────────────
  slither:
    name: Slither Static Analysis
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive
      
      - name: Run Slither
        uses: crytic/slither-action@v0.4.0
        with:
          target: 'src/'
          slither-args: '--filter-paths "lib/"'
          fail-on: high
          sarif: results.sarif
      
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: results.sarif

  # ─────────────────────────────────────────────
  # Gas Snapshot
  # ─────────────────────────────────────────────
  gas-snapshot:
    name: Gas Snapshot
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive
      
      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1
      
      - name: Create Gas Snapshot
        run: forge snapshot --check
        # ถ้า gas เพิ่มขึ้น PR จะ fail

  # ─────────────────────────────────────────────
  # NatSpec Check
  # ─────────────────────────────────────────────
  natspec:
    name: NatSpec Completeness
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1
      
      - name: Build with NatSpec
        run: forge build --doc
      
      - name: Check NatSpec Coverage
        run: |
          # Custom script to check NatSpec completeness
          python3 scripts/check-natspec.py src/
```

### 3.2 Automated Review Bot Configuration

```yaml
# .github/workflows/review-bot.yml
name: Automated Review Suggestions

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  suggest-improvements:
    name: Suggest Code Improvements
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Get Changed Files
        id: changed-files
        uses: tj-actions/changed-files@v44
        with:
          files: "**/*.sol"
      
      - name: Run Custom Checks
        if: steps.changed-files.outputs.any_changed == 'true'
        run: |
          for file in ${{ steps.changed-files.outputs.all_changed_files }}; do
            echo "Checking: $file"
            # ตรวจสอบ require strings (should use custom errors)
            if grep -n "require(.*\"" "$file"; then
              echo "::warning file=$file::Consider using custom errors instead of require strings"
            fi
            # ตรวจสอบ tx.origin
            if grep -n "tx.origin" "$file"; then
              echo "::error file=$file::tx.origin usage detected - security risk"
            fi
          done
```

### 3.3 PR Template

```markdown
<!-- .github/PULL_REQUEST_TEMPLATE.md -->
## Summary

<!-- Brief description of changes -->

## Type of Change

- [ ] Bug fix (ไม่ทำให้ feature เดิมเสียหาย)
- [ ] New feature (เพิ่ม functionality ใหม่)
- [ ] Breaking change (เปลี่ยน API หรือ behavior)
- [ ] Documentation update
- [ ] Gas optimization

## Security Checklist

- [ ] ตรวจสอบ reentrancy vulnerabilities
- [ ] Access control ถูกต้อง
- [ ] Input validation ครบถ้วน
- [ ] ไม่มี integer overflow/underflow
- [ ] Events emit ถูกต้อง

## Testing

- [ ] Unit tests เพิ่ม/อัปเดต
- [ ] Integration tests ผ่าน
- [ ] Edge cases ถูกทดสอบ
- [ ] `forge test` ผ่านทั้งหมด
- [ ] Gas snapshot อัปเดต

## Gas Impact

<!-- รายงาน gas changes -->
| Function | Before | After | Change |
|----------|--------|-------|--------|
| deposit() | 45000 | 43000 | -2000 (-4.4%) |

## Deployment Notes

<!-- ถ้า require migration หรือ specific deployment steps -->

## Reviewers

@reviewer1 @reviewer2
```

## 4. Pair Programming Patterns สำหรับ Smart Contract Development

### 4.1 Driver-Navigator Pattern

```
ใน Pair Programming สำหรับ Smart Contract:

Navigator (คิด): 
- วางแผน Security Model
- ตรวจสอบ Business Logic
- ระวัง Edge Cases
- ดู Big Picture

Driver (เขียน code):
- Focus ที่ current task
- Type code ตาม direction
- ถามเมื่อไม่แน่ใจ
- หา syntax/API ที่ถูกต้อง
```

### 4.2 Test-First Pair Programming

```solidity
// ขั้นตอน: เขียน test ก่อน แล้วค่อย implement

// 1. Navigator เขียน test spec
// 2. Driver เขียน test code
// 3. ทั้งคู่ discuss expected behavior
// 4. Driver implement ให้ test ผ่าน
// 5. Navigator review implementation

// STEP 1: เขียน test ก่อน
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test} from "forge-std/Test.sol";

contract EscrowTest is Test {
    Escrow escrow;
    address buyer = makeAddr("buyer");
    address seller = makeAddr("seller");
    address arbiter = makeAddr("arbiter");

    function setUp() public {
        escrow = new Escrow(arbiter);
    }

    // ✅ Test spec เขียนก่อน implement
    function test_deposit_storesCorrectAmount() public {
        vm.deal(buyer, 1 ether);
        vm.prank(buyer);
        escrow.createDeal{value: 1 ether}(seller, 30 days);
        
        assertEq(escrow.getDealAmount(0), 1 ether);
        assertEq(escrow.getDealSeller(0), seller);
    }

    function test_release_transfersToSeller() public {
        // setup
        vm.deal(buyer, 1 ether);
        vm.prank(buyer);
        escrow.createDeal{value: 1 ether}(seller, 30 days);
        
        // action
        vm.prank(buyer);
        escrow.confirmDelivery(0);
        
        // assert
        assertEq(seller.balance, 1 ether);
    }

    function test_release_revertsIfNotBuyer() public {
        vm.deal(buyer, 1 ether);
        vm.prank(buyer);
        escrow.createDeal{value: 1 ether}(seller, 30 days);
        
        // ✅ Test revert cases
        vm.prank(makeAddr("random"));
        vm.expectRevert(Escrow.NotBuyer.selector);
        escrow.confirmDelivery(0);
    }

    function test_dispute_allowsArbiterToDecide() public {
        vm.deal(buyer, 1 ether);
        vm.prank(buyer);
        escrow.createDeal{value: 1 ether}(seller, 30 days);
        
        vm.prank(buyer);
        escrow.raiseDispute(0);
        
        vm.prank(arbiter);
        escrow.resolveDispute(0, seller, 7000, 3000); // 70% seller, 30% buyer
        
        assertApproxEqAbs(seller.balance, 0.7 ether, 1);
        assertApproxEqAbs(buyer.balance, 0.3 ether, 1);
    }
}

// STEP 2: Implement ให้ test ผ่าน
contract Escrow {
    struct Deal {
        address payable buyer;
        address payable seller;
        uint256 amount;
        uint256 deadline;
        DealState state;
    }

    enum DealState { Active, Completed, Disputed, Resolved }

    address public immutable arbiter;
    Deal[] public deals;

    error NotBuyer();
    error NotArbiter();
    error InvalidDeal();
    error InvalidState();

    event DealCreated(uint256 indexed dealId, address buyer, address seller, uint256 amount);
    event DealCompleted(uint256 indexed dealId);
    event DisputeRaised(uint256 indexed dealId);
    event DisputeResolved(uint256 indexed dealId, uint256 sellerPct, uint256 buyerPct);

    constructor(address _arbiter) {
        arbiter = _arbiter;
    }

    function createDeal(address payable _seller, uint256 _duration) external payable returns (uint256 dealId) {
        dealId = deals.length;
        deals.push(Deal({
            buyer: payable(msg.sender),
            seller: _seller,
            amount: msg.value,
            deadline: block.timestamp + _duration,
            state: DealState.Active
        }));
        emit DealCreated(dealId, msg.sender, _seller, msg.value);
    }

    function confirmDelivery(uint256 dealId) external {
        Deal storage deal = deals[dealId];
        if (msg.sender != deal.buyer) revert NotBuyer();
        if (deal.state != DealState.Active) revert InvalidState();
        
        deal.state = DealState.Completed;
        deal.seller.transfer(deal.amount);
        emit DealCompleted(dealId);
    }

    function raiseDispute(uint256 dealId) external {
        Deal storage deal = deals[dealId];
        if (msg.sender != deal.buyer) revert NotBuyer();
        if (deal.state != DealState.Active) revert InvalidState();
        
        deal.state = DealState.Disputed;
        emit DisputeRaised(dealId);
    }

    function resolveDispute(
        uint256 dealId, 
        address, // ignored, use deal.seller
        uint256 sellerPct, 
        uint256 buyerPct
    ) external {
        if (msg.sender != arbiter) revert NotArbiter();
        Deal storage deal = deals[dealId];
        if (deal.state != DealState.Disputed) revert InvalidState();
        require(sellerPct + buyerPct == 10_000, "Must sum to 100%");
        
        deal.state = DealState.Resolved;
        
        uint256 sellerAmount = deal.amount * sellerPct / 10_000;
        uint256 buyerAmount = deal.amount - sellerAmount;
        
        if (sellerAmount > 0) deal.seller.transfer(sellerAmount);
        if (buyerAmount > 0) deal.buyer.transfer(buyerAmount);
        
        emit DisputeResolved(dealId, sellerPct, buyerPct);
    }

    function getDealAmount(uint256 dealId) external view returns (uint256) {
        return deals[dealId].amount;
    }

    function getDealSeller(uint256 dealId) external view returns (address) {
        return deals[dealId].seller;
    }
}
```

## 5. Common Review Comments และวิธีแก้ไข

### 5.1 Review Comment Examples

```solidity
// ─────────────────────────────────────────────
// COMMENT 1: "Use custom errors instead of require strings"
// ─────────────────────────────────────────────

// ❌ Before
function transfer(address to, uint256 amount) external {
    require(to != address(0), "Transfer to zero address");
    require(amount > 0, "Amount must be positive");
    require(balances[msg.sender] >= amount, "Insufficient balance");
}

// ✅ After
error TransferToZeroAddress();
error ZeroAmount();
error InsufficientBalance(uint256 requested, uint256 available);

function transfer(address to, uint256 amount) external {
    if (to == address(0)) revert TransferToZeroAddress();
    if (amount == 0) revert ZeroAmount();
    if (balances[msg.sender] < amount) {
        revert InsufficientBalance(amount, balances[msg.sender]);
    }
}

// ─────────────────────────────────────────────
// COMMENT 2: "Consider using SafeERC20 for token transfers"
// ─────────────────────────────────────────────

// ❌ Before - USDT doesn't return bool!
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

function depositToken(address token, uint256 amount) external {
    IERC20(token).transferFrom(msg.sender, address(this), amount); // ❌ ignores return value
}

// ✅ After
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

contract TokenVault {
    using SafeERC20 for IERC20;

    function depositToken(address token, uint256 amount) external {
        IERC20(token).safeTransferFrom(msg.sender, address(this), amount); // ✅ handles non-standard tokens
    }
}

// ─────────────────────────────────────────────
// COMMENT 3: "Magic numbers should be named constants"
// ─────────────────────────────────────────────

// ❌ Before
function calculateFee(uint256 amount) external pure returns (uint256) {
    return amount * 250 / 10000; // ❌ What is 250? What is 10000?
}

// ✅ After
uint256 private constant PROTOCOL_FEE_BPS = 250;  // 2.5%
uint256 private constant BASIS_POINTS = 10_000;    // 100%

function calculateFee(uint256 amount) external pure returns (uint256) {
    return amount * PROTOCOL_FEE_BPS / BASIS_POINTS; // ✅ self-documenting
}

// ─────────────────────────────────────────────
// COMMENT 4: "State variable can be immutable"
// ─────────────────────────────────────────────

// ❌ Before - wastes 1 SLOAD per access
contract TokenPool {
    address public token; // ❌ never changed after constructor

    constructor(address _token) {
        token = _token;
    }
}

// ✅ After - reads from code, saves gas
contract TokenPoolOptimized {
    address public immutable token; // ✅ read from bytecode

    constructor(address _token) {
        token = _token;
    }
}

// ─────────────────────────────────────────────
// COMMENT 5: "Missing zero-address check"
// ─────────────────────────────────────────────

// ❌ Before
constructor(address _treasury) {
    treasury = _treasury; // ❌ if address(0) passed, fees are lost forever
}

// ✅ After
error ZeroAddress(string param);

constructor(address _treasury) {
    if (_treasury == address(0)) revert ZeroAddress("treasury");
    treasury = _treasury;
}
```

### 5.2 Review Response Templates

```markdown
<!-- การตอบ review comments อย่างมืออาชีพ -->

<!-- เมื่อ agree และ fix -->
"Fixed in commit abc1234. Changed from `require` string to custom error `InsufficientBalance`."

<!-- เมื่อ agree แต่ defer -->
"Good catch! This is out of scope for this PR but I've created issue #123 to track this."

<!-- เมื่อ disagree ด้วยเหตุผล -->
"I considered this but decided against it because:
1. The additional SLOAD cost is negligible here (called once in tx)
2. Using immutable would require significant refactoring
3. We plan to add upgradeability in v2 anyway

Happy to discuss further if you feel strongly about it."

<!-- เมื่อไม่เข้าใจ -->
"Could you elaborate on what specific scenario you're concerned about?
I want to make sure I understand the risk before making changes."
```

## 6. Automated Security Scanning Integration

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title ReviewChecks
/// @notice ตัวอย่าง Contract ที่ผ่าน automated review checks ทั้งหมด
/// @dev ใช้เป็น template สำหรับ new contracts
contract ReviewChecks {
    // ✅ Named imports
    // import {IERC20} from "..."; // ไม่ใช้ wildcard import

    // ✅ Custom errors (ไม่ใช้ require strings)
    error Unauthorized(address caller, address expected);
    error InvalidAmount(uint256 amount, string reason);
    error ZeroAddress(string param);

    // ✅ Named constants (ไม่ใช้ magic numbers)
    uint256 public constant MAX_FEE_BPS = 1_000;  // 10%
    uint256 public constant BASIS_POINTS = 10_000;
    uint256 public constant LOCK_PERIOD = 7 days;

    // ✅ Immutables สำหรับ values ที่ไม่เปลี่ยน
    address public immutable owner;
    uint256 public immutable deployedAt;

    // ✅ Storage packing
    struct Config {
        uint128 feeBps;
        uint64 lastUpdate;
        bool paused;
        // 15 bytes free in this slot
    }

    Config public config;

    // ✅ Events ที่ index ถูกต้อง
    event ConfigUpdated(uint128 indexed oldFee, uint128 indexed newFee, uint64 timestamp);
    event Paused(address indexed by, string reason);

    // ✅ Constructor validation
    constructor(address _owner) {
        if (_owner == address(0)) revert ZeroAddress("owner");
        owner = _owner;
        deployedAt = block.timestamp;
        config.lastUpdate = uint64(block.timestamp);
    }

    // ✅ Modifier ที่ใช้ custom error
    modifier onlyOwner() {
        if (msg.sender != owner) revert Unauthorized(msg.sender, owner);
        _;
    }

    // ✅ Input validation, CEI pattern, events
    function updateFee(uint128 newFee) external onlyOwner {
        // Checks
        if (newFee > MAX_FEE_BPS) revert InvalidAmount(newFee, "exceeds max fee");

        // Effects
        uint128 oldFee = config.feeBps;
        config.feeBps = newFee;
        config.lastUpdate = uint64(block.timestamp);

        // Interactions (none)

        emit ConfigUpdated(oldFee, newFee, uint64(block.timestamp));
    }
}
```

### 6.1 Slither Configuration

```yaml
# slither.config.json
{
    "detectors_to_exclude": [
        "timestamp"     // เราตั้งใจใช้ block.timestamp
    ],
    "filter_paths": "lib/",
    "exclude_informational": false,
    "exclude_low": false,
    "exclude_medium": false,
    "exclude_high": false
}
```

### 6.2 Foundry Test Standards

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test, console2} from "forge-std/Test.sol";
import {ReviewChecks} from "../src/ReviewChecks.sol";

/// @title ReviewChecksTest
/// @notice Comprehensive tests for ReviewChecks contract
contract ReviewChecksTest is Test {
    ReviewChecks checks;
    address owner = makeAddr("owner");
    address alice = makeAddr("alice");

    // ✅ Naming convention: test_functionName_condition_outcome
    function test_constructor_setsOwner() public {
        checks = new ReviewChecks(owner);
        assertEq(checks.owner(), owner);
    }

    function test_constructor_revertsOnZeroAddress() public {
        vm.expectRevert(abi.encodeWithSelector(ReviewChecks.ZeroAddress.selector, "owner"));
        new ReviewChecks(address(0));
    }

    function test_updateFee_succeeds() public {
        checks = new ReviewChecks(owner);
        
        vm.prank(owner);
        checks.updateFee(500); // 5%

        (uint128 feeBps,,) = checks.config();
        assertEq(feeBps, 500);
    }

    function test_updateFee_revertsIfNotOwner() public {
        checks = new ReviewChecks(owner);
        
        vm.prank(alice);
        vm.expectRevert(
            abi.encodeWithSelector(ReviewChecks.Unauthorized.selector, alice, owner)
        );
        checks.updateFee(500);
    }

    function test_updateFee_revertsIfExceedsMax() public {
        checks = new ReviewChecks(owner);
        
        vm.prank(owner);
        vm.expectRevert(
            abi.encodeWithSelector(ReviewChecks.InvalidAmount.selector, 1001, "exceeds max fee")
        );
        checks.updateFee(1001); // > MAX_FEE_BPS (1000)
    }

    // ✅ Fuzz testing
    function testFuzz_updateFee_alwaysValidUnderMax(uint128 fee) public {
        checks = new ReviewChecks(owner);
        vm.assume(fee <= checks.MAX_FEE_BPS());
        
        vm.prank(owner);
        checks.updateFee(fee);
        
        (uint128 feeBps,,) = checks.config();
        assertEq(feeBps, fee);
    }

    // ✅ Invariant: fee should never exceed MAX
    function invariant_feeNeverExceedsMax() public {
        (uint128 feeBps,,) = checks.config();
        assertLe(feeBps, checks.MAX_FEE_BPS());
    }
}
```

## สรุป Part 62

- **Security Checklist** ครอบคลุม reentrancy, access control, arithmetic, oracle, input validation
- **Before/After examples** แสดง 5 patterns ที่พบบ่อย: reentrancy, access control, gas, precision, events
- **GitHub Actions CI** ตรวจสอบ formatting, tests, coverage, static analysis, gas snapshot อัตโนมัติ
- **Pair Programming** แบบ Driver-Navigator และ Test-First ช่วยให้ค้นพบ bugs เร็วขึ้น
- **Common review comments**: custom errors, SafeERC20, named constants, immutables, zero-address checks
- **Test naming convention**: `test_functionName_condition_outcome` ทำให้อ่านง่าย
- **Fuzz tests** และ **invariant tests** ช่วยค้นพบ edge cases ที่ manual tests พลาด

## Next: Part 63 - Protocol Design Patterns
