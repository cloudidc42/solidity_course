# Part 36: Advanced Gas Optimization

## สารบัญ
1. EVM Opcodes และ Cost
2. Assembly Optimization
3. Memory vs Calldata vs Storage
4. Custom Errors vs Strings
5. Workshop: Ultra-Optimized Contract

---

## 1. EVM Opcodes Gas Cost

```
Key Opcodes:
- SLOAD: 2100 gas (cold), 100 gas (warm)
- SSTORE: 20000 gas (new), 2900 gas (update), 100 gas (same)
- CALL: 2600 gas (cold), 100 gas (warm)
- CREATE: 32000 gas + init code
- CALLDATALOAD: 3 gas
- MLOAD/MSTORE: 3 gas
- ADD/SUB/MUL: 3-5 gas
- DIV/MOD: 5 gas

Zero byte vs Non-zero calldata:
- Zero byte: 4 gas
- Non-zero byte: 16 gas
→ Compress calldata ด้วย zeros

Memory Expansion:
- Cost เพิ่มขึ้น quadratically
- words^2 / 512 + 3 × words

Stack depth limit: 1024
```

---

## 2. Assembly Optimization

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Using inline assembly for gas savings
 * ระวัง: assembly ไม่มี safety checks
 */
contract AssemblyOptimized {
    
    // Standard: ~500 gas
    function standardAdd(uint256 a, uint256 b) external pure returns (uint256) {
        return a + b; // Includes overflow check
    }
    
    // Optimized: ~100 gas (no overflow check)
    function assemblyAdd(uint256 a, uint256 b) external pure returns (uint256 result) {
        assembly {
            result := add(a, b)
        }
    }
    
    // Efficient memory copy
    function memoryCopy(bytes calldata data) external pure returns (bytes memory) {
        uint256 len = data.length;
        bytes memory result = new bytes(len);
        
        assembly {
            // Copy 32 bytes at a time
            let dst := add(result, 32)
            let src := data.offset
            let end := add(src, len)
            
            for {} lt(src, end) {} {
                mstore(dst, calldataload(src))
                dst := add(dst, 32)
                src := add(src, 32)
            }
        }
        
        return result;
    }
    
    // Efficient address comparison
    function addressEquals(address a, address b) external pure returns (bool result) {
        assembly {
            result := eq(a, b)
        }
    }
    
    // Check if contract exists (cheaper than calling)
    function hasCode(address addr) external view returns (bool result) {
        assembly {
            result := gt(extcodesize(addr), 0)
        }
    }
    
    // Read specific storage slot (useful for proxy patterns)
    function readSlot(bytes32 slot) external view returns (bytes32 value) {
        assembly {
            value := sload(slot)
        }
    }
    
    // Write to specific storage slot
    function writeSlot(bytes32 slot, bytes32 value) external {
        assembly {
            sstore(slot, value)
        }
    }
    
    // Efficient revert with message
    function revertWithError(bytes4 selector) external pure {
        assembly {
            mstore(0x00, selector)
            revert(0x00, 0x04)
        }
    }
    
    // Get calldata without copying
    function processCalldata() external pure returns (bytes4 selector) {
        assembly {
            selector := calldataload(0)
        }
    }
}
```

---

## 3. Storage Layout Optimization

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Storage optimization techniques
 */

// ❌ Bad: 5 storage slots
contract BadLayout {
    uint256 a;    // slot 0
    uint8 b;      // slot 1 (wastes 31 bytes!)
    uint256 c;    // slot 2
    bool d;       // slot 3 (wastes 31 bytes!)
    address e;    // slot 4
}

// ✅ Good: 3 storage slots
contract GoodLayout {
    uint256 a;    // slot 0
    uint256 c;    // slot 1
    // Pack small variables together:
    uint8 b;      // slot 2
    bool d;       // slot 2 (same slot!)
    address e;    // slot 2 (same slot! 20+1+1 = 22 bytes)
}

/**
 * Tight Packing with Structs
 */
contract PackedStruct {
    // ❌ Bad: 3 slots
    struct UserBad {
        uint256 balance;   // 32 bytes
        address addr;      // 20 bytes → slot 1
        uint256 timestamp; // 32 bytes → slot 2
        bool active;       // 1 byte → slot 3
    }
    
    // ✅ Good: 2 slots
    struct UserGood {
        uint256 balance;   // slot 0: 32 bytes
        address addr;      // slot 1: 20 bytes
        uint64 timestamp;  // slot 1: +8 bytes = 28 bytes
        bool active;       // slot 1: +1 byte = 29 bytes ✓
    }
    
    mapping(uint256 => UserGood) public users;
    
    // Read multiple fields in one SLOAD
    function getUserInfo(uint256 id) external view returns (
        address addr,
        uint64 timestamp,
        bool active
    ) {
        UserGood storage user = users[id];
        // All three fields are in the same slot → 1 SLOAD
        addr = user.addr;
        timestamp = user.timestamp;
        active = user.active;
    }
}

/**
 * Transient Storage (EIP-1153, available Cancun+)
 * Cheap temporary storage (same as memory but accessible across calls)
 */
contract TransientExample {
    bytes32 constant LOCKED_SLOT = keccak256("reentrancy.lock");
    
    modifier nonReentrantTransient() {
        assembly {
            if tload(LOCKED_SLOT) { revert(0, 0) }
            tstore(LOCKED_SLOT, 1)
        }
        _;
        assembly {
            tstore(LOCKED_SLOT, 0)
        }
    }
    
    function secureWithdraw(uint256 amount) external nonReentrantTransient {
        // Safe from reentrancy, cheaper than SSTORE
    }
}
```

