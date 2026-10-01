# Part 63: Protocol Design Patterns

## บทนำ

การออกแบบ Protocol ที่ดีคือหัวใจของ DeFi ที่ยั่งยืน ไม่ใช่แค่การเขียน Smart Contract ให้ทำงานได้ แต่ต้องออกแบบให้รองรับการเติบโต การ Upgrade และสถานการณ์ฉุกเฉินได้อย่างปลอดภัย

ใน Part นี้เราจะเรียนรู้ Pattern ที่ Protocol ระดับ Production ใช้จริง เช่น Uniswap, Aave, Compound และ MakerDAO

---

## ทำไม Protocol Design ถึงสำคัญ?

ลองนึกภาพ Protocol ที่:
- มี Logic กับ Storage อยู่ในไฟล์เดียวกัน → ถ้าต้อง Upgrade ต้อง Migrate ข้อมูลทั้งหมด
- ไม่มีระบบ Access Control ที่ชัดเจน → ใครก็เรียก Admin Function ได้
- ไม่มี Emergency System → เมื่อเกิด Bug ไม่สามารถหยุด Protocol ได้ทัน

Protocol Design Patterns แก้ปัญหาเหล่านี้อย่างเป็นระบบ

---

## 1. Separation of Concerns: Logic vs Storage vs Access Control

### หลักการ

**Separation of Concerns** คือการแยก Contract ตามหน้าที่:
- **Storage Contract**: เก็บ State ทั้งหมด ไม่มี Logic
- **Logic Contract**: มี Business Logic ไม่เก็บ State
- **Access Control Contract**: กำหนดสิทธิ์การเข้าถึง

ประโยชน์:
1. **Upgradeability**: เปลี่ยน Logic ได้โดยไม่กระทบ Storage
2. **Modularity**: ทดสอบแต่ละส่วนแยกกัน
3. **Security**: Access Control อยู่ที่เดียว แก้ Bug ได้จุดเดียว

### โครงสร้าง Diamond Pattern (EIP-2535 แบบง่าย)

```
┌─────────────────┐     ┌─────────────────┐
│  Logic Contract │────▶│ Storage Contract│
│  (Stateless)    │     │  (Pure Storage) │
└─────────────────┘     └─────────────────┘
         │
         ▼
┌─────────────────┐
│ Access Control  │
│    Contract     │
└─────────────────┘
```

### Code: Storage Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title ProtocolStorage
/// @notice Pure storage contract - contains no logic
/// @dev All state variables live here; logic contracts read/write via interface
contract ProtocolStorage {
    // ─────────────────────────────────────────────────────────────
    // State Variables
    // ─────────────────────────────────────────────────────────────

    /// @notice Address of the logic contract allowed to modify state
    address public logicContract;

    /// @notice Address of the access control contract
    address public accessControl;

    /// @notice Protocol admin (set once at deploy)
    address public admin;

    // User balances: user => token => amount
    mapping(address => mapping(address => uint256)) public balances;

    // Total deposits per token
    mapping(address => uint256) public totalDeposits;

    // Protocol parameters
    uint256 public feeRate;         // basis points (e.g., 30 = 0.3%)
    uint256 public minDeposit;
    bool    public paused;

    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────

    event LogicContractUpdated(address indexed oldLogic, address indexed newLogic);
    event BalanceUpdated(address indexed user, address indexed token, uint256 newBalance);

    // ─────────────────────────────────────────────────────────────
    // Modifiers
    // ─────────────────────────────────────────────────────────────

    modifier onlyLogic() {
        require(msg.sender == logicContract, "Storage: caller is not logic contract");
        _;
    }

    modifier onlyAdmin() {
        require(msg.sender == admin, "Storage: caller is not admin");
        _;
    }

    // ─────────────────────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────────────────────

    constructor(address _admin) {
        admin = _admin;
        feeRate = 30;    // 0.3% default
        minDeposit = 1e15; // 0.001 ETH equivalent
    }

    // ─────────────────────────────────────────────────────────────
    // Admin Functions
    // ─────────────────────────────────────────────────────────────

    /// @notice Update the logic contract address
    function setLogicContract(address _logic) external onlyAdmin {
        require(_logic != address(0), "Storage: zero address");
        address old = logicContract;
        logicContract = _logic;
        emit LogicContractUpdated(old, _logic);
    }

    // ─────────────────────────────────────────────────────────────
    // Write Functions (only callable by logic contract)
    // ─────────────────────────────────────────────────────────────

    function setBalance(
        address user,
        address token,
        uint256 amount
    ) external onlyLogic {
        balances[user][token] = amount;
        emit BalanceUpdated(user, token, amount);
    }

    function addToBalance(
        address user,
        address token,
        uint256 amount
    ) external onlyLogic {
        balances[user][token] += amount;
        totalDeposits[token] += amount;
    }

    function subtractFromBalance(
        address user,
        address token,
        uint256 amount
    ) external onlyLogic {
        require(balances[user][token] >= amount, "Storage: insufficient balance");
        balances[user][token] -= amount;
        totalDeposits[token] -= amount;
    }

    function setFeeRate(uint256 _feeRate) external onlyLogic {
        require(_feeRate <= 1000, "Storage: fee too high"); // max 10%
        feeRate = _feeRate;
    }

    function setPaused(bool _paused) external onlyLogic {
        paused = _paused;
    }

    // ─────────────────────────────────────────────────────────────
    // View Functions (public - anyone can read)
    // ─────────────────────────────────────────────────────────────

    function getBalance(address user, address token) external view returns (uint256) {
        return balances[user][token];
    }

    function getTotalDeposit(address token) external view returns (uint256) {
        return totalDeposits[token];
    }
}
```

### Code: Access Control Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title AccessControlManager
/// @notice Centralized access control for the entire protocol
/// @dev Role-based with 3-tier hierarchy: ADMIN > OPERATOR > USER
contract AccessControlManager {
    // ─────────────────────────────────────────────────────────────
    // Constants
    // ─────────────────────────────────────────────────────────────

    bytes32 public constant ADMIN_ROLE    = keccak256("ADMIN_ROLE");
    bytes32 public constant OPERATOR_ROLE = keccak256("OPERATOR_ROLE");
    bytes32 public constant PAUSER_ROLE   = keccak256("PAUSER_ROLE");

    // ─────────────────────────────────────────────────────────────
    // State
    // ─────────────────────────────────────────────────────────────

    // role => account => hasRole
    mapping(bytes32 => mapping(address => bool)) private _roles;

    // role => admin role (who can grant this role)
    mapping(bytes32 => bytes32) private _roleAdmin;

    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────

    event RoleGranted(bytes32 indexed role, address indexed account, address indexed sender);
    event RoleRevoked(bytes32 indexed role, address indexed account, address indexed sender);

    // ─────────────────────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────────────────────

    constructor(address initialAdmin) {
        // Setup role hierarchy
        _roleAdmin[ADMIN_ROLE]    = ADMIN_ROLE;    // Admin manages itself
        _roleAdmin[OPERATOR_ROLE] = ADMIN_ROLE;    // Admin grants Operator
        _roleAdmin[PAUSER_ROLE]   = ADMIN_ROLE;    // Admin grants Pauser

        // Grant initial admin
        _roles[ADMIN_ROLE][initialAdmin] = true;
        emit RoleGranted(ADMIN_ROLE, initialAdmin, address(0));
    }

    // ─────────────────────────────────────────────────────────────
    // View Functions
    // ─────────────────────────────────────────────────────────────

    function hasRole(bytes32 role, address account) public view returns (bool) {
        return _roles[role][account];
    }

    function getRoleAdmin(bytes32 role) public view returns (bytes32) {
        return _roleAdmin[role];
    }

    // ─────────────────────────────────────────────────────────────
    // Write Functions
    // ─────────────────────────────────────────────────────────────

    function grantRole(bytes32 role, address account) external {
        bytes32 adminRole = _roleAdmin[role];
        require(_roles[adminRole][msg.sender], "AccessControl: sender lacks role admin");
        _roles[role][account] = true;
        emit RoleGranted(role, account, msg.sender);
    }

    function revokeRole(bytes32 role, address account) external {
        bytes32 adminRole = _roleAdmin[role];
        require(_roles[adminRole][msg.sender], "AccessControl: sender lacks role admin");
        _roles[role][account] = false;
        emit RoleRevoked(role, account, msg.sender);
    }

    function renounceRole(bytes32 role) external {
        _roles[role][msg.sender] = false;
        emit RoleRevoked(role, msg.sender, msg.sender);
    }

    // ─────────────────────────────────────────────────────────────
    // Modifier Helper
    // ─────────────────────────────────────────────────────────────

    /// @notice Revert if caller doesn't have the role
    function checkRole(bytes32 role, address account) external view {
        require(_roles[role][account], "AccessControl: account lacks role");
    }
}
```

