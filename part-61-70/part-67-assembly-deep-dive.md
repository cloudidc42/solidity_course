# Part 67: Yul / Inline Assembly Deep Dive

## บทนำ

Yul เป็น intermediate language ของ Solidity ที่ให้เราเข้าถึง EVM ได้โดยตรง ทั้งนี้ทำให้เราสามารถ optimize gas ได้มากกว่า Solidity ปกติ แต่ก็ต้องระวังมากขึ้น บทนี้จะครอบคลุมตั้งแต่ Yul basics ไปจนถึงการเขียน ERC-20 ใน assembly เกือบทั้งหมด

---

## 1. Yul Basics

### 1.1 โครงสร้างพื้นฐาน

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract YulBasics {

    /// @notice ตัวแปรและ arithmetic ใน Yul
    function basicArithmetic(uint256 a, uint256 b) external pure returns (uint256 result) {
        assembly {
            // ประกาศตัวแปร
            let x := add(a, b)        // x = a + b
            let y := mul(a, b)        // y = a * b
            let z := div(x, 2)        // z = x / 2
            let w := mod(a, b)        // w = a % b
            let v := exp(a, 2)        // v = a^2
            
            // bitwise operations
            let andVal := and(a, b)   // a & b
            let orVal  := or(a, b)    // a | b
            let xorVal := xor(a, b)   // a ^ b
            let notVal := not(a)      // ~a
            let shlVal := shl(3, a)   // a << 3
            let shrVal := shr(3, a)   // a >> 3
            
            result := add(x, y)
        }
    }

    /// @notice Comparison operations
    function comparison(uint256 a, uint256 b) external pure returns (bool) {
        assembly {
            let eq_result  := eq(a, b)   // a == b
            let lt_result  := lt(a, b)   // a < b  (unsigned)
            let gt_result  := gt(a, b)   // a > b  (unsigned)
            let slt_result := slt(a, b)  // a < b  (signed)
            let sgt_result := sgt(a, b)  // a > b  (signed)
            let iszero_res := iszero(a)  // a == 0

            mstore(0x00, lt_result)
            return(0x00, 0x20)
        }
    }

    /// @notice Control flow: if-else
    function controlFlow(uint256 x) external pure returns (uint256 result) {
        assembly {
            // if statement (ไม่มี else)
            if gt(x, 100) {
                result := 1
            }

            // if-else ต้องใช้ switch
            switch gt(x, 50)
            case 0 {
                result := 10
            }
            default {
                result := 20
            }
        }
    }

    /// @notice Loop ใน Yul
    function loopSum(uint256 n) external pure returns (uint256 sum) {
        assembly {
            // for loop
            for { let i := 0 } lt(i, n) { i := add(i, 1) } {
                sum := add(sum, i)
            }
        }
    }

    /// @notice Functions ใน Yul
    function yulFunctions(uint256 a, uint256 b) external pure returns (uint256) {
        assembly {
            // ประกาศ function ใน Yul
            function max(x, y) -> result {
                result := x
                if gt(y, x) {
                    result := y
                }
            }

            function min(x, y) -> result {
                result := y
                if gt(y, x) {
                    result := x
                }
            }

            function abs_diff(x, y) -> result {
                switch gt(x, y)
                case 1 { result := sub(x, y) }
                default { result := sub(y, x) }
            }

            let maxVal := max(a, b)
            let minVal := min(a, b)
            mstore(0x00, add(maxVal, minVal))
            return(0x00, 0x20)
        }
    }
}
```

### 1.2 Memory Layout ใน EVM

```
Memory Layout:
┌──────────────────────────────────────────────┐
│  0x00 - 0x3f  │  Scratch space (64 bytes)    │
│               │  ใช้สำหรับ hashing           │
├───────────────┼──────────────────────────────┤
│  0x40 - 0x5f  │  Free Memory Pointer         │
│               │  mload(0x40) = next free slot │
├───────────────┼──────────────────────────────┤
│  0x60 - 0x7f  │  Zero slot                   │
│               │  ห้ามเขียน, ใช้เป็น initial  │
│               │  value ของ dynamic arrays    │
├───────────────┼──────────────────────────────┤
│  0x80+        │  Allocated memory             │
│               │  เริ่มต้นที่ 0x80            │
└───────────────┴──────────────────────────────┘
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract MemoryLayout {

    /// @notice อ่านและเขียน memory โดยตรง
    function memoryOperations() external pure returns (bytes32 hash) {
        assembly {
            // อ่าน free memory pointer
            let fmp := mload(0x40)

            // เขียนข้อมูลที่ position ปัจจุบัน
            mstore(fmp, 0xdeadbeef)         // เขียน 32 bytes
            mstore8(add(fmp, 32), 0xff)     // เขียน 1 byte

            // hash ข้อมูล
            hash := keccak256(fmp, 33)

            // update free memory pointer
            mstore(0x40, add(fmp, 64))
        }
    }

    /// @notice เปรียบเทียบ Solidity vs Assembly สำหรับ memory allocation
    function allocateMemorySolidity() external pure returns (bytes memory) {
        bytes memory data = new bytes(100); // Solidity allocate
        return data;
        // Solidity จะ:
        // 1. อ่าน free memory pointer (mload 0x40)
        // 2. เขียน length (mstore)
        // 3. zero out memory
        // 4. update free memory pointer
    }

    function allocateMemoryAssembly(uint256 size) external pure returns (bytes32 ptr) {
        assembly {
            // Manual memory allocation
            ptr := mload(0x40)           // อ่าน free memory pointer
            mstore(0x40, add(ptr, size)) // update free memory pointer
            // ไม่ต้อง zero out ถ้าเราจะเขียนทับทั้งหมด
        }
    }
}
```

---

## 2. Reading/Writing Storage Slots Directly

### 2.1 Storage Layout ของ Solidity

```
Mapping storage slot:
keccak256(abi.encode(key, baseSlot))

