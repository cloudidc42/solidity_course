# Part 51: Modular Protocol Architecture

## บทนำ

ในโลกของ DeFi สมัยใหม่ การออกแบบโปรโตคอลให้มีความ **Modular** และ **Composable** เป็นสิ่งสำคัญมาก แนวคิดนี้ช่วยให้เราสามารถ:

- เพิ่มฟีเจอร์ใหม่โดยไม่ต้อง deploy สัญญาใหม่ทั้งหมด
- ใช้ฟีเจอร์จากโปรโตคอลอื่นได้อย่างปลอดภัย
- สร้าง plugin ecosystem ที่ชุมชนสามารถมีส่วนร่วมได้
- ลดความเสี่ยงด้าน security โดยแยก concern ออกจากกัน

ในบทนี้เราจะเรียนรู้ pattern ที่ใช้จริงใน protocol ชั้นนำอย่าง Uniswap v4, Aave, Compound และ MakerDAO

---

## 1. Plugin/Module System

### ทฤษฎี: IModule Interface

แนวคิดพื้นฐานของ module system คือการกำหนด **interface มาตรฐาน** ที่ module ทุกตัวต้องปฏิบัติตาม ทำให้ core protocol ไม่จำเป็นต้องรู้รายละเอียดของแต่ละ module

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title IModule - Interface มาตรฐานสำหรับทุก module
/// @notice ทุก module ที่ต้องการ register ในระบบต้อง implement interface นี้
interface IModule {
    /// @notice ดึง identifier ของ module (ควรเป็น bytes32 ที่ unique)
    function moduleId() external view returns (bytes32);
    
    /// @notice ดึง version ของ module สำหรับ compatibility check
    function moduleVersion() external view returns (uint256);
    
    /// @notice ตรวจสอบว่า module รองรับ interface ที่กำหนดหรือไม่
    /// @param interfaceId bytes4 selector ของ interface
    function supportsInterface(bytes4 interfaceId) external view returns (bool);
    
    /// @notice เรียกเมื่อ module ถูก enable
    /// @param registry address ของ ModuleRegistry
    /// @param data ข้อมูล initialization เพิ่มเติม
    function onEnable(address registry, bytes calldata data) external;
    
    /// @notice เรียกเมื่อ module ถูก disable
    function onDisable() external;
}

/// @title IModuleRegistry - Interface สำหรับ registry ที่จัดการ modules
interface IModuleRegistry {
    event ModuleRegistered(bytes32 indexed moduleId, address indexed module, address indexed registrar);
    event ModuleEnabled(bytes32 indexed moduleId, address indexed enabledBy);
    event ModuleDisabled(bytes32 indexed moduleId, address indexed disabledBy);
    event ModuleUpgraded(bytes32 indexed moduleId, address indexed oldModule, address indexed newModule);
    
    /// @notice Register module ใหม่
    function registerModule(address module, bytes calldata initData) external;
    
    /// @notice Enable module ที่ register แล้ว
    function enableModule(bytes32 moduleId) external;
    
    /// @notice Disable module ที่ enable อยู่
    function disableModule(bytes32 moduleId) external;
    
    /// @notice ดึง address ของ module
    function getModule(bytes32 moduleId) external view returns (address);
    
    /// @notice ตรวจสอบว่า module enable อยู่หรือไม่
    function isModuleEnabled(bytes32 moduleId) external view returns (bool);
    