### Code: Logic Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

interface IProtocolStorage {
    function addToBalance(address user, address token, uint256 amount) external;
    function subtractFromBalance(address user, address token, uint256 amount) external;
    function getBalance(address user, address token) external view returns (uint256);
    function feeRate() external view returns (uint256);
    function paused() external view returns (bool);
    function minDeposit() external view returns (uint256);
}

interface IAccessControlManager {
    function hasRole(bytes32 role, address account) external view returns (bool);
    function checkRole(bytes32 role, address account) external view;
}

/// @title ProtocolLogic
/// @notice Business logic for the protocol - stateless
/// @dev Reads/writes state through ProtocolStorage; checks permissions via AccessControlManager
contract ProtocolLogic {
    using SafeERC20 for IERC20;

    // ─────────────────────────────────────────────────────────────
    // Constants
    // ─────────────────────────────────────────────────────────────

    bytes32 public constant OPERATOR_ROLE = keccak256("OPERATOR_ROLE");
    uint256 public constant BASIS_POINTS = 10_000;

    // ─────────────────────────────────────────────────────────────
    // Immutables
    // ─────────────────────────────────────────────────────────────

    IProtocolStorage  public immutable storageContract;
    IAccessControlManager public immutable acm;

    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────

    event Deposited(address indexed user, address indexed token, uint256 amount, uint256 fee);
    event Withdrawn(address indexed user, address indexed token, uint256 amount);

    // ─────────────────────────────────────────────────────────────
    // Modifiers
    // ─────────────────────────────────────────────────────────────

    modifier notPaused() {
        require(!storageContract.paused(), "Logic: protocol is paused");
        _;
    }

    // ─────────────────────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────────────────────

    constructor(address _storage, address _acm) {
        storageContract = IProtocolStorage(_storage);
        acm = IAccessControlManager(_acm);
    }

    // ─────────────────────────────────────────────────────────────
    // Core Functions
    // ─────────────────────────────────────────────────────────────

    /// @notice Deposit tokens into the protocol
    /// @param token ERC20 token address
    /// @param amount Amount to deposit
    function deposit(address token, uint256 amount) external notPaused {
        require(amount >= storageContract.minDeposit(), "Logic: below minimum deposit");

        // Calculate fee
        uint256 feeRate = storageContract.feeRate();
        uint256 fee = (amount * feeRate) / BASIS_POINTS;
        uint256 netAmount = amount - fee;

        // Transfer tokens from user
        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);

        // Update storage (net amount after fee)
        storageContract.addToBalance(msg.sender, token, netAmount);

        emit Deposited(msg.sender, token, netAmount, fee);
    }

    /// @notice Withdraw tokens from the protocol
    /// @param token ERC20 token address
    /// @param amount Amount to withdraw
    function withdraw(address token, uint256 amount) external notPaused {
        // This will revert if insufficient balance (handled in storage)
        storageContract.subtractFromBalance(msg.sender, token, amount);

        // Transfer tokens to user
        IERC20(token).safeTransfer(msg.sender, amount);

        emit Withdrawn(msg.sender, token, amount);
    }

    /// @notice Get user's balance (view function - reads from storage)
    function balanceOf(address user, address token) external view returns (uint256) {
        return storageContract.getBalance(user, token);
    }
}
```

---

## 2. Versioned Protocol Design: V1 → V2 Migration

### ปัญหาของการ Upgrade Protocol

เมื่อ Protocol V1 มี User 10,000 คน และ TVL $50M:
- ไม่สามารถหยุด Protocol แล้วบอกให้ทุกคน Withdraw ได้
- User ต้องสามารถ Migrate ไป V2 ได้เอง ตามจังหวะที่สะดวก
- V1 ยังต้อง Read-Only ต่อไปหลัง Migration ปิด

### Pattern: Versioned Protocol with MigrationHelper

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

// ─────────────────────────────────────────────────────────────────────────────
// V1 Protocol (Legacy - still running)
// ─────────────────────────────────────────────────────────────────────────────

/// @title ProtocolV1
/// @notice Original protocol - frozen after V2 launch
/// @dev Migration withdrawals allowed even when deprecated
contract ProtocolV1 is Ownable {
    using SafeERC20 for IERC20;

    // ─────────────────────────────────────────────────────────────
    // State
    // ─────────────────────────────────────────────────────────────

    mapping(address => mapping(address => uint256)) public balances;

    address public migrationHelper;
    bool    public deprecated;

    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────

    event Deposited(address indexed user, address indexed token, uint256 amount);
    event Withdrawn(address indexed user, address indexed token, uint256 amount);
    event Deprecated(address indexed migrationHelper);

    // ─────────────────────────────────────────────────────────────
    // Modifiers
    // ─────────────────────────────────────────────────────────────

    modifier notDeprecated() {
        require(!deprecated, "V1: protocol deprecated - please migrate to V2");
        _;
    }

    modifier onlyMigrationHelper() {
        require(msg.sender == migrationHelper, "V1: caller is not migration helper");
        _;
    }

    // ─────────────────────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────────────────────

    constructor() Ownable(msg.sender) {}

    // ─────────────────────────────────────────────────────────────
    // Core Functions (blocked after deprecation)
    // ─────────────────────────────────────────────────────────────

    function deposit(address token, uint256 amount) external notDeprecated {
        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
        balances[msg.sender][token] += amount;
        emit Deposited(msg.sender, token, amount);
    }

    function withdraw(address token, uint256 amount) external {
        // Note: withdraw works even when deprecated (users can always get funds back)
        require(balances[msg.sender][token] >= amount, "V1: insufficient balance");
        balances[msg.sender][token] -= amount;
        IERC20(token).safeTransfer(msg.sender, amount);
        emit Withdrawn(msg.sender, token, amount);
    }

    // ─────────────────────────────────────────────────────────────
    // Migration Functions (callable only by MigrationHelper)
    // ─────────────────────────────────────────────────────────────

    /// @notice Withdraw on behalf of user during migration
    /// @dev Called by MigrationHelper - transfers directly to V2
    function migrateWithdraw(
        address user,
        address token,
        uint256 amount
    ) external onlyMigrationHelper returns (uint256 withdrawn) {
        uint256 bal = balances[user][token];
        withdrawn = amount == 0 ? bal : amount;  // 0 = migrate all

        require(bal >= withdrawn, "V1: insufficient balance for migration");
        balances[user][token] -= withdrawn;

        // Transfer to migration helper (which will deposit to V2)
        IERC20(token).safeTransfer(migrationHelper, withdrawn);

        emit Withdrawn(user, token, withdrawn);
    }

    // ─────────────────────────────────────────────────────────────
    // Admin Functions
    // ─────────────────────────────────────────────────────────────

    function setMigrationHelper(address _helper) external onlyOwner {
        require(_helper != address(0), "V1: zero address");
        migrationHelper = _helper;
    }

    function deprecate() external onlyOwner {
        require(migrationHelper != address(0), "V1: set migration helper first");
        deprecated = true;
        emit Deprecated(migrationHelper);
    }
}

// ─────────────────────────────────────────────────────────────────────────────
// V2 Protocol (New version with improved features)
// ─────────────────────────────────────────────────────────────────────────────

/// @title ProtocolV2
/// @notice Improved protocol with yield-bearing positions
contract ProtocolV2 is Ownable {
    using SafeERC20 for IERC20;

    // ─────────────────────────────────────────────────────────────
    // State
    // ─────────────────────────────────────────────────────────────

    mapping(address => mapping(address => uint256)) public balances;
    mapping(address => uint256) public yieldAccrued;

    // New V2 feature: track deposit timestamp for yield calculation
    mapping(address => mapping(address => uint256)) public depositTimestamp;

    address public migrationHelper;
    uint256 public constant YIELD_RATE_PER_YEAR = 500; // 5% APY in basis points

    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────

    event Deposited(address indexed user, address indexed token, uint256 amount, bool isMigration);
    event Withdrawn(address indexed user, address indexed token, uint256 amount);

    // ─────────────────────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────────────────────

    constructor() Ownable(msg.sender) {}

    // ─────────────────────────────────────────────────────────────
    // Core Functions
    // ─────────────────────────────────────────────────────────────

    function deposit(address token, uint256 amount) external {
        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
        _creditDeposit(msg.sender, token, amount, false);
    }

    function withdraw(address token, uint256 amount) external {
        require(balances[msg.sender][token] >= amount, "V2: insufficient balance");
        balances[msg.sender][token] -= amount;
        IERC20(token).safeTransfer(msg.sender, amount);
        emit Withdrawn(msg.sender, token, amount);
    }

    /// @notice Receive migrated funds from MigrationHelper
    function receiveMigration(
        address user,
        address token,
        uint256 amount
    ) external {
        require(msg.sender == migrationHelper, "V2: caller is not migration helper");
        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
        _creditDeposit(user, token, amount, true);
    }

    // ─────────────────────────────────────────────────────────────
    // Internal
    // ─────────────────────────────────────────────────────────────

    function _creditDeposit(
        address user,
        address token,
        uint256 amount,
        bool isMigration
    ) internal {
        balances[user][token] += amount;
        depositTimestamp[user][token] = block.timestamp;
        emit Deposited(user, token, amount, isMigration);
    }

    // ─────────────────────────────────────────────────────────────
    // Admin
    // ─────────────────────────────────────────────────────────────

    function setMigrationHelper(address _helper) external onlyOwner {
        migrationHelper = _helper;
    }
}

// ─────────────────────────────────────────────────────────────────────────────
// Migration Helper
// ─────────────────────────────────────────────────────────────────────────────

/// @title MigrationHelper
/// @notice Orchestrates user migration from V1 to V2 atomically
/// @dev Pulls from V1, approves V2, deposits - all in one transaction
contract MigrationHelper {
    using SafeERC20 for IERC20;

    // ─────────────────────────────────────────────────────────────
    // State
    // ─────────────────────────────────────────────────────────────

    ProtocolV1 public immutable v1;
    ProtocolV2 public immutable v2;

    address public immutable owner;
    bool    public migrationOpen;

    // Track migration status
    mapping(address => mapping(address => bool)) public hasMigrated;

    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────

    event MigrationCompleted(
        address indexed user,
        address indexed token,
        uint256 amount
    );

    // ─────────────────────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────────────────────

    constructor(address _v1, address _v2) {
        v1 = ProtocolV1(_v1);
        v2 = ProtocolV2(_v2);
        owner = msg.sender;
        migrationOpen = true;
    }

    // ─────────────────────────────────────────────────────────────
    // Migration Functions
    // ─────────────────────────────────────────────────────────────

    /// @notice Migrate user's entire balance of a token from V1 to V2
    /// @param token The token to migrate
    function migrate(address token) external {
        require(migrationOpen, "Migration: migration period closed");
        _migrateAmount(msg.sender, token, 0); // 0 = all
    }

    /// @notice Migrate a specific amount from V1 to V2
    /// @param token The token to migrate
    /// @param amount Amount to migrate (0 = migrate all)
    function migrateAmount(address token, uint256 amount) external {
        require(migrationOpen, "Migration: migration period closed");
        _migrateAmount(msg.sender, token, amount);
    }

    /// @notice Deposit to V1 and immediately migrate to V2 (for new users during transition)
    /// @dev Useful during the migration period when users want to enter directly to V2
    function depositAndMigrate(address token, uint256 amount) external {
        require(migrationOpen, "Migration: migration period closed");

        // First approve and deposit to V1
        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
        IERC20(token).forceApprove(address(v1), amount);

        // Deposit to V1 on behalf of user (need V1 to support this pattern)
        // In practice, user would deposit directly - this shows the concept
        // Then immediately migrate
        _migrateAmount(msg.sender, token, amount);
    }

    // ─────────────────────────────────────────────────────────────
    // Internal
    // ─────────────────────────────────────────────────────────────

    function _migrateAmount(address user, address token, uint256 amount) internal {
        // Check V1 balance
        uint256 v1Balance = v1.balances(user, token);
        require(v1Balance > 0, "Migration: no V1 balance to migrate");

        // Pull funds from V1 (goes to this contract)
        uint256 received = v1.migrateWithdraw(user, token, amount);

        // Approve V2 to take tokens
        IERC20(token).forceApprove(address(v2), received);

        // Deposit to V2 on behalf of user
        v2.receiveMigration(user, token, received);

        hasMigrated[user][token] = true;

        emit MigrationCompleted(user, token, received);
    }

    // ─────────────────────────────────────────────────────────────
    // Admin
    // ─────────────────────────────────────────────────────────────

    function closeMigration() external {
        require(msg.sender == owner, "Migration: not owner");
        migrationOpen = false;
    }
}
```

