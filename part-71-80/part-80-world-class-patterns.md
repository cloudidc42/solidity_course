# Part 80: World-Class Protocol Patterns (รูปแบบโปรโตคอลระดับโลก)

## บทนำ

Part สุดท้ายของ series นี้จะพาเราไปสู่ระดับ world-class protocol design — รูปแบบที่ใช้ใน Uniswap V4, Balancer V3 และ protocols ที่ leading ที่สุดในวงการ

เราจะเรียนรู้:
- **Singleton Vault Pattern**: PoolManager ที่ถือทุก pool ในเดียว (Balancer/Uniswap V4)
- **Transient Storage (EIP-1153)**: TSTORE/TLOAD สำหรับ within-transaction state
- **Hooks as Extension System**: IHooks interface ที่ทำให้ pool customizable ได้ไม่จำกัด
- **Flash Accounting**: track net balances และ enforce settlement ท้าย transaction
- **Protocol-Level Composability**: callback pattern สำหรับ atomic multi-step operations

---

## 1. Singleton Vault Pattern (Uniswap V4 / Balancer)

### 1.1 ปัญหาของ Per-Pool Architecture

**Uniswap V2/V3 (แบบเก่า):**
- แต่ละ pair มี contract แยก
- Liquidity กระจายใน contracts หลายร้อยอัน
- Multi-hop swap ต้องส่ง token ข้าม contracts หลายครั้ง (gas แพง)
- Flash loan ข้าม pools ทำได้ยาก

**Singleton Pattern (Uniswap V4):**
- Pool ทุกอันอยู่ใน `PoolManager` เดียว
- Token อยู่ใน vault เดียว
- Multi-hop เป็น internal accounting (ไม่ต้อง transfer token จริง)
- Flash accounting: net zero ท้าย transaction

### 1.2 PoolManager Architecture

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title PoolManager (Simplified Uniswap V4 style)
 * @dev Singleton contract ที่จัดการ pools ทั้งหมด
 *
 * Key Design Decisions:
 * 1. Pools เป็น structs ใน storage ของ contract เดียว
 * 2. Balances tracked ด้วย flash accounting (transient storage)
 * 3. Hooks เป็น extension points ที่ deploy แยก
 * 4. Currency ใช้แทน address (รองรับ native ETH)
 */

// Currency type: address(0) = ETH
type Currency is address;

library CurrencyLibrary {
    Currency public constant NATIVE = Currency.wrap(address(0));

    function isNative(Currency currency) internal pure returns (bool) {
        return Currency.unwrap(currency) == address(0);
    }

    function toId(Currency currency) internal pure returns (uint256) {
        return uint256(uint160(Currency.unwrap(currency)));
    }
}

// PoolKey identifies a unique pool
struct PoolKey {
    Currency currency0;    // token0 (smaller address)
    Currency currency1;    // token1 (larger address)
    uint24 fee;           // fee tier (100, 500, 3000, 10000)
    int24 tickSpacing;    // minimum tick spacing
    address hooks;        // hooks contract (address(0) = no hooks)
}

// Pool state stored in singleton
struct PoolState {
    uint160 sqrtPriceX96;  // current sqrt price
    int24 tick;             // current tick
    uint128 liquidity;      // active liquidity
    uint256 feeGrowthGlobal0X128; // accumulated fees token0
    uint256 feeGrowthGlobal1X128; // accumulated fees token1
    bool initialized;
}

// Balance delta from a swap/liquidity action
struct BalanceDelta {
    int128 amount0; // positive = owed to contract, negative = owed to user
    int128 amount1;
}

/**
 * @title IHooks
 * @dev Interface สำหรับ hooks contract
 * Pool Manager เรียก hooks ก่อน/หลัง swap และ liquidity operations
 */
interface IHooks {
    // Before/After hooks สำหรับ swap
    function beforeSwap(
        address sender,
        PoolKey calldata key,
        bool zeroForOne,
        int256 amountSpecified,
        bytes calldata hookData
    ) external returns (bytes4 selector, BeforeSwapDelta delta, uint24 lpFeeOverride);

    function afterSwap(
        address sender,
        PoolKey calldata key,
        bool zeroForOne,
        int256 amountSpecified,
        BalanceDelta delta,
        bytes calldata hookData
    ) external returns (bytes4 selector, int128 hookDeltaUnspecified);

    // Before/After hooks สำหรับ add liquidity
    function beforeAddLiquidity(
        address sender,
        PoolKey calldata key,
        AddLiquidityParams calldata params,
        bytes calldata hookData
    ) external returns (bytes4 selector);

    function afterAddLiquidity(
        address sender,
        PoolKey calldata key,
        AddLiquidityParams calldata params,
        BalanceDelta delta,
        BalanceDelta feesAccrued,
        bytes calldata hookData
    ) external returns (bytes4 selector, BalanceDelta hookDelta);

    // Remove liquidity hooks
    function beforeRemoveLiquidity(
        address sender,
        PoolKey calldata key,
        RemoveLiquidityParams calldata params,
        bytes calldata hookData
    ) external returns (bytes4 selector);

    function afterRemoveLiquidity(
        address sender,
        PoolKey calldata key,
        RemoveLiquidityParams calldata params,
        BalanceDelta delta,
        BalanceDelta feesAccrued,
        bytes calldata hookData
    ) external returns (bytes4 selector, BalanceDelta hookDelta);
}

struct BeforeSwapDelta {
    int128 deltaSpecified;
    int128 deltaUnspecified;
}

struct AddLiquidityParams {
    int24 tickLower;
    int24 tickUpper;
    int256 liquidityDelta;
    bytes32 salt;
}

struct RemoveLiquidityParams {
    int24 tickLower;
    int24 tickUpper;
    int256 liquidityDelta;
    bytes32 salt;
}

/**
 * @title PoolManager
 * @dev Singleton PoolManager แบบ simplified
 * ใช้ flash accounting สำหรับ efficient multi-hop swaps
 */