    /// @notice ดึงรายการ modules ทั้งหมด
    function getEnabledModules() external view returns (bytes32[] memory);
}
```

### ModuleRegistry Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "./IModule.sol";

/// @title ModuleRegistry - ระบบจัดการ modules สำหรับ protocol
/// @notice ทำหน้าที่เป็น central registry สำหรับ modules ทั้งหมด
/// @dev ใช้ pattern: owner-controlled registry with timelock for upgrades
contract ModuleRegistry is IModuleRegistry {
    // ============================================================
    //                          ERRORS
    // ============================================================
    error ModuleAlreadyRegistered(bytes32 moduleId);
    error ModuleNotRegistered(bytes32 moduleId);
    error ModuleAlreadyEnabled(bytes32 moduleId);
    error ModuleNotEnabled(bytes32 moduleId);
    error InvalidModule(address module);
    error Unauthorized(address caller);
    error TimelockNotExpired(uint256 unlockTime);
    error ZeroAddress();
    
    // ============================================================
    //                          STRUCTS
    // ============================================================
    struct ModuleInfo {
        address implementation;     // address ของ contract ที่ implement module
        bool enabled;               // สถานะว่า active อยู่หรือไม่
        uint256 enabledAt;          // timestamp ที่ enable
        uint256 disabledAt;         // timestamp ที่ disable (0 ถ้ายังไม่เคย)
        address registrar;          // ใครเป็นคน register
        bytes32 moduleId;           // unique identifier
    }
    
    struct PendingUpgrade {
        address newImplementation;
        uint256 scheduledAt;
        uint256 executeAfter;       // timelock: execute ได้หลังจาก timestamp นี้
    }
    
    // ============================================================
    //                          STORAGE
    // ============================================================
    
    /// @notice owner ของ registry มีสิทธิ์จัดการทุกอย่าง
    address public owner;
    
    /// @notice mapping จาก moduleId → ModuleInfo
    mapping(bytes32 => ModuleInfo) public modules;
    
    /// @notice รายการ moduleId ทั้งหมดที่ register แล้ว
    bytes32[] public allModuleIds;
    
    /// @notice รายการ moduleId ที่กำลัง enable อยู่
    bytes32[] private _enabledModuleIds;
    mapping(bytes32 => uint256) private _enabledIndex; // สำหรับ O(1) lookup
    
    /// @notice pending upgrades ที่รอ timelock
    mapping(bytes32 => PendingUpgrade) public pendingUpgrades;
    
    /// @notice ระยะเวลา timelock สำหรับ upgrade (เพื่อความปลอดภัย)
    uint256 public constant UPGRADE_TIMELOCK = 2 days;
    
    /// @notice mapping สำหรับ whitelist ของ registrars ที่ได้รับอนุญาต
    mapping(address => bool) public authorizedRegistrars;
    
    // ============================================================
    //                        CONSTRUCTOR
    // ============================================================
    
    constructor(address _owner) {
        if (_owner == address(0)) revert ZeroAddress();
        owner = _owner;
        authorizedRegistrars[_owner] = true;
    }
    
    // ============================================================
    //                         MODIFIERS
    // ============================================================
    
    modifier onlyOwner() {
        if (msg.sender != owner) revert Unauthorized(msg.sender);
        _;
    }
    
    modifier onlyAuthorized() {
        if (!authorizedRegistrars[msg.sender]) revert Unauthorized(msg.sender);
        _;
    }
    
    // ============================================================
    //                      ADMIN FUNCTIONS
    // ============================================================
    
    /// @notice เพิ่ม registrar ที่ได้รับอนุญาต
    function addAuthorizedRegistrar(address registrar) external onlyOwner {
        if (registrar == address(0)) revert ZeroAddress();
        authorizedRegistrars[registrar] = true;
    }
    
    /// @notice ลบ registrar
    function removeAuthorizedRegistrar(address registrar) external onlyOwner {
        authorizedRegistrars[registrar] = false;
    }
    
    // ============================================================
    //                    CORE MODULE FUNCTIONS
    // ============================================================
    
    /// @notice Register module ใหม่เข้าสู่ registry
    /// @param module address ของ module contract
    /// @param initData ข้อมูลสำหรับ initialize module
    function registerModule(address module, bytes calldata initData) external override onlyAuthorized {
        if (module == address(0)) revert ZeroAddress();
        
        // ตรวจสอบว่า module implement IModule interface
        try IModule(module).moduleId() returns (bytes32 moduleId) {
            if (modules[moduleId].implementation != address(0)) {
                revert ModuleAlreadyRegistered(moduleId);
            }
            
            // บันทึก module info
            modules[moduleId] = ModuleInfo({
                implementation: module,
                enabled: false,
                enabledAt: 0,
                disabledAt: 0,
                registrar: msg.sender,
                moduleId: moduleId
            });
            
            allModuleIds.push(moduleId);
            
            // เรียก onEnable ถ้าต้องการ auto-enable
            if (initData.length > 0) {
                IModule(module).onEnable(address(this), initData);
                _enableModule(moduleId);
            }
            
            emit ModuleRegistered(moduleId, module, msg.sender);
            
        } catch {
            revert InvalidModule(module);
        }
    }
    
    /// @notice Enable module ที่ register แล้ว
    function enableModule(bytes32 moduleId) external override onlyAuthorized {
        ModuleInfo storage info = modules[moduleId];
        if (info.implementation == address(0)) revert ModuleNotRegistered(moduleId);
        if (info.enabled) revert ModuleAlreadyEnabled(moduleId);
        
        // เรียก onEnable callback
        IModule(info.implementation).onEnable(address(this), "");
        _enableModule(moduleId);
        
        emit ModuleEnabled(moduleId, msg.sender);
    }
    
    /// @notice Disable module
    function disableModule(bytes32 moduleId) external override onlyAuthorized {
        ModuleInfo storage info = modules[moduleId];
        if (info.implementation == address(0)) revert ModuleNotRegistered(moduleId);
        if (!info.enabled) revert ModuleNotEnabled(moduleId);
        
        // เรียก onDisable callback
        IModule(info.implementation).onDisable();
        _disableModule(moduleId);
        
        emit ModuleDisabled(moduleId, msg.sender);
    }
    
    /// @notice Schedule upgrade สำหรับ module (ต้องรอ timelock)
    function scheduleUpgrade(bytes32 moduleId, address newImplementation) external onlyOwner {
        if (modules[moduleId].implementation == address(0)) revert ModuleNotRegistered(moduleId);
        if (newImplementation == address(0)) revert ZeroAddress();
        
        pendingUpgrades[moduleId] = PendingUpgrade({
            newImplementation: newImplementation,
            scheduledAt: block.timestamp,
            executeAfter: block.timestamp + UPGRADE_TIMELOCK
        });
    }
    
    /// @notice Execute upgrade หลังจาก timelock ผ่านไปแล้ว
    function executeUpgrade(bytes32 moduleId) external onlyOwner {
        PendingUpgrade memory upgrade = pendingUpgrades[moduleId];
        if (upgrade.newImplementation == address(0)) revert ModuleNotRegistered(moduleId);
        if (block.timestamp < upgrade.executeAfter) {
            revert TimelockNotExpired(upgrade.executeAfter);
        }
        
        address oldImplementation = modules[moduleId].implementation;
        modules[moduleId].implementation = upgrade.newImplementation;
        
        delete pendingUpgrades[moduleId];
        
        emit ModuleUpgraded(moduleId, oldImplementation, upgrade.newImplementation);
    }
    
    // ============================================================
    //                        VIEW FUNCTIONS
    // ============================================================
    
    function getModule(bytes32 moduleId) external view override returns (address) {
        return modules[moduleId].implementation;
    }
    
    function isModuleEnabled(bytes32 moduleId) external view override returns (bool) {
        return modules[moduleId].enabled;
    }
    
    function getEnabledModules() external view override returns (bytes32[] memory) {
        return _enabledModuleIds;
    }
    
    function getAllModules() external view returns (bytes32[] memory) {
        return allModuleIds;
    }
    
    // ============================================================
    //                      INTERNAL FUNCTIONS
    // ============================================================
    
    function _enableModule(bytes32 moduleId) internal {
        modules[moduleId].enabled = true;
        modules[moduleId].enabledAt = block.timestamp;
        _enabledIndex[moduleId] = _enabledModuleIds.length;
        _enabledModuleIds.push(moduleId);
    }
    
    function _disableModule(bytes32 moduleId) internal {
        modules[moduleId].enabled = false;
        modules[moduleId].disabledAt = block.timestamp;
        
        // ลบออกจาก enabled list อย่างมีประสิทธิภาพ (swap and pop)
        uint256 index = _enabledIndex[moduleId];
        uint256 lastIndex = _enabledModuleIds.length - 1;
        
        if (index != lastIndex) {
            bytes32 lastModuleId = _enabledModuleIds[lastIndex];
            _enabledModuleIds[index] = lastModuleId;
            _enabledIndex[lastModuleId] = index;
        }
        
        _enabledModuleIds.pop();
        delete _enabledIndex[moduleId];
    }
}
```