---

## 4. Calldata Tricks

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Calldata is cheapest data location on L2
 * ใช้ calldata สำหรับ read-only parameters
 */
contract CalldataOptimized {
    
    // ❌ Memory: copies data (expensive for large arrays)
    function sumMemory(uint256[] memory arr) external pure returns (uint256 sum) {
        for (uint256 i = 0; i < arr.length; i++) {
            sum += arr[i];
        }
    }
    
    // ✅ Calldata: no copy (cheap for large arrays)
    function sumCalldata(uint256[] calldata arr) external pure returns (uint256 sum) {
        for (uint256 i = 0; i < arr.length; i++) {
            sum += arr[i];
        }
    }
    
    // Zero-padding trick: ใช้ address ที่ 0-padded
    // address: 0x0000000000000000000000001234... (12 zero bytes)
    // → ถ้า calldata หลาย address ประหยัด gas มาก
    
    // Compact encoding: ใช้ custom decode แทน abi.decode
    function compactDecode(bytes calldata data) external pure returns (
        address addr,
        uint128 amount,
        uint32 deadline
    ) {
        assembly {
            // Manually decode packed bytes
            addr := shr(96, calldataload(data.offset))
            amount := shr(128, calldataload(add(data.offset, 20)))
            deadline := shr(224, calldataload(add(data.offset, 36)))
        }
    }
    
    // Encode compactly off-chain:
    // bytes.concat(bytes20(addr), bytes16(amount), bytes4(deadline))
    // = 20 + 16 + 4 = 40 bytes instead of 96 bytes with abi.encode
}

/**
 * Custom Error vs Require String
 */
contract CustomErrorOptimized {
    
    // ❌ String error: expensive (string stored in bytecode)
    function validateString(uint256 value) external pure {
        require(value > 0, "Value must be greater than zero"); // ~50 extra bytes
    }
    
    // ✅ Custom error: 4 bytes selector only
    error ValueTooLow(uint256 provided);
    
    function validateCustom(uint256 value) external pure {
        if (value == 0) revert ValueTooLow(value); // 4 bytes
    }
    
    // ✅ Even more minimal: no parameters
    error Invalid();
    
    function validateMinimal(uint256 value) external pure {
        if (value == 0) revert Invalid(); // minimal gas
    }
}

/**
 * Loop Optimization
 */