Array storage slot:
length: baseSlot
element[i]: keccak256(baseSlot) + i

Packed struct:
ตัวแปรหลาย ๆ ตัวที่รวมกันได้ใน 32 bytes
จะอยู่ใน slot เดียวกัน
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title StorageAssembly
 * @notice อ่านเขียน storage โดยตรงด้วย assembly
 */
contract StorageAssembly {
    // slot 0
    uint256 private value1;
    // slot 1
    uint256 private value2;
    // slot 2 (packed)
    uint128 private packedA; // bits 0-127
    uint128 private packedB; // bits 128-255
    // slot 3
    mapping(address => uint256) private balances;
    // slot 4
    uint256[] private dynamicArray;

    /// @notice อ่าน storage slot โดยตรง
    function readSlot(uint256 slotNumber) external view returns (bytes32 value) {
        assembly {
            value := sload(slotNumber)
        }
    }

    /// @notice เขียน storage slot โดยตรง
    function writeSlot(uint256 slotNumber, bytes32 value) external {
        assembly {
            sstore(slotNumber, value)
        }
    }

    /// @notice อ่าน mapping value โดยตรง
    function readMapping(address key) external view returns (uint256) {
        assembly {
            // คำนวณ slot สำหรับ balances[key]
            // slot = keccak256(abi.encode(key, 3)) เพราะ balances อยู่ที่ slot 3
            mstore(0x00, key)   // key
            mstore(0x20, 3)     // base slot
            let slot := keccak256(0x00, 0x40)
            mstore(0x00, sload(slot))
            return(0x00, 0x20)
        }
    }

    /// @notice เขียน mapping value โดยตรง
    function writeMapping(address key, uint256 val) external {
        assembly {
            mstore(0x00, key)
            mstore(0x20, 3)
            let slot := keccak256(0x00, 0x40)
            sstore(slot, val)
        }
    }

    /// @notice อ่าน packed values จาก slot เดียว
    function readPackedSlot() external view returns (uint128 a, uint128 b) {
        assembly {
            let packed := sload(2) // slot 2
            a := and(packed, 0xffffffffffffffffffffffffffffffff)         // lower 128 bits
            b := shr(128, packed)                                         // upper 128 bits
        }
    }

    /// @notice เขียน packed values ใน 1 SSTORE
    function writePackedSlot(uint128 a, uint128 b) external {
        assembly {
            // pack: b (upper 128 bits) | a (lower 128 bits)
            let packed := or(a, shl(128, b))
            sstore(2, packed)
        }
    }

    /// @notice อ่าน dynamic array element
    function readArrayElement(uint256 index) external view returns (uint256) {
        assembly {
            // base slot สำหรับ elements = keccak256(4)
            mstore(0x00, 4)
            let baseSlot := keccak256(0x00, 0x20)
            let elementSlot := add(baseSlot, index)
            mstore(0x00, sload(elementSlot))
            return(0x00, 0x20)
        }
    }

    function setValues(uint256 v1, uint256 v2, uint128 a, uint128 b) external {
        value1 = v1;
        value2 = v2;
        assembly {
            sstore(2, or(a, shl(128, b)))
        }
    }

    function pushToArray(uint256 val) external {
        dynamicArray.push(val);
    }
}
```

---

## 3. Efficient Calldata Parsing in Assembly

### 3.1 Calldata Layout

```
Calldata layout:
[0:4]    - function selector (keccak256(signature)[0:4])
[4:36]   - first parameter (32 bytes)
[36:68]  - second parameter (32 bytes)
...