---

## 2. Hook Patterns (Uniswap v4 Style)

### ทฤษฎี: Hooks คืออะไร?

**Hooks** คือ callback functions ที่ถูกเรียกก่อน/หลังการดำเนินการสำคัญๆ ใน pool เช่น:
- ก่อน/หลัง swap
- ก่อน/หลังเพิ่ม/ถอน liquidity
- ก่อน/หลัง initialize pool

Uniswap v4 ใช้ **HookFlags bitmask** เพื่อบอก pool ว่า hook ต้องการรับ callback ใดบ้าง ทำให้ประหยัด gas โดยไม่ต้อง call hook ถ้าไม่ได้ implement

```
HookFlags:
Bit 0: BEFORE_INITIALIZE
Bit 1: AFTER_INITIALIZE  
Bit 2: BEFORE_ADD_LIQUIDITY
Bit 3: AFTER_ADD_LIQUIDITY
Bit 4: BEFORE_REMOVE_LIQUIDITY
Bit 5: AFTER_REMOVE_LIQUIDITY
Bit 6: BEFORE_SWAP
Bit 7: AFTER_SWAP
Bit 8: BEFORE_DONATE
Bit 9: AFTER_DONATE
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title IHooks - Interface สำหรับ Uniswap v4 style hooks
/// @notice Contract ที่ implement interface นี้สามารถ plug เข้า pool ได้
interface IHooks {
    // ============================================================
    //                    INITIALIZATION HOOKS
    // ============================================================
    
    function beforeInitialize(
        address sender,
        PoolKey calldata key,
        uint160 sqrtPriceX96
    ) external returns (bytes4);
    
    function afterInitialize(
        address sender,
        PoolKey calldata key,
        uint160 sqrtPriceX96,
        int24 tick
    ) external returns (bytes4);
    
    // ============================================================
    //                    LIQUIDITY HOOKS
    // ============================================================
    
    function beforeAddLiquidity(
        address sender,
        PoolKey calldata key,
        ModifyLiquidityParams calldata params,
        bytes calldata hookData
    ) external returns (bytes4);
    
    function afterAddLiquidity(
        address sender,
        PoolKey calldata key,
        ModifyLiquidityParams calldata params,
        BalanceDelta delta,
        BalanceDelta feesAccrued,
        bytes calldata hookData
    ) external returns (bytes4, BalanceDelta);
    
    function beforeRemoveLiquidity(
        address sender,
        PoolKey calldata key,
        ModifyLiquidityParams calldata params,
        bytes calldata hookData
    ) external returns (bytes4);
    
    function afterRemoveLiquidity(
        address sender,
        PoolKey calldata key,
        ModifyLiquidityParams calldata params,
        BalanceDelta delta,
        BalanceDelta feesAccrued,
        bytes calldata hookData
    ) external returns (bytes4, BalanceDelta);
    
    // ============================================================
    //                      SWAP HOOKS
    // ============================================================
    
    function beforeSwap(
        address sender,
        PoolKey calldata key,
        SwapParams calldata params,
        bytes calldata hookData
    ) external returns (bytes4, BeforeSwapDelta, uint24);
    
    function afterSwap(
        address sender,
        PoolKey calldata key,
        SwapParams calldata params,
        BalanceDelta delta,
        bytes calldata hookData
    ) external returns (bytes4, int128);
}

// ============================================================
//                    SUPPORTING STRUCTS
// ============================================================

struct PoolKey {
    address currency0;
    address currency1;
    uint24 fee;
    int24 tickSpacing;
    address hooks;
}

struct ModifyLiquidityParams {
    int24 tickLower;
    int24 tickUpper;
    int256 liquidityDelta;
    bytes32 salt;
}

struct SwapParams {
    bool zeroForOne;
    int256 amountSpecified;
    uint160 sqrtPriceLimitX96;
}

type BalanceDelta is int256;
type BeforeSwapDelta is int256;

/// @title HookFlags - Library สำหรับจัดการ permission bitmask
library HookFlags {
    // แต่ละ bit แทน permission หนึ่งชนิด
    uint160 internal constant BEFORE_INITIALIZE_FLAG   = 1 << 13;
    uint160 internal constant AFTER_INITIALIZE_FLAG    = 1 << 12;
    uint160 internal constant BEFORE_ADD_LIQUIDITY_FLAG = 1 << 11;
    uint160 internal constant AFTER_ADD_LIQUIDITY_FLAG  = 1 << 10;
    uint160 internal constant BEFORE_REMOVE_LIQUIDITY_FLAG = 1 << 9;
    uint160 internal constant AFTER_REMOVE_LIQUIDITY_FLAG  = 1 << 8;
    uint160 internal constant BEFORE_SWAP_FLAG         = 1 << 7;
    uint160 internal constant AFTER_SWAP_FLAG          = 1 << 6;
    uint160 internal constant BEFORE_DONATE_FLAG       = 1 << 5;
    uint160 internal constant AFTER_DONATE_FLAG        = 1 << 4;
    
    // Special flags
    uint160 internal constant AFTER_SWAP_RETURNS_DELTA_FLAG     = 1 << 3;
    uint160 internal constant AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG  = 1 << 2;
    uint160 internal constant AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG = 1 << 1;
    
    /// @notice ตรวจสอบว่า hook address มี flag ที่ต้องการหรือไม่
    /// Hook address ใน Uniswap v4 encode flags ไว้ใน lower bits ของ address
    function hasPermission(address hook, uint160 flag) internal pure returns (bool) {
        return uint160(hook) & flag != 0;
    }
    
    /// @notice ดึง flags ทั้งหมดจาก hook address
    function getFlags(address hook) internal pure returns (uint160) {
        return uint160(hook) & 0x3FFF; // lower 14 bits
    }
}
```

