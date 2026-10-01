# Part 65: Advanced Security Patterns

## บทนำ

Security ใน Solidity ไม่ใช่แค่การหลีกเลี่ยง Reentrancy หรือ Integer Overflow เบื้องต้น แต่คือการออกแบบ **Defense-in-Depth** ที่ผสม Pattern หลายชั้นเข้าด้วยกัน ทำให้ Attacker ต้องทะลุหลายชั้นพร้อมกัน

ใน Part นี้เราจะสร้างระบบ Security ที่ Protocol ระดับ Billions-in-TVL ใช้จริง

---

## ทำไม Security Patterns จึงสำคัญถึงขนาดนั้น?

สถิติที่น่ากลัว:
- DeFi สูญเสียมากกว่า $4B จาก Exploit ในปี 2022
- 60%+ ของ Exploit มาจาก Access Control ที่ผิดพลาด
- 30%+ มาจากการ Upgrade Contract ที่ไม่ปลอดภัย
- Pattern ใน Part นี้ป้องกันได้ทั้งหมด

---

## 1. Multi-Layer Access Control: RoleManager

### ปัญหาของ Access Control ทั่วไป

```solidity
// BAD: Single admin = single point of failure
modifier onlyOwner() {
    require(msg.sender == owner, "not owner");
    _;
}
```

ปัญหา:
1. Owner Key ถูก Compromise → Protocol ถูกยึดทั้งหมด
2. Owner ลาออก → ไม่มีใครดูแล Protocol
3. ไม่มีกำหนดเวลา → Role อยู่ตลอดไป

