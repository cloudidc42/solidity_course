# Part 14: Access Control และ Role Management

## สารบัญ
1. Ownable Pattern
2. Role-Based Access Control (RBAC)
3. OpenZeppelin AccessControl
4. Timelocked Admin
5. Multi-Signature Control
6. Workshop: Protocol Admin

---

## 1. Ownable Pattern (Advanced)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// Ownable2Step: transfer ownership ต้องมี 2 ขั้นตอน
contract Ownable2Step {
    
    address private _owner;
    address private _pendingOwner;
    
    event OwnershipTransferStarted(address indexed previousOwner, address indexed newOwner);
    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
    
    error OwnableUnauthorizedAccount(address account);
    error OwnableInvalidOwner(address owner);
    
    constructor(address initialOwner) {
        if (initialOwner == address(0)) revert OwnableInvalidOwner(address(0));
        _transferOwnership(initialOwner);
    }
    
    modifier onlyOwner() {
        if (owner() != msg.sender) revert OwnableUnauthorizedAccount(msg.sender);
        _;
    }
    
    function owner() public view virtual returns (address) { return _owner; }
    function pendingOwner() public view virtual returns (address) { return _pendingOwner; }
    
    function transferOwnership(address newOwner) public virtual onlyOwner {
        _pendingOwner = newOwner;
        emit OwnershipTransferStarted(owner(), newOwner);
    }
    
    function acceptOwnership() public virtual {
        address sender = msg.sender;
        if (pendingOwner() != sender) {
            revert OwnableUnauthorizedAccount(sender);
        }
        _transferOwnership(sender);
    }
    
    function renounceOwnership() public virtual onlyOwner {
        _transferOwnership(address(0));
    }
    
    function _transferOwnership(address newOwner) internal virtual {
        delete _pendingOwner;
        address oldOwner = _owner;
        _owner = newOwner;
        emit OwnershipTransferred(oldOwner, newOwner);
    }
}
```

---

## 2. Role-Based Access Control (RBAC)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract AccessControl {
    
    struct RoleData {
        mapping(address account => bool) hasRole;
        bytes32 adminRole;
    }
    
    mapping(bytes32 role => RoleData) private _roles;
    
    bytes32 public constant DEFAULT_ADMIN_ROLE = 0x00;
    
    event RoleAdminChanged(
        bytes32 indexed role,
        bytes32 indexed previousAdminRole,
        bytes32 indexed newAdminRole
    );
    event RoleGranted(bytes32 indexed role, address indexed account, address indexed sender);
    event RoleRevoked(bytes32 indexed role, address indexed account, address indexed sender);
    
    error AccessControlUnauthorizedAccount(address account, bytes32 neededRole);
    error AccessControlBadConfirmation();
    
    modifier onlyRole(bytes32 role) {
        _checkRole(role);
        _;
    }
    
    function supportsInterface(bytes4 interfaceId) public view virtual returns (bool) {
        return interfaceId == type(IAccessControl).interfaceId;
    }
    
    function hasRole(bytes32 role, address account) public view virtual returns (bool) {
        return _roles[role].hasRole[account];
    }
    
    function _checkRole(bytes32 role) internal view virtual {
        _checkRole(role, msg.sender);
    }
    
    function _checkRole(bytes32 role, address account) internal view virtual {
        if (!hasRole(role, account)) {
            revert AccessControlUnauthorizedAccount(account, role);
        }
    }
    
    function getRoleAdmin(bytes32 role) public view virtual returns (bytes32) {
        return _roles[role].adminRole;
    }
    
    function grantRole(bytes32 role, address account) public virtual onlyRole(getRoleAdmin(role)) {
        _grantRole(role, account);
    }
    
    function revokeRole(bytes32 role, address account) public virtual onlyRole(getRoleAdmin(role)) {
        _revokeRole(role, account);
    }
    
    function renounceRole(bytes32 role, address callerConfirmation) public virtual {
        if (callerConfirmation != msg.sender) {
            revert AccessControlBadConfirmation();
        }
        _revokeRole(role, callerConfirmation);
    }
    
    function _setRoleAdmin(bytes32 role, bytes32 adminRole) internal virtual {
        bytes32 previousAdminRole = getRoleAdmin(role);
        _roles[role].adminRole = adminRole;
        emit RoleAdminChanged(role, previousAdminRole, adminRole);
    }
    
    function _grantRole(bytes32 role, address account) internal virtual returns (bool) {
        if (!hasRole(role, account)) {
            _roles[role].hasRole[account] = true;
            emit RoleGranted(role, account, msg.sender);
            return true;
        }
        return false;
    }
    
    function _revokeRole(bytes32 role, address account) internal virtual returns (bool) {
        if (hasRole(role, account)) {
            _roles[role].hasRole[account] = false;
            emit RoleRevoked(role, account, msg.sender);
            return true;
        }
        return false;
    }
}

interface IAccessControl {
    function hasRole(bytes32 role, address account) external view returns (bool);
    function getRoleAdmin(bytes32 role) external view returns (bytes32);
    function grantRole(bytes32 role, address account) external;
    function revokeRole(bytes32 role, address account) external;
    function renounceRole(bytes32 role, address callerConfirmation) external;
}
```