### Dynamic Fee Hook - ตัวอย่าง Hook จริง

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "./IHooks.sol";
import "./HookFlags.sol";

/// @title DynamicFeeHook - Hook ที่ปรับ fee แบบ dynamic ตามความผันผวน
/// @notice ใช้ volatility ของ price เพื่อปรับ fee ขึ้นลง
/// @dev Hook นี้ implement beforeSwap เพื่อ return dynamic fee
contract DynamicFeeHook is IHooks {
    using HookFlags for address;
    
    // ============================================================
    //                          ERRORS
    // ============================================================
    error OnlyPoolManager();
    error PoolNotInitialized();
    
    // ============================================================
    //                          STORAGE
    // ============================================================
    
    /// @notice address ของ PoolManager ที่ authorized
    address public immutable poolManager;
    
    /// @notice Base fee ในหน่วย pips (1 pip = 0.0001%)
    uint24 public constant BASE_FEE = 3000; // 0.3%
    
    /// @notice fee สูงสุดที่ hook สามารถเก็บได้
    uint24 public constant MAX_FEE = 10000; // 1%
    
    /// @notice fee ต่ำสุด
    uint24 public constant MIN_FEE = 500; // 0.05%
    
    struct PoolState {
        uint160 lastSqrtPrice;
        uint256 lastBlock;
        uint256 volatilityAccumulator;  // สะสม volatility ช่วง window
        uint24 currentFee;
        bool initialized;
    }
    
    mapping(bytes32 => PoolState) public poolStates;
    
    /// @notice จำนวน blocks ที่ใช้คำนวณ volatility window
    uint256 public constant VOLATILITY_WINDOW = 10;
    
    // ============================================================
    //                        CONSTRUCTOR
    // ============================================================
    
    constructor(address _poolManager) {
        poolManager = _poolManager;
    }
    
    // ============================================================
    //                          MODIFIERS
    // ============================================================
    
    modifier onlyPoolManager() {
        if (msg.sender != poolManager) revert OnlyPoolManager();
        _;
    }
    
    // ============================================================
    //                    HOOK IMPLEMENTATIONS
    // ============================================================
    
    /// @notice ถูกเรียกเมื่อ pool ถูก initialize
    function afterInitialize(
        address,
        PoolKey calldata key,
        uint160 sqrtPriceX96,
        int24
    ) external override onlyPoolManager returns (bytes4) {
        bytes32 poolId = _getPoolId(key);
        
        poolStates[poolId] = PoolState({
            lastSqrtPrice: sqrtPriceX96,
            lastBlock: block.number,
            volatilityAccumulator: 0,
            currentFee: BASE_FEE,
            initialized: true
        });
        
        return IHooks.afterInitialize.selector;
    }
    
    /// @notice ถูกเรียกก่อน swap - นี่คือจุดที่เราปรับ fee
    function beforeSwap(
        address,
        PoolKey calldata key,
        SwapParams calldata,
        bytes calldata
    ) external override onlyPoolManager returns (bytes4, BeforeSwapDelta, uint24) {
        bytes32 poolId = _getPoolId(key);
        PoolState storage state = poolStates[poolId];
        
        if (!state.initialized) revert PoolNotInitialized();
        
        // คำนวณ dynamic fee จาก volatility
        uint24 dynamicFee = _calculateDynamicFee(poolId);
        state.currentFee = dynamicFee;
        
        // Return: selector, delta (0 = no override), fee
        return (IHooks.beforeSwap.selector, BeforeSwapDelta.wrap(0), dynamicFee);
    }
    
    /// @notice ถูกเรียกหลัง swap - อัปเดต price state
    function afterSwap(
        address,
        PoolKey calldata key,
        SwapParams calldata,
        BalanceDelta,
        bytes calldata
    ) external override onlyPoolManager returns (bytes4, int128) {
        bytes32 poolId = _getPoolId(key);
        PoolState storage state = poolStates[poolId];
        
        // อัปเดต volatility tracker (simplified)
        // ใน production จะใช้ sqrtPriceX96 จริงๆ จาก slot0
        state.lastBlock = block.number;
        
        return (IHooks.afterSwap.selector, 0);
    }
    
    // Hook functions ที่ไม่ได้ implement (return selector เพื่อบอกว่า success)
    function beforeInitialize(address, PoolKey calldata, uint160) external pure override returns (bytes4) {
        return IHooks.beforeInitialize.selector;
    }
    
    function beforeAddLiquidity(address, PoolKey calldata, ModifyLiquidityParams calldata, bytes calldata) 
        external pure override returns (bytes4) {
        return IHooks.beforeAddLiquidity.selector;
    }
    
    function afterAddLiquidity(address, PoolKey calldata, ModifyLiquidityParams calldata, BalanceDelta, BalanceDelta, bytes calldata) 
        external pure override returns (bytes4, BalanceDelta) {
        return (IHooks.afterAddLiquidity.selector, BalanceDelta.wrap(0));
    }
    
    function beforeRemoveLiquidity(address, PoolKey calldata, ModifyLiquidityParams calldata, bytes calldata) 
        external pure override returns (bytes4) {
        return IHooks.beforeRemoveLiquidity.selector;
    }
    
    function afterRemoveLiquidity(address, PoolKey calldata, ModifyLiquidityParams calldata, BalanceDelta, BalanceDelta, bytes calldata) 
        external pure override returns (bytes4, BalanceDelta) {
        return (IHooks.afterRemoveLiquidity.selector, BalanceDelta.wrap(0));
    }
    
    // ============================================================
    //                      INTERNAL FUNCTIONS
    // ============================================================
    
    /// @notice คำนวณ fee จาก volatility
    /// @dev Algorithm: fee = BASE_FEE * (1 + volatilityMultiplier)
    function _calculateDynamicFee(bytes32 poolId) internal view returns (uint24) {
        PoolState storage state = poolStates[poolId];
        
        uint256 blocksDelta = block.number - state.lastBlock;
        
        // ถ้าเพิ่งมี swap ใน block ล่าสุด ความผันผวนสูง = fee สูง
        if (blocksDelta == 0) {
            // Multiple swaps in same block = high activity = higher fee
            return uint24(_clamp(BASE_FEE * 2, MIN_FEE, MAX_FEE));
        } else if (blocksDelta <= 2) {
            // Active block range
            return uint24(_clamp(BASE_FEE * 150 / 100, MIN_FEE, MAX_FEE));
        } else if (blocksDelta > VOLATILITY_WINDOW) {
            // Quiet market = lower fee to attract volume
            return uint24(_clamp(BASE_FEE * 80 / 100, MIN_FEE, MAX_FEE));
        }
        
        return BASE_FEE;
    }
    
    function _getPoolId(PoolKey calldata key) internal pure returns (bytes32) {
        return keccak256(abi.encode(key));
    }
    
    function _clamp(uint256 value, uint256 min, uint256 max) internal pure returns (uint256) {
        if (value < min) return min;
        if (value > max) return max;
        return value;
    }
}
```

---

## 3. Composable Contracts - การ Compose Protocol อย่างปลอดภัย

### ทฤษฎี: Re-entrancy ใน Composable Systems

เมื่อเรา call contracts ข้าม protocol ต้องระวัง:
1. **Re-entrancy attacks** - ต้องใช้ Checks-Effects-Interactions pattern
2. **Flash loan re-entrancy** - ต้องใช้ re-entrancy locks
3. **Cross-contract state inconsistency** - ต้องใช้ transient storage (EIP-1153)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title ComposableProtocol - ตัวอย่างการ compose protocols อย่างปลอดภัย
/// @notice แสดงให้เห็น pattern การ call external protocols + callback
contract ComposableProtocol {
    // ============================================================
    //                          ERRORS
    // ============================================================
    error ReentrantCall();
    error CallbackNotExpected();
    error InsufficientOutput(uint256 got, uint256 expected);
    error InvalidCaller(address caller, address expected);
    
    // ============================================================
    //                          STORAGE
    // ============================================================
    
    /// @notice Re-entrancy guard
    /// @dev ใช้ transient storage (EIP-1153) ใน Solidity 0.8.24+
    /// สำหรับ Cancun upgrade ที่มี TSTORE/TLOAD opcodes
    uint256 private _locked;
    uint256 private constant NOT_LOCKED = 1;
    uint256 private constant LOCKED = 2;
    
    /// @notice ติดตาม pending callbacks
    address private _pendingCallbackCaller;
    bytes32 private _pendingCallbackId;
    
    // ============================================================
    //                          MODIFIERS
    // ============================================================
    
    modifier nonReentrant() {
        if (_locked == LOCKED) revert ReentrantCall();
        _locked = LOCKED;
        _;
        _locked = NOT_LOCKED;
    }
    
    modifier onlyDuringCallback(bytes32 callbackId) {
        if (_pendingCallbackId != callbackId) revert CallbackNotExpected();
        if (_pendingCallbackCaller != msg.sender) {
            revert InvalidCaller(msg.sender, _pendingCallbackCaller);
        }
        _;
    }
    
    // ============================================================
    //                    COMPOSABLE OPERATIONS
    // ============================================================
    
    /// @notice Flash swap - ยืม tokens แล้ว callback กลับมาชำระ
    /// @param token0 token ที่ต้องการยืม
    /// @param amount จำนวนที่ยืม
    /// @param callbackData ข้อมูลที่ส่งกลับใน callback
    function flashSwap(
        address token0,
        uint256 amount,
        bytes calldata callbackData
    ) external nonReentrant {
        // 1. CHECKS
        // ...validation...
        
        // 2. EFFECTS - บันทึก state ก่อน external call
        _pendingCallbackCaller = address(this); // ในกรณีนี้ internal callback
        _pendingCallbackId = keccak256("FLASH_SWAP");
        
        // 3. INTERACTIONS - เรียก external contract
        // ส่ง tokens และคาดหวัง callback
        IFlashLender(token0).flashLoan(
            address(this),
            amount,
            callbackData
        );
        
        // หลัง callback เสร็จ ตรวจสอบว่าชำระครบแล้ว
        // (ในกรณีจริง จะตรวจสอบ balance ที่นี่)
    }
    
    /// @notice Callback จาก flash loan provider
    function onFlashLoan(
        address initiator,
        address token,
        uint256 amount,
        uint256 fee,
        bytes calldata data
    ) external returns (bytes32) {
        // ตรวจสอบว่า callback มาจาก source ที่ถูกต้อง
        if (_pendingCallbackId != keccak256("FLASH_SWAP")) {
            revert CallbackNotExpected();
        }
        
        // ดำเนินการกับ borrowed funds
        // (เช่น arbitrage, liquidation, etc.)
        _executeArbitrageLogic(token, amount, data);
        
        // อนุมัติ repayment
        uint256 repayAmount = amount + fee;
        IERC20(token).approve(msg.sender, repayAmount);
        
        return keccak256("ERC3156FlashBorrower.onFlashLoan");
    }
    
    /// @notice ดำเนินการ arbitrage (simplified)
    function _executeArbitrageLogic(
        address token,
        uint256 amount,
        bytes calldata data
    ) internal {
        // Decode arbitrage params
        (address targetDex, uint256 minOutput) = abi.decode(data, (address, uint256));
        
        // Swap บน target DEX
        uint256 output = IDex(targetDex).swap(token, amount);
        
        if (output < minOutput) {
            revert InsufficientOutput(output, minOutput);
        }
    }
}

// ============================================================
//                    CALLBACK INTERFACE PATTERN
// ============================================================

/// @title CallbackRouter - จัดการ callbacks จาก protocols ต่างๆ
/// @notice Pattern สำหรับ routing callbacks อย่างปลอดภัย
contract CallbackRouter {
    error UnknownCallback(bytes4 selector);
    error NotAuthorizedCallback(address caller);
    
    struct CallbackConfig {
        address authorizedCaller;   // contract เดียวที่ allowed
        bool active;
    }
    
    mapping(bytes4 => CallbackConfig) private _callbackConfigs;
    
    modifier validCallback(bytes4 selector) {
        CallbackConfig memory config = _callbackConfigs[selector];
        if (!config.active) revert UnknownCallback(selector);
        if (msg.sender != config.authorizedCaller) {
            revert NotAuthorizedCallback(msg.sender);
        }
        _;
    }
    
    /// @notice Register callback handler
    function _registerCallback(
        bytes4 selector,
        address authorizedCaller
    ) internal {
        _callbackConfigs[selector] = CallbackConfig({
            authorizedCaller: authorizedCaller,
            active: true
        });
    }
    
    /// @notice Deregister callback
    function _deregisterCallback(bytes4 selector) internal {
        delete _callbackConfigs[selector];
    }
}

// Minimal interfaces ที่ต้องการ
interface IFlashLender {
    function flashLoan(address receiver, uint256 amount, bytes calldata data) external;
}

interface IDex {
    function swap(address tokenIn, uint256 amountIn) external returns (uint256 amountOut);
}

interface IERC20 {
    function approve(address spender, uint256 amount) external returns (bool);
    function transfer(address to, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}
```