### Pattern: Multi-Tier Role with 2-Step Transfer and Expiry

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title RoleManager
/// @notice Multi-tier role system with 2-step transfer and time-bounded roles
/// @custom:invariant Only ADMIN can grant OPERATOR roles
/// @custom:invariant Only OPERATOR can grant USER roles
/// @custom:invariant A role with expiry=0 never expires; otherwise expires at that timestamp
/// @custom:invariant Role transfer requires acceptance by the new holder
contract RoleManager {

    // ─────────────────────────────────────────────────────────────
    // Constants
    // ─────────────────────────────────────────────────────────────

    bytes32 public constant ADMIN_ROLE    = keccak256("ADMIN_ROLE");
    bytes32 public constant OPERATOR_ROLE = keccak256("OPERATOR_ROLE");
    bytes32 public constant USER_ROLE     = keccak256("USER_ROLE");

    // ─────────────────────────────────────────────────────────────
    // Types
    // ─────────────────────────────────────────────────────────────

    struct RoleData {
        bool    active;
        uint256 expiry;      // 0 = no expiry; otherwise: unix timestamp
        address grantedBy;   // Track who granted this role
        uint256 grantedAt;
    }

    struct PendingTransfer {
        bytes32 role;
        address from;
        address to;
        uint256 deadline;    // Offer expires if not accepted in time
    }

    // ─────────────────────────────────────────────────────────────
    // State
    // ─────────────────────────────────────────────────────────────

    // role => account => RoleData
    mapping(bytes32 => mapping(address => RoleData)) private _roleData;

    // role => adminRole
    mapping(bytes32 => bytes32) private _roleAdmin;

    // Pending 2-step role transfers: transferId => PendingTransfer
    mapping(bytes32 => PendingTransfer) public pendingTransfers;

    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────

    event RoleGranted(
        bytes32 indexed role,
        address indexed account,
        address indexed grantor,
        uint256 expiry
    );
    event RoleRevoked(bytes32 indexed role, address indexed account, address indexed revoker);
    event RoleTransferInitiated(bytes32 indexed transferId, bytes32 indexed role, address indexed from, address to);
    event RoleTransferAccepted(bytes32 indexed transferId, bytes32 indexed role, address indexed newHolder);
    event RoleTransferCancelled(bytes32 indexed transferId);
    event RoleExpired(bytes32 indexed role, address indexed account);

    // ─────────────────────────────────────────────────────────────
    // Modifiers
    // ─────────────────────────────────────────────────────────────

    modifier onlyRole(bytes32 role) {
        _checkRole(role, msg.sender);
        _;
    }

    modifier onlyRoleAdmin(bytes32 role) {
        _checkRole(_roleAdmin[role], msg.sender);
        _;
    }

    // ─────────────────────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────────────────────

    constructor(address initialAdmin) {
        // Set role hierarchy
        _roleAdmin[ADMIN_ROLE]    = ADMIN_ROLE;    // Admin manages itself
        _roleAdmin[OPERATOR_ROLE] = ADMIN_ROLE;    // Admin grants Operator
        _roleAdmin[USER_ROLE]     = OPERATOR_ROLE; // Operator grants User

        // Grant initial admin with no expiry
        _roleData[ADMIN_ROLE][initialAdmin] = RoleData({
            active:    true,
            expiry:    0,          // No expiry for initial admin
            grantedBy: address(0),
            grantedAt: block.timestamp
        });

        emit RoleGranted(ADMIN_ROLE, initialAdmin, address(0), 0);
    }

    // ─────────────────────────────────────────────────────────────
    // Core Role Functions
    // ─────────────────────────────────────────────────────────────

    /// @notice Grant a role to an account
    /// @param role The role to grant
    /// @param account The recipient
    /// @param expiry Unix timestamp expiry (0 = no expiry)
    function grantRole(
        bytes32 role,
        address account,
        uint256 expiry
    ) external onlyRoleAdmin(role) {
        require(account != address(0), "RoleManager: zero address");
        if (expiry != 0) {
            require(expiry > block.timestamp, "RoleManager: expiry in the past");
        }

        _roleData[role][account] = RoleData({
            active:    true,
            expiry:    expiry,
            grantedBy: msg.sender,
            grantedAt: block.timestamp
        });

        emit RoleGranted(role, account, msg.sender, expiry);
    }

    /// @notice Revoke a role from an account
    function revokeRole(bytes32 role, address account) external onlyRoleAdmin(role) {
        require(_roleData[role][account].active, "RoleManager: role not active");
        _roleData[role][account].active = false;
        emit RoleRevoked(role, account, msg.sender);
    }

    /// @notice Renounce your own role
    function renounceRole(bytes32 role) external {
        require(_roleData[role][msg.sender].active, "RoleManager: not active");
        _roleData[role][msg.sender].active = false;
        emit RoleRevoked(role, msg.sender, msg.sender);
    }

    // ─────────────────────────────────────────────────────────────
    // 2-Step Role Transfer
    // ─────────────────────────────────────────────────────────────

    /// @notice Initiate role transfer (Step 1: current holder offers transfer)
    /// @param role The role to transfer
    /// @param newHolder The intended recipient
    /// @param acceptDeadline How long the recipient has to accept (seconds from now)
    function initiateTransfer(
        bytes32 role,
        address newHolder,
        uint256 acceptDeadline
    ) external returns (bytes32 transferId) {
        _checkRole(role, msg.sender);
        require(newHolder != address(0), "RoleManager: zero address");
        require(newHolder != msg.sender, "RoleManager: transfer to self");
        require(acceptDeadline >= 1 hours, "RoleManager: deadline too short");
        require(acceptDeadline <= 7 days, "RoleManager: deadline too long");

        transferId = keccak256(abi.encodePacked(role, msg.sender, newHolder, block.timestamp));

        pendingTransfers[transferId] = PendingTransfer({
            role:     role,
            from:     msg.sender,
            to:       newHolder,
            deadline: block.timestamp + acceptDeadline
        });

        emit RoleTransferInitiated(transferId, role, msg.sender, newHolder);
    }

    /// @notice Accept a pending role transfer (Step 2: new holder accepts)
    function acceptTransfer(bytes32 transferId) external {
        PendingTransfer storage transfer = pendingTransfers[transferId];

        require(transfer.deadline != 0, "RoleManager: transfer not found");
        require(transfer.to == msg.sender, "RoleManager: caller is not intended recipient");
        require(block.timestamp <= transfer.deadline, "RoleManager: transfer expired");

        // Check that `from` still has the role
        _checkRole(transfer.role, transfer.from);

        // Revoke from old holder
        _roleData[transfer.role][transfer.from].active = false;
        emit RoleRevoked(transfer.role, transfer.from, msg.sender);

        // Grant to new holder (inherit expiry from old role)
        uint256 oldExpiry = _roleData[transfer.role][transfer.from].expiry;
        _roleData[transfer.role][msg.sender] = RoleData({
            active:    true,
            expiry:    oldExpiry,
            grantedBy: transfer.from,
            grantedAt: block.timestamp
        });

        emit RoleGranted(transfer.role, msg.sender, transfer.from, oldExpiry);
        emit RoleTransferAccepted(transferId, transfer.role, msg.sender);

        delete pendingTransfers[transferId];
    }

    /// @notice Cancel a pending transfer (Step 1 initiator can cancel)
    function cancelTransfer(bytes32 transferId) external {
        PendingTransfer storage transfer = pendingTransfers[transferId];
        require(transfer.from == msg.sender, "RoleManager: not transfer initiator");
        delete pendingTransfers[transferId];
        emit RoleTransferCancelled(transferId);
    }

    // ─────────────────────────────────────────────────────────────
    // View Functions
    // ─────────────────────────────────────────────────────────────

    /// @notice Check if account has a role (respects expiry)
    function hasRole(bytes32 role, address account) public view returns (bool) {
        RoleData memory d = _roleData[role][account];
        if (!d.active) return false;
        if (d.expiry != 0 && block.timestamp > d.expiry) return false;
        return true;
    }

    /// @notice Get full role metadata
    function getRoleData(bytes32 role, address account) external view returns (RoleData memory) {
        return _roleData[role][account];
    }

    /// @notice Check if a role is expired (was active but now past expiry)
    function isExpired(bytes32 role, address account) external view returns (bool) {
        RoleData memory d = _roleData[role][account];
        return d.active && d.expiry != 0 && block.timestamp > d.expiry;
    }

    // ─────────────────────────────────────────────────────────────
    // Internal
    // ─────────────────────────────────────────────────────────────

    function _checkRole(bytes32 role, address account) internal view {
        require(hasRole(role, account), "RoleManager: account lacks role");
    }
}
```

---

## 2. TimelockController Integration

### หลักการ Timelock

**Timelock** คือการบังคับ Delay ระหว่าง:
- **Schedule**: Admin ประกาศ Transaction ที่จะทำ
- **Execute**: ต้องรอ Minimum Delay ก่อน Execute ได้
- **Cancel**: Admin หรือ Guardian สามารถ Cancel ได้ในช่วง Delay

ผู้ใช้มีเวลาตรวจสอบและออกจาก Protocol ก่อน Transaction จะ Execute

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title TimelockController
/// @notice Schedule, delay, and execute transactions with governance-enforced wait period
/// @custom:invariant No transaction can execute before block.timestamp >= scheduleTime + minDelay
/// @custom:invariant Cancelled transactions cannot be re-scheduled with the same ID
contract TimelockController {

    // ─────────────────────────────────────────────────────────────
    // Types
    // ─────────────────────────────────────────────────────────────

    enum OperationState {
        UNSET,      // Operation doesn't exist
        PENDING,    // Scheduled, waiting for delay
        READY,      // Delay passed, ready to execute
        DONE,       // Executed
        CANCELLED   // Cancelled
    }

    // ─────────────────────────────────────────────────────────────
    // Constants
    // ─────────────────────────────────────────────────────────────

    bytes32 public constant PROPOSER_ROLE = keccak256("PROPOSER_ROLE");
    bytes32 public constant EXECUTOR_ROLE = keccak256("EXECUTOR_ROLE");
    bytes32 public constant CANCELLER_ROLE = keccak256("CANCELLER_ROLE");

    // ─────────────────────────────────────────────────────────────
    // State
    // ─────────────────────────────────────────────────────────────

    uint256 public minDelay;

    mapping(bytes32 => uint256)   private _timestamps;  // opId => executeAfter
    mapping(bytes32 => bool)      private _cancelled;   // opId => cancelled
    mapping(bytes32 => address)   private _roles;       // simplified: role => single account

    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────

    event CallScheduled(
        bytes32 indexed id,
        address indexed target,
        uint256 value,
        bytes data,
        bytes32 predecessor,
        uint256 delay
    );
    event CallExecuted(bytes32 indexed id, address indexed target, uint256 value, bytes data);
    event Cancelled(bytes32 indexed id);
    event MinDelayChange(uint256 oldDuration, uint256 newDuration);

    // ─────────────────────────────────────────────────────────────
    // Modifiers
    // ─────────────────────────────────────────────────────────────

    modifier onlyRole(bytes32 role) {
        require(_roles[role] == msg.sender, "Timelock: access denied");
        _;
    }

    modifier onlySelf() {
        require(msg.sender == address(this), "Timelock: must be self");
        _;
    }

    // ─────────────────────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────────────────────

    constructor(
        uint256 _minDelay,
        address proposer,
        address executor,
        address canceller
    ) {
        require(_minDelay >= 1 hours, "Timelock: delay too short");

        minDelay = _minDelay;
        _roles[PROPOSER_ROLE]  = proposer;
        _roles[EXECUTOR_ROLE]  = executor;
        _roles[CANCELLER_ROLE] = canceller;

        emit MinDelayChange(0, _minDelay);
    }

    // ─────────────────────────────────────────────────────────────
    // Schedule
    // ─────────────────────────────────────────────────────────────

    /// @notice Schedule a transaction for future execution
    /// @param target Contract to call
    /// @param value ETH to send
    /// @param data Calldata for the call
    /// @param predecessor Optional: another operation that must execute first (0 = none)
    /// @param salt Random value to allow scheduling identical calls multiple times
    /// @param delay Must be >= minDelay
    function schedule(
        address target,
        uint256 value,
        bytes calldata data,
        bytes32 predecessor,
        bytes32 salt,
        uint256 delay
    ) external onlyRole(PROPOSER_ROLE) returns (bytes32 id) {
        require(delay >= minDelay, "Timelock: insufficient delay");

        id = hashOperation(target, value, data, predecessor, salt);

        require(_timestamps[id] == 0, "Timelock: operation already scheduled");
        require(!_cancelled[id], "Timelock: operation was cancelled");

        _timestamps[id] = block.timestamp + delay;

        emit CallScheduled(id, target, value, data, predecessor, delay);
    }

    /// @notice Batch schedule multiple operations
    function scheduleBatch(
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata payloads,
        bytes32 predecessor,
        bytes32 salt,
        uint256 delay
    ) external onlyRole(PROPOSER_ROLE) returns (bytes32 id) {
        require(targets.length == values.length, "Timelock: length mismatch");
        require(targets.length == payloads.length, "Timelock: length mismatch");
        require(delay >= minDelay, "Timelock: insufficient delay");

        id = hashOperationBatch(targets, values, payloads, predecessor, salt);

        require(_timestamps[id] == 0, "Timelock: already scheduled");

        _timestamps[id] = block.timestamp + delay;

        for (uint256 i = 0; i < targets.length; i++) {
            emit CallScheduled(id, targets[i], values[i], payloads[i], predecessor, delay);
        }
    }

    // ─────────────────────────────────────────────────────────────
    // Cancel
    // ─────────────────────────────────────────────────────────────

    /// @notice Cancel a pending operation
    function cancel(bytes32 id) external onlyRole(CANCELLER_ROLE) {
        require(getOperationState(id) == OperationState.PENDING, "Timelock: not pending");
        _timestamps[id] = 0;
        _cancelled[id] = true;
        emit Cancelled(id);
    }

    // ─────────────────────────────────────────────────────────────
    // Execute
    // ─────────────────────────────────────────────────────────────

    /// @notice Execute a ready operation
    function execute(
        address target,
        uint256 value,
        bytes calldata data,
        bytes32 predecessor,
        bytes32 salt
    ) external payable onlyRole(EXECUTOR_ROLE) {
        bytes32 id = hashOperation(target, value, data, predecessor, salt);

        require(getOperationState(id) == OperationState.READY, "Timelock: not ready");

        // Check predecessor completed
        if (predecessor != bytes32(0)) {
            require(
                getOperationState(predecessor) == OperationState.DONE,
                "Timelock: predecessor not done"
            );
        }

        _timestamps[id] = 1; // Mark as done (non-zero, won't be re-scheduled)

        _execute(target, value, data);

        emit CallExecuted(id, target, value, data);
    }

    /// @notice Execute a batch of operations
    function executeBatch(
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata payloads,
        bytes32 predecessor,
        bytes32 salt
    ) external payable onlyRole(EXECUTOR_ROLE) {
        bytes32 id = hashOperationBatch(targets, values, payloads, predecessor, salt);

        require(getOperationState(id) == OperationState.READY, "Timelock: not ready");

        if (predecessor != bytes32(0)) {
            require(
                getOperationState(predecessor) == OperationState.DONE,
                "Timelock: predecessor not done"
            );
        }

        _timestamps[id] = 1;

        for (uint256 i = 0; i < targets.length; i++) {
            _execute(targets[i], values[i], payloads[i]);
            emit CallExecuted(id, targets[i], values[i], payloads[i]);
        }
    }

    // ─────────────────────────────────────────────────────────────
    // Self-Calls (changing Timelock config requires going through Timelock itself)
    // ─────────────────────────────────────────────────────────────

    /// @notice Update minimum delay (must go through Timelock queue)
    function updateDelay(uint256 newDelay) external onlySelf {
        require(newDelay >= 1 hours, "Timelock: delay too short");
        emit MinDelayChange(minDelay, newDelay);
        minDelay = newDelay;
    }

    // ─────────────────────────────────────────────────────────────
    // View Functions
    // ─────────────────────────────────────────────────────────────

    function getOperationState(bytes32 id) public view returns (OperationState) {
        if (_cancelled[id]) return OperationState.CANCELLED;

        uint256 ts = _timestamps[id];
        if (ts == 0) return OperationState.UNSET;
        if (ts == 1) return OperationState.DONE;
        if (ts > block.timestamp) return OperationState.PENDING;
        return OperationState.READY;
    }

    function isOperation(bytes32 id) external view returns (bool) {
        return _timestamps[id] > 0;
    }

    function isOperationReady(bytes32 id) external view returns (bool) {
        return getOperationState(id) == OperationState.READY;
    }

    function getTimestamp(bytes32 id) external view returns (uint256) {
        return _timestamps[id];
    }

    function hashOperation(
        address target,
        uint256 value,
        bytes calldata data,
        bytes32 predecessor,
        bytes32 salt
    ) public pure returns (bytes32) {
        return keccak256(abi.encode(target, value, data, predecessor, salt));
    }

    function hashOperationBatch(
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata payloads,
        bytes32 predecessor,
        bytes32 salt
    ) public pure returns (bytes32) {
        return keccak256(abi.encode(targets, values, payloads, predecessor, salt));
    }

    // ─────────────────────────────────────────────────────────────
    // Internal
    // ─────────────────────────────────────────────────────────────

    function _execute(address target, uint256 value, bytes calldata data) internal {
        (bool success, bytes memory returndata) = target.call{value: value}(data);
        if (!success) {
            // Bubble up the revert reason
            if (returndata.length > 0) {
                assembly {
                    let returndata_size := mload(returndata)
                    revert(add(32, returndata), returndata_size)
                }
            } else {
                revert("Timelock: execution failed");
            }
        }
    }

    receive() external payable {}
}
```