Dynamic types (string, bytes, arrays):
[offset] - absolute offset from start of parameters
[at offset] - length
[at offset+32] - data...
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title CalldataParser
 * @notice Efficient calldata parsing ด้วย assembly
 */
contract CalldataParser {

    /// @notice อ่าน parameter แรกจาก calldata
    function getFirstParam() external pure returns (uint256 param) {
        assembly {
            // calldata layout:
            // [0:4]   selector
            // [4:36]  first param
            param := calldataload(4)
        }
    }

    /// @notice อ่านหลาย parameters พร้อมกัน
    function getMultipleParams()
        external
        pure
        returns (uint256 a, uint256 b, address c)
    {
        assembly {
            a := calldataload(4)
            b := calldataload(36)
            c := calldataload(68)  // address อยู่ใน 32 bytes แต่ 12 bytes แรกเป็น 0
        }
    }

    /// @notice Parse bytes calldata อย่างมีประสิทธิภาพ
    function parseBytesCalldata(bytes calldata data)
        external
        pure
        returns (bytes32 word1, bytes32 word2)
    {
        assembly {
            // data.offset = position ใน calldata ที่ข้อมูลเริ่มต้น
            word1 := calldataload(data.offset)
            word2 := calldataload(add(data.offset, 32))
        }
    }

    /// @notice Multicall ที่ parse calldata เอง
    function multicall(bytes[] calldata calls) external returns (bytes[] memory results) {
        results = new bytes[](calls.length);
        for (uint256 i = 0; i < calls.length; i++) {
            (bool success, bytes memory result) = address(this).delegatecall(calls[i]);
            require(success, "Call failed");
            results[i] = result;
        }
    }

    /// @notice Ultra-efficient batch transfer parser
    /// Format: [address(20)][amount(12)] packed ใน 32 bytes ต่อ recipient
    function batchTransferPacked(bytes calldata packed) external {
        assembly {
            let len := packed.length
            let offset := packed.offset

            for { let i := 0 } lt(i, len) { i := add(i, 32) } {
                let word := calldataload(add(offset, i))
                let recipient := shr(96, word)           // upper 160 bits = address
                let amount := and(word, 0xffffffffffffffffffffffffffff) // lower 96 bits
                // ทำ transfer...
                // (simplified: แค่แสดงการ parse)
                pop(recipient)
                pop(amount)
            }
        }
    }

    /// @notice คำนวณ calldata cost
    function calldataCostEstimate(bytes calldata data) external pure returns (uint256 cost) {
        assembly {
            let len := data.length
            let offset := data.offset

            for { let i := 0 } lt(i, len) { i := add(i, 1) } {
                let byte_ := byte(0, calldataload(add(offset, i)))
                switch iszero(byte_)
                case 1 { cost := add(cost, 4) }   // zero byte: 4 gas
                default { cost := add(cost, 16) }  // non-zero byte: 16 gas
            }
        }
    }
}
```

---

## 4. Custom Memory Allocator in Yul

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title YulMemoryAllocator
 * @notice Custom memory allocator ที่ efficient กว่า Solidity default
 *
 * ปัญหาของ Solidity default allocator:
 * - Zero out memory ทุกครั้ง (เพื่อ safety)
 * - ไม่มี deallocation
 * - Overhead จาก bounds checking
 *
 * Custom allocator:
 * - ไม่ zero out ถ้าไม่จำเป็น (fast path)
 * - Arena allocator สำหรับ batch operations
 * - Stack-like allocator สำหรับ temporary data
 */
contract YulMemoryAllocator {

    /// @notice Arena allocator: จอง memory block ใหญ่ไว้ก่อน
    function arenaAllocator(uint256 numItems) external pure returns (uint256 total) {
        assembly {
            // จอง arena ใหญ่ไว้ก่อน
            let arenaSize := mul(numItems, 32)
            let arena := mload(0x40)
            mstore(0x40, add(arena, arenaSize))

            // เขียนข้อมูลใน arena โดยไม่ต้อง update free pointer ทุกครั้ง
            for { let i := 0 } lt(i, numItems) { i := add(i, 1) } {
                mstore(add(arena, mul(i, 32)), mul(i, i)) // i^2
            }

            // คำนวณผลรวม
            for { let i := 0 } lt(i, numItems) { i := add(i, 1) } {
                total := add(total, mload(add(arena, mul(i, 32))))
            }
        }
    }

    /// @notice Stack allocator สำหรับ temporary computation
    function stackAllocator(
        uint256[] calldata a,
        uint256[] calldata b
    ) external pure returns (uint256[] memory result) {
        uint256 len = a.length;
        require(len == b.length, "Length mismatch");

        result = new uint256[](len);

        assembly {
            // ใช้ scratch space สำหรับ temporary values เล็ก ๆ
            let scratch := 0x00

            let resultOffset := add(result, 32) // skip length prefix

            for { let i := 0 } lt(i, len) { i := add(i, 1) } {
                let ai := calldataload(add(a.offset, mul(i, 32)))
                let bi := calldataload(add(b.offset, mul(i, 32)))

                // ใช้ scratch space สำหรับ intermediate result
                mstore(scratch, add(ai, bi))
                let sum := mload(scratch)

                mstore(add(resultOffset, mul(i, 32)), sum)
            }
        }
    }

    /// @notice Return data โดยตรงจาก memory (bypass ABI encoding)
    function rawReturn(uint256[] calldata data) external pure {
        assembly {
            // คัดลอก calldata ไปยัง memory
            let len := data.length
            let totalBytes := add(mul(len, 32), 32) // length + data
            let ptr := mload(0x40)

            // เขียน length
            mstore(ptr, len)

            // คัดลอก data
            calldatacopy(add(ptr, 32), data.offset, mul(len, 32))

            // Return โดยตรง
            return(ptr, totalBytes)
        }
    }

    /// @notice Packed struct allocator
    struct PackedData {
        uint128 value1;
        uint128 value2;
    }

    function packedAlloc(uint256 n) external pure returns (bytes memory packed) {
        assembly {
            // allocate: n * 32 bytes + 32 bytes length prefix
            let totalSize := add(mul(n, 32), 32)
            packed := mload(0x40)
            mstore(0x40, add(packed, totalSize))
            mstore(packed, n) // length

            let dataStart := add(packed, 32)

            for { let i := 0 } lt(i, n) { i := add(i, 1) } {
                // pack สอง uint128 ใน 1 word
                let v1 := add(mul(i, 2), 1)
                let v2 := add(mul(i, 2), 2)
                let word := or(v1, shl(128, v2))
                mstore(add(dataStart, mul(i, 32)), word)
            }
        }
    }
}
```