contract PoolManager {
    using CurrencyLibrary for Currency;

    // ==================== Constants ====================

    // Hook flags (encoded in hooks address bits)
    uint160 private constant BEFORE_SWAP_FLAG = 1 << 7;
    uint160 private constant AFTER_SWAP_FLAG = 1 << 6;
    uint160 private constant BEFORE_ADD_LIQUIDITY_FLAG = 1 << 5;
    uint160 private constant AFTER_ADD_LIQUIDITY_FLAG = 1 << 4;
    uint160 private constant BEFORE_REMOVE_LIQUIDITY_FLAG = 1 << 3;
    uint160 private constant AFTER_REMOVE_LIQUIDITY_FLAG = 1 << 2;

    // ==================== State ====================

    // Pool storage: poolId => PoolState
    mapping(bytes32 => PoolState) private pools;

    // Token balances held by PoolManager
    mapping(Currency => uint256) public reservesOf;

    // Flash accounting: net deltas during unlock (using transient storage)
    // Transient storage slots
    uint256 private constant UNLOCK_SLOT = uint256(keccak256("UNLOCK_SLOT")) - 1;
    uint256 private constant NON_ZERO_DELTA_COUNT_SLOT =
        uint256(keccak256("NON_ZERO_DELTA_COUNT_SLOT")) - 1;

    // ==================== Events ====================

    event Initialize(
        bytes32 indexed id,
        Currency indexed currency0,
        Currency indexed currency1,
        uint24 fee,
        int24 tickSpacing,
        address hooks,
        uint160 sqrtPriceX96,
        int24 tick
    );

    event Swap(
        bytes32 indexed id,
        address indexed sender,
        int128 amount0,
        int128 amount1,
        uint160 sqrtPriceX96,
        uint128 liquidity,
        int24 tick,
        uint24 fee
    );

    event ModifyLiquidity(
        bytes32 indexed id,
        address indexed sender,
        int24 tickLower,
        int24 tickUpper,
        int256 liquidityDelta,
        bytes32 salt
    );

    // ==================== Unlock Mechanism ====================

    /**
     * @dev Unlock pattern: ใช้ callback เพื่อ execute operations
     * ภายใน unlock callback เท่านั้น swap/liquidity operations ถูกอนุญาต
     *
     * ทุก operation ที่ทำจะสร้าง "delta" ใน transient storage
     * ท้าย callback ต้องมี net delta = 0 (settlement)
     */
    function unlock(bytes calldata data) external returns (bytes memory result) {
        // Mark as unlocked ใน transient storage
        assembly {
            tstore(UNLOCK_SLOT, 1)
        }

        // Execute callback
        result = IUnlockCallback(msg.sender).unlockCallback(data);

        // Verify ว่าทุก delta ถูก settle แล้ว
        uint256 nonZeroDeltaCount;
        assembly {
            nonZeroDeltaCount := tload(NON_ZERO_DELTA_COUNT_SLOT)
        }
        require(nonZeroDeltaCount == 0, "Unsettled deltas");

        // Reset unlock state
        assembly {
            tstore(UNLOCK_SLOT, 0)
        }
    }

    modifier onlyWhenUnlocked() {
        uint256 unlocked;
        assembly {
            unlocked := tload(UNLOCK_SLOT)
        }
        require(unlocked == 1, "Manager locked");
        _;
    }

    // ==================== Pool Initialization ====================

    /**
     * @dev Initialize pool ใหม่
     * PoolKey ถูก hash เป็น poolId
     */
    function initialize(
        PoolKey calldata key,
        uint160 sqrtPriceX96
    ) external returns (int24 tick) {
        bytes32 id = _poolId(key);
        require(!pools[id].initialized, "Pool already initialized");

        // Validate key
        require(Currency.unwrap(key.currency0) < Currency.unwrap(key.currency1), "Currency order");
        require(key.fee < 1_000_000, "Invalid fee"); // max 100%

        // Validate hooks address flags
        if (key.hooks != address(0)) {
            _validateHookAddress(key.hooks);
        }

        // Calculate tick from sqrtPrice
        tick = _getTickAtSqrtPrice(sqrtPriceX96);

        pools[id] = PoolState({
            sqrtPriceX96: sqrtPriceX96,
            tick: tick,
            liquidity: 0,
            feeGrowthGlobal0X128: 0,
            feeGrowthGlobal1X128: 0,
            initialized: true
        });

        emit Initialize(
            id,
            key.currency0,
            key.currency1,
            key.fee,
            key.tickSpacing,
            key.hooks,
            sqrtPriceX96,
            tick
        );
    }

    // ==================== Swap ====================

    /**
     * @dev Swap tokens ใน pool
     * ต้อง call ใน unlock callback
     * Flash accounting: delta ถูก track ใน transient storage
     *
     * @param key pool key
     * @param zeroForOne true = swap token0→token1
     * @param amountSpecified positive = exactIn, negative = exactOut
     */
    function swap(
        PoolKey calldata key,
        bool zeroForOne,
        int256 amountSpecified,
        uint160 sqrtPriceLimitX96,
        bytes calldata hookData
    ) external onlyWhenUnlocked returns (BalanceDelta delta) {
        bytes32 id = _poolId(key);
        PoolState storage pool = pools[id];
        require(pool.initialized, "Pool not initialized");

        // Call beforeSwap hook ถ้ามี
        if (key.hooks != address(0) && _hasHookFlag(key.hooks, BEFORE_SWAP_FLAG)) {
            (bytes4 selector, BeforeSwapDelta hookDelta, uint24 lpFeeOverride) =
                IHooks(key.hooks).beforeSwap(msg.sender, key, zeroForOne, amountSpecified, hookData);
            require(selector == IHooks.beforeSwap.selector, "Invalid hook return");
            // hookDelta และ lpFeeOverride ถูก apply ใน production
            hookDelta; // suppress warning
            lpFeeOverride;
        }

        // Execute swap (simplified)
        delta = _executeSwap(pool, key, zeroForOne, amountSpecified, sqrtPriceLimitX96);

        // Update flash accounting deltas ใน transient storage
        _accountDelta(key.currency0, delta.amount0);
        _accountDelta(key.currency1, delta.amount1);

        // Call afterSwap hook ถ้ามี
        if (key.hooks != address(0) && _hasHookFlag(key.hooks, AFTER_SWAP_FLAG)) {
            IHooks(key.hooks).afterSwap(msg.sender, key, zeroForOne, amountSpecified, delta, hookData);
        }

        emit Swap(
            id,
            msg.sender,
            delta.amount0,
            delta.amount1,
            pool.sqrtPriceX96,
            pool.liquidity,
            pool.tick,
            key.fee
        );
    }

    // ==================== Settlement ====================

    /**
     * @dev ชำระ positive delta (ส่ง token เข้า PoolManager)
     * เรียกหลังจาก swap/liquidity operations เพื่อ settle debt
     */
    function settle(Currency currency) external payable onlyWhenUnlocked returns (uint256 paid) {
        if (currency.isNative()) {
            paid = msg.value;
        } else {
            // ดึง token จาก caller
            uint256 balanceBefore = _balance(currency);
            // caller ต้อง transfer token ก่อน
            paid = _balance(currency) - balanceBefore;
        }

        reservesOf[currency] += paid;
        _accountDelta(currency, -int128(int256(paid)));
    }

    /**
     * @dev รับ negative delta (รับ token จาก PoolManager)
     */
    function take(
        Currency currency,
        address to,
        uint256 amount
    ) external onlyWhenUnlocked {
        _accountDelta(currency, int128(int256(amount)));
        reservesOf[currency] -= amount;
        _transfer(currency, to, amount);
    }

    // ==================== Flash Accounting Helpers ====================

    /**
     * @dev อัปเดต delta สำหรับ currency ใน transient storage
     * Transient storage จะถูก reset ท้าย transaction อัตโนมัติ
     */
    function _accountDelta(Currency currency, int128 delta) internal {
        if (delta == 0) return;

        uint256 slot = _currencyDeltaSlot(currency);
        int128 currentDelta;
        assembly {
            currentDelta := tload(slot)
        }

        int128 newDelta = currentDelta + delta;

        // Track จำนวน non-zero deltas
        if (currentDelta == 0 && newDelta != 0) {
            // เพิ่ม count
            assembly {
                let count := tload(NON_ZERO_DELTA_COUNT_SLOT)
                tstore(NON_ZERO_DELTA_COUNT_SLOT, add(count, 1))
            }
        } else if (currentDelta != 0 && newDelta == 0) {
            // ลด count
            assembly {
                let count := tload(NON_ZERO_DELTA_COUNT_SLOT)
                tstore(NON_ZERO_DELTA_COUNT_SLOT, sub(count, 1))
            }
        }

        assembly {
            tstore(slot, newDelta)
        }
    }

    function _currencyDeltaSlot(Currency currency) internal pure returns (uint256) {
        return uint256(keccak256(abi.encode("DELTA", Currency.unwrap(currency))));
    }

    /**
     * @dev อ่าน delta ปัจจุบันของ currency
     */
    function getCurrencyDelta(Currency currency) external view returns (int128 delta) {
        uint256 slot = _currencyDeltaSlot(currency);
        assembly {
            delta := tload(slot)
        }
    }

    // ==================== Internal Helpers ====================

    function _poolId(PoolKey calldata key) internal pure returns (bytes32) {
        return keccak256(abi.encode(key));
    }

    function _hasHookFlag(address hooks, uint160 flag) internal pure returns (bool) {
        return uint160(hooks) & flag != 0;
    }

    function _validateHookAddress(address hooks) internal pure {
        // Hook address ต้องมี flag bits ที่ถูกต้อง
        // ใน Uniswap V4 จริง มีการ validate ผ่าน CREATE2 deployment
        require(hooks != address(0), "Zero hooks address");
    }

    function _executeSwap(
        PoolState storage pool,
        PoolKey calldata key,
        bool zeroForOne,
        int256 amountSpecified,
        uint160 sqrtPriceLimitX96
    ) internal returns (BalanceDelta delta) {
        // Simplified swap logic
        // ใน production ใช้ TickMath และ SwapMath
        sqrtPriceLimitX96; // suppress warning

        if (amountSpecified > 0) {
            // Exact input
            if (zeroForOne) {
                delta = BalanceDelta({
                    amount0: int128(amountSpecified),
                    amount1: -int128(amountSpecified * 997 / 1000) // simplified fee
                });
            } else {
                delta = BalanceDelta({
                    amount0: -int128(amountSpecified * 997 / 1000),
                    amount1: int128(amountSpecified)
                });
            }
        }

        // Update fee growth (simplified)
        pool.feeGrowthGlobal0X128 += uint256(key.fee);
    }

    function _getTickAtSqrtPrice(uint160 sqrtPriceX96) internal pure returns (int24) {
        // Simplified - ใน production ใช้ TickMath
        sqrtPriceX96;
        return 0;
    }

    function _balance(Currency currency) internal view returns (uint256) {
        if (currency.isNative()) return address(this).balance;
        return IERC20(Currency.unwrap(currency)).balanceOf(address(this));
    }

    function _transfer(Currency currency, address to, uint256 amount) internal {
        if (currency.isNative()) {
            payable(to).transfer(amount);
        } else {
            IERC20(Currency.unwrap(currency)).transfer(to, amount);
        }
    }
}