---

## 4. Protocol Integration Pattern

### ตัวอย่างสมบูรณ์: Multi-Protocol Yield Optimizer

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title MultiProtocolYieldOptimizer
/// @notice Automatically routes deposits to highest yielding protocol
/// @dev Demonstrates composable protocol interaction pattern
contract MultiProtocolYieldOptimizer is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;
    
    // ============================================================
    //                          ERRORS
    // ============================================================
    error ProtocolAlreadyAdded(bytes32 protocolId);
    error ProtocolNotFound(bytes32 protocolId);
    error InsufficientBalance(uint256 available, uint256 required);
    error WithdrawalFailed();
    error NoActiveProtocols();
    error SlippageTooHigh(uint256 actual, uint256 maximum);
    
    // ============================================================
    //                          EVENTS
    // ============================================================
    event DepositRouted(address indexed user, bytes32 indexed protocolId, uint256 amount);
    event YieldHarvested(bytes32 indexed protocolId, uint256 yieldAmount);
    event ProtocolAdded(bytes32 indexed protocolId, address indexed adapter);
    event ProtocolRemoved(bytes32 indexed protocolId);
    event Rebalanced(bytes32 indexed fromProtocol, bytes32 indexed toProtocol, uint256 amount);
    
    // ============================================================
    //                          STRUCTS
    // ============================================================
    
    struct ProtocolInfo {
        address adapter;        // Adapter contract ที่ wrap protocol นั้น
        bool active;
        uint256 totalDeposited; // ยอดรวมที่ deposit ใน protocol นี้
        uint256 lastApy;        // APY ล่าสุดที่ query ได้ (scaled by 1e18)
        uint256 lastApyUpdate;  // timestamp ที่อัปเดต APY ล่าสุด
    }
    
    struct UserPosition {
        mapping(bytes32 => uint256) shares;  // shares ใน protocol แต่ละตัว
        uint256 totalValue;
    }
    
    // ============================================================
    //                          STORAGE
    // ============================================================
    
    IERC20 public immutable asset;  // underlying asset (เช่น USDC)
    
    mapping(bytes32 => ProtocolInfo) public protocols;
    bytes32[] public protocolIds;
    
    mapping(address => UserPosition) private userPositions;
    
    uint256 public constant APY_PRECISION = 1e18;
    uint256 public constant MAX_PROTOCOLS = 10;
    
    // ============================================================
    //                        CONSTRUCTOR
    // ============================================================
    
    constructor(address _asset, address _owner) Ownable(_owner) {
        asset = IERC20(_asset);
    }
    
    // ============================================================
    //                      ADMIN FUNCTIONS
    // ============================================================
    
    /// @notice เพิ่ม protocol ใหม่
    function addProtocol(bytes32 protocolId, address adapter) external onlyOwner {
        if (protocols[protocolId].adapter != address(0)) {
            revert ProtocolAlreadyAdded(protocolId);
        }
        
        protocols[protocolId] = ProtocolInfo({
            adapter: adapter,
            active: true,
            totalDeposited: 0,
            lastApy: 0,
            lastApyUpdate: 0
        });
        
        protocolIds.push(protocolId);
        
        emit ProtocolAdded(protocolId, adapter);
    }
    
    /// @notice ปิดการใช้งาน protocol (ไม่รับ deposit ใหม่)
    function deactivateProtocol(bytes32 protocolId) external onlyOwner {
        if (protocols[protocolId].adapter == address(0)) {
            revert ProtocolNotFound(protocolId);
        }
        protocols[protocolId].active = false;
    }
    
    // ============================================================
    //                    CORE DEPOSIT LOGIC
    // ============================================================
    
    /// @notice Deposit และ route ไปยัง protocol ที่ให้ yield สูงสุด
    function deposit(uint256 amount, uint256 minShares) external nonReentrant {
        // 1. CHECKS
        if (amount == 0) return;
        
        // 2. EFFECTS - อัปเดต state ก่อน external call
        // (ที่นี่เราอัปเดต internal accounting)
        
        // 3. INTERACTIONS
        asset.safeTransferFrom(msg.sender, address(this), amount);
        
        // หา protocol ที่ดีที่สุด
        bytes32 bestProtocol = _findBestProtocol();
        if (bestProtocol == bytes32(0)) revert NoActiveProtocols();
        
        // Deposit ไป protocol ที่ดีที่สุด
        uint256 sharesReceived = _depositToProtocol(bestProtocol, amount);
        
        if (sharesReceived < minShares) {
            revert SlippageTooHigh(sharesReceived, minShares);
        }
        
        userPositions[msg.sender].shares[bestProtocol] += sharesReceived;
        protocols[bestProtocol].totalDeposited += amount;
        
        emit DepositRouted(msg.sender, bestProtocol, amount);
    }
    
    /// @notice ถอนเงินจาก protocol
    function withdraw(bytes32 protocolId, uint256 shares, uint256 minAmount) external nonReentrant {
        uint256 userShares = userPositions[msg.sender].shares[protocolId];
        if (userShares < shares) {
            revert InsufficientBalance(userShares, shares);
        }
        
        // EFFECTS ก่อน INTERACTION
        userPositions[msg.sender].shares[protocolId] -= shares;
        
        // INTERACTION
        uint256 amountReceived = _withdrawFromProtocol(protocolId, shares);
        
        if (amountReceived < minAmount) {
            revert SlippageTooHigh(amountReceived, minAmount);
        }
        
        protocols[protocolId].totalDeposited -= amountReceived;
        asset.safeTransfer(msg.sender, amountReceived);
    }
    
    /// @notice Rebalance: ย้ายเงินจาก protocol ที่ yield ต่ำ ไปยัง protocol ที่ yield สูง
    function rebalance(bytes32 fromProtocol, bytes32 toProtocol, uint256 amount) external onlyOwner {
        if (protocols[fromProtocol].adapter == address(0)) revert ProtocolNotFound(fromProtocol);
        if (protocols[toProtocol].adapter == address(0)) revert ProtocolNotFound(toProtocol);
        
        // ถอนจาก source
        uint256 sharesToWithdraw = _amountToShares(fromProtocol, amount);
        uint256 received = _withdrawFromProtocol(fromProtocol, sharesToWithdraw);
        
        // Deposit ไป destination
        _depositToProtocol(toProtocol, received);
        
        protocols[fromProtocol].totalDeposited -= amount;
        protocols[toProtocol].totalDeposited += received;
        
        emit Rebalanced(fromProtocol, toProtocol, received);
    }
    
    // ============================================================
    //                      INTERNAL FUNCTIONS
    // ============================================================
    
    /// @notice หา protocol ที่ให้ APY สูงสุด
    function _findBestProtocol() internal view returns (bytes32 bestId) {
        uint256 bestApy = 0;
        
        for (uint256 i = 0; i < protocolIds.length; i++) {
            bytes32 pid = protocolIds[i];
            if (!protocols[pid].active) continue;
            
            uint256 apy = IYieldProtocolAdapter(protocols[pid].adapter).getCurrentApy();
            if (apy > bestApy) {
                bestApy = apy;
                bestId = pid;
            }
        }
    }
    
    /// @notice Deposit ไปยัง protocol ผ่าน adapter
    function _depositToProtocol(bytes32 protocolId, uint256 amount) internal returns (uint256 shares) {
        address adapter = protocols[protocolId].adapter;
        asset.safeApprove(adapter, amount);
        shares = IYieldProtocolAdapter(adapter).deposit(amount);
    }
    
    /// @notice ถอนจาก protocol ผ่าน adapter
    function _withdrawFromProtocol(bytes32 protocolId, uint256 shares) internal returns (uint256 amount) {
        address adapter = protocols[protocolId].adapter;
        amount = IYieldProtocolAdapter(adapter).withdraw(shares);
    }
    
    function _amountToShares(bytes32 protocolId, uint256 amount) internal view returns (uint256) {
        return IYieldProtocolAdapter(protocols[protocolId].adapter).amountToShares(amount);
    }
    
    // ============================================================
    //                        VIEW FUNCTIONS
    // ============================================================
    
    function getUserShares(address user, bytes32 protocolId) external view returns (uint256) {
        return userPositions[user].shares[protocolId];
    }
    
    function getProtocolInfo(bytes32 protocolId) external view returns (ProtocolInfo memory) {
        return protocols[protocolId];
    }
}