---

## 5. Assembly-Optimized ERC-20

### 5.1 Full ERC-20 ใน Yul/Assembly

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title AssemblyERC20
 * @notice ERC-20 token ที่ transfer และ approve ใช้ assembly
 * @dev เป็นการ optimize ส่วน hot path ที่สุด
 *
 * Storage layout:
 * slot 0: totalSupply
 * slot 1: name (string)
 * slot 2: symbol (string)
 * slot 3: decimals (uint8)
 * slot 4: balanceOf mapping
 * slot 5: allowance mapping (nested)
 */
contract AssemblyERC20 {

    // Custom errors
    error InsufficientBalance();
    error InsufficientAllowance();
    error ZeroAddress();
    error ZeroAmount();

    // Events
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    // State variables
    uint256 public totalSupply;
    string public name;
    string public symbol;
    uint8 public decimals;

    // slot 4: balanceOf
    mapping(address => uint256) public balanceOf;

    // slot 5: allowance
    mapping(address => mapping(address => uint256)) public allowance;

    // Event signatures
    bytes32 private constant TRANSFER_EVENT_SIG =
        0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef;
    bytes32 private constant APPROVAL_EVENT_SIG =
        0x8c5be1e5ebec7d5bd14f71427d1e84f3dd0314c0f7b2291e5b200ac8c7c3b925;

    constructor(
        string memory _name,
        string memory _symbol,
        uint8 _decimals,
        uint256 initialSupply
    ) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
        _mint(msg.sender, initialSupply);
    }

    /// @notice Transfer ที่ optimize ด้วย assembly
    function transfer(address to, uint256 amount) external returns (bool) {
        assembly {
            // ตรวจสอบ to != address(0)
            if iszero(to) {
                // revert ZeroAddress()
                mstore(0x00, 0xd92e233d)
                revert(0x1c, 0x04)
            }

            // ตรวจสอบ amount != 0 (optional optimization)
            // if iszero(amount) { ... }

            // คำนวณ balanceOf[msg.sender] slot
            // slot = keccak256(abi.encode(msg.sender, 4))
            mstore(0x00, caller())
            mstore(0x20, 4)  // balanceOf is at slot 4
            let senderSlot := keccak256(0x00, 0x40)
            let senderBalance := sload(senderSlot)

            // ตรวจสอบ balance
            if lt(senderBalance, amount) {
                // revert InsufficientBalance()
                mstore(0x00, 0xf4d678b8)
                revert(0x1c, 0x04)
            }

            // คำนวณ balanceOf[to] slot
            mstore(0x00, to)
            mstore(0x20, 4)
            let toSlot := keccak256(0x00, 0x40)

            // Update balances
            sstore(senderSlot, sub(senderBalance, amount))
            sstore(toSlot, add(sload(toSlot), amount))

            // Emit Transfer event
            // event Transfer(address indexed from, address indexed to, uint256 value)
            mstore(0x00, amount)
            log3(
                0x00,
                0x20,
                TRANSFER_EVENT_SIG,
                caller(),
                to
            )

            // return true
            mstore(0x00, 1)
            return(0x00, 0x20)
        }
    }

    /// @notice TransferFrom ที่ optimize ด้วย assembly
    function transferFrom(
        address from,
        address to,
        uint256 amount
    ) external returns (bool) {
        assembly {
            // ตรวจสอบ addresses
            if or(iszero(from), iszero(to)) {
                mstore(0x00, 0xd92e233d)
                revert(0x1c, 0x04)
            }

            // ตรวจสอบและ update allowance
            // allowance slot = keccak256(abi.encode(caller(), keccak256(abi.encode(from, 5))))
            mstore(0x00, from)
            mstore(0x20, 5)  // allowance is at slot 5
            let innerSlot := keccak256(0x00, 0x40)

            mstore(0x00, caller())
            mstore(0x20, innerSlot)
            let allowanceSlot := keccak256(0x00, 0x40)
            let currentAllowance := sload(allowanceSlot)

            // ตรวจสอบ allowance (ยกเว้น max uint256)
            if and(
                lt(currentAllowance, amount),
                not(iszero(not(currentAllowance)))  // not(currentAllowance != max_uint256)
            ) {
                // revert InsufficientAllowance()
                mstore(0x00, 0x13be252b)
                revert(0x1c, 0x04)
            }

            // Update allowance ถ้าไม่ใช่ max_uint256
            if not(iszero(not(currentAllowance))) {
                sstore(allowanceSlot, sub(currentAllowance, amount))
            }

            // ตรวจสอบ balance ของ from
            mstore(0x00, from)
            mstore(0x20, 4)
            let fromSlot := keccak256(0x00, 0x40)
            let fromBalance := sload(fromSlot)

            if lt(fromBalance, amount) {
                mstore(0x00, 0xf4d678b8)
                revert(0x1c, 0x04)
            }

            // Update balances
            mstore(0x00, to)
            mstore(0x20, 4)
            let toSlot := keccak256(0x00, 0x40)

            sstore(fromSlot, sub(fromBalance, amount))
            sstore(toSlot, add(sload(toSlot), amount))

            // Emit Transfer event
            mstore(0x00, amount)
            log3(0x00, 0x20, TRANSFER_EVENT_SIG, from, to)

            mstore(0x00, 1)
            return(0x00, 0x20)
        }
    }

    /// @notice Approve ที่ optimize ด้วย assembly
    function approve(address spender, uint256 amount) external returns (bool) {
        assembly {
            // ตรวจสอบ spender
            if iszero(spender) {
                mstore(0x00, 0xd92e233d)
                revert(0x1c, 0x04)
            }

            // คำนวณ allowance[msg.sender][spender] slot
            mstore(0x00, caller())
            mstore(0x20, 5)
            let innerSlot := keccak256(0x00, 0x40)

            mstore(0x00, spender)
            mstore(0x20, innerSlot)
            let allowanceSlot := keccak256(0x00, 0x40)

            // เขียน allowance
            sstore(allowanceSlot, amount)

            // Emit Approval event
            mstore(0x00, amount)
            log3(0x00, 0x20, APPROVAL_EVENT_SIG, caller(), spender)

            mstore(0x00, 1)
            return(0x00, 0x20)
        }
    }

    /// @notice Mint tokens
    function _mint(address to, uint256 amount) internal {
        assembly {
            if iszero(to) {
                mstore(0x00, 0xd92e233d)
                revert(0x1c, 0x04)
            }

            // Update totalSupply (slot 0)
            let newTotalSupply := add(sload(0), amount)
            sstore(0, newTotalSupply)

            // Update balanceOf[to]
            mstore(0x00, to)
            mstore(0x20, 4)
            let toSlot := keccak256(0x00, 0x40)
            sstore(toSlot, add(sload(toSlot), amount))

            // Emit Transfer(address(0), to, amount)
            mstore(0x00, amount)
            log3(0x00, 0x20, TRANSFER_EVENT_SIG, 0, to)
        }
    }

    /// @notice Burn tokens
    function burn(uint256 amount) external {
        assembly {
            // ตรวจสอบ balance
            mstore(0x00, caller())
            mstore(0x20, 4)
            let senderSlot := keccak256(0x00, 0x40)
            let senderBalance := sload(senderSlot)

            if lt(senderBalance, amount) {
                mstore(0x00, 0xf4d678b8)
                revert(0x1c, 0x04)
            }

            // Update balance
            sstore(senderSlot, sub(senderBalance, amount))

            // Update totalSupply
            sstore(0, sub(sload(0), amount))

            // Emit Transfer(sender, address(0), amount)
            mstore(0x00, amount)
            log3(0x00, 0x20, TRANSFER_EVENT_SIG, caller(), 0)
        }
    }

    /// @notice Mint function สาธารณะสำหรับ testing
    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
}
```

---

## 6. SafeTransfer Library ใน Assembly

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title SafeTransferLib
 * @notice SafeTransfer implementation ด้วย assembly
 * @dev คล้าย solmate's SafeTransferLib
 *
 * ทำไมต้องมี SafeTransfer?
 * - ERC-20 บาง token ไม่ return value (เช่น USDT)
 * - บาง token return false แทน revert
 * - ต้องจัดการทั้ง 2 กรณี
 */
library SafeTransferLib {

    error TransferFailed();
    error TransferFromFailed();
    error ApproveFailed();

    /// @notice Safe transfer ที่รองรับ non-standard ERC-20
    function safeTransfer(address token, address to, uint256 amount) internal {
        bool success;
        assembly {
            // ใช้ scratch space
            let ptr := mload(0x40)

            // encode transfer(address,uint256) call
            // selector: 0xa9059cbb
            mstore(ptr, 0xa9059cbb00000000000000000000000000000000000000000000000000000000)
            mstore(add(ptr, 4), to)
            mstore(add(ptr, 36), amount)

            // call token
            success := call(
                gas(),    // forward all gas
                token,    // target
                0,        // no ETH
                ptr,      // calldata start
                68,       // calldata length (4 + 32 + 32)
                ptr,      // returndata destination (overwrite input)
                32        // returndata max size
            )

            // Check success
            // success = true AND (no returndata OR returndata == true)
            success := and(
                success,
                or(
                    iszero(returndatasize()),  // no return value (non-standard tokens)
                    and(
                        gt(returndatasize(), 31),
                        mload(ptr)             // return value == true
                    )
                )
            )
        }
        if (!success) revert TransferFailed();
    }

    /// @notice Safe transferFrom
    function safeTransferFrom(
        address token,
        address from,
        address to,
        uint256 amount
    ) internal {
        bool success;
        assembly {
            let ptr := mload(0x40)

            // transferFrom(address,address,uint256) selector: 0x23b872dd
            mstore(ptr, 0x23b872dd00000000000000000000000000000000000000000000000000000000)
            mstore(add(ptr, 4), from)
            mstore(add(ptr, 36), to)
            mstore(add(ptr, 68), amount)

            success := call(gas(), token, 0, ptr, 100, ptr, 32)

            success := and(
                success,
                or(
                    iszero(returndatasize()),
                    and(gt(returndatasize(), 31), mload(ptr))
                )
            )
        }
        if (!success) revert TransferFromFailed();
    }

    /// @notice Safe approve
    function safeApprove(address token, address spender, uint256 amount) internal {
        bool success;
        assembly {
            let ptr := mload(0x40)

            // approve(address,uint256) selector: 0x095ea7b3
            mstore(ptr, 0x095ea7b300000000000000000000000000000000000000000000000000000000)
            mstore(add(ptr, 4), spender)
            mstore(add(ptr, 36), amount)

            success := call(gas(), token, 0, ptr, 68, ptr, 32)

            success := and(
                success,
                or(
                    iszero(returndatasize()),
                    and(gt(returndatasize(), 31), mload(ptr))
                )
            )
        }
        if (!success) revert ApproveFailed();
    }

    /// @notice Safe approve with reset (สำหรับ tokens ที่ต้อง reset ก่อน)
    function safeApproveWithReset(
        address token,
        address spender,
        uint256 amount
    ) internal {
        // Reset approval ก่อน (สำหรับ USDT-like tokens)
        safeApprove(token, spender, 0);
        safeApprove(token, spender, amount);
    }

    /// @notice อ่าน balance ด้วย static call
    function balanceOf(address token, address account) internal view returns (uint256 bal) {
        assembly {
            let ptr := mload(0x40)

            // balanceOf(address) selector: 0x70a08231
            mstore(ptr, 0x70a0823100000000000000000000000000000000000000000000000000000000)
            mstore(add(ptr, 4), account)

            let success := staticcall(gas(), token, ptr, 36, ptr, 32)

            if iszero(success) {
                // return 0 ถ้า call fail
                bal := 0
            }

            if success {
                bal := mload(ptr)
            }
        }
    }
}

/**
 * @title SafeTransferExample
 * @notice ตัวอย่างการใช้ SafeTransferLib
 */
contract SafeTransferExample {
    using SafeTransferLib for address;

    event Deposited(address token, address from, uint256 amount);
    event Withdrawn(address token, address to, uint256 amount);

    mapping(address => mapping(address => uint256)) public deposits;

    function deposit(address token, uint256 amount) external {
        token.safeTransferFrom(msg.sender, address(this), amount);
        deposits[token][msg.sender] += amount;
        emit Deposited(token, msg.sender, amount);
    }

    function withdraw(address token, uint256 amount) external {
        require(deposits[token][msg.sender] >= amount, "Insufficient deposit");
        deposits[token][msg.sender] -= amount;
        token.safeTransfer(msg.sender, amount);
        emit Withdrawn(token, msg.sender, amount);
    }

    function getBalance(address token, address user) external view returns (uint256) {
        return token.balanceOf(user);
    }
}
```