---

## 3. Minimal Proxy Factory with CREATE2

### หลักการ Minimal Proxy (EIP-1167)

**Minimal Proxy** = Clone Factory Pattern:
- Deploy ครั้งแรก: Logic Contract (full bytecode)
- Clone แต่ละครั้ง: Proxy ขนาดเล็ก (~55 bytes) ที่ Delegate ไปยัง Logic
- ประหยัด Gas สูงมาก (90%+ ถูกกว่า Deploy Contract ใหม่)

**CREATE2** ทำให้ Address คาดเดาได้ล่วงหน้า:
```
address = keccak256(0xFF, deployerAddress, salt, initCodeHash)[12:]
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title ProxyFactory
/// @notice Deploys EIP-1167 Minimal Proxies at deterministic CREATE2 addresses
/// @custom:invariant Each proxy is initialized exactly once (initialized flag)
/// @custom:invariant A proxy deployed with the same salt deploys to the same address
contract ProxyFactory {

    // ─────────────────────────────────────────────────────────────
    // State
    // ─────────────────────────────────────────────────────────────

    address public immutable implementation; // Logic contract to clone

    // Track deployed proxies
    mapping(address => bool)   public isProxy;
    mapping(bytes32 => address) public deployedAt; // salt => proxy address

    address[] public allProxies;

    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────

    event ProxyDeployed(
        address indexed proxy,
        address indexed deployer,
        bytes32 indexed salt,
        bytes initData
    );

    // ─────────────────────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────────────────────

    constructor(address _implementation) {
        require(_implementation != address(0), "Factory: zero implementation");
        implementation = _implementation;
    }

    // ─────────────────────────────────────────────────────────────
    // Deploy Functions
    // ─────────────────────────────────────────────────────────────

    /// @notice Deploy a new proxy with CREATE2 (deterministic address)
    /// @param salt Unique salt (typically includes msg.sender to prevent front-running)
    /// @param initData ABI-encoded initialization call (e.g., abi.encodeCall(init, (args)))
    /// @return proxy Address of the newly deployed proxy
    function deployProxy(
        bytes32 salt,
        bytes calldata initData
    ) external returns (address proxy) {
        // Incorporate msg.sender into salt to prevent front-running
        bytes32 safeSalt = keccak256(abi.encodePacked(salt, msg.sender));

        require(deployedAt[safeSalt] == address(0), "Factory: already deployed");

        // Deploy EIP-1167 minimal proxy using CREATE2
        proxy = _deployMinimalProxy(safeSalt);

        // Record deployment
        isProxy[proxy] = true;
        deployedAt[safeSalt] = proxy;
        allProxies.push(proxy);

        // Initialize if data provided
        if (initData.length > 0) {
            (bool success, bytes memory returndata) = proxy.call(initData);
            if (!success) {
                if (returndata.length > 0) {
                    assembly {
                        revert(add(32, returndata), mload(returndata))
                    }
                }
                revert("Factory: initialization failed");
            }
        }

        emit ProxyDeployed(proxy, msg.sender, salt, initData);
    }

    /// @notice Predict the address where a proxy will be deployed
    /// @param salt User-provided salt (same as deploy call)
    /// @param deployer The address that will call deployProxy
    function predictAddress(bytes32 salt, address deployer) external view returns (address) {
        bytes32 safeSalt = keccak256(abi.encodePacked(salt, deployer));
        bytes memory creationCode = _getCreationCode();

        return address(uint160(uint256(keccak256(abi.encodePacked(
            bytes1(0xFF),
            address(this),
            safeSalt,
            keccak256(creationCode)
        )))));
    }

    // ─────────────────────────────────────────────────────────────
    // Internal
    // ─────────────────────────────────────────────────────────────

    /// @dev Deploy EIP-1167 minimal proxy bytecode
    function _deployMinimalProxy(bytes32 salt) internal returns (address proxy) {
        bytes memory creationCode = _getCreationCode();

        assembly {
            proxy := create2(
                0,                              // No ETH
                add(creationCode, 0x20),        // Skip length prefix
                mload(creationCode),            // Bytecode length
                salt                            // Salt for CREATE2
            )
        }

        require(proxy != address(0), "Factory: deployment failed");
    }

    /// @dev EIP-1167 minimal proxy bytecode for the implementation address
    function _getCreationCode() internal view returns (bytes memory) {
        // EIP-1167 bytecode template:
        // 3d602d80600a3d3981f3363d3d373d3d3d363d73<address>5af43d82803e903d91602b57fd5bf3
        address impl = implementation;

        return abi.encodePacked(
            hex"3d602d80600a3d3981f3363d3d373d3d3d363d73",
            impl,
            hex"5af43d82803e903d91602b57fd5bf3"
        );
    }

    // ─────────────────────────────────────────────────────────────
    // View Functions
    // ─────────────────────────────────────────────────────────────

    function getProxyCount() external view returns (uint256) {
        return allProxies.length;
    }

    function getProxies(uint256 offset, uint256 limit) external view returns (address[] memory) {
        uint256 end = offset + limit;
        if (end > allProxies.length) end = allProxies.length;

        address[] memory result = new address[](end - offset);
        for (uint256 i = offset; i < end; i++) {
            result[i - offset] = allProxies[i];
        }
        return result;
    }
}

// ─────────────────────────────────────────────────────────────────────────────
// Example: Upgradeable Implementation with Initialization Guard
// ─────────────────────────────────────────────────────────────────────────────

/// @title InitializableImplementation
/// @notice Logic contract designed to be deployed as minimal proxy
/// @dev Uses initialization guard to prevent re-initialization
contract InitializableImplementation {

    // ─────────────────────────────────────────────────────────────
    // State
    // ─────────────────────────────────────────────────────────────

    bool    private _initialized;
    address public  owner;
    string  public  name;
    uint256 public  value;

    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────

    event Initialized(address indexed owner, string name);
    event ValueUpdated(uint256 oldValue, uint256 newValue);

    // ─────────────────────────────────────────────────────────────
    // Modifiers
    // ─────────────────────────────────────────────────────────────

    modifier initializer() {
        require(!_initialized, "Init: already initialized");
        _initialized = true;
        _;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Init: not owner");
        _;
    }

    // ─────────────────────────────────────────────────────────────
    // Initialization (called once by ProxyFactory after deployment)
    // ─────────────────────────────────────────────────────────────

    /// @notice Initialize the proxy instance
    /// @dev Replaces constructor for clone pattern
    function initialize(address _owner, string calldata _name) external initializer {
        require(_owner != address(0), "Init: zero owner");
        owner = _owner;
        name  = _name;
        emit Initialized(_owner, _name);
    }

    // ─────────────────────────────────────────────────────────────
    // Business Logic
    // ─────────────────────────────────────────────────────────────

    function setValue(uint256 newValue) external onlyOwner {
        uint256 old = value;
        value = newValue;
        emit ValueUpdated(old, newValue);
    }

    function isInitialized() external view returns (bool) {
        return _initialized;
    }
}
```

