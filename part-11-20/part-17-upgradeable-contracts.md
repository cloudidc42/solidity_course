# Part 17: Upgradeable Contracts

## สารบัญ
1. Why Upgradeable?
2. Proxy Patterns Overview
3. Transparent Proxy Pattern
4. UUPS Pattern
5. Storage Collision
6. Initialize vs Constructor
7. OpenZeppelin Upgrades
8. Workshop: Upgradeable Token

---

## 1. Why Upgradeable?

```
ปัญหาของ Smart Contracts ปกติ:
- Deploy แล้วเปลี่ยน logic ไม่ได้
- Bug ต้องออก version ใหม่ (ผู้ใช้ต้อง migrate)
- Feature ใหม่ไม่สามารถเพิ่มได้

Proxy Pattern แก้ปัญหาด้วย:
1. แยก Storage (Proxy) กับ Logic (Implementation)
2. Proxy เก็บ state, forward calls ไปที่ Implementation
3. ต้องการ upgrade: deploy Implementation ใหม่, เปลี่ยน pointer
   ├── State ยังอยู่ใน Proxy (ไม่หาย)
   └── Logic เปลี่ยนเป็น version ใหม่

Tradeoffs:
✅ Bug fixes without user migration
✅ Feature additions
❌ Centralization risk (admin can upgrade anything)
❌ More complex, more attack surface
❌ Storage collision risk
```

---

## 2. delegatecall คืออะไร

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// delegatecall: เรียก function ของ contract อื่น
// แต่ใช้ storage และ context ของ caller

contract Storage {
    uint256 public value;
    
    function setValue(uint256 _value) external {
        value = _value;
    }
}