---

## 7. Workshop: Gas Comparison Tests

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";

/**
 * @title AssemblyGasTest
 * @notice เปรียบเทียบ gas ระหว่าง Solidity และ Assembly implementations
 */
contract AssemblyGasTest is Test {
    AssemblyERC20 public asmToken;
    StandardERC20 public stdToken;

    address constant ALICE = address(0xA11CE);
    address constant BOB = address(0xB0B);

    function setUp() public {
        asmToken = new AssemblyERC20("Assembly Token", "ASM", 18, 1_000_000e18);
        stdToken = new StandardERC20("Standard Token", "STD", 18, 1_000_000e18);

        asmToken.transfer(ALICE, 100_000e18);
        stdToken.transfer(ALICE, 100_000e18);
    }

    /// @notice วัด gas ของ assembly transfer
    function testAssemblyTransfer() public {
        vm.prank(ALICE);
        asmToken.transfer(BOB, 1000e18);
    }

    /// @notice วัด gas ของ standard transfer
    function testStandardTransfer() public {
        vm.prank(ALICE);
        stdToken.transfer(BOB, 1000e18);
    }

    /// @notice วัด gas ของ assembly approve + transferFrom
    function testAssemblyApproveAndTransferFrom() public {
        vm.prank(ALICE);
        asmToken.approve(address(this), type(uint256).max);

        asmToken.transferFrom(ALICE, BOB, 1000e18);
    }

    /// @notice วัด gas ของ standard approve + transferFrom
    function testStandardApproveAndTransferFrom() public {
        vm.prank(ALICE);
        stdToken.approve(address(this), type(uint256).max);

        stdToken.transferFrom(ALICE, BOB, 1000e18);
    }

    /// @notice Test correctness
    function testCorrectness() public {
        uint256 aliceBalanceBefore = asmToken.balanceOf(ALICE);
        uint256 bobBalanceBefore = asmToken.balanceOf(BOB);

        vm.prank(ALICE);
        asmToken.transfer(BOB, 1000e18);

        assertEq(asmToken.balanceOf(ALICE), aliceBalanceBefore - 1000e18);
        assertEq(asmToken.balanceOf(BOB), bobBalanceBefore + 1000e18);
    }

    /// @notice Test edge cases
    function testInsufficientBalance() public {
        vm.prank(ALICE);
        vm.expectRevert(AssemblyERC20.InsufficientBalance.selector);
        asmToken.transfer(BOB, type(uint256).max);
    }

    function testZeroAddress() public {
        vm.prank(ALICE);
        vm.expectRevert(AssemblyERC20.ZeroAddress.selector);
        asmToken.transfer(address(0), 1000e18);
    }
}