---

## 4. GuardianSystem: Off-Chain Monitoring + On-Chain Approval

### หลักการ Guardian System

Protocol บางอย่างต้องการ Off-chain Monitoring ที่ Trigger On-chain Action:
1. **Off-chain Monitor**: Bots ตรวจสอบ Oracle Price, Pool Ratio, TVL Changes
2. **Guardian Approval**: เมื่อตรวจพบ Anomaly ต้องได้รับ N-of-M Guardian Approval
3. **Time Window**: Action ต้องทำภายใน Time Window มิฉะนั้น Expire

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title GuardianSystem
/// @notice Off-chain monitoring with on-chain N-of-M approval for emergency actions
/// @custom:invariant An action requires >= threshold guardian approvals
/// @custom:invariant An approved action must execute within executionWindow
/// @custom:invariant A guardian cannot approve the same action twice
contract GuardianSystem {

    // ─────────────────────────────────────────────────────────────
    // Types
    // ─────────────────────────────────────────────────────────────

    struct GuardianAction {
        bytes32   actionType;    // keccak256 of action name (e.g., keccak256("PAUSE"))
        bytes     data;          // ABI-encoded action parameters
        uint256   proposedAt;
        uint256   executeAfter;  // Can't execute before this
        uint256   executeBefore; // Expires after this (execution window)
        uint256   approvals;     // Current approval count
        bool      executed;
        bool      cancelled;
        mapping(address => bool) hasApproved;
    }

    // ─────────────────────────────────────────────────────────────
    // Constants
    // ─────────────────────────────────────────────────────────────

    uint256 public constant MIN_EXECUTION_WINDOW = 10 minutes;
    uint256 public constant MAX_EXECUTION_WINDOW = 24 hours;

    // ─────────────────────────────────────────────────────────────
    // State
    // ─────────────────────────────────────────────────────────────

    mapping(address => bool) public guardians;
    uint256 public guardianCount;
    uint256 public threshold;         // Minimum approvals needed

    mapping(bytes32 => GuardianAction) private _actions;
    uint256 public executionWindow;   // How long after threshold is met to execute

    address public owner;

    // Authorized action handlers (contracts that execute actions)
    mapping(bytes32 => address) public actionHandlers;

    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────

    event ActionProposed(
        bytes32 indexed actionId,
        bytes32 indexed actionType,
        address indexed proposer,
        bytes data,
        uint256 executeBefore
    );
    event ActionApproved(bytes32 indexed actionId, address indexed guardian, uint256 totalApprovals);
    event ActionExecuted(bytes32 indexed actionId, bytes32 indexed actionType);
    event ActionExpired(bytes32 indexed actionId);
    event ActionCancelled(bytes32 indexed actionId);
    event GuardianAdded(address indexed guardian);
    event GuardianRemoved(address indexed guardian);
    event ThresholdUpdated(uint256 oldThreshold, uint256 newThreshold);

    // ─────────────────────────────────────────────────────────────
    // Modifiers
    // ─────────────────────────────────────────────────────────────

    modifier onlyOwner() {
        require(msg.sender == owner, "Guardian: not owner");
        _;
    }

    modifier onlyGuardian() {
        require(guardians[msg.sender], "Guardian: not a guardian");
        _;
    }

    // ─────────────────────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────────────────────

    constructor(
        address[] memory initialGuardians,
        uint256 _threshold,
        uint256 _executionWindow
    ) {
        require(_threshold > 0, "Guardian: zero threshold");
        require(initialGuardians.length >= _threshold, "Guardian: insufficient guardians for threshold");
        require(_executionWindow >= MIN_EXECUTION_WINDOW, "Guardian: window too short");
        require(_executionWindow <= MAX_EXECUTION_WINDOW, "Guardian: window too long");

        for (uint256 i = 0; i < initialGuardians.length; i++) {
            _addGuardian(initialGuardians[i]);
        }

        threshold = _threshold;
        executionWindow = _executionWindow;
        owner = msg.sender;
    }

    // ─────────────────────────────────────────────────────────────
    // Action Lifecycle
    // ─────────────────────────────────────────────────────────────

    /// @notice Propose an emergency action (typically called by monitoring bot)
    /// @param actionType Type identifier (e.g., keccak256("PAUSE_PROTOCOL"))
    /// @param data ABI-encoded parameters for the action
    /// @return actionId Unique identifier for this action
    function proposeAction(
        bytes32 actionType,
        bytes calldata data
    ) external onlyGuardian returns (bytes32 actionId) {
        actionId = keccak256(abi.encodePacked(actionType, data, block.timestamp));

        GuardianAction storage action = _actions[actionId];
        require(action.proposedAt == 0, "Guardian: action already exists");

        action.actionType    = actionType;
        action.data          = data;
        action.proposedAt    = block.timestamp;
        action.executeAfter  = block.timestamp; // Can execute immediately once approved
        action.executeBefore = block.timestamp + executionWindow;
        action.approvals     = 1;
        action.hasApproved[msg.sender] = true;

        emit ActionProposed(actionId, actionType, msg.sender, data, action.executeBefore);
        emit ActionApproved(actionId, msg.sender, 1);

        // Auto-execute if threshold is 1
        if (threshold == 1 && actionHandlers[actionType] != address(0)) {
            _executeAction(actionId);
        }
    }

    /// @notice Approve a proposed action
    function approveAction(bytes32 actionId) external onlyGuardian {
        GuardianAction storage action = _actions[actionId];

        require(action.proposedAt != 0, "Guardian: action not found");
        require(!action.executed, "Guardian: already executed");
        require(!action.cancelled, "Guardian: cancelled");
        require(block.timestamp <= action.executeBefore, "Guardian: action expired");
        require(!action.hasApproved[msg.sender], "Guardian: already approved");

        action.hasApproved[msg.sender] = true;
        action.approvals++;

        emit ActionApproved(actionId, msg.sender, action.approvals);

        // Auto-execute if threshold reached and handler registered
        if (action.approvals >= threshold && actionHandlers[action.actionType] != address(0)) {
            _executeAction(actionId);
        }
    }

    /// @notice Execute an approved action (can be called by anyone once approved)
    function executeAction(bytes32 actionId) external {
        GuardianAction storage action = _actions[actionId];

        require(action.proposedAt != 0, "Guardian: action not found");
        require(action.approvals >= threshold, "Guardian: insufficient approvals");
        require(!action.executed, "Guardian: already executed");
        require(!action.cancelled, "Guardian: cancelled");
        require(block.timestamp >= action.executeAfter, "Guardian: too early");
        require(block.timestamp <= action.executeBefore, "Guardian: action expired");

        _executeAction(actionId);
    }

    /// @notice Cancel an action (guardian or owner)
    function cancelAction(bytes32 actionId) external {
        require(guardians[msg.sender] || msg.sender == owner, "Guardian: unauthorized");

        GuardianAction storage action = _actions[actionId];
        require(!action.executed, "Guardian: already executed");
        require(!action.cancelled, "Guardian: already cancelled");

        action.cancelled = true;
        emit ActionCancelled(actionId);
    }

    // ─────────────────────────────────────────────────────────────
    // Internal
    // ─────────────────────────────────────────────────────────────

    function _executeAction(bytes32 actionId) internal {
        GuardianAction storage action = _actions[actionId];
        action.executed = true;

        address handler = actionHandlers[action.actionType];
        require(handler != address(0), "Guardian: no handler registered");

        (bool success,) = handler.call(action.data);
        require(success, "Guardian: action execution failed");

        emit ActionExecuted(actionId, action.actionType);
    }

    function _addGuardian(address guardian) internal {
        require(guardian != address(0), "Guardian: zero address");
        require(!guardians[guardian], "Guardian: already guardian");
        guardians[guardian] = true;
        guardianCount++;
        emit GuardianAdded(guardian);
    }

    // ─────────────────────────────────────────────────────────────
    // View Functions
    // ─────────────────────────────────────────────────────────────

    function getActionApprovals(bytes32 actionId) external view returns (uint256) {
        return _actions[actionId].approvals;
    }

    function hasApproved(bytes32 actionId, address guardian) external view returns (bool) {
        return _actions[actionId].hasApproved[guardian];
    }

    function isActionExpired(bytes32 actionId) external view returns (bool) {
        GuardianAction storage action = _actions[actionId];
        return action.proposedAt != 0 && block.timestamp > action.executeBefore && !action.executed;
    }

    function isReadyToExecute(bytes32 actionId) external view returns (bool) {
        GuardianAction storage action = _actions[actionId];
        return action.approvals >= threshold
            && !action.executed
            && !action.cancelled
            && block.timestamp >= action.executeAfter
            && block.timestamp <= action.executeBefore;
    }

    // ─────────────────────────────────────────────────────────────
    // Admin
    // ─────────────────────────────────────────────────────────────

    function addGuardian(address guardian) external onlyOwner {
        _addGuardian(guardian);
    }

    function removeGuardian(address guardian) external onlyOwner {
        require(guardians[guardian], "Guardian: not a guardian");
        require(guardianCount > threshold, "Guardian: would break threshold");
        guardians[guardian] = false;
        guardianCount--;
        emit GuardianRemoved(guardian);
    }

    function updateThreshold(uint256 newThreshold) external onlyOwner {
        require(newThreshold > 0, "Guardian: zero threshold");
        require(newThreshold <= guardianCount, "Guardian: exceeds guardian count");
        emit ThresholdUpdated(threshold, newThreshold);
        threshold = newThreshold;
    }

    function registerActionHandler(bytes32 actionType, address handler) external onlyOwner {
        actionHandlers[actionType] = handler;
    }
}
```

---

## 5. Security Invariants as NatSpec + Foundry Invariant Tests

### หลักการ Security Invariants

**Invariant** คือเงื่อนไขที่ต้องเป็นจริงเสมอ ไม่ว่าจะเรียก Function ใดก็ตาม:
- `totalDeposits >= sum(userDeposits)` → Protocol ไม่ขาดทุน
- `vePower <= lockedAmount` → Voting Power ไม่เกิน Lock Amount
- `initialized == true` → Contract ถูก Initialize แล้ว

**Foundry Invariant Tests** รัน Random Function Calls แล้วตรวจสอบ Invariants หลังทุก Call

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "forge-std/StdInvariant.sol";

// ─────────────────────────────────────────────────────────────────────────────
// Contract Under Test: SecureVault
// ─────────────────────────────────────────────────────────────────────────────

/// @title SecureVault
/// @notice Example vault with explicit security invariants
/// @custom:invariant INVARIANT-1: totalDeposited == sum of all userDeposits
/// @custom:invariant INVARIANT-2: contract token balance >= totalDeposited
/// @custom:invariant INVARIANT-3: no user can withdraw more than their deposit
/// @custom:invariant INVARIANT-4: pausedAt[L1] => no new deposits accepted
/// @custom:invariant INVARIANT-5: pausedAt[L2] => no withdrawals accepted
contract SecureVault {
    using SafeERC20 for IERC20;

    // ─────────────────────────────────────────────────────────────
    // State
    // ─────────────────────────────────────────────────────────────

    IERC20 public immutable token;

    mapping(address => uint256) public deposits;
    uint256 public totalDeposited;

    address public owner;
    bool    public depositsPaused;
    bool    public withdrawalsPaused;

    // ─────────────────────────────────────────────────────────────
    // Events
    // ─────────────────────────────────────────────────────────────

    event Deposited(address indexed user, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);

    // ─────────────────────────────────────────────────────────────
    // Constructor
    // ─────────────────────────────────────────────────────────────

    constructor(address _token) {
        token = IERC20(_token);
        owner = msg.sender;
    }

    // ─────────────────────────────────────────────────────────────
    // Functions
    // ─────────────────────────────────────────────────────────────

    function deposit(uint256 amount) external {
        require(!depositsPaused, "Vault: deposits paused");
        require(amount > 0, "Vault: zero amount");

        token.safeTransferFrom(msg.sender, address(this), amount);
        deposits[msg.sender] += amount;
        totalDeposited += amount;

        emit Deposited(msg.sender, amount);
    }

    function withdraw(uint256 amount) external {
        require(!withdrawalsPaused, "Vault: withdrawals paused");
        require(deposits[msg.sender] >= amount, "Vault: insufficient balance");
        require(amount > 0, "Vault: zero amount");

        deposits[msg.sender] -= amount;
        totalDeposited -= amount;

        token.safeTransfer(msg.sender, amount);

        emit Withdrawn(msg.sender, amount);
    }

    function pauseDeposits()   external { require(msg.sender == owner); depositsPaused = true; }
    function pauseWithdrawals() external { require(msg.sender == owner); withdrawalsPaused = true; }
    function unpause()          external { require(msg.sender == owner); depositsPaused = false; withdrawalsPaused = false; }
}

// Need SafeERC20 import for the above
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

// ─────────────────────────────────────────────────────────────────────────────
// Handler Contract (for Foundry Invariant Testing)
// ─────────────────────────────────────────────────────────────────────────────

/// @title VaultHandler
/// @notice Wraps SecureVault calls for invariant fuzzing
/// @dev Foundry calls random functions on this handler between invariant checks
contract VaultHandler is Test {

    SecureVault public vault;
    MockERC20   public token;

    // Ghost variables: track expected state independently
    uint256 public ghost_totalDeposited;
    mapping(address => uint256) public ghost_deposits;

    address[] public actors;
    address   public currentActor;

    // Bound inputs to valid ranges
    uint256 constant MAX_AMOUNT = 1_000_000 ether;

    constructor(SecureVault _vault, MockERC20 _token) {
        vault = _vault;
        token = _token;

        // Setup actors
        actors.push(address(0x1111));
        actors.push(address(0x2222));
        actors.push(address(0x3333));

        // Fund actors
        for (uint256 i = 0; i < actors.length; i++) {
            token.mint(actors[i], MAX_AMOUNT);
            vm.prank(actors[i]);
            token.approve(address(vault), type(uint256).max);
        }
    }

    // ─────────────────────────────────────────────────────────────
    // Handler Functions (Foundry randomly calls these)
    // ─────────────────────────────────────────────────────────────

    /// @notice Handler: deposit random amount for random actor
    function deposit(uint256 actorSeed, uint256 amount) external {
        amount = bound(amount, 1, MAX_AMOUNT / actors.length);
        currentActor = _selectActor(actorSeed);

        vm.prank(currentActor);
        try vault.deposit(amount) {
            ghost_deposits[currentActor] += amount;
            ghost_totalDeposited += amount;
        } catch {
            // Deposits paused or insufficient balance - fine, skip
        }
    }

    /// @notice Handler: withdraw random amount for random actor
    function withdraw(uint256 actorSeed, uint256 amount) external {
        currentActor = _selectActor(actorSeed);
        uint256 maxWithdraw = vault.deposits(currentActor);
        if (maxWithdraw == 0) return;

        amount = bound(amount, 1, maxWithdraw);

        vm.prank(currentActor);
        try vault.withdraw(amount) {
            ghost_deposits[currentActor] -= amount;
            ghost_totalDeposited -= amount;
        } catch {
            // Withdrawals paused - fine
        }
    }

    /// @notice Handler: pause deposits
    function pauseDeposits() external {
        address _owner = vault.owner();
        vm.prank(_owner);
        vault.pauseDeposits();
    }

    /// @notice Handler: unpause
    function unpause() external {
        address _owner = vault.owner();
        vm.prank(_owner);
        vault.unpause();
    }

    // ─────────────────────────────────────────────────────────────
    // Internal
    // ─────────────────────────────────────────────────────────────

    function _selectActor(uint256 seed) internal view returns (address) {
        return actors[seed % actors.length];
    }

    function getActorCount() external view returns (uint256) {
        return actors.length;
    }
}

// ─────────────────────────────────────────────────────────────────────────────
// Invariant Test Contract
// ─────────────────────────────────────────────────────────────────────────────

/// @title VaultInvariantTest
/// @notice Foundry invariant tests for SecureVault
contract VaultInvariantTest is StdInvariant, Test {

    SecureVault  public vault;
    MockERC20    public token;
    VaultHandler public handler;

    function setUp() public {
        token   = new MockERC20("USD", "USDC", 6);
        vault   = new SecureVault(address(token));
        handler = new VaultHandler(vault, token);

        // Fund vault to simulate existing deposits
        token.mint(address(vault), 0); // Start empty

        // Tell Foundry to call handler functions randomly
        targetContract(address(handler));

        // Set which functions Foundry should call
        bytes4[] memory selectors = new bytes4[](4);
        selectors[0] = VaultHandler.deposit.selector;
        selectors[1] = VaultHandler.withdraw.selector;
        selectors[2] = VaultHandler.pauseDeposits.selector;
        selectors[3] = VaultHandler.unpause.selector;

        targetSelector(FuzzSelector({addr: address(handler), selectors: selectors}));
    }

    // ─────────────────────────────────────────────────────────────
    // Invariant Tests
    // ─────────────────────────────────────────────────────────────

    /// @custom:invariant INVARIANT-1: totalDeposited matches ghost variable
    function invariant_totalDepositedMatchesGhost() public view {
        assertEq(
            vault.totalDeposited(),
            handler.ghost_totalDeposited(),
            "INVARIANT-1 BROKEN: totalDeposited mismatch"
        );
    }

    /// @custom:invariant INVARIANT-2: vault token balance >= totalDeposited
    function invariant_vaultSolvent() public view {
        uint256 vaultBalance = token.balanceOf(address(vault));
        assertGe(
            vaultBalance,
            vault.totalDeposited(),
            "INVARIANT-2 BROKEN: vault is insolvent"
        );
    }

    /// @custom:invariant INVARIANT-3: each user deposit <= their ghost deposit
    function invariant_userDepositsConsistent() public view {
        address[3] memory actors = [address(0x1111), address(0x2222), address(0x3333)];

        for (uint256 i = 0; i < actors.length; i++) {
            assertLe(
                vault.deposits(actors[i]),
                handler.ghost_deposits(actors[i]),
                "INVARIANT-3 BROKEN: user deposit exceeds ghost"
            );
        }
    }

    /// @custom:invariant INVARIANT-4: no user deposit exceeds vault balance
    function invariant_noUserExceedsVaultBalance() public view {
        address[3] memory actors = [address(0x1111), address(0x2222), address(0x3333)];

        uint256 totalUserDeposits = 0;
        for (uint256 i = 0; i < actors.length; i++) {
            totalUserDeposits += vault.deposits(actors[i]);
        }

        assertLe(
            totalUserDeposits,
            token.balanceOf(address(vault)),
            "INVARIANT-4 BROKEN: sum of user deposits exceeds vault balance"
        );
    }
}

// ─────────────────────────────────────────────────────────────────────────────
// Unit Tests Alongside Invariants
// ─────────────────────────────────────────────────────────────────────────────

contract SecurityPatternsUnitTest is Test {

    RoleManager rm;
    TimelockController timelock;
    ProxyFactory factory;
    InitializableImplementation impl;
    GuardianSystem guardian;

    address admin     = address(0xAD);
    address operator  = address(0x0B);
    address user      = address(0x5E);
    address proposer  = address(0xA1);
    address executor_ = address(0xA2);
    address canceller = address(0xA3);

    function setUp() public {
        vm.startPrank(admin);
        rm       = new RoleManager(admin);
        timelock = new TimelockController(2 hours, proposer, executor_, canceller);
        impl     = new InitializableImplementation();
        factory  = new ProxyFactory(address(impl));

        address[] memory guardians = new address[](3);
        guardians[0] = address(0xG1);
        guardians[1] = address(0xG2);
        guardians[2] = address(0xG3);
        guardian = new GuardianSystem(guardians, 2, 30 minutes);

        vm.stopPrank();
    }

    // ─────────────────────────────────────────────────────────────
    // RoleManager Tests
    // ─────────────────────────────────────────────────────────────

    function test_RoleExpiry() public {
        vm.prank(admin);
        rm.grantRole(RoleManager.OPERATOR_ROLE, operator, block.timestamp + 1 days);

        // Should have role now
        assertTrue(rm.hasRole(RoleManager.OPERATOR_ROLE, operator));

        // After expiry
        vm.warp(block.timestamp + 2 days);
        assertFalse(rm.hasRole(RoleManager.OPERATOR_ROLE, operator));
    }

    function test_2StepTransfer() public {
        // Grant role to operator first
        vm.prank(admin);
        rm.grantRole(RoleManager.OPERATOR_ROLE, operator, 0);

        // Step 1: Operator initiates transfer to user
        vm.prank(operator);
        bytes32 transferId = rm.initiateTransfer(RoleManager.OPERATOR_ROLE, user, 2 hours);

        // User doesn't have role yet
        assertFalse(rm.hasRole(RoleManager.OPERATOR_ROLE, user));

        // Step 2: User accepts
        vm.prank(user);
        rm.acceptTransfer(transferId);

        // Now user has the role, operator doesn't
        assertTrue(rm.hasRole(RoleManager.OPERATOR_ROLE, user));
        assertFalse(rm.hasRole(RoleManager.OPERATOR_ROLE, operator));
    }

    function test_TransferExpires() public {
        vm.prank(admin);
        rm.grantRole(RoleManager.OPERATOR_ROLE, operator, 0);

        vm.prank(operator);
        bytes32 transferId = rm.initiateTransfer(RoleManager.OPERATOR_ROLE, user, 1 hours);

        // Warp past deadline
        vm.warp(block.timestamp + 2 hours);

        vm.expectRevert("RoleManager: transfer expired");
        vm.prank(user);
        rm.acceptTransfer(transferId);
    }

    // ─────────────────────────────────────────────────────────────
    // TimelockController Tests
    // ─────────────────────────────────────────────────────────────

    function test_TimelockMinimumDelay() public {
        address target   = address(0xDEAD);
        bytes memory data = abi.encodeWithSignature("someFunction()");

        // Try to schedule with delay less than minDelay
        vm.expectRevert("Timelock: insufficient delay");
        vm.prank(proposer);
        timelock.schedule(target, 0, data, bytes32(0), bytes32(0), 1 hours); // Need >= 2 hours
    }

    function test_TimelockFullCycle() public {
        MockTarget target = new MockTarget();
        bytes memory data = abi.encodeWithSignature("doSomething()");

        // Schedule
        vm.prank(proposer);
        bytes32 id = timelock.schedule(address(target), 0, data, bytes32(0), bytes32(uint256(1)), 2 hours);

        assertEq(
            uint8(timelock.getOperationState(id)),
            uint8(TimelockController.OperationState.PENDING)
        );

        // Can't execute yet
        vm.warp(block.timestamp + 1 hours);
        assertEq(
            uint8(timelock.getOperationState(id)),
            uint8(TimelockController.OperationState.PENDING)
        );

        // After 2 hours, should be ready
        vm.warp(block.timestamp + 1 hours + 1);
        assertEq(
            uint8(timelock.getOperationState(id)),
            uint8(TimelockController.OperationState.READY)
        );

        // Execute
        vm.prank(executor_);
        timelock.execute(address(target), 0, data, bytes32(0), bytes32(uint256(1)));

        assertTrue(target.called());
        assertEq(
            uint8(timelock.getOperationState(id)),
            uint8(TimelockController.OperationState.DONE)
        );
    }

    // ─────────────────────────────────────────────────────────────
    // ProxyFactory Tests
    // ─────────────────────────────────────────────────────────────

    function test_DeterministicAddress() public {
        bytes32 salt = bytes32(uint256(42));
        address predicted = factory.predictAddress(salt, address(this));

        bytes memory initData = abi.encodeCall(
            InitializableImplementation.initialize,
            (address(this), "TestProxy")
        );

        address deployed = factory.deployProxy(salt, initData);

        assertEq(deployed, predicted, "Proxy address should be deterministic");
    }

    function test_CannotInitializeTwice() public {
        bytes32 salt = bytes32(uint256(99));
        bytes memory initData = abi.encodeCall(
            InitializableImplementation.initialize,
            (address(this), "TestProxy")
        );

        address proxy = factory.deployProxy(salt, initData);

        // Try to initialize again
        vm.expectRevert("Init: already initialized");
        InitializableImplementation(proxy).initialize(address(this), "Again");
    }

    function test_CannotDeployWithSameSalt() public {
        bytes32 salt = bytes32(uint256(77));

        factory.deployProxy(salt, "");

        vm.expectRevert("Factory: already deployed");
        factory.deployProxy(salt, "");
    }

    // ─────────────────────────────────────────────────────────────
    // GuardianSystem Tests
    // ─────────────────────────────────────────────────────────────

    function test_ActionNeedsThresholdApprovals() public {
        address g1 = address(0xG1);
        address g2 = address(0xG2);

        bytes32 actionType = keccak256("PAUSE_PROTOCOL");
        bytes memory data = abi.encodeWithSignature("pause()");

        // G1 proposes (counts as 1 approval)
        vm.prank(g1);
        bytes32 actionId = guardian.proposeAction(actionType, data);

        // Should need 1 more (threshold=2)
        assertEq(guardian.getActionApprovals(actionId), 1);
        assertFalse(guardian.isReadyToExecute(actionId)); // No handler = won't auto-execute

        // G2 approves
        vm.prank(g2);
        guardian.approveAction(actionId);

        assertEq(guardian.getActionApprovals(actionId), 2);
    }

    function test_ActionExpiresAfterWindow() public {
        address g1 = address(0xG1);
        bytes32 actionType = keccak256("PAUSE_PROTOCOL");

        vm.prank(g1);
        bytes32 actionId = guardian.proposeAction(actionType, "");

        // Warp past execution window
        vm.warp(block.timestamp + 31 minutes);

        assertTrue(guardian.isActionExpired(actionId));

        // Cannot approve expired action
        vm.expectRevert("Guardian: action expired");
        vm.prank(address(0xG2));
        guardian.approveAction(actionId);
    }
}

// Helpers for testing
contract MockTarget {
    bool public called;
    function doSomething() external { called = true; }
}

contract MockERC20 {
    string public name;
    string public symbol;
    uint8  public decimals;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    uint256 public totalSupply;

    constructor(string memory _n, string memory _s, uint8 _d) {
        name = _n; symbol = _s; decimals = _d;
    }

    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount; totalSupply += amount;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount; return true;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount);
        balanceOf[msg.sender] -= amount; balanceOf[to] += amount; return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(allowance[from][msg.sender] >= amount);
        require(balanceOf[from] >= amount);
        allowance[from][msg.sender] -= amount;
        balanceOf[from] -= amount; balanceOf[to] += amount; return true;
    }
}
```