---

## 3. Fee Routing Architecture: FeeCollector

### ทำไมต้องมี Fee Routing ที่ซับซ้อน?

Protocol ส่วนใหญ่ต้องแบ่ง Fee หลายทาง:
- **Protocol Treasury**: เพื่อการพัฒนา Protocol
- **Stakers**: สิ่งตอบแทนสำหรับ Token Staker
- **Insurance Fund**: กองทุนชดเชยกรณีเกิด Exploit

การ Hard-code เปอร์เซ็นต์เสี่ยงต่อ Upgrade Complexity จึงใช้ Configurable Basis Points

### Code: FeeCollector

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title FeeCollector
/// @notice Collects protocol fees and routes them to designated recipients
/// @custom:invariant sum of all recipient shares == BASIS_POINTS
contract FeeCollector is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ─────────────────────────────────────────────────────────────
    // Types
    // ─────────────────────────────────────────────────────────────

    struct FeeRecipient {
        address account;    // Recipient address
        uint256 bps;        // Basis points share (out of 10000)
        string  label;      // Human-readable label (e.g., "Treasury")
        bool    active;     // Can be deactivated without removing
    }

    // ─────────────────────────────────────────────────────────────
    // Constants
    // ─────────────────────────────────────────────────────────────

    uint256 public constant BASIS_POINTS = 10_000;
    uint256 public constant MAX_RECIPIENTS = 10;

    // ─────────────────────────────────────────────────────────────
    // State
    // ─────────────────────────────────────────────────────────────

    FeeRecipient[] public recipients;

    // Accumulated fees per recipient per token (push vs pull pattern)
    mapping(uint256 => mapping(address => uint256)) public pendingFees;
    // recipientIndex => token => amount

    // Total fees collected per token
    mapping(address => uint256) public totalFeesCollected;

    // Approved callers that can route fees (e.g., Logic contracts)
    mapping(address => bool) public approvedRouters;

    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────

    event FeesReceived(address indexed token, uint256 amount, address indexed from);
    event FeesDistributed(address indexed token, uint256 totalAmount);
    event FeesClaimed(uint256 indexed recipientIndex, address indexed token, uint256 amount);
    event RecipientAdded(uint256 indexed index, address account, uint256 bps, string label);
    event RecipientUpdated(uint256 indexed index, address account, uint256 bps);
    event RouterApproved(address indexed router, bool approved);

    // ─────────────────────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────────────────────

    constructor(
        address _protocol,   // e.g., 5000 bps = 50% to protocol
        address _treasury,   // e.g., 3000 bps = 30% to treasury
        address _stakers     // e.g., 2000 bps = 20% to stakers
    ) Ownable(msg.sender) {
        // Initialize with 3 default recipients (total = 10000)
        recipients.push(FeeRecipient({
            account: _protocol,
            bps: 5000,
            label: "Protocol",
            active: true
        }));
        recipients.push(FeeRecipient({
            account: _treasury,
            bps: 3000,
            label: "Treasury",
            active: true
        }));
        recipients.push(FeeRecipient({
            account: _stakers,
            bps: 2000,
            label: "Stakers",
            active: true
        }));

        emit RecipientAdded(0, _protocol, 5000, "Protocol");
        emit RecipientAdded(1, _treasury, 3000, "Treasury");
        emit RecipientAdded(2, _stakers, 2000, "Stakers");
    }

    // ─────────────────────────────────────────────────────────────
    // Modifiers
    // ─────────────────────────────────────────────────────────────

    modifier onlyRouter() {
        require(approvedRouters[msg.sender], "FeeCollector: caller is not approved router");
        _;
    }

    // ─────────────────────────────────────────────────────────────
    // Router Functions
    // ─────────────────────────────────────────────────────────────

    /// @notice Receive fee tokens and queue them for distribution
    /// @param token ERC20 token address
    /// @param amount Total fee amount
    function receiveFees(address token, uint256 amount) external onlyRouter nonReentrant {
        require(amount > 0, "FeeCollector: zero amount");

        // Pull tokens from caller
        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);

        // Distribute to recipients based on BPS
        _splitFees(token, amount);

        totalFeesCollected[token] += amount;
        emit FeesReceived(token, amount, msg.sender);
    }

    // ─────────────────────────────────────────────────────────────
    // Claim Functions (pull pattern - recipients claim their own)
    // ─────────────────────────────────────────────────────────────

    /// @notice Recipient claims their pending fees for a token
    /// @param recipientIndex Index in the recipients array
    /// @param token Token to claim
    function claimFees(uint256 recipientIndex, address token) external nonReentrant {
        require(recipientIndex < recipients.length, "FeeCollector: invalid index");
        FeeRecipient memory r = recipients[recipientIndex];
        require(r.account == msg.sender, "FeeCollector: caller is not recipient");

        uint256 amount = pendingFees[recipientIndex][token];
        require(amount > 0, "FeeCollector: nothing to claim");

        pendingFees[recipientIndex][token] = 0;
        IERC20(token).safeTransfer(r.account, amount);

        emit FeesClaimed(recipientIndex, token, amount);
    }

    /// @notice Batch claim across multiple tokens
    function claimMultiple(uint256 recipientIndex, address[] calldata tokens) external nonReentrant {
        require(recipientIndex < recipients.length, "FeeCollector: invalid index");
        FeeRecipient memory r = recipients[recipientIndex];
        require(r.account == msg.sender, "FeeCollector: caller is not recipient");

        for (uint256 i = 0; i < tokens.length; i++) {
            uint256 amount = pendingFees[recipientIndex][tokens[i]];
            if (amount > 0) {
                pendingFees[recipientIndex][tokens[i]] = 0;
                IERC20(tokens[i]).safeTransfer(r.account, amount);
                emit FeesClaimed(recipientIndex, tokens[i], amount);
            }
        }
    }

    // ─────────────────────────────────────────────────────────────
    // Admin Functions
    // ─────────────────────────────────────────────────────────────

    /// @notice Update recipient shares (must sum to BASIS_POINTS)
    /// @dev All active recipients' BPS must be provided
    function updateShares(
        uint256[] calldata indices,
        uint256[] calldata newBps
    ) external onlyOwner {
        require(indices.length == newBps.length, "FeeCollector: length mismatch");

        uint256 total = 0;
        for (uint256 i = 0; i < newBps.length; i++) {
            total += newBps[i];
        }
        require(total == BASIS_POINTS, "FeeCollector: shares must sum to 10000");

        for (uint256 i = 0; i < indices.length; i++) {
            require(indices[i] < recipients.length, "FeeCollector: invalid index");
            recipients[indices[i]].bps = newBps[i];
            emit RecipientUpdated(indices[i], recipients[indices[i]].account, newBps[i]);
        }
    }

    /// @notice Update recipient address (e.g., treasury moves to multisig)
    function updateRecipientAddress(uint256 index, address newAccount) external onlyOwner {
        require(index < recipients.length, "FeeCollector: invalid index");
        require(newAccount != address(0), "FeeCollector: zero address");
        recipients[index].account = newAccount;
        emit RecipientUpdated(index, newAccount, recipients[index].bps);
    }

    /// @notice Approve or remove a fee router
    function setRouter(address router, bool approved) external onlyOwner {
        approvedRouters[router] = approved;
        emit RouterApproved(router, approved);
    }

    // ─────────────────────────────────────────────────────────────
    // Internal
    // ─────────────────────────────────────────────────────────────

    function _splitFees(address token, uint256 amount) internal {
        uint256 distributed = 0;

        for (uint256 i = 0; i < recipients.length; i++) {
            if (!recipients[i].active) continue;

            uint256 share;
            if (i == recipients.length - 1) {
                // Last recipient gets remainder (avoid dust from rounding)
                share = amount - distributed;
            } else {
                share = (amount * recipients[i].bps) / BASIS_POINTS;
            }

            pendingFees[i][token] += share;
            distributed += share;
        }

        emit FeesDistributed(token, amount);
    }

    // ─────────────────────────────────────────────────────────────
    // View Functions
    // ─────────────────────────────────────────────────────────────

    function getRecipientCount() external view returns (uint256) {
        return recipients.length;
    }

    function getPendingFees(
        uint256 recipientIndex,
        address token
    ) external view returns (uint256) {
        return pendingFees[recipientIndex][token];
    }

    function validateShares() external view returns (bool valid, uint256 total) {
        total = 0;
        for (uint256 i = 0; i < recipients.length; i++) {
            if (recipients[i].active) {
                total += recipients[i].bps;
            }
        }
        valid = (total == BASIS_POINTS);
    }
}
```

---

## 4. ParameterStore: Governance-Controlled Parameters with Timelock

### ทำไมต้องมี ParameterStore?

Protocol Parameters เช่น Fee Rate, Max Deposit, Liquidation Threshold มักต้องการ:
1. **Governance Control**: เปลี่ยนได้ผ่าน Governance Vote
2. **Timelock**: รอ 48 ชม. ก่อน Apply เพื่อให้ Users ตรวจสอบ
3. **Bounds Validation**: ป้องกัน Admin Set ค่าที่อันตราย (เช่น Fee = 100%)

### Code: ParameterStore

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/Ownable.sol";

/// @title ParameterStore
/// @notice Governance-controlled parameters with timelock and bounds validation
/// @custom:invariant Every queued change must wait >= MIN_DELAY before execution
/// @custom:invariant Every applied parameter must satisfy: minBound[key] <= value <= maxBound[key]
contract ParameterStore is Ownable {

    // ─────────────────────────────────────────────────────────────
    // Types
    // ─────────────────────────────────────────────────────────────

    struct Parameter {
        uint256 value;          // Current live value
        uint256 minBound;       // Absolute minimum allowed
        uint256 maxBound;       // Absolute maximum allowed
        string  description;    // Human-readable description
        bool    initialized;
    }

    struct PendingChange {
        bytes32 key;            // Parameter key
        uint256 newValue;       // Proposed value
        uint256 executeAfter;   // Timestamp when it can be applied
        address proposer;       // Who queued this
        bool    executed;
        bool    cancelled;
    }

    // ─────────────────────────────────────────────────────────────
    // Constants
    // ─────────────────────────────────────────────────────────────

    uint256 public constant MIN_DELAY = 48 hours;
    uint256 public constant MAX_DELAY = 30 days;

    // ─────────────────────────────────────────────────────────────
    // State
    // ─────────────────────────────────────────────────────────────

    mapping(bytes32 => Parameter)     public parameters;
    mapping(bytes32 => PendingChange) public pendingChanges;
    // changeId => PendingChange (changeId = keccak256(key, newValue, timestamp))

    mapping(address => bool) public governors; // Can propose changes
    uint256 public timelockDelay;

    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────

    event ParameterInitialized(bytes32 indexed key, uint256 value, uint256 min, uint256 max);
    event ChangeQueued(bytes32 indexed changeId, bytes32 indexed key, uint256 newValue, uint256 executeAfter);
    event ChangeExecuted(bytes32 indexed changeId, bytes32 indexed key, uint256 oldValue, uint256 newValue);
    event ChangeCancelled(bytes32 indexed changeId, bytes32 indexed key);
    event GovernorUpdated(address indexed governor, bool status);

    // ─────────────────────────────────────────────────────────────
    // Modifiers
    // ─────────────────────────────────────────────────────────────

    modifier onlyGovernor() {
        require(governors[msg.sender] || msg.sender == owner(), "ParamStore: not governor");
        _;
    }

    // ─────────────────────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────────────────────

    constructor(uint256 _timelockDelay) Ownable(msg.sender) {
        require(_timelockDelay >= MIN_DELAY, "ParamStore: delay too short");
        require(_timelockDelay <= MAX_DELAY, "ParamStore: delay too long");
        timelockDelay = _timelockDelay;
    }

    // ─────────────────────────────────────────────────────────────
    // Initialization
    // ─────────────────────────────────────────────────────────────

    /// @notice Register a new parameter
    function initParameter(
        bytes32 key,
        uint256 initialValue,
        uint256 minBound,
        uint256 maxBound,
        string calldata description
    ) external onlyOwner {
        require(!parameters[key].initialized, "ParamStore: already initialized");
        require(minBound <= maxBound, "ParamStore: invalid bounds");
        require(initialValue >= minBound && initialValue <= maxBound, "ParamStore: initial value out of bounds");

        parameters[key] = Parameter({
            value: initialValue,
            minBound: minBound,
            maxBound: maxBound,
            description: description,
            initialized: true
        });

        emit ParameterInitialized(key, initialValue, minBound, maxBound);
    }

    // ─────────────────────────────────────────────────────────────
    // Change Lifecycle
    // ─────────────────────────────────────────────────────────────

    /// @notice Queue a parameter change (starts timelock)
    /// @param key Parameter identifier
    /// @param newValue Proposed new value
    /// @return changeId Unique identifier for this change
    function queueChange(
        bytes32 key,
        uint256 newValue
    ) external onlyGovernor returns (bytes32 changeId) {
        Parameter memory p = parameters[key];
        require(p.initialized, "ParamStore: parameter not initialized");
        require(newValue >= p.minBound && newValue <= p.maxBound, "ParamStore: value out of bounds");
        require(newValue != p.value, "ParamStore: same value");

        uint256 executeAfter = block.timestamp + timelockDelay;

        changeId = keccak256(abi.encodePacked(key, newValue, block.timestamp, msg.sender));

        pendingChanges[changeId] = PendingChange({
            key: key,
            newValue: newValue,
            executeAfter: executeAfter,
            proposer: msg.sender,
            executed: false,
            cancelled: false
        });

        emit ChangeQueued(changeId, key, newValue, executeAfter);
    }

    /// @notice Execute a queued change after timelock
    /// @param changeId The change to execute
    function executeChange(bytes32 changeId) external onlyGovernor {
        PendingChange storage change = pendingChanges[changeId];

        require(change.executeAfter != 0, "ParamStore: change not found");
        require(!change.executed, "ParamStore: already executed");
        require(!change.cancelled, "ParamStore: change cancelled");
        require(block.timestamp >= change.executeAfter, "ParamStore: timelock not expired");

        Parameter storage p = parameters[change.key];

        // Re-validate bounds (in case bounds changed)
        require(change.newValue >= p.minBound && change.newValue <= p.maxBound, "ParamStore: value out of bounds");

        uint256 oldValue = p.value;
        p.value = change.newValue;
        change.executed = true;

        emit ChangeExecuted(changeId, change.key, oldValue, change.newValue);
    }

    /// @notice Cancel a queued change
    function cancelChange(bytes32 changeId) external onlyOwner {
        PendingChange storage change = pendingChanges[changeId];
        require(change.executeAfter != 0, "ParamStore: change not found");
        require(!change.executed, "ParamStore: already executed");
        require(!change.cancelled, "ParamStore: already cancelled");

        change.cancelled = true;

        emit ChangeCancelled(changeId, change.key);
    }

    // ─────────────────────────────────────────────────────────────
    // View Functions
    // ─────────────────────────────────────────────────────────────

    /// @notice Get current value of a parameter
    function get(bytes32 key) external view returns (uint256) {
        require(parameters[key].initialized, "ParamStore: not initialized");
        return parameters[key].value;
    }

    /// @notice Get parameter with full metadata
    function getParameter(bytes32 key) external view returns (Parameter memory) {
        return parameters[key];
    }

    /// @notice Check if a change can be executed now
    function canExecute(bytes32 changeId) external view returns (bool) {
        PendingChange memory c = pendingChanges[changeId];
        return c.executeAfter != 0
            && !c.executed
            && !c.cancelled
            && block.timestamp >= c.executeAfter;
    }

    // ─────────────────────────────────────────────────────────────
    // Admin
    // ─────────────────────────────────────────────────────────────

    function setGovernor(address governor, bool status) external onlyOwner {
        governors[governor] = status;
        emit GovernorUpdated(governor, status);
    }

    function updateBounds(
        bytes32 key,
        uint256 newMin,
        uint256 newMax
    ) external onlyOwner {
        Parameter storage p = parameters[key];
        require(p.initialized, "ParamStore: not initialized");
        require(newMin <= newMax, "ParamStore: invalid bounds");
        p.minBound = newMin;
        p.maxBound = newMax;
    }
}
```