/// @title IYieldProtocolAdapter - Interface มาตรฐานสำหรับ protocol adapters
interface IYieldProtocolAdapter {
    function deposit(uint256 amount) external returns (uint256 shares);
    function withdraw(uint256 shares) external returns (uint256 amount);
    function getCurrentApy() external view returns (uint256);
    function amountToShares(uint256 amount) external view returns (uint256);
    function getTotalValue() external view returns (uint256);
}

/// @title AaveAdapter - Adapter สำหรับ Aave protocol
contract AaveAdapter is IYieldProtocolAdapter {
    IERC20 public immutable underlying;
    IAavePool public immutable aavePool;
    address public immutable aToken;
    
    constructor(address _underlying, address _aavePool, address _aToken) {
        underlying = IERC20(_underlying);
        aavePool = IAavePool(_aavePool);
        aToken = _aToken;
    }
    
    function deposit(uint256 amount) external override returns (uint256 shares) {
        underlying.approve(address(aavePool), amount);
        aavePool.supply(address(underlying), amount, address(this), 0);
        return amount; // Aave aTokens are 1:1 with underlying
    }
    
    function withdraw(uint256 shares) external override returns (uint256 amount) {
        amount = aavePool.withdraw(address(underlying), shares, msg.sender);
    }
    
    function getCurrentApy() external view override returns (uint256) {
        // ดึง APY จาก Aave (แปลงจาก ray precision เป็น 1e18)
        uint256 liquidityRate = aavePool.getReserveData(address(underlying)).currentLiquidityRate;
        return liquidityRate / 1e9; // แปลงจาก 1e27 เป็น 1e18
    }
    
    function amountToShares(uint256 amount) external pure override returns (uint256) {
        return amount; // 1:1 ratio ใน Aave
    }
    
    function getTotalValue() external view override returns (uint256) {
        return IERC20(aToken).balanceOf(address(this));
    }
}