---

## 3. TimelockController

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title TimelockController
 * @dev ทุก operation ต้องรอ delay ก่อน execute
 * ป้องกัน admin ทำอะไรฉับพลันโดยไม่แจ้ง community
 */
contract TimelockController is AccessControl {
    
    bytes32 public constant TIMELOCK_ADMIN_ROLE = keccak256("TIMELOCK_ADMIN_ROLE");
    bytes32 public constant PROPOSER_ROLE = keccak256("PROPOSER_ROLE");
    bytes32 public constant EXECUTOR_ROLE = keccak256("EXECUTOR_ROLE");
    bytes32 public constant CANCELLER_ROLE = keccak256("CANCELLER_ROLE");
    
    uint256 internal constant _DONE_TIMESTAMP = uint256(1);
    
    mapping(bytes32 => uint256) private _timestamps;
    uint256 private _minDelay;
    
    event CallScheduled(
        bytes32 indexed id,
        uint256 indexed index,
        address target,
        uint256 value,
        bytes data,
        bytes32 predecessor,
        uint256 delay
    );
    event CallExecuted(
        bytes32 indexed id,
        uint256 indexed index,
        address target,
        uint256 value,
        bytes data
    );
    event CallSalt(bytes32 indexed id, bytes32 salt);
    event Cancelled(bytes32 indexed id);
    event MinDelayChange(uint256 oldDuration, uint256 newDuration);
    
    error TimelockInvalidOperationLength(uint256 targets, uint256 payloads, uint256 values);
    error TimelockInsufficientDelay(uint256 delay, uint256 minDelay);
    error TimelockUnexpectedOperationState(bytes32 operationId, bytes32 expectedStates);
    error TimelockUnexecutedPredecessor(bytes32 predecessorId);
    error TimelockUnauthorizedCaller(address caller);
    
    enum OperationState { Unset, Waiting, Ready, Done }
    
    constructor(
        uint256 minDelay,
        address[] memory proposers,
        address[] memory executors,
        address admin
    ) {
        _minDelay = minDelay;
        emit MinDelayChange(0, minDelay);
        
        // Admin setup
        _setRoleAdmin(TIMELOCK_ADMIN_ROLE, TIMELOCK_ADMIN_ROLE);
        _setRoleAdmin(PROPOSER_ROLE, TIMELOCK_ADMIN_ROLE);
        _setRoleAdmin(EXECUTOR_ROLE, TIMELOCK_ADMIN_ROLE);
        _setRoleAdmin(CANCELLER_ROLE, TIMELOCK_ADMIN_ROLE);
        
        _grantRole(TIMELOCK_ADMIN_ROLE, address(this));
        
        if (admin != address(0)) {
            _grantRole(TIMELOCK_ADMIN_ROLE, admin);
        }
        
        for (uint256 i = 0; i < proposers.length; ++i) {
            _grantRole(PROPOSER_ROLE, proposers[i]);
            _grantRole(CANCELLER_ROLE, proposers[i]);
        }
        
        for (uint256 i = 0; i < executors.length; ++i) {
            _grantRole(EXECUTOR_ROLE, executors[i]);
        }
    }
    
    receive() external payable {}
    
    function getMinDelay() public view virtual returns (uint256) {
        return _minDelay;
    }
    
    function getTimestamp(bytes32 id) public view virtual returns (uint256) {
        return _timestamps[id];
    }
    
    function getOperationState(bytes32 id) public view virtual returns (OperationState) {
        uint256 timestamp = getTimestamp(id);
        if (timestamp == 0) return OperationState.Unset;
        if (timestamp == _DONE_TIMESTAMP) return OperationState.Done;
        if (timestamp > block.timestamp) return OperationState.Waiting;
        return OperationState.Ready;
    }
    
    function isOperation(bytes32 id) public view returns (bool) {
        return getTimestamp(id) > 0;
    }
    
    function isOperationPending(bytes32 id) public view returns (bool) {
        OperationState state = getOperationState(id);
        return state == OperationState.Waiting || state == OperationState.Ready;
    }
    
    function isOperationReady(bytes32 id) public view returns (bool) {
        return getOperationState(id) == OperationState.Ready;
    }
    
    function isOperationDone(bytes32 id) public view returns (bool) {
        return getOperationState(id) == OperationState.Done;
    }
    
    function hashOperation(
        address target,
        uint256 value,
        bytes calldata data,
        bytes32 predecessor,
        bytes32 salt
    ) public pure virtual returns (bytes32) {
        return keccak256(abi.encode(target, value, data, predecessor, salt));
    }
    
    function hashOperationBatch(
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata payloads,
        bytes32 predecessor,
        bytes32 salt
    ) public pure virtual returns (bytes32) {
        return keccak256(abi.encode(targets, values, payloads, predecessor, salt));
    }
    
    function schedule(
        address target,
        uint256 value,
        bytes calldata data,
        bytes32 predecessor,
        bytes32 salt,
        uint256 delay
    ) public virtual onlyRole(PROPOSER_ROLE) {
        bytes32 id = hashOperation(target, value, data, predecessor, salt);
        _schedule(id, delay);
        emit CallScheduled(id, 0, target, value, data, predecessor, delay);
        if (salt != bytes32(0)) {
            emit CallSalt(id, salt);
        }
    }
    
    function _schedule(bytes32 id, uint256 delay) private {
        if (isOperation(id)) {
            revert TimelockUnexpectedOperationState(id, _encodeStateBitmap(OperationState.Unset));
        }
        if (delay < getMinDelay()) {
            revert TimelockInsufficientDelay(delay, getMinDelay());
        }
        _timestamps[id] = block.timestamp + delay;
    }
    
    function cancel(bytes32 id) public virtual onlyRole(CANCELLER_ROLE) {
        if (!isOperationPending(id)) {
            revert TimelockUnexpectedOperationState(
                id,
                _encodeStateBitmap(OperationState.Waiting) | _encodeStateBitmap(OperationState.Ready)
            );
        }
        delete _timestamps[id];
        emit Cancelled(id);
    }
    
    function execute(
        address target,
        uint256 value,
        bytes calldata payload,
        bytes32 predecessor,
        bytes32 salt
    ) public payable virtual onlyRoleOrOpenRole(EXECUTOR_ROLE) {
        bytes32 id = hashOperation(target, value, payload, predecessor, salt);
        
        _beforeCall(id, predecessor);
        _execute(target, value, payload);
        emit CallExecuted(id, 0, target, value, payload);
        _afterCall(id);
    }
    
    function _execute(address target, uint256 value, bytes calldata data) internal virtual {
        (bool success, bytes memory returndata) = target.call{value: value}(data);
        if (!success) {
            if (returndata.length > 0) {
                assembly { revert(add(returndata, 0x20), mload(returndata)) }
            } else {
                revert("TimelockController: execution reverted");
            }
        }
    }
    
    function _beforeCall(bytes32 id, bytes32 predecessor) private view {
        if (!isOperationReady(id)) {
            revert TimelockUnexpectedOperationState(id, _encodeStateBitmap(OperationState.Ready));
        }
        if (predecessor != bytes32(0) && !isOperationDone(predecessor)) {
            revert TimelockUnexecutedPredecessor(predecessor);
        }
    }
    
    function _afterCall(bytes32 id) private {
        if (!isOperationReady(id)) {
            revert TimelockUnexpectedOperationState(id, _encodeStateBitmap(OperationState.Ready));
        }
        _timestamps[id] = _DONE_TIMESTAMP;
    }
    
    modifier onlyRoleOrOpenRole(bytes32 role) {
        if (!hasRole(role, address(0))) {
            _checkRole(role, msg.sender);
        }
        _;
    }
    
    function updateDelay(uint256 newDelay) external virtual {
        address sender = msg.sender;
        if (sender != address(this)) {
            revert TimelockUnauthorizedCaller(sender);
        }
        emit MinDelayChange(_minDelay, newDelay);
        _minDelay = newDelay;
    }
    
    function _encodeStateBitmap(OperationState operationState) internal pure returns (bytes32) {
        return bytes32(1 << uint8(operationState));
    }
}
```

---

## 4. Workshop: Protocol Admin

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title ProtocolAdmin
 * @dev Admin contract with RBAC, Timelock, Emergency
 */
contract ProtocolAdmin is AccessControl, Ownable2Step {
    
    // Role definitions
    bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");
    bytes32 public constant UPGRADER_ROLE = keccak256("UPGRADER_ROLE");
    bytes32 public constant FEE_MANAGER_ROLE = keccak256("FEE_MANAGER_ROLE");
    bytes32 public constant TREASURY_ROLE = keccak256("TREASURY_ROLE");
    bytes32 public constant OPERATOR_ROLE = keccak256("OPERATOR_ROLE");
    
    // Protocol state
    bool public paused;
    uint256 public protocolFee = 30; // 0.3% in basis points / 100
    address public feeRecipient;
    address public treasury;
    
    // Timelock for sensitive operations
    uint256 public constant TIMELOCK_DELAY = 2 days;
    
    struct PendingChange {
        bytes32 changeType;
        bytes data;
        uint256 eta; // earliest execution time
        bool executed;
        bool cancelled;
    }
    
    mapping(bytes32 => PendingChange) public pendingChanges;
    
    event ProtocolPaused(address indexed by);
    event ProtocolUnpaused(address indexed by);
    event FeeUpdated(uint256 oldFee, uint256 newFee);
    event ChangeQueued(bytes32 indexed changeId, bytes32 changeType, uint256 eta);
    event ChangeExecuted(bytes32 indexed changeId);
    event ChangeCancelled(bytes32 indexed changeId);
    
    error ProtocolPausedError();
    error InvalidFee(uint256 fee);
    error ChangeNotReady(bytes32 changeId, uint256 eta);
    error ChangeAlreadyExecuted(bytes32 changeId);
    error ChangeCancelledError(bytes32 changeId);
    error NotQueued(bytes32 changeId);
    
    modifier whenNotPaused() {
        if (paused) revert ProtocolPausedError();
        _;
    }
    
    constructor(
        address admin,
        address _feeRecipient,
        address _treasury
    ) Ownable2Step(admin) {
        feeRecipient = _feeRecipient;
        treasury = _treasury;
        
        // Grant admin all roles
        _grantRole(DEFAULT_ADMIN_ROLE, admin);
        _grantRole(PAUSER_ROLE, admin);
        _grantRole(UPGRADER_ROLE, admin);
        _grantRole(FEE_MANAGER_ROLE, admin);
        _grantRole(TREASURY_ROLE, admin);
        _grantRole(OPERATOR_ROLE, admin);
    }
    
    // === Pause ===
    
    function pause() external onlyRole(PAUSER_ROLE) {
        paused = true;
        emit ProtocolPaused(msg.sender);
    }
    
    function unpause() external onlyRole(PAUSER_ROLE) {
        paused = false;
        emit ProtocolUnpaused(msg.sender);
    }
    
    // === Timelocked Operations ===
    
    function queueFeeChange(uint256 newFee) external onlyRole(FEE_MANAGER_ROLE) returns (bytes32 changeId) {
        if (newFee > 1000) revert InvalidFee(newFee); // max 10%
        
        changeId = keccak256(abi.encodePacked("FEE_CHANGE", newFee, block.timestamp));
        uint256 eta = block.timestamp + TIMELOCK_DELAY;
        
        pendingChanges[changeId] = PendingChange({
            changeType: keccak256("FEE_CHANGE"),
            data: abi.encode(newFee),
            eta: eta,
            executed: false,
            cancelled: false
        });
        
        emit ChangeQueued(changeId, keccak256("FEE_CHANGE"), eta);
    }
    
    function executeFeeChange(bytes32 changeId) external onlyRole(FEE_MANAGER_ROLE) {
        PendingChange storage change = pendingChanges[changeId];
        
        if (change.eta == 0) revert NotQueued(changeId);
        if (change.executed) revert ChangeAlreadyExecuted(changeId);
        if (change.cancelled) revert ChangeCancelledError(changeId);
        if (block.timestamp < change.eta) revert ChangeNotReady(changeId, change.eta);
        
        change.executed = true;
        
        uint256 newFee = abi.decode(change.data, (uint256));
        uint256 oldFee = protocolFee;
        protocolFee = newFee;
        
        emit FeeUpdated(oldFee, newFee);
        emit ChangeExecuted(changeId);
    }
    
    function cancelChange(bytes32 changeId) external onlyRole(DEFAULT_ADMIN_ROLE) {
        PendingChange storage change = pendingChanges[changeId];
        
        if (change.eta == 0) revert NotQueued(changeId);
        if (change.executed) revert ChangeAlreadyExecuted(changeId);
        
        change.cancelled = true;
        emit ChangeCancelled(changeId);
    }
    
    // === Emergency ===
    
    function emergencyWithdraw(address token, uint256 amount) external onlyRole(TREASURY_ROLE) {
        if (token == address(0)) {
            (bool sent,) = treasury.call{value: amount}("");
            require(sent, "ETH transfer failed");
        } else {
            IERC20(token).transfer(treasury, amount);
        }
    }
    
    receive() external payable {}
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
}
```