interface IUnlockCallback {
    function unlockCallback(bytes calldata data) external returns (bytes memory);
}

interface IERC20 {
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
}
```

---

## 2. Transient Storage (EIP-1153)

### 2.1 ทำไม Transient Storage จึงสำคัญ

**ปัญหาเดิม:**
- Reentrancy lock ต้อง SSTORE (cold write = 20,000 gas) เมื่อ lock
- และ SSTORE อีกครั้งเมื่อ unlock (warm write = 2,900 gas)
- รวม ~22,900 gas แค่สำหรับ reentrancy protection

**EIP-1153 Solution:**
- TSTORE: 100 gas (vs SSTORE 20,000)
- TLOAD: 100 gas (vs SLOAD 100-2,100)
- ข้อมูลถูก reset ท้าย transaction อัตโนมัติ
- ไม่ต้อง cleanup แต่ก็ไม่ persist ข้าม transactions

### 2.2 Transient Storage ใน Solidity ^0.8.24

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title TransientStorageExamples
 * @dev ตัวอย่างการใช้ Transient Storage (EIP-1153)
 * Solidity ^0.8.24 รองรับ assembly: tstore/tload
 */

/**
 * @title TransientReentrancyGuard
 * @dev Reentrancy guard ที่ใช้ transient storage แทน persistent storage
 * ประหยัด gas ได้ ~22,900 gas ต่อ transaction (ที่มี lock/unlock)
 */
contract TransientReentrancyGuard {
    // Transient storage slot สำหรับ reentrancy lock
    uint256 private constant REENTRANCY_GUARD_SLOT =
        uint256(keccak256("REENTRANCY_GUARD")) - 1;

    uint256 private constant NOT_ENTERED = 1;
    uint256 private constant ENTERED = 2;

    error ReentrancyGuardReentrantCall();

    modifier nonReentrant() {
        _checkAndSetReentrancyGuard();
        _;
        _clearReentrancyGuard();
    }

    function _checkAndSetReentrancyGuard() private {
        uint256 status;
        assembly {
            status := tload(REENTRANCY_GUARD_SLOT)
        }

        if (status == ENTERED) {
            revert ReentrancyGuardReentrantCall();
        }

        assembly {
            tstore(REENTRANCY_GUARD_SLOT, ENTERED)
        }
    }

    function _clearReentrancyGuard() private {
        assembly {
            tstore(REENTRANCY_GUARD_SLOT, NOT_ENTERED)
        }
    }

    /**
     * @dev ตรวจสอบว่า currently inside a reentrant call
     */
    function _isReentrant() internal view returns (bool) {
        uint256 status;
        assembly {
            status := tload(REENTRANCY_GUARD_SLOT)
        }
        return status == ENTERED;
    }
}

/**
 * @title TransientAccumulator
 * @dev ใช้ transient storage เพื่อสะสมค่าระหว่าง transaction
 * ประโยชน์: multi-step operations ที่ต้อง track running total
 * โดยไม่ต้องเขียนลง storage ถาวรในแต่ละ step
 */
contract TransientAccumulator {
    // Slot สำหรับ accumulated value
    uint256 private constant ACCUMULATOR_SLOT =
        uint256(keccak256("ACCUMULATOR")) - 1;

    // Slot สำหรับนับจำนวน operations
    uint256 private constant OPERATION_COUNT_SLOT =
        uint256(keccak256("OPERATION_COUNT")) - 1;

    // Unlock mechanism
    uint256 private constant UNLOCK_SLOT =
        uint256(keccak256("UNLOCK")) - 1;

    event BatchCompleted(uint256 totalAccumulated, uint256 operationCount);

    /**
     * @dev เริ่ม batch operation
     * เปิด unlock และ reset accumulator
     */
    function startBatch() external {
        assembly {
            tstore(UNLOCK_SLOT, 1)
            tstore(ACCUMULATOR_SLOT, 0)
            tstore(OPERATION_COUNT_SLOT, 0)
        }
    }

    /**
     * @dev เพิ่มค่าเข้า accumulator (ใช้ transient storage)
     * Gas ถูกมากเพราะ TSTORE แทน SSTORE
     */
    function accumulate(uint256 value) external {
        uint256 unlocked;
        assembly {
            unlocked := tload(UNLOCK_SLOT)
        }
        require(unlocked == 1, "Not in batch");

        assembly {
            let current := tload(ACCUMULATOR_SLOT)
            tstore(ACCUMULATOR_SLOT, add(current, value))

            let count := tload(OPERATION_COUNT_SLOT)
            tstore(OPERATION_COUNT_SLOT, add(count, 1))
        }
    }

    /**
     * @dev อ่านค่าปัจจุบันของ accumulator
     */
    function getCurrentAccumulated() external view returns (uint256 total) {
        assembly {
            total := tload(ACCUMULATOR_SLOT)
        }
    }

    /**
     * @dev จบ batch และ finalize ผลลัพธ์
     * ค่าใน transient storage จะถูก reset ท้าย transaction อัตโนมัติ
     */
    function finalizeBatch() external returns (uint256 total, uint256 count) {
        uint256 unlocked;
        assembly {
            unlocked := tload(UNLOCK_SLOT)
        }
        require(unlocked == 1, "Not in batch");

        assembly {
            total := tload(ACCUMULATOR_SLOT)
            count := tload(OPERATION_COUNT_SLOT)
            tstore(UNLOCK_SLOT, 0)
        }

        emit BatchCompleted(total, count);
    }
}

/**
 * @title TransientBalanceTracker
 * @dev Track balance changes ใน flash accounting แบบ Uniswap V4
 * ใช้ transient storage สำหรับ within-transaction balance deltas
 */
contract TransientBalanceTracker {
    /**
     * @dev slot สำหรับ delta ของ token address
     */
    function _deltaSlot(address token) internal pure returns (uint256) {
        return uint256(keccak256(abi.encode("DELTA", token)));
    }

    /**
     * @dev เพิ่ม delta ให้ token (ใน transient storage)
     */
    function _addDelta(address token, int256 delta) internal {
        uint256 slot = _deltaSlot(token);
        assembly {
            let current := tload(slot)
            tstore(slot, add(current, delta))
        }
    }

    /**
     * @dev อ่าน delta ปัจจุบัน
     */
    function _getDelta(address token) internal view returns (int256 delta) {
        uint256 slot = _deltaSlot(token);
        assembly {
            delta := tload(slot)
        }
    }

    /**
     * @dev Reset delta (เรียกหลัง settle)
     */
    function _resetDelta(address token) internal {
        uint256 slot = _deltaSlot(token);
        assembly {
            tstore(slot, 0)
        }
    }
}
```