// Interfaces ที่ต้องการ
interface IAavePool {
    struct ReserveData {
        uint256 currentLiquidityRate;
        // ... fields อื่นๆ
    }
    
    function supply(address asset, uint256 amount, address onBehalfOf, uint16 referralCode) external;
    function withdraw(address asset, uint256 amount, address to) external returns (uint256);
    function getReserveData(address asset) external view returns (ReserveData memory);
}
```

---

## Workshop / แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง Custom Hook

**โจทย์**: สร้าง `LimitOrderHook` ที่:
1. เก็บ limit orders ไว้ใน storage
2. เมื่อ price ถึง target ใน `afterSwap`, execute order อัตโนมัติ
3. ผู้ใช้สามารถ cancel order ได้

```solidity
// แบบฝึกหัด: LimitOrderHook
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract LimitOrderHook {
    struct LimitOrder {
        address owner;
        bool zeroForOne;        // direction ของ swap
        uint256 inputAmount;
        uint256 minOutputAmount;
        int24 targetTick;       // price target ใน tick form
        bool executed;
        bool cancelled;
    }
    
    mapping(bytes32 => LimitOrder[]) public poolOrders;
    mapping(address => uint256[]) public userOrderIndices;
    
    event OrderPlaced(bytes32 indexed poolId, uint256 indexed orderId, address indexed owner);
    event OrderExecuted(bytes32 indexed poolId, uint256 indexed orderId, uint256 output);
    event OrderCancelled(bytes32 indexed poolId, uint256 indexed orderId);
    
    // TODO: Implement placeOrder()
    // TODO: Implement afterSwap() - check if any orders should execute
    // TODO: Implement cancelOrder()
    // TODO: Implement _executeOrder() internal
}
```

### แบบฝึกหัดที่ 2: Module ที่มี Dependencies

**โจทย์**: สร้าง `OracleModule` และ `PriceCapModule` ที่:
- `OracleModule` ดึงราคาจาก Chainlink
- `PriceCapModule` ขึ้นอยู่กับ `OracleModule` และใช้ price เพื่อ cap position size

```solidity
// แบบฝึกหัด: Module with Dependencies
contract OracleModule is IModule {
    bytes32 constant MODULE_ID = keccak256("ORACLE_V1");
    
    function moduleId() external pure override returns (bytes32) {
        return MODULE_ID;
    }
    
    // TODO: Implement getPrice(address token) returns (uint256 price, uint256 timestamp)
    // TODO: Use Chainlink AggregatorV3Interface
    // TODO: Handle stale prices (revert if > 1 hour old)
}