---

## Workshop: Security Audit Checklist ที่นำไปใช้ได้จริง

### โจทย์

ตรวจสอบ Contract ต่อไปนี้ว่ามีช่องโหว่อะไรบ้าง และแก้ไขให้ถูกต้อง

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// ❌ BROKEN: Multiple security issues - find and fix them all
contract VulnerableVault {

    mapping(address => uint256) public balances;
    address public admin;

    // ISSUE 1: No access control check
    function setAdmin(address newAdmin) external {
        admin = newAdmin;
    }

    // ISSUE 2: Reentrancy vulnerability
    function withdraw() external {
        uint256 amount = balances[msg.sender];
        require(amount > 0, "No balance");

        // BUG: External call BEFORE state update
        (bool success,) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");

        balances[msg.sender] = 0; // Too late! Attacker already re-entered
    }

    // ISSUE 3: No bounds check on fee
    function setFeeRate(uint256 fee) external {
        // BUG: fee can be 100% or higher
        // feeRate = fee;
    }

    // ISSUE 4: Integer overflow potential (Solidity 0.8+ checks, but logic error)
    function calculateReward(uint256 amount, uint256 multiplier) external pure returns (uint256) {
        // BUG: multiplier not bounded, can overflow practical limits
        return amount * multiplier;
    }
}
```

### Solution: Secure Version

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/// @title SecureVaultFixed
/// @notice Workshop solution: all vulnerabilities patched
contract SecureVaultFixed is ReentrancyGuard, Ownable {

    mapping(address => uint256) public balances;
    uint256 public feeRate;
    uint256 public constant MAX_FEE_RATE    = 1000; // 10% max
    uint256 public constant MAX_MULTIPLIER  = 100;  // 100x max

    constructor() Ownable(msg.sender) {}

    // FIX 1: Only owner can set admin (already via Ownable)
    // No need for separate setAdmin - Ownable handles it via transferOwnership(2-step)

    // FIX 2: ReentrancyGuard + CEI pattern
    function withdraw() external nonReentrant {
        uint256 amount = balances[msg.sender];
        require(amount > 0, "No balance");

        // CORRECT ORDER: Checks-Effects-Interactions
        // 1. CHECKS: require above
        // 2. EFFECTS: state update BEFORE external call
        balances[msg.sender] = 0;

        // 3. INTERACTIONS: external call AFTER state update
        (bool success,) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }

    // FIX 3: Bounds check on fee
    function setFeeRate(uint256 fee) external onlyOwner {
        require(fee <= MAX_FEE_RATE, "SecureVault: fee too high");
        feeRate = fee;
    }

    // FIX 4: Bounded multiplier
    function calculateReward(uint256 amount, uint256 multiplier) external pure returns (uint256) {
        require(multiplier <= MAX_MULTIPLIER, "SecureVault: multiplier too high");
        return amount * multiplier;
    }

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }
}
```

---

## สรุป Part 65

- **RoleManager**: 3-Tier Hierarchy (ADMIN > OPERATOR > USER) + 2-Step Transfer ป้องกัน Key Loss + Role Expiry จำกัดสิทธิ์ชั่วคราว
- **TimelockController**: Schedule → (ผ่าน minDelay) → Execute หรือ Cancel ทุก Config Change ต้องผ่าน Timelock ให้ Users ตรวจสอบก่อน
- **ProxyFactory + CREATE2**: Deterministic Address = Predictable Deployment, Initialization Guard ป้องกัน Re-init Attack
- **GuardianSystem**: N-of-M Off-chain Monitoring + On-chain Approval + Time Window ทำให้ Bot Monitoring มี Security Layer
- **Invariant Tests**: `@custom:invariant` NatSpec + Foundry Handler Pattern ทดสอบ Protocol ด้วย Random Call Sequence

---

## Next: Part 66 - Gas Profiling Workshop