---

## 3. Hooks as Extension System

### 3.1 Hook Address Encoding

ใน Uniswap V4 hooks contract address ต้องมี specific bits set เพื่อบอกว่า hook ไหน active:

```
Address bits (lowest 14 bits):
Bit 13: BEFORE_INITIALIZE
Bit 12: AFTER_INITIALIZE
Bit 11: BEFORE_ADD_LIQUIDITY
Bit 10: AFTER_ADD_LIQUIDITY
Bit  9: BEFORE_REMOVE_LIQUIDITY
Bit  8: AFTER_REMOVE_LIQUIDITY
Bit  7: BEFORE_SWAP
Bit  6: AFTER_SWAP
Bit  5: BEFORE_DONATE
Bit  4: AFTER_DONATE
Bit  3: BEFORE_SWAP_RETURNS_DELTA
Bit  2: AFTER_SWAP_RETURNS_DELTA
Bit  1: AFTER_ADD_LIQUIDITY_RETURNS_DELTA
Bit  0: AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title DynamicFeeHook
 * @dev Hook ที่ implement dynamic fee based on volatility
 * Fee เพิ่มขึ้นเมื่อ price volatility สูง (เพื่อ protect LPs จาก IL)
 *
 * Hook flags ที่ต้องการ (ต้องอยู่ใน address bits):
 * - BEFORE_SWAP (bit 7)
 *
 * Deploy ด้วย CREATE2 เพื่อให้ address มี bits ถูกต้อง
 */
contract DynamicFeeHook {
    // ==================== Constants ====================

    // Fee boundaries
    uint24 public constant MIN_FEE = 100;   // 0.01%
    uint24 public constant MAX_FEE = 10000; // 1.00%
    uint24 public constant BASE_FEE = 3000; // 0.30%

    // Volatility thresholds (basis points per block)
    uint256 public constant LOW_VOLATILITY = 10;   // <0.1% per block
    uint256 public constant HIGH_VOLATILITY = 100; // >1% per block

    // ==================== State ====================

    address public immutable poolManager;

    // Price history for volatility calculation
    struct PriceSnapshot {
        uint160 sqrtPriceX96;
        uint256 timestamp;
        uint256 blockNumber;
    }

    // Pool price history (poolId => snapshots)
    mapping(bytes32 => PriceSnapshot[]) public priceHistory;
    mapping(bytes32 => uint256) public currentFee;

    // ==================== Events ====================

    event FeeUpdated(bytes32 indexed poolId, uint24 newFee, uint256 volatility);
    event PriceSnapshotRecorded(bytes32 indexed poolId, uint160 sqrtPrice);

    // ==================== Constructor ====================

    constructor(address _poolManager) {
        poolManager = _poolManager;
    }

    // ==================== Hook Implementation ====================

    /**
     * @dev beforeSwap: คำนวณและ override fee ตาม volatility
     * PoolManager จะ call function นี้ก่อนทุก swap
     */
    function beforeSwap(
        address sender,
        PoolKey calldata key,
        bool zeroForOne,
        int256 amountSpecified,
        bytes calldata hookData
    ) external returns (bytes4 selector, BeforeSwapDelta delta, uint24 lpFeeOverride) {
        require(msg.sender == poolManager, "Only pool manager");

        bytes32 poolId = keccak256(abi.encode(key));

        // คำนวณ volatility จาก price history
        uint256 volatility = _calculateVolatility(poolId);

        // กำหนด fee ตาม volatility
        uint24 dynamicFee = _calculateDynamicFee(volatility);
        currentFee[poolId] = dynamicFee;

        emit FeeUpdated(poolId, dynamicFee, volatility);

        // Suppress unused warnings
        sender;
        zeroForOne;
        amountSpecified;
        hookData;

        return (
            this.beforeSwap.selector,
            BeforeSwapDelta({deltaSpecified: 0, deltaUnspecified: 0}),
            dynamicFee // override the pool's base fee
        );
    }

    /**
     * @dev afterSwap: บันทึก price snapshot หลัง swap
     */
    function afterSwap(
        address sender,
        PoolKey calldata key,
        bool zeroForOne,
        int256 amountSpecified,
        BalanceDelta delta,
        bytes calldata hookData
    ) external returns (bytes4 selector, int128 hookDeltaUnspecified) {
        require(msg.sender == poolManager, "Only pool manager");

        bytes32 poolId = keccak256(abi.encode(key));

        // บันทึก price snapshot
        // ใน production ดึง sqrtPrice จาก pool state
        priceHistory[poolId].push(PriceSnapshot({
            sqrtPriceX96: 0, // placeholder
            timestamp: block.timestamp,
            blockNumber: block.number
        }));

        // เก็บแค่ 10 snapshots ล่าสุด
        if (priceHistory[poolId].length > 10) {
            _removeOldestSnapshot(poolId);
        }

        // Suppress unused warnings
        sender;
        zeroForOne;
        amountSpecified;
        delta;
        hookData;

        emit PriceSnapshotRecorded(poolId, 0);

        return (this.afterSwap.selector, 0);
    }

    // Stub implementations for other hooks
    function beforeAddLiquidity(address, PoolKey calldata, AddLiquidityParams calldata, bytes calldata)
        external pure returns (bytes4) { return this.beforeAddLiquidity.selector; }

    function afterAddLiquidity(address, PoolKey calldata, AddLiquidityParams calldata,
        BalanceDelta, BalanceDelta, bytes calldata)
        external pure returns (bytes4, BalanceDelta) {
            return (this.afterAddLiquidity.selector, BalanceDelta(0, 0));
        }

    function beforeRemoveLiquidity(address, PoolKey calldata, RemoveLiquidityParams calldata, bytes calldata)
        external pure returns (bytes4) { return this.beforeRemoveLiquidity.selector; }

    function afterRemoveLiquidity(address, PoolKey calldata, RemoveLiquidityParams calldata,
        BalanceDelta, BalanceDelta, bytes calldata)
        external pure returns (bytes4, BalanceDelta) {
            return (this.afterRemoveLiquidity.selector, BalanceDelta(0, 0));
        }

    // ==================== Volatility Calculation ====================

    /**
     * @dev คำนวณ realized volatility จาก price history
     * ใช้ log returns ระหว่าง snapshots ต่อๆ กัน
     */
    function _calculateVolatility(bytes32 poolId) internal view returns (uint256) {
        PriceSnapshot[] storage history = priceHistory[poolId];
        uint256 n = history.length;

        if (n < 2) return LOW_VOLATILITY; // ไม่มีข้อมูลพอ

        // คำนวณ average price change ระหว่าง snapshots
        uint256 totalChange = 0;
        for (uint256 i = 1; i < n; i++) {
            uint256 older = history[i-1].sqrtPriceX96;
            uint256 newer = history[i].sqrtPriceX96;

            if (older == 0) continue;

            uint256 change;
            if (newer > older) {
                change = (newer - older) * 10000 / older;
            } else {
                change = (older - newer) * 10000 / older;
            }
            totalChange += change;
        }

        return totalChange / (n - 1);
    }

    /**
     * @dev คำนวณ dynamic fee จาก volatility
     * Low vol → low fee (ดึงดูด volume)
     * High vol → high fee (ป้องกัน LP จาก IL)
     */
    function _calculateDynamicFee(uint256 volatility) internal pure returns (uint24) {
        if (volatility <= LOW_VOLATILITY) {
            return MIN_FEE;
        } else if (volatility >= HIGH_VOLATILITY) {
            return MAX_FEE;
        } else {
            // Linear interpolation
            uint256 range = HIGH_VOLATILITY - LOW_VOLATILITY;
            uint256 feeRange = MAX_FEE - MIN_FEE;
            return uint24(MIN_FEE + feeRange * (volatility - LOW_VOLATILITY) / range);
        }
    }

    function _removeOldestSnapshot(bytes32 poolId) internal {
        PriceSnapshot[] storage history = priceHistory[poolId];
        for (uint256 i = 0; i < history.length - 1; i++) {
            history[i] = history[i + 1];
        }
        history.pop();
    }
}
```