---

## 5. Emergency System Design

### โครงสร้าง Emergency Levels

ระบบ Emergency ต้องมี Granularity:
- **Level 0**: ปกติ ทุกอย่างทำงาน
- **Level 1**: หยุดเฉพาะ Deposits ใหม่ (แต่ Withdraw ยังได้)
- **Level 2**: หยุด Withdrawals ด้วย (เพื่อป้องกัน Bank Run)
- **Level 3**: หยุดทุกอย่าง (Full Pause) ใช้เฉพาะกรณีวิกฤต

### Code: EmergencySystem

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/Ownable.sol";

/// @title EmergencySystem
/// @notice Multi-level emergency pause system for protocol safety
/// @dev Level 0=normal, 1=no deposits, 2=no withdrawals, 3=full pause
contract EmergencySystem is Ownable {

    // ─────────────────────────────────────────────────────────────
    // Types
    // ─────────────────────────────────────────────────────────────

    enum PauseLevel {
        NONE,        // 0: Normal operation
        DEPOSITS,    // 1: Deposits paused, withdrawals allowed
        WITHDRAWALS, // 2: Both deposits AND withdrawals paused
        FULL         // 3: Everything paused (emergency)
    }

    // ─────────────────────────────────────────────────────────────
    // State
    // ─────────────────────────────────────────────────────────────

    PauseLevel public pauseLevel;

    // EmergencyCouncil members (can trigger L1 pause)
    mapping(address => bool) public council;
    uint256 public councilCount;

    // Multisig requirement for high-level pauses
    uint256 public constant COUNCIL_THRESHOLD = 2; // Need 2 of N council members

    // Pending L2+ pause approvals
    struct PauseProposal {
        PauseLevel targetLevel;
        uint256    approvals;
        uint256    expiresAt;
        bool       executed;
        mapping(address => bool) hasApproved;
    }

    mapping(bytes32 => PauseProposal) public pauseProposals;

    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────

    event PauseLevelChanged(PauseLevel indexed oldLevel, PauseLevel indexed newLevel, address indexed by);
    event EmergencyProposed(bytes32 indexed proposalId, PauseLevel targetLevel, address indexed proposer);
    event EmergencyApproved(bytes32 indexed proposalId, address indexed approver, uint256 totalApprovals);
    event EmergencyExecuted(bytes32 indexed proposalId, PauseLevel newLevel);
    event CouncilMemberAdded(address indexed member);
    event CouncilMemberRemoved(address indexed member);

    // ─────────────────────────────────────────────────────────────
    // Modifiers
    // ─────────────────────────────────────────────────────────────

    modifier onlyCouncil() {
        require(council[msg.sender], "Emergency: caller is not council member");
        _;
    }

    modifier whenNotFullPaused() {
        require(pauseLevel != PauseLevel.FULL, "Emergency: full pause active");
        _;
    }

    // ─────────────────────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────────────────────

    constructor(address[] memory initialCouncil) Ownable(msg.sender) {
        require(initialCouncil.length >= COUNCIL_THRESHOLD, "Emergency: insufficient council members");

        for (uint256 i = 0; i < initialCouncil.length; i++) {
            _addCouncilMember(initialCouncil[i]);
        }
    }

    // ─────────────────────────────────────────────────────────────
    // Level 1 Pause: Single council member can trigger
    // ─────────────────────────────────────────────────────────────

    /// @notice Pause deposits immediately (single council member)
    function pauseDeposits() external onlyCouncil {
        PauseLevel old = pauseLevel;
        if (uint8(pauseLevel) < uint8(PauseLevel.DEPOSITS)) {
            pauseLevel = PauseLevel.DEPOSITS;
            emit PauseLevelChanged(old, PauseLevel.DEPOSITS, msg.sender);
        }
    }

    /// @notice Resume deposits (owner only)
    function resumeDeposits() external onlyOwner {
        require(pauseLevel == PauseLevel.DEPOSITS, "Emergency: wrong pause level");
        pauseLevel = PauseLevel.NONE;
        emit PauseLevelChanged(PauseLevel.DEPOSITS, PauseLevel.NONE, msg.sender);
    }

    // ─────────────────────────────────────────────────────────────
    // Level 2 & 3 Pause: Requires council multisig
    // ─────────────────────────────────────────────────────────────

    /// @notice Propose a higher-level pause (L2 or L3)
    function proposePause(PauseLevel targetLevel) external onlyCouncil returns (bytes32 proposalId) {
        require(
            targetLevel == PauseLevel.WITHDRAWALS || targetLevel == PauseLevel.FULL,
            "Emergency: use pauseDeposits() for L1"
        );
        require(uint8(targetLevel) > uint8(pauseLevel), "Emergency: target level not higher");

        proposalId = keccak256(abi.encodePacked(targetLevel, block.timestamp, msg.sender));

        PauseProposal storage proposal = pauseProposals[proposalId];
        proposal.targetLevel = targetLevel;
        proposal.expiresAt = block.timestamp + 1 hours; // Proposal expires in 1 hour
        proposal.approvals = 1;
        proposal.hasApproved[msg.sender] = true;

        emit EmergencyProposed(proposalId, targetLevel, msg.sender);

        // Auto-execute if threshold already met (e.g., 1-of-1 council)
        if (proposal.approvals >= COUNCIL_THRESHOLD) {
            _executePause(proposalId);
        }
    }

    /// @notice Approve a pending pause proposal
    function approvePause(bytes32 proposalId) external onlyCouncil {
        PauseProposal storage proposal = pauseProposals[proposalId];

        require(proposal.expiresAt != 0, "Emergency: proposal not found");
        require(!proposal.executed, "Emergency: already executed");
        require(block.timestamp <= proposal.expiresAt, "Emergency: proposal expired");
        require(!proposal.hasApproved[msg.sender], "Emergency: already approved");

        proposal.hasApproved[msg.sender] = true;
        proposal.approvals++;

        emit EmergencyApproved(proposalId, msg.sender, proposal.approvals);

        if (proposal.approvals >= COUNCIL_THRESHOLD) {
            _executePause(proposalId);
        }
    }

    /// @notice Full unpause (owner only, after incident resolved)
    function unpause() external onlyOwner {
        PauseLevel old = pauseLevel;
        pauseLevel = PauseLevel.NONE;
        emit PauseLevelChanged(old, PauseLevel.NONE, msg.sender);
    }

    // ─────────────────────────────────────────────────────────────
    // Check Functions (used by other contracts)
    // ─────────────────────────────────────────────────────────────

    /// @notice Revert if deposits are paused
    function checkDepositsAllowed() external view {
        require(
            pauseLevel == PauseLevel.NONE,
            "Emergency: deposits are paused"
        );
    }

    /// @notice Revert if withdrawals are paused
    function checkWithdrawalsAllowed() external view {
        require(
            pauseLevel != PauseLevel.WITHDRAWALS && pauseLevel != PauseLevel.FULL,
            "Emergency: withdrawals are paused"
        );
    }

    /// @notice Revert if fully paused
    function checkNotFullPause() external view {
        require(pauseLevel != PauseLevel.FULL, "Emergency: protocol fully paused");
    }

    // ─────────────────────────────────────────────────────────────
    // Internal
    // ─────────────────────────────────────────────────────────────

    function _executePause(bytes32 proposalId) internal {
        PauseProposal storage proposal = pauseProposals[proposalId];
        proposal.executed = true;

        PauseLevel old = pauseLevel;
        pauseLevel = proposal.targetLevel;

        emit EmergencyExecuted(proposalId, proposal.targetLevel);
        emit PauseLevelChanged(old, proposal.targetLevel, msg.sender);
    }

    function _addCouncilMember(address member) internal {
        require(member != address(0), "Emergency: zero address");
        require(!council[member], "Emergency: already council member");
        council[member] = true;
        councilCount++;
        emit CouncilMemberAdded(member);
    }

    // ─────────────────────────────────────────────────────────────
    // Admin
    // ─────────────────────────────────────────────────────────────

    function addCouncilMember(address member) external onlyOwner {
        _addCouncilMember(member);
    }

    function removeCouncilMember(address member) external onlyOwner {
        require(council[member], "Emergency: not a council member");
        require(councilCount > COUNCIL_THRESHOLD, "Emergency: would go below threshold");
        council[member] = false;
        councilCount--;
        emit CouncilMemberRemoved(member);
    }
}
```

---

## Workshop: สร้าง Mini-Protocol ที่ใช้ Pattern ทั้งหมด

### โจทย์

สร้าง `IntegratedProtocol` ที่รวม:
1. ProtocolStorage (แยก state)
2. FeeCollector (แบ่ง fees)
3. ParameterStore (ควบคุม parameters)
4. EmergencySystem (หยุดกรณีฉุกเฉิน)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title IntegratedProtocol
/// @notice Workshop: Demonstrates all 5 Protocol Design Patterns working together
contract IntegratedProtocol is ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ─────────────────────────────────────────────────────────────
    // Contract References (injected for separation of concerns)
    // ─────────────────────────────────────────────────────────────

    address public immutable storageAddr;
    address public immutable feeCollectorAddr;
    address public immutable paramStoreAddr;
    address public immutable emergencyAddr;

    // ─────────────────────────────────────────────────────────────
    // State (minimal - most in storage contract)
    // ─────────────────────────────────────────────────────────────

    // Parameter keys
    bytes32 public constant PARAM_FEE_RATE    = keccak256("FEE_RATE");
    bytes32 public constant PARAM_MIN_DEPOSIT = keccak256("MIN_DEPOSIT");
    bytes32 public constant PARAM_MAX_DEPOSIT = keccak256("MAX_DEPOSIT");

    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────

    event Deposited(address indexed user, address indexed token, uint256 net, uint256 fee);
    event Withdrawn(address indexed user, address indexed token, uint256 amount);

    // ─────────────────────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────────────────────

    constructor(
        address _storage,
        address _feeCollector,
        address _paramStore,
        address _emergency
    ) {
        storageAddr     = _storage;
        feeCollectorAddr = _feeCollector;
        paramStoreAddr  = _paramStore;
        emergencyAddr   = _emergency;
    }

    // ─────────────────────────────────────────────────────────────
    // Core Functions
    // ─────────────────────────────────────────────────────────────

    function deposit(address token, uint256 amount) external nonReentrant {
        // 1. Check emergency state
        IEmergency(emergencyAddr).checkDepositsAllowed();

        // 2. Read parameters from ParameterStore
        uint256 minDeposit = IParamStore(paramStoreAddr).get(PARAM_MIN_DEPOSIT);
        uint256 maxDeposit = IParamStore(paramStoreAddr).get(PARAM_MAX_DEPOSIT);
        uint256 feeRate    = IParamStore(paramStoreAddr).get(PARAM_FEE_RATE);

        require(amount >= minDeposit, "Protocol: below minimum");
        require(amount <= maxDeposit, "Protocol: above maximum");

        // 3. Pull tokens
        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);

        // 4. Calculate and route fees
        uint256 fee = (amount * feeRate) / 10_000;
        uint256 net = amount - fee;

        if (fee > 0) {
            IERC20(token).forceApprove(feeCollectorAddr, fee);
            IFeeCollector(feeCollectorAddr).receiveFees(token, fee);
        }

        // 5. Update storage
        IStorage(storageAddr).addToBalance(msg.sender, token, net);

        emit Deposited(msg.sender, token, net, fee);
    }

    function withdraw(address token, uint256 amount) external nonReentrant {
        // 1. Check emergency state
        IEmergency(emergencyAddr).checkWithdrawalsAllowed();

        // 2. Update storage (reverts if insufficient)
        IStorage(storageAddr).subtractFromBalance(msg.sender, token, amount);

        // 3. Transfer to user
        IERC20(token).safeTransfer(msg.sender, amount);

        emit Withdrawn(msg.sender, token, amount);
    }
}

// Minimal interfaces for IntegratedProtocol
interface IStorage {
    function addToBalance(address user, address token, uint256 amount) external;
    function subtractFromBalance(address user, address token, uint256 amount) external;
}

interface IFeeCollector {
    function receiveFees(address token, uint256 amount) external;
}

interface IParamStore {
    function get(bytes32 key) external view returns (uint256);
}

interface IEmergency {
    function checkDepositsAllowed() external view;
    function checkWithdrawalsAllowed() external view;
}
```