// Standard ERC-20 สำหรับ comparison
contract StandardERC20 {
    string public name;
    string public symbol;
    uint8 public decimals;
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    constructor(string memory _name, string memory _symbol, uint8 _dec, uint256 supply) {
        name = _name;
        symbol = _symbol;
        decimals = _dec;
        totalSupply = supply;
        balanceOf[msg.sender] = supply;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        require(to != address(0), "Zero address");
        require(balanceOf[msg.sender] >= amount, "Insufficient balance");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(from != address(0) && to != address(0), "Zero address");
        require(balanceOf[from] >= amount, "Insufficient balance");
        if (allowance[from][msg.sender] != type(uint256).max) {
            require(allowance[from][msg.sender] >= amount, "Insufficient allowance");
            allowance[from][msg.sender] -= amount;
        }
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        emit Transfer(from, to, amount);
        return true;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        require(spender != address(0), "Zero address");
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }
}
```

---

## 8. Pure Yul Contract (Advanced)

```yul
// pure-yul-token.yul
// @notice Token ที่เขียนใน pure Yul (ไม่ใช่ inline assembly ใน Solidity)
// ใช้สำหรับเรียนรู้ ไม่ใช่ production

object "PureYulToken" {
    // Constructor
    code {
        // Deploy runtime code
        datacopy(0, dataoffset("runtime"), datasize("runtime"))
        return(0, datasize("runtime"))
    }