---

## 4. Flash Accounting

### 4.1 หลักการ Flash Accounting

Flash accounting เป็น pattern ที่ทำให้ multi-step operations เป็น atomic โดย:

1. ทุก operation สร้าง "delta" (ยอดที่ค้างชำระ)
2. Deltas ถูก track ใน transient storage
3. ท้าย transaction ทุก delta ต้องเป็น 0 (net settlement)
4. ถ้า delta ไม่ใช่ 0 → revert ทั้ง transaction

**ประโยชน์:**
- Multi-hop swap ไม่ต้อง transfer token จริงในแต่ละ step
- Flash loan ใน single pool manager
- Atomic arbitrage operations
- Gas savings มหาศาล

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title FlashAccountingRouter
 * @dev Router สำหรับ atomic multi-step operations ด้วย flash accounting
 * ใช้ PoolManager.unlock() callback pattern
 *
 * ตัวอย่าง: Arbitrage ETH/USDC/DAI
 * 1. Buy ETH ด้วย USDC (delta: +ETH, -USDC)
 * 2. Sell ETH เอา DAI (delta: -ETH, +DAI)
 * 3. Swap DAI→USDC (delta: -DAI, +USDC)
 * ท้าย transaction: ETH delta=0, USDC delta≥0 (profit), DAI delta=0
 * ชำระด้วยการ settle USDC ที่เหลือ
 */