contract PriceCapModule is IModule {
    bytes32 constant MODULE_ID = keccak256("PRICE_CAP_V1");
    
    // TODO: ดึง OracleModule address จาก registry
    // TODO: Implement checkPositionSize(address token, uint256 amount) returns (bool valid)
    // TODO: Cap position ที่ $100,000 USD equivalent
}
```

### แบบฝึกหัดที่ 3: Composable Flash Arbitrage

**โจทย์**: สร้าง `FlashArbitrageBot` ที่:
1. กู้ USDC จาก Aave (flash loan)
2. Swap USDC → ETH บน DEX A
3. Swap ETH → USDC บน DEX B (ได้ราคาดีกว่า)
4. ชำระคืน Aave
5. เก็บกำไร

```solidity
// แบบฝึกหัด: Flash Arbitrage
contract FlashArbitrageBot is IFlashLoanReceiver {
    // TODO: Implement executeArbitrage(
    //           address tokenBorrow,
    //           uint256 amount,
    //           address dexA,
    //           address dexB
    //       )
    // TODO: Implement executeOperation() - Aave flash loan callback
    // TODO: ตรวจสอบ profitability ก่อน execute
    // TODO: ส่งกำไรกลับ msg.sender
}
```

---

## สรุป Part 51

- **IModule Interface** กำหนดมาตรฐานที่ทุก module ต้องปฏิบัติตาม ทำให้ระบบ composable ได้
- **ModuleRegistry** เป็น central hub สำหรับ register, enable, disable และ upgrade modules
- **Timelock** ใน upgrade ช่วยป้องกัน malicious upgrades โดยให้เวลา community ตรวจสอบ
- **HookFlags bitmask** ทำให้ Uniswap v4 style hooks มีประสิทธิภาพโดยไม่ต้อง call hooks ที่ไม่ได้ implement
- **Checks-Effects-Interactions** เป็น pattern สำคัญเมื่อ call external contracts
- **Adapter pattern** ช่วย normalize interfaces ของ protocols ต่างๆ ให้ใช้งานร่วมกันได้

## Next: Part 52 - Yield Aggregators (Yearn-style)