    object "runtime" {
        code {
            // Memory layout:
            // 0x00-0x3f: scratch space
            // 0x40-0x5f: free memory pointer

            // Initialize free memory pointer
            mstore(0x40, 0x80)

            // Dispatch table
            let selector := shr(224, calldataload(0))

            switch selector
            // transfer(address,uint256) = 0xa9059cbb
            case 0xa9059cbb {
                let to := calldataload(4)
                let amount := calldataload(36)
                transfer(caller(), to, amount)
                mstore(0, 1)
                return(0, 32)
            }
            // balanceOf(address) = 0x70a08231
            case 0x70a08231 {
                let account := calldataload(4)
                mstore(0, balanceOf(account))
                return(0, 32)
            }
            // totalSupply() = 0x18160ddd
            case 0x18160ddd {
                mstore(0, sload(0)) // totalSupply at slot 0
                return(0, 32)
            }
            default {
                revert(0, 0)
            }

            // Functions
            function transfer(from, to, amount) {
                let fromBal := balanceOf(from)
                if lt(fromBal, amount) {
                    revert(0, 0)
                }
                setBalance(from, sub(fromBal, amount))
                setBalance(to, add(balanceOf(to), amount))
            }

            function balanceOf(account) -> bal {
                mstore(0, account)
                mstore(32, 1) // balanceOf at slot 1
                bal := sload(keccak256(0, 64))
            }

            function setBalance(account, amount) {
                mstore(0, account)
                mstore(32, 1)
                sstore(keccak256(0, 64), amount)
            }
        }
    }
}
```

---

## สรุป Part 67

- **Yul basics**: ตัวแปร, control flow (if/switch/for), functions, arithmetic ops ทั้งหมด
- **Memory layout**: scratch space (0x00-0x3f), free memory pointer (0x40), data starts at 0x80
- **Storage slots**: อ่านเขียนโดยตรงด้วย `sload`/`sstore`, คำนวณ mapping slots ด้วย keccak256
- **Calldata parsing**: `calldataload`, `calldatacopy`, ใช้ `data.offset` กับ `calldata` parameters
- **Custom allocator**: arena allocator ลด overhead ของ Solidity memory management
- **Assembly ERC-20**: transfer/approve/transferFrom เร็วกว่า standard ~15-20%
- **SafeTransfer library**: รองรับ non-standard tokens ที่ไม่ return value
- **Pure Yul**: เขียน contract ทั้งหมดใน Yul สำหรับ maximum control

## Next: Part 68 - Writing & Proposing EIPs