contract FlashAccountingRouter is IUnlockCallback {
    PoolManager public immutable manager;

    struct SwapStep {
        PoolKey key;
        bool zeroForOne;
        int256 amountSpecified;
        uint160 sqrtPriceLimitX96;
        bytes hookData;
    }

    struct RouteParams {
        SwapStep[] steps;
        address tokenIn;
        address tokenOut;
        uint256 amountIn;
        uint256 minAmountOut;
        address recipient;
    }

    event RouteExecuted(
        address indexed trader,
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 amountOut
    );

    constructor(address _manager) {
        manager = PoolManager(_manager);
    }

    /**
     * @dev Execute multi-hop route ด้วย flash accounting
     * @param params route parameters
     * @return amountOut จำนวน token ที่ได้รับ
     */
    function executeRoute(RouteParams calldata params) external payable returns (uint256 amountOut) {
        // Encode params สำหรับ unlock callback
        bytes memory callData = abi.encode(params, msg.sender);

        // Unlock PoolManager และ execute ใน callback
        bytes memory result = manager.unlock(callData);

        amountOut = abi.decode(result, (uint256));

        require(amountOut >= params.minAmountOut, "Slippage exceeded");
    }

    /**
     * @dev Callback จาก PoolManager.unlock()
     * ทำ multi-step swap ทั้งหมดที่นี่
     * ท้าย callback ต้อง settle deltas ทั้งหมด
     */
    function unlockCallback(bytes calldata data) external override returns (bytes memory) {
        require(msg.sender == address(manager), "Only manager");

        (RouteParams memory params, address trader) = abi.decode(data, (RouteParams, address));

        // Execute แต่ละ swap step
        for (uint256 i = 0; i < params.steps.length; i++) {
            SwapStep memory step = params.steps[i];

            manager.swap(
                step.key,
                step.zeroForOne,
                step.amountSpecified,
                step.sqrtPriceLimitX96,
                step.hookData
            );
        }

        // ณ จุดนี้ PoolManager มี outstanding deltas สำหรับแต่ละ token
        // ต้อง settle ด้วยการ:
        // 1. ส่ง tokenIn เข้า manager (settle positive delta)
        // 2. รับ tokenOut จาก manager (take negative delta)

        Currency tokenIn = Currency.wrap(params.tokenIn);
        Currency tokenOut = Currency.wrap(params.tokenOut);

        // อ่าน deltas
        int128 deltaIn = manager.getCurrencyDelta(tokenIn);
        int128 deltaOut = manager.getCurrencyDelta(tokenOut);

        // Settle tokenIn (ส่ง token เข้า manager)
        if (deltaIn > 0) {
            // Manager ต้องการ token from us
            IERC20(params.tokenIn).transferFrom(trader, address(manager), uint256(uint128(deltaIn)));
            manager.settle(tokenIn);
        }

        // Take tokenOut (รับ token จาก manager)
        uint256 amountOut = 0;
        if (deltaOut < 0) {
            amountOut = uint256(uint128(-deltaOut));
            manager.take(tokenOut, params.recipient, amountOut);
        }

        emit RouteExecuted(trader, params.tokenIn, params.tokenOut, params.amountIn, amountOut);

        return abi.encode(amountOut);
    }
}

/**
 * @title FlashLoanViaAccounting
 * @dev Flash loan ผ่าน flash accounting ของ PoolManager
 * ยืม token โดยไม่ต้องมี dedicated flash loan contract
 */
contract FlashLoanViaAccounting is IUnlockCallback {
    PoolManager public immutable manager;

    struct FlashLoanParams {
        Currency currency;
        uint256 amount;
        address callbackContract;
        bytes callbackData;
    }

    constructor(address _manager) {
        manager = PoolManager(_manager);
    }

    function flashLoan(FlashLoanParams calldata params) external {
        bytes memory data = abi.encode(params, msg.sender);
        manager.unlock(data);
    }

    function unlockCallback(bytes calldata data) external override returns (bytes memory) {
        require(msg.sender == address(manager), "Only manager");

        (FlashLoanParams memory params, address initiator) = abi.decode(data, (FlashLoanParams, address));

        // 1. Take tokens (สร้าง negative delta)
        manager.take(params.currency, params.callbackContract, params.amount);

        // 2. Execute user's callback (user ต้องคืน token + fee)
        IFlashLoanCallback(params.callbackContract).onFlashLoan(
            initiator,
            params.currency,
            params.amount,
            params.callbackData
        );

        // 3. Settle (รับ token คืน)
        // User contract ต้อง approve และ transfer ก่อน
        manager.settle(params.currency);

        // ณ จุดนี้ delta ควรเป็น 0 (หรือ negative ถ้ามี profit ให้ manager)

        return "";
    }
}