contract Proxy {
    uint256 public value; // ต้อง layout เหมือน Storage!
    address public implementation;
    
    constructor(address _impl) {
        implementation = _impl;
    }
    
    fallback() external payable {
        address impl = implementation;
        assembly {
            // Copy calldata to memory
            calldatacopy(0, 0, calldatasize())
            
            // delegatecall: use impl's code, proxy's storage
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            
            // Copy return data
            returndatacopy(0, 0, returndatasize())
            
            // Return or revert
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}

// ทดสอบ
// proxy.value() ≠ storage.value
// เพราะ delegatecall เปลี่ยน proxy.value ไม่ใช่ storage.value
```

---

## 3. Transparent Proxy Pattern

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Transparent Proxy:
 * - Admin: เรียก proxy admin functions (upgrade, etc.)
 * - Non-admin: calls forwarded ไป implementation
 * 
 * Storage layout (EIP-1967):
 * - Implementation slot: keccak256("eip1967.proxy.implementation") - 1
 * - Admin slot: keccak256("eip1967.proxy.admin") - 1
 * - Beacon slot: keccak256("eip1967.proxy.beacon") - 1
 */
contract TransparentUpgradeableProxy {
    
    // EIP-1967 storage slots
    bytes32 private constant IMPLEMENTATION_SLOT = 
        0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    bytes32 private constant ADMIN_SLOT = 
        0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103;
    
    event Upgraded(address indexed implementation);
    event AdminChanged(address previousAdmin, address newAdmin);
    
    constructor(address _implementation, address _admin, bytes memory _data) payable {
        _setImplementation(_implementation);
        _setAdmin(_admin);
        
        if (_data.length > 0) {
            (bool success,) = _implementation.delegatecall(_data);
            require(success, "Init failed");
        }
    }
    
    modifier ifAdmin() {
        if (msg.sender == _getAdmin()) {
            _;
        } else {
            _fallback();
        }
    }
    
    function admin() external ifAdmin returns (address) {
        return _getAdmin();
    }
    
    function implementation() external ifAdmin returns (address) {
        return _getImplementation();
    }
    
    function upgradeTo(address newImplementation) external ifAdmin {
        _setImplementation(newImplementation);
        emit Upgraded(newImplementation);
    }
    
    function upgradeToAndCall(address newImplementation, bytes memory data) external payable ifAdmin {
        _setImplementation(newImplementation);
        emit Upgraded(newImplementation);
        
        if (data.length > 0) {
            (bool success,) = newImplementation.delegatecall(data);
            require(success, "Upgrade call failed");
        }
    }
    
    function changeAdmin(address newAdmin) external ifAdmin {
        emit AdminChanged(_getAdmin(), newAdmin);
        _setAdmin(newAdmin);
    }
    
    fallback() external payable {
        _fallback();
    }
    
    receive() external payable {
        _fallback();
    }
    
    function _fallback() internal {
        _delegate(_getImplementation());
    }
    
    function _delegate(address impl) internal {
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
    
    function _getImplementation() internal view returns (address impl) {
        assembly { impl := sload(IMPLEMENTATION_SLOT) }
    }
    
    function _setImplementation(address newImpl) private {
        require(newImpl.code.length > 0, "Not a contract");
        assembly { sstore(IMPLEMENTATION_SLOT, newImpl) }
    }
    
    function _getAdmin() internal view returns (address adm) {
        assembly { adm := sload(ADMIN_SLOT) }
    }
    
    function _setAdmin(address newAdmin) private {
        assembly { sstore(ADMIN_SLOT, newAdmin) }
    }
}
```

---

## 4. UUPS Pattern (Universal Upgradeable Proxy Standard)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * UUPS: Upgrade logic อยู่ใน Implementation (ไม่ใช่ Proxy)
 * Pros: Proxy ถูกกว่า (simpler), Implementation control upgrade
 * Cons: ถ้า deploy impl ใหม่โดยไม่มี upgrade function = stuck!
 */

// ERC-1822 Proxiable
abstract contract UUPSUpgradeable {
    
    bytes32 private constant IMPLEMENTATION_SLOT = 
        0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    
    event Upgraded(address indexed implementation);
    
    modifier onlyProxy() {
        require(_isUUPS(), "Not called via proxy");
        _;
    }
    
    function proxiableUUID() external pure virtual returns (bytes32) {
        return IMPLEMENTATION_SLOT;
    }
    
    function upgradeTo(address newImplementation) external virtual;
    
    function upgradeToAndCall(address newImplementation, bytes memory data) external payable virtual {
        upgradeTo(newImplementation);
        if (data.length > 0) {
            (bool success,) = address(this).delegatecall(data);
            require(success, "Reinit failed");
        }
    }
    
    function _authorizeUpgrade(address newImplementation) internal virtual;
    
    function _upgradeToAndCallUUPS(address newImplementation, bytes memory data, bool forceCall) internal {
        // Check new impl is UUPS compatible
        try UUPSUpgradeable(newImplementation).proxiableUUID() returns (bytes32 slot) {
            require(slot == IMPLEMENTATION_SLOT, "Incompatible UUPS");
        } catch {
            revert("Bad UUPS");
        }
        
        _setImplementation(newImplementation);
        emit Upgraded(newImplementation);
        
        if (data.length > 0 || forceCall) {
            (bool success,) = newImplementation.delegatecall(data);
            if (!success && data.length > 0) revert("Reinit failed");
        }
    }
    
    function _setImplementation(address newImpl) private {
        assembly { sstore(IMPLEMENTATION_SLOT, newImpl) }
    }
    
    function _getImplementation() internal view returns (address impl) {
        assembly { impl := sload(IMPLEMENTATION_SLOT) }
    }
    
    function _isUUPS() private view returns (bool) {
        return _getImplementation() != address(0);
    }
}

// Simple UUPS Proxy
contract ERC1967Proxy {
    
    bytes32 private constant IMPLEMENTATION_SLOT = 
        0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    
    constructor(address implementation, bytes memory data) payable {
        assembly { sstore(IMPLEMENTATION_SLOT, implementation) }
        
        if (data.length > 0) {
            (bool success,) = implementation.delegatecall(data);
            require(success, "Init failed");
        }
    }
    
    fallback() external payable {
        assembly {
            let impl := sload(IMPLEMENTATION_SLOT)
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
    
    receive() external payable {}
}
```

---

## 5. Storage Collision Prevention

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// ❌ Storage Collision Example
contract ProxyBad {
    address public implementation; // slot 0
    address public admin;          // slot 1
}

contract TokenV1 {
    address public owner;    // slot 0 ← COLLISION! maps to implementation in proxy
    uint256 public supply;   // slot 1 ← COLLISION! maps to admin in proxy
}

// ✅ EIP-1967: Use pseudo-random storage slots
// These slots are extremely unlikely to collide with normal storage

contract EIP1967Storage {
    // Derived from keccak256("eip1967.proxy.implementation") - 1
    bytes32 internal constant IMPLEMENTATION_SLOT = 
        0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    
    // Derived from keccak256("eip1967.proxy.admin") - 1
    bytes32 internal constant ADMIN_SLOT = 
        0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103;
}

// ✅ Unstructured Storage: 
// Implementation starts at slot 0, Proxy uses slot defined by hash
// No collision possible!
```

---

## 6. Initialize vs Constructor

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// ✅ Initializable: prevents multiple initialization
abstract contract Initializable {
    
    uint8 private _initialized;
    bool private _initializing;
    
    event Initialized(uint8 version);
    
    modifier initializer() {
        bool isTopLevelCall = !_initializing;
        
        require(
            isTopLevelCall && _initialized < 1 ||
            !isTopLevelCall && _initialized == 1,
            "Already initialized"
        );
        
        bool isSetV1 = isTopLevelCall && _initialized < 1;
        if (isSetV1) {
            _initialized = 1;
            _initializing = true;
        }
        
        _;
        
        if (isSetV1) {
            _initializing = false;
            emit Initialized(1);
        }
    }
    
    modifier reinitializer(uint8 version) {
        require(
            !_initializing && _initialized < version,
            "Already initialized"
        );
        
        _initialized = version;
        _initializing = true;
        
        _;
        
        _initializing = false;
        emit Initialized(version);
    }
    
    function _disableInitializers() internal virtual {
        require(!_initializing, "Still initializing");
        if (_initialized != type(uint8).max) {
            _initialized = type(uint8).max;
            emit Initialized(type(uint8).max);
        }
    }
    
    function getInitializedVersion() external view returns (uint8) {
        return _initialized;
    }
    
    function isInitializing() external view returns (bool) {
        return _initializing;
    }
}

// Implementation V1
contract TokenV1 is Initializable, UUPSUpgradeable {
    
    string public name;
    string public symbol;
    uint256 public totalSupply;
    address public owner;
    
    mapping(address => uint256) public balanceOf;
    
    // ❌ No constructor! Use initialize instead
    // constructor() {} // This would be in implementation's code, not proxy's state
    
    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers(); // Prevent init on impl directly
    }
    
    // ✅ initializer modifier: runs only once
    function initialize(
        string memory _name,
        string memory _symbol,
        address _owner
    ) public initializer {
        name = _name;
        symbol = _symbol;
        owner = _owner;
        
        // Mint initial supply
        totalSupply = 1_000_000 * 1e18;
        balanceOf[_owner] = totalSupply;
    }
    
    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount, "Insufficient");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }
    
    // UUPS upgrade authorization
    function upgradeTo(address newImplementation) external override onlyProxy {
        require(msg.sender == owner, "Not owner");
        _authorizeUpgrade(newImplementation);
        _upgradeToAndCallUUPS(newImplementation, "", false);
    }
    
    function _authorizeUpgrade(address) internal override {
        require(msg.sender == owner, "Not owner");
    }
}

// Implementation V2: adds new features
contract TokenV2 is TokenV1 {
    
    // ✅ New state variable appended at end (safe)
    uint256 public mintFee;
    mapping(address => bool) public minters;
    
    // ❌ DO NOT change existing variable order!
    // string public name; // move this anywhere = STORAGE COLLISION
    
    // ✅ reinitializer: run V2-specific init
    function initializeV2(uint256 _mintFee) public reinitializer(2) {
        mintFee = _mintFee;
        minters[owner] = true;
    }
    
    // New function in V2
    function mint(address to, uint256 amount) external {
        require(minters[msg.sender], "Not minter");
        totalSupply += amount;
        balanceOf[to] += amount;
    }
    
    function addMinter(address minter) external {
        require(msg.sender == owner, "Not owner");
        minters[minter] = true;
    }
}
```

---

## 7. Workshop: Upgradeable DeFi Protocol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title UpgradeableLending (V1)
 * @dev UUPS upgradeable lending protocol
 */
contract LendingV1 is Initializable, UUPSUpgradeable {
    
    struct Market {
        address asset;
        uint256 totalDeposited;
        uint256 totalBorrowed;
        uint256 supplyRate;    // per year, basis points
        uint256 borrowRate;    // per year, basis points
        bool active;
    }
    
    mapping(address => Market) public markets;
    mapping(address => mapping(address => uint256)) public deposits; // user => asset => amount
    mapping(address => mapping(address => uint256)) public borrows;  // user => asset => amount
    
    address public admin;
    uint256 public protocolFee = 10; // 0.1%
    
    event MarketAdded(address indexed asset);
    event Deposited(address indexed user, address indexed asset, uint256 amount);
    event Borrowed(address indexed user, address indexed asset, uint256 amount);
    
    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() { _disableInitializers(); }
    
    function initialize(address _admin) public initializer {
        admin = _admin;
    }
    
    function addMarket(address asset, uint256 supplyRate, uint256 borrowRate) external {
        require(msg.sender == admin, "Not admin");
        markets[asset] = Market({
            asset: asset,
            totalDeposited: 0,
            totalBorrowed: 0,
            supplyRate: supplyRate,
            borrowRate: borrowRate,
            active: true
        });
        emit MarketAdded(asset);
    }
    
    function deposit(address asset, uint256 amount) external {
        require(markets[asset].active, "Market inactive");
        IERC20(asset).transferFrom(msg.sender, address(this), amount);
        deposits[msg.sender][asset] += amount;
        markets[asset].totalDeposited += amount;
        emit Deposited(msg.sender, asset, amount);
    }
    
    function borrow(address asset, uint256 amount) external {
        require(markets[asset].active, "Market inactive");
        require(markets[asset].totalDeposited - markets[asset].totalBorrowed >= amount, "Insufficient liquidity");
        borrows[msg.sender][asset] += amount;
        markets[asset].totalBorrowed += amount;
        IERC20(asset).transfer(msg.sender, amount);
        emit Borrowed(msg.sender, asset, amount);
    }
    
    function upgradeTo(address newImpl) external override onlyProxy {
        require(msg.sender == admin, "Not admin");
        _authorizeUpgrade(newImpl);
        _upgradeToAndCallUUPS(newImpl, "", false);
    }
    
    function _authorizeUpgrade(address) internal override {
        require(msg.sender == admin, "Not admin");
    }
    
    function version() external pure virtual returns (string memory) {
        return "1.0.0";
    }
}

// V2: adds liquidation
contract LendingV2 is LendingV1 {
    
    // New state: appended at end
    mapping(address => address) public priceOracles;
    uint256 public liquidationThreshold = 8000; // 80%
    
    event Liquidated(address indexed borrower, address indexed liquidator, uint256 amount);
    
    function initializeV2(uint256 _threshold) public reinitializer(2) {
        liquidationThreshold = _threshold;
    }
    
    function setOracle(address asset, address oracle) external {
        require(msg.sender == admin, "Not admin");
        priceOracles[asset] = oracle;
    }
    
    function liquidate(
        address borrower,
        address asset,
        uint256 amount
    ) external {
        // Check health factor
        uint256 healthFactor = _getHealthFactor(borrower, asset);
        require(healthFactor < liquidationThreshold, "Position healthy");
        
        // Repay debt
        IERC20(asset).transferFrom(msg.sender, address(this), amount);
        borrows[borrower][asset] -= amount;
        markets[asset].totalBorrowed -= amount;
        
        // Give collateral to liquidator (simplified)
        uint256 reward = (amount * 10500) / 10000; // 5% bonus
        deposits[borrower][asset] -= reward;
        deposits[msg.sender][asset] += reward;
        
        emit Liquidated(borrower, msg.sender, amount);
    }
    
    function _getHealthFactor(address user, address asset) internal view returns (uint256) {
        if (borrows[user][asset] == 0) return type(uint256).max;
        return (deposits[user][asset] * 10000) / borrows[user][asset];
    }
    
    function version() external pure override returns (string memory) {
        return "2.0.0";
    }
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
}
```

---

## Upgrade Safety Checklist

```
Before Upgrading:

[ ] Storage Layout
    - New variables appended ONLY (never insert/remove)
    - Never change variable types
    - Use gaps for reserved storage

[ ] Function Changes
    - Can add new functions
    - Can modify function body
    - Cannot remove events (breaks indexers)
    - Deprecated functions: mark but keep stub

[ ] Initialization
    - V1: use initializer modifier
    - V2+: use reinitializer(version) modifier
    - Never re-initialize same version

[ ] Testing
    - Run upgrade tests with hardhat-upgrades
    - Check storage layout compatibility
    - Test all existing functions still work

[ ] Multi-sig / Timelock
    - Upgrade via timelock (2-day delay)
    - Multi-sig for admin
    - Community vote for major upgrades

Tools:
- @openzeppelin/hardhat-upgrades
- OpenZeppelin Defender
```

---

## สรุป Part 17

Upgradeable Contracts ที่เรียนรู้:
- ✅ delegatecall mechanism
- ✅ Transparent Proxy Pattern
- ✅ UUPS Pattern
- ✅ EIP-1967 storage slots
- ✅ Storage collision prevention
- ✅ Initializable pattern
- ✅ Upgradeable DeFi protocol

## Quiz

1. Transparent Proxy กับ UUPS ต่างกันตรงไหน?
2. ทำไม constructor ใน upgradeable contract ถึงอันตราย?
3. Storage collision คืออะไร จะป้องกันได้อย่างไร?
4. EIP-1967 ใช้ทำอะไร?

---

## Next: Part 18 - DeFi Fundamentals