---

## 5. TypeScript Tests

```typescript
// test/ProtocolAdmin.test.ts
import { expect } from "chai";
import { ethers } from "hardhat";
import { ProtocolAdmin } from "../typechain-types";
import { Signer } from "ethers";
import { time } from "@nomicfoundation/hardhat-network-helpers";

describe("ProtocolAdmin", function () {
  let admin: ProtocolAdmin;
  let owner: Signer;
  let feeMgr: Signer;
  let pauser: Signer;
  let user: Signer;
  
  const PAUSER_ROLE = ethers.keccak256(ethers.toUtf8Bytes("PAUSER_ROLE"));
  const FEE_MANAGER_ROLE = ethers.keccak256(ethers.toUtf8Bytes("FEE_MANAGER_ROLE"));
  
  beforeEach(async function () {
    [owner, feeMgr, pauser, user] = await ethers.getSigners();
    
    const Admin = await ethers.getContractFactory("ProtocolAdmin");
    admin = await Admin.deploy(
      await owner.getAddress(),
      await owner.getAddress(),
      await owner.getAddress()
    );
    
    // Grant roles
    await admin.grantRole(PAUSER_ROLE, await pauser.getAddress());
    await admin.grantRole(FEE_MANAGER_ROLE, await feeMgr.getAddress());
  });
  
  describe("Roles", function () {
    it("should grant and check roles", async function () {
      expect(await admin.hasRole(PAUSER_ROLE, await pauser.getAddress())).to.be.true;
      expect(await admin.hasRole(FEE_MANAGER_ROLE, await feeMgr.getAddress())).to.be.true;
      expect(await admin.hasRole(PAUSER_ROLE, await user.getAddress())).to.be.false;
    });
    
    it("should only allow role holders to use role functions", async function () {
      await expect(admin.connect(user).pause())
        .to.be.revertedWithCustomError(admin, "AccessControlUnauthorizedAccount");
    });
  });
  
  describe("Pause", function () {
    it("should pause and unpause", async function () {
      await admin.connect(pauser).pause();
      expect(await admin.paused()).to.be.true;
      
      await admin.connect(pauser).unpause();
      expect(await admin.paused()).to.be.false;
    });
  });
  
  describe("Fee Change with Timelock", function () {
    it("should queue and execute fee change after delay", async function () {
      const newFee = 50n; // 0.5%
      
      // Queue
      const tx = await admin.connect(feeMgr).queueFeeChange(newFee);
      const receipt = await tx.wait();
      const event = receipt?.logs.find(
        log => admin.interface.parseLog(log as any)?.name === "ChangeQueued"
      );
      const parsed = admin.interface.parseLog(event as any);
      const changeId = parsed?.args.changeId;
      
      // Cannot execute before delay
      await expect(
        admin.connect(feeMgr).executeFeeChange(changeId)
      ).to.be.revertedWithCustomError(admin, "ChangeNotReady");
      
      // Advance time by 2 days
      await time.increase(2 * 24 * 3600 + 1);
      
      // Execute
      await admin.connect(feeMgr).executeFeeChange(changeId);
      
      expect(await admin.protocolFee()).to.equal(newFee);
    });
    
    it("should allow admin to cancel queued change", async function () {
      const tx = await admin.connect(feeMgr).queueFeeChange(50n);
      const receipt = await tx.wait();
      const event = receipt?.logs.find(
        log => admin.interface.parseLog(log as any)?.name === "ChangeQueued"
      );
      const parsed = admin.interface.parseLog(event as any);
      const changeId = parsed?.args.changeId;
      
      await admin.cancelChange(changeId);
      
      await time.increase(2 * 24 * 3600 + 1);
      
      await expect(
        admin.connect(feeMgr).executeFeeChange(changeId)
      ).to.be.revertedWithCustomError(admin, "ChangeCancelledError");
    });
  });
  
  describe("Ownership Transfer (2-step)", function () {
    it("should require acceptance for ownership transfer", async function () {
      const userAddr = await user.getAddress();
      
      // Start transfer
      await admin.transferOwnership(userAddr);
      expect(await admin.pendingOwner()).to.equal(userAddr);
      expect(await admin.owner()).to.equal(await owner.getAddress());
      
      // User accepts
      await admin.connect(user).acceptOwnership();
      expect(await admin.owner()).to.equal(userAddr);
    });
  });
});
```

---

## สรุป Part 14

Access Control และ Role Management ที่เรียนรู้:
- ✅ Ownable2Step (2-step ownership transfer)
- ✅ Role-Based Access Control (RBAC)
- ✅ TimelockController สำหรับ sensitive operations
- ✅ Multi-role Protocol Admin
- ✅ Emergency functions

## Quiz

1. ทำไม Ownable2Step ปลอดภัยกว่า Ownable?
2. DEFAULT_ADMIN_ROLE คือ `bytes32(0)` มีความหมายพิเศษอะไร?
3. Timelock ช่วยป้องกันอะไรใน DeFi?
4. การ `renounceRole` ต่างจาก `revokeRole` อย่างไร?

---

## Next: Part 15 - Security Patterns