interface IFlashLoanCallback {
    function onFlashLoan(
        address initiator,
        Currency currency,
        uint256 amount,
        bytes calldata data
    ) external;
}
```

---

## 5. Protocol-Level Composability

### 5.1 AggregatedOperation Pattern

Pattern นี้ช่วยให้ทำ complex operations หลาย steps ใน single transaction:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title AggregatedOperation
 * @dev Atomic multi-step operation ด้วย callback pattern
 *
 * ตัวอย่าง Use Cases:
 * 1. Zap: ETH → LP Token ในขั้นตอนเดียว
 * 2. Rebalance: ถอน LP → swap → ใส่ LP ใหม่
 * 3. Leverage: borrow → buy → deposit collateral
 * 4. Arbitrage: multi-hop cross-pool
 */
contract AggregatedOperation {

    // ==================== Types ====================

    enum OperationType {
        SWAP,           // swap tokens
        ADD_LIQUIDITY,  // add LP
        REMOVE_LIQUIDITY, // remove LP
        BORROW,         // borrow from lending
        REPAY,          // repay loan
        DEPOSIT,        // deposit to yield protocol
        WITHDRAW        // withdraw from yield protocol
    }

    struct Operation {
        OperationType opType;
        address protocol;      // protocol contract
        bytes params;          // encoded parameters
        bool critical;         // ถ้า true: revert ถ้า op ล้มเหลว
    }

    struct OperationResult {
        bool success;
        bytes returnData;
        uint256 gasUsed;
    }

    // ==================== State ====================

    // Trusted protocol adapters
    mapping(address => bool) public trustedProtocols;
    address public owner;

    // ==================== Events ====================

    event OperationExecuted(uint256 opIndex, OperationType opType, bool success);
    event BatchCompleted(uint256 totalOps, uint256 successOps);

    // ==================== Constructor ====================

    constructor() {
        owner = msg.sender;
    }

    // ==================== Admin ====================

    function addTrustedProtocol(address protocol) external {
        require(msg.sender == owner, "Only owner");
        trustedProtocols[protocol] = true;
    }

    // ==================== Core ====================

    /**
     * @dev Execute batch ของ operations แบบ atomic
     * ถ้า critical operation ใด fail → revert ทั้งหมด
     * ถ้า non-critical fail → continue
     */
    function executeBatch(Operation[] calldata operations)
        external
        payable
        returns (OperationResult[] memory results)
    {
        results = new OperationResult[](operations.length);
        uint256 successCount = 0;

        for (uint256 i = 0; i < operations.length; i++) {
            Operation calldata op = operations[i];

            // ตรวจสอบ trusted protocol
            require(trustedProtocols[op.protocol], "Untrusted protocol");

            uint256 gasBefore = gasleft();

            // Execute operation ผ่าน adapter
            (bool success, bytes memory returnData) = _executeOperation(op);

            results[i] = OperationResult({
                success: success,
                returnData: returnData,
                gasUsed: gasBefore - gasleft()
            });

            if (success) {
                successCount++;
            } else if (op.critical) {
                // Revert ทั้ง batch ถ้า critical operation ล้มเหลว
                revert(string(abi.encodePacked("Critical op failed at index: ", _uintToString(i))));
            }

            emit OperationExecuted(i, op.opType, success);
        }

        emit BatchCompleted(operations.length, successCount);
    }

    /**
     * @dev Execute single operation ผ่าน low-level call
     */
    function _executeOperation(Operation calldata op) internal returns (bool success, bytes memory data) {
        // Encode function call ตาม operation type
        bytes memory callData = _encodeOperation(op);

        (success, data) = op.protocol.call{value: 0}(callData);
    }

    function _encodeOperation(Operation calldata op) internal pure returns (bytes memory) {
        if (op.opType == OperationType.SWAP) {
            return abi.encodeWithSignature("swap(bytes)", op.params);
        } else if (op.opType == OperationType.ADD_LIQUIDITY) {
            return abi.encodeWithSignature("addLiquidity(bytes)", op.params);
        } else if (op.opType == OperationType.REMOVE_LIQUIDITY) {
            return abi.encodeWithSignature("removeLiquidity(bytes)", op.params);
        } else {
            return op.params;
        }
    }

    function _uintToString(uint256 v) internal pure returns (string memory) {
        if (v == 0) return "0";
        uint256 tmp = v;
        uint256 digits;
        while (tmp != 0) { digits++; tmp /= 10; }
        bytes memory buf = new bytes(digits);
        while (v != 0) { digits--; buf[digits] = bytes1(uint8(48 + v % 10)); v /= 10; }
        return string(buf);
    }
}

/**
 * @title ZapIntoLP
 * @dev Zap: ใส่ single token เข้า LP position ใน single transaction
 * ใช้ flash accounting ของ PoolManager
 *
 * Flow:
 * 1. รับ tokenA จาก user
 * 2. Swap ครึ่งหนึ่งเป็น tokenB
 * 3. Add liquidity ด้วย tokenA + tokenB
 * 4. ส่ง LP token กลับให้ user
 */
contract ZapIntoLP is IUnlockCallback {
    PoolManager public immutable manager;

    struct ZapParams {
        PoolKey key;         // pool ที่ต้องการ zap เข้า
        address tokenIn;     // token ที่ user มี
        uint256 amountIn;    // จำนวน
        address recipient;   // ผู้รับ LP token
        uint256 minLiquidity; // minimum LP amount (slippage protection)
    }

    event Zapped(address indexed user, uint256 amountIn, uint256 liquidityMinted);

    constructor(address _manager) {
        manager = PoolManager(_manager);
    }

    function zap(ZapParams calldata params) external returns (uint256 liquidity) {
        // Transfer tokenIn จาก user
        IERC20(params.tokenIn).transferFrom(msg.sender, address(this), params.amountIn);

        // Execute zap ใน unlock callback
        bytes memory result = manager.unlock(abi.encode(params, msg.sender));

        liquidity = abi.decode(result, (uint256));
        require(liquidity >= params.minLiquidity, "Insufficient liquidity");
    }

    function unlockCallback(bytes calldata data) external override returns (bytes memory) {
        require(msg.sender == address(manager), "Only manager");

        (ZapParams memory params, address user) = abi.decode(data, (ZapParams, address));

        // Step 1: Swap ครึ่งหนึ่งของ tokenIn เป็นอีก token
        uint256 halfAmount = params.amountIn / 2;
        Currency tokenIn = Currency.wrap(params.tokenIn);

        bool zeroForOne = params.tokenIn == Currency.unwrap(params.key.currency0);

        BalanceDelta swapDelta = manager.swap(
            params.key,
            zeroForOne,
            int256(halfAmount),
            0, // no price limit
            ""
        );

        // Step 2: เพิ่ม liquidity ด้วยทั้งสอง tokens
        // (simplified - ใน production ต้องคำนวณ optimal amounts)
        uint256 liquidity = _calculateLiquidity(swapDelta, halfAmount);

        // Step 3: Settle deltas
        // Settle tokenIn (ส่ง full amount เข้า manager)
        IERC20(params.tokenIn).approve(address(manager), params.amountIn);
        manager.settle(tokenIn);

        // Take tokenOut ที่ได้จาก swap
        Currency tokenOut = zeroForOne ? params.key.currency1 : params.key.currency0;
        uint256 tokenOutAmount = uint256(uint128(-swapDelta.amount1));
        manager.take(tokenOut, params.recipient, tokenOutAmount);

        // Suppress unused
        user;

        emit Zapped(user, params.amountIn, liquidity);

        return abi.encode(liquidity);
    }

    function _calculateLiquidity(BalanceDelta delta, uint256 amount0) internal pure returns (uint256) {
        // Simplified liquidity calculation
        uint256 amount1 = uint256(uint128(-delta.amount1));
        return _sqrt(amount0 * amount1);
    }

    function _sqrt(uint256 x) internal pure returns (uint256 y) {
        if (x == 0) return 0;
        uint256 z = (x + 1) / 2;
        y = x;
        while (z < y) { y = z; z = (x / z + z) / 2; }
    }
}
```

---

## Workshop: Deploy Singleton PoolManager พร้อม Dynamic Fee Hook

### เป้าหมาย
สร้างระบบ pool management ที่ครบถ้วน:
1. Deploy PoolManager (Singleton)
2. Deploy DynamicFeeHook
3. Initialize pool ETH/USDC พร้อม hook
4. Execute multi-hop swap ผ่าน FlashAccountingRouter

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title WorldClassPoolSystem
 * @dev ระบบ pool ที่รวม:
 * - Singleton PoolManager
 * - Dynamic Fee Hooks
 * - Flash Accounting
 * - Composable Operations
 *
 * Workshop steps:
 * 1. Deploy PoolManager
 * 2. Deploy DynamicFeeHook (ต้อง mine address ที่มี BEFORE_SWAP bit)
 * 3. Initialize ETH/USDC pool
 * 4. Add liquidity ผ่าน ZapIntoLP
 * 5. Swap ผ่าน FlashAccountingRouter
 */