### Deployment Script

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @notice Deployment script showing proper initialization order
contract DeployScript {
    function run() external {
        address deployer = msg.sender;
        address treasury = address(0x...); // Replace with real address
        address stakers  = address(0x...); // Replace with real address
        address[] memory council = new address[](3);
        council[0] = address(0x...); // Replace with real addresses
        council[1] = address(0x...);
        council[2] = address(0x...);

        // Step 1: Deploy independent contracts
        ProtocolStorage storage_ = new ProtocolStorage(deployer);
        AccessControlManager acm = new AccessControlManager(deployer);
        EmergencySystem emergency = new EmergencySystem(council);

        // Step 2: Deploy FeeCollector with recipient addresses
        FeeCollector feeCollector = new FeeCollector(deployer, treasury, stakers);

        // Step 3: Deploy ParameterStore with timelock
        ParameterStore paramStore = new ParameterStore(48 hours);

        // Step 4: Initialize parameters
        paramStore.initParameter(keccak256("FEE_RATE"), 30, 0, 500, "Fee rate in bps");
        paramStore.initParameter(keccak256("MIN_DEPOSIT"), 1e15, 1e12, 1e18, "Min deposit amount");
        paramStore.initParameter(keccak256("MAX_DEPOSIT"), 1e24, 1e18, 1e27, "Max deposit amount");

        // Step 5: Deploy Logic contract with all dependencies
        IntegratedProtocol protocol = new IntegratedProtocol(
            address(storage_),
            address(feeCollector),
            address(paramStore),
            address(emergency)
        );

        // Step 6: Wire permissions
        storage_.setLogicContract(address(protocol));
        feeCollector.setRouter(address(protocol), true);
    }
}
```

---

## Foundry Tests

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "../src/ProtocolStorage.sol";
import "../src/EmergencySystem.sol";
import "../src/ParameterStore.sol";
import "../src/FeeCollector.sol";

contract ProtocolDesignTest is Test {

    ProtocolStorage storage_;
    EmergencySystem emergency;
    ParameterStore  paramStore;
    FeeCollector    feeCollector;

    address admin    = address(1);
    address council1 = address(2);
    address council2 = address(3);
    address user     = address(4);
    address treasury = address(5);
    address stakers  = address(6);

    function setUp() public {
        vm.startPrank(admin);

        // Deploy
        storage_  = new ProtocolStorage(admin);
        paramStore = new ParameterStore(48 hours);

        address[] memory c = new address[](2);
        c[0] = council1;
        c[1] = council2;
        emergency = new EmergencySystem(c);

        feeCollector = new FeeCollector(admin, treasury, stakers);

        // Init parameters
        paramStore.initParameter(keccak256("FEE_RATE"), 30, 0, 1000, "Fee rate");

        vm.stopPrank();
    }

    // ─────────────────────────────────────────────────────────────
    // Storage Tests
    // ─────────────────────────────────────────────────────────────

    function test_OnlyLogicCanWriteStorage() public {
        // Set logic contract
        vm.prank(admin);
        storage_.setLogicContract(address(this));

        // Now this contract can write
        storage_.addToBalance(user, address(0xDEAD), 100 ether);
        assertEq(storage_.getBalance(user, address(0xDEAD)), 100 ether);
    }

    function test_RevertIf_NonLogicWritesStorage() public {
        vm.expectRevert("Storage: caller is not logic contract");
        vm.prank(user);
        storage_.addToBalance(user, address(0xDEAD), 100 ether);
    }

    // ─────────────────────────────────────────────────────────────
    // Emergency Tests
    // ─────────────────────────────────────────────────────────────

    function test_CouncilCanPauseDepositsL1() public {
        assertEq(uint8(emergency.pauseLevel()), 0);

        vm.prank(council1);
        emergency.pauseDeposits();

        assertEq(uint8(emergency.pauseLevel()), 1);
    }

    function test_L2PauseRequiresMultisig() public {
        // Propose L2 pause
        vm.prank(council1);
        bytes32 proposalId = emergency.proposePause(EmergencySystem.PauseLevel.WITHDRAWALS);

        // Still at L0 (only 1 approval so far)
        assertEq(uint8(emergency.pauseLevel()), 0);

        // Second approval executes it
        vm.prank(council2);
        emergency.approvePause(proposalId);

        assertEq(uint8(emergency.pauseLevel()), 2);
    }

    // ─────────────────────────────────────────────────────────────
    // ParameterStore Tests
    // ─────────────────────────────────────────────────────────────

    function test_TimelockEnforced() public {
        bytes32 key = keccak256("FEE_RATE");

        vm.prank(admin);
        bytes32 changeId = paramStore.queueChange(key, 50);

        // Try to execute immediately - should fail
        vm.expectRevert("ParamStore: timelock not expired");
        vm.prank(admin);
        paramStore.executeChange(changeId);

        // Warp 48 hours + 1
        vm.warp(block.timestamp + 48 hours + 1);

        // Now it should succeed
        vm.prank(admin);
        paramStore.executeChange(changeId);

        assertEq(paramStore.get(key), 50);
    }

    function test_BoundsValidation() public {
        bytes32 key = keccak256("FEE_RATE");

        // max bound is 1000 (10%), try to set 1001
        vm.expectRevert("ParamStore: value out of bounds");
        vm.prank(admin);
        paramStore.queueChange(key, 1001);
    }

    // ─────────────────────────────────────────────────────────────
    // FeeCollector Tests
    // ─────────────────────────────────────────────────────────────

    function test_FeeSplitIs50_30_20() public {
        // Setup mock token and router
        MockERC20 token = new MockERC20("USD", "USDC", 6);
        token.mint(address(this), 1000e6);

        vm.prank(admin);
        feeCollector.setRouter(address(this), true);

        token.approve(address(feeCollector), 1000e6);
        feeCollector.receiveFees(address(token), 1000e6);

        // Check splits: 50% = 500, 30% = 300, 20% = 200
        assertEq(feeCollector.getPendingFees(0, address(token)), 500e6);
        assertEq(feeCollector.getPendingFees(1, address(token)), 300e6);
        // Last recipient gets remainder
        assertEq(feeCollector.getPendingFees(2, address(token)), 200e6);
    }
}

// Mock ERC20 for testing
contract MockERC20 {
    string public name;
    string public symbol;
    uint8  public decimals;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    uint256 public totalSupply;

    constructor(string memory _name, string memory _symbol, uint8 _decimals) {
        name = _name; symbol = _symbol; decimals = _decimals;
    }

    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount;
        totalSupply += amount;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount);
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(allowance[from][msg.sender] >= amount);
        require(balanceOf[from] >= amount);
        allowance[from][msg.sender] -= amount;
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        return true;
    }
}
```

---

## สรุป Part 63

- **Separation of Concerns**: แยก Storage/Logic/Access Control ทำให้ Upgrade ง่ายและทดสอบได้แยกส่วน
- **MigrationHelper**: ช่วยให้ User ย้ายจาก V1→V2 อย่างปลอดภัยใน 1 Transaction โดยไม่ต้อง Withdraw มือ
- **FeeCollector**: ใช้ Basis Points + Pull Pattern ทำให้ Fee Routing ยืดหยุ่นและปลอดภัยจาก Reentrancy
- **ParameterStore**: Governance + Timelock + Bounds Validation คือ 3 ชั้นป้องกัน Parameter Manipulation
- **EmergencySystem**: 3 Levels ของ Pause ตามความรุนแรง โดย L1 ใช้ Council คนเดียว แต่ L2+ ต้องการ Multisig

---

## Next: Part 64 - Advanced Tokenomics Engineering