contract LoopOptimized {
    
    uint256[] private data;
    
    // ❌ Standard loop: ~21000 gas per iteration for storage read
    function sumStandard() external view returns (uint256 sum) {
        for (uint256 i = 0; i < data.length; i++) {
            sum += data[i];
        }
    }
    
    // ✅ Optimized: cache length, unchecked increment
    function sumOptimized() external view returns (uint256 sum) {
        uint256[] storage d = data; // cache storage reference
        uint256 len = d.length;    // cache length (1 SLOAD)
        
        for (uint256 i; i < len;) {
            sum += d[i];
            unchecked { ++i; } // pre-increment, no overflow check
        }
    }
    
    // ✅ Most optimized: load entire array to memory first (if small)
    function sumCached() external view returns (uint256 sum) {
        uint256[] memory cached = data; // copy to memory
        uint256 len = cached.length;
        
        for (uint256 i; i < len;) {
            sum += cached[i];
            unchecked { ++i; }
        }
    }
}
```

---

## 5. Workshop: Ultra-Optimized ERC-20

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Ultra-optimized ERC-20
 * Using every trick in the book
 */
contract UltraToken {
    
    // Events (required by standard)
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    // Errors (cheaper than strings)
    error InsufficientBalance();
    error InsufficientAllowance();
    error ZeroAddress();
    
    // Pack: name + symbol stored as bytes32 (cheaper than string)
    bytes32 immutable _name;
    bytes32 immutable _symbol;
    
    // Pack totalSupply + owner into same slot
    uint224 private _totalSupply;
    address private _owner; // 20 bytes, fits with 4 bytes to spare
    
    mapping(address => uint256) private _balances;
    mapping(address => mapping(address => uint256)) private _allowances;
    
    constructor(bytes32 name_, bytes32 symbol_, uint224 initialSupply_) {
        _name = name_;
        _symbol = symbol_;
        _totalSupply = initialSupply_;
        _owner = msg.sender;
        _balances[msg.sender] = initialSupply_;
        
        // Emit without memory allocation
        assembly {
            log3(
                0, 0,
                // Transfer(address,address,uint256) topic
                0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef,
                0,              // from = address(0)
                caller(),       // to = msg.sender
                initialSupply_
            )
        }
    }
    
    function name() external view returns (string memory) {
        return _bytes32ToString(_name);
    }
    
    function symbol() external view returns (string memory) {
        return _bytes32ToString(_symbol);
    }
    
    function decimals() external pure returns (uint8) { return 18; }
    
    function totalSupply() external view returns (uint256) {
        return _totalSupply;
    }
    
    function balanceOf(address account) external view returns (uint256) {
        return _balances[account];
    }
    
    function allowance(address owner_, address spender) external view returns (uint256) {
        return _allowances[owner_][spender];
    }
    
    function transfer(address to, uint256 amount) external returns (bool) {
        if (to == address(0)) revert ZeroAddress();
        
        uint256 fromBalance = _balances[msg.sender];
        if (fromBalance < amount) revert InsufficientBalance();
        
        unchecked {
            _balances[msg.sender] = fromBalance - amount;
            _balances[to] += amount;
        }
        
        emit Transfer(msg.sender, to, amount);
        return true;
    }
    
    function approve(address spender, uint256 amount) external returns (bool) {
        _allowances[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        if (to == address(0)) revert ZeroAddress();
        
        uint256 currentAllowance = _allowances[from][msg.sender];
        if (currentAllowance != type(uint256).max) {
            if (currentAllowance < amount) revert InsufficientAllowance();
            unchecked { _allowances[from][msg.sender] = currentAllowance - amount; }
        }
        
        uint256 fromBalance = _balances[from];
        if (fromBalance < amount) revert InsufficientBalance();
        
        unchecked {
            _balances[from] = fromBalance - amount;
            _balances[to] += amount;
        }
        
        emit Transfer(from, to, amount);
        return true;
    }
    
    function _bytes32ToString(bytes32 b) internal pure returns (string memory str) {
        uint256 len;
        while (len < 32 && b[len] != 0) len++;
        assembly {
            str := mload(0x40)
            mstore(0x40, add(str, add(len, 32)))
            mstore(str, len)
            mstore(add(str, 32), b)
        }
    }
}
```

---

## สรุป Part 36

Advanced Gas Optimization ที่เรียนรู้:
- ✅ EVM opcode costs (SLOAD, SSTORE, CALL)
- ✅ Inline assembly for hot paths
- ✅ Struct packing + slot optimization
- ✅ Calldata vs memory
- ✅ Custom errors (4 bytes)
- ✅ Loop optimization techniques
- ✅ Ultra-optimized ERC-20

## Quiz

1. ทำไม SLOAD ถึงแพงกว่า MLOAD?
2. Custom error ประหยัด gas ได้อย่างไร?
3. `unchecked { ++i; }` ดีกว่า `i++` อย่างไร?
4. Transient storage (EIP-1153) ต่างจาก regular storage อย่างไร?

---

## Next: Part 37 - Protocol Governance Advanced