contract WorldClassPoolSystem {
    // Pool configurations
    struct PoolConfig {
        Currency token0;
        Currency token1;
        uint24 baseFee;
        int24 tickSpacing;
        address hook;
        uint160 initialSqrtPrice;
    }

    PoolManager public poolManager;
    mapping(bytes32 => bool) public deployedPools;
    bytes32[] public poolIds;

    event SystemDeployed(address poolManager);
    event PoolDeployed(bytes32 indexed poolId, address hook);

    constructor() {
        poolManager = new PoolManager();
        emit SystemDeployed(address(poolManager));
    }

    /**
     * @dev Initialize pool ใหม่พร้อม configuration
     */
    function deployPool(PoolConfig calldata config) external returns (bytes32 poolId) {
        PoolKey memory key = PoolKey({
            currency0: config.token0,
            currency1: config.token1,
            fee: config.baseFee,
            tickSpacing: config.tickSpacing,
            hooks: config.hook
        });

        poolId = keccak256(abi.encode(key));
        require(!deployedPools[poolId], "Pool already exists");

        poolManager.initialize(key, config.initialSqrtPrice);

        deployedPools[poolId] = true;
        poolIds.push(poolId);

        emit PoolDeployed(poolId, config.hook);
    }

    /**
     * @dev คำนวณ initial sqrtPrice จากราคา
     * @param price token1 per token0 (scaled 1e18)
     * @return sqrtPriceX96 = sqrt(price) * 2^96
     */
    function priceToSqrtPriceX96(uint256 price) external pure returns (uint160) {
        // sqrt(price * 2^192) = sqrt(price) * 2^96
        uint256 sqrtPrice = _sqrt(price * (1 << 96));
        require(sqrtPrice <= type(uint160).max, "Price overflow");
        return uint160(sqrtPrice);
    }

    function _sqrt(uint256 x) internal pure returns (uint256 y) {
        if (x == 0) return 0;
        uint256 z = (x + 1) / 2;
        y = x;
        while (z < y) { y = z; z = (x / z + z) / 2; }
    }

    /**
     * @dev Helper สำหรับ mine hook address ที่มี specific flags
     * ใช้ CREATE2 เพื่อ deploy hook ที่ address ที่มี bits ถูกต้อง
     *
     * ใน production ใช้ script ที่ iterate salt จนพบ address ที่ต้องการ
     */
    function computeHookAddress(
        bytes32 salt,
        bytes32 bytecodeHash,
        address deployer,
        uint160 requiredFlags
    ) external pure returns (address hookAddress, bool isValid) {
        hookAddress = address(uint160(uint256(keccak256(abi.encodePacked(
            bytes1(0xff),
            deployer,
            salt,
            bytecodeHash
        )))));

        // ตรวจสอบว่า address มี flags ที่ต้องการ
        isValid = (uint160(hookAddress) & requiredFlags) == requiredFlags;
    }
}
```

---

## Workshop 2: Gas Comparison Test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title GasComparisonTest
 * @dev เปรียบเทียบ gas usage ระหว่าง patterns ต่างๆ
 * เพื่อ demonstrate ประโยชน์ของ transient storage และ flash accounting
 */
contract GasComparisonTest {

    // ==================== Traditional Reentrancy Guard ====================

    uint256 private _status = 1; // persistent storage

    modifier nonReentrantTraditional() {
        require(_status == 1, "Reentrant");
        _status = 2;
        _;
        _status = 1;
    }

    function functionWithTraditionalGuard() external nonReentrantTraditional {
        // ~22,900 gas overhead (SSTORE cold + SSTORE warm)
    }

    // ==================== Transient Reentrancy Guard ====================

    uint256 private constant T_GUARD_SLOT = uint256(keccak256("T_GUARD")) - 1;

    modifier nonReentrantTransient() {
        assembly {
            if tload(T_GUARD_SLOT) { revert(0, 0) }
            tstore(T_GUARD_SLOT, 1)
        }
        _;
        assembly {
            tstore(T_GUARD_SLOT, 0)
        }
    }

    function functionWithTransientGuard() external nonReentrantTransient {
        // ~200 gas overhead (TSTORE + TSTORE)
        // ประหยัด ~22,700 gas ต่อ call
    }

    // ==================== Multi-hop Swap Comparison ====================

    address[] public tokenPath;
    uint256 public hopCount;

    /**
     * @dev Simulate traditional multi-hop swap (V2 style)
     * ต้อง transfer token จริงในแต่ละ hop
     * Gas: O(n) transfers
     */
    function traditionalMultiHop(address[] calldata path, uint256 amountIn)
        external returns (uint256 amountOut) {
        uint256 currentAmount = amountIn;

        for (uint256 i = 0; i < path.length - 1; i++) {
            // แต่ละ hop ต้อง:
            // 1. SLOAD (read balance): 100 gas
            // 2. SSTORE (update balance): 5000-20000 gas
            // 3. Transfer token: 5000-30000 gas
            // รวม ~30,000-50,000 gas per hop
            currentAmount = currentAmount * 997 / 1000; // simplified swap
        }

        amountOut = currentAmount;
    }

    /**
     * @dev Simulate flash accounting multi-hop (V4 style)
     * Track deltas ใน transient storage เท่านั้น
     * Token transfer เกิดแค่ ต้น/ท้าย transaction
     * Gas: O(1) transfers + O(n) delta updates (cheap)
     */
    function flashAccountingMultiHop(address[] calldata path, uint256 amountIn)
        external returns (uint256 amountOut) {
        uint256 currentAmount = amountIn;

        for (uint256 i = 0; i < path.length - 1; i++) {
            // แต่ละ hop ต้องแค่:
            // 1. TLOAD + TSTORE: 200 gas total
            // 2. Internal accounting: ~100 gas
            // รวม ~300 gas per hop (vs 30,000-50,000!)
            uint256 slot = uint256(keccak256(abi.encode(path[i])));
            assembly {
                let currentDelta := tload(slot)
                tstore(slot, add(currentDelta, amountIn))
            }
            currentAmount = currentAmount * 997 / 1000;
        }

        // Settlement ครั้งเดียวตอนจบ (1 transfer)
        amountOut = currentAmount;
    }

    /**
     * @dev Gas savings estimation
     */
    function estimateGasSavings(uint256 hops) external pure returns (
        uint256 traditionalGas,
        uint256 flashAccountingGas,
        uint256 savings
    ) {
        uint256 gasPerHopTraditional = 40_000; // rough estimate
        uint256 gasPerHopFlash = 300;
        uint256 settlementGas = 50_000; // 2 token transfers

        traditionalGas = hops * gasPerHopTraditional;
        flashAccountingGas = hops * gasPerHopFlash + settlementGas;

        if (traditionalGas > flashAccountingGas) {
            savings = traditionalGas - flashAccountingGas;
        }
    }
}
```

---

## สรุป Part 80

- **Singleton Vault Pattern**: Pool ทั้งหมดอยู่ใน PoolManager เดียว; Token อยู่ใน vault เดียว; Multi-hop swap เป็น internal accounting ประหยัด gas มหาศาล; Pattern ใช้ใน Uniswap V4 และ Balancer V3
- **Transient Storage (EIP-1153)**: TSTORE/TLOAD ราคา 100 gas (vs SSTORE 20,000); ข้อมูลถูก reset ท้าย transaction อัตโนมัติ; เหมาะสำหรับ reentrancy locks, within-tx accumulators, และ flash accounting; ประหยัด ~22,700 gas ต่อ reentrancy check
- **Hooks as Extension System**: IHooks interface ให้ customization ไม่จำกัด; Hook flags encoded ใน address bits (ต้อง mine address ด้วย CREATE2); Dynamic fee hook ปรับ fee ตาม volatility แบบ real-time
- **Flash Accounting**: Track net balance deltas ใน transient storage; Settlement เดียวท้าย transaction; Enforce invariant: sum(deltas) = 0; ทำให้ atomic arbitrage และ flash loans ง่ายขึ้น
- **Protocol Composability**: Callback pattern (unlock/unlockCallback) สำหรับ atomic multi-step ops; ZapIntoLP ใช้ single transaction; AggregatedOperation สำหรับ complex strategies; Gas savings 10-100x สำหรับ multi-hop operations

## Next: Part 81 - Research Frontier: MEV & PBS
