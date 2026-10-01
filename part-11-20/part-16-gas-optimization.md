# Part 16: Gas Optimization

## สารบัญ
1. Gas Basics
2. Storage Optimization
3. Computation Optimization
4. Calldata vs Memory
5. Loop Optimization
6. Event vs Storage
7. Bytecode Optimization
8. Workshop: Gas-Optimized Contract

---

## 1. Gas Basics

```
Gas = หน่วยวัด computational work ใน EVM

Transaction Cost = Gas Used × Gas Price
                 = Gas Used × (Base Fee + Priority Fee)

Base Gas Costs:
- ADD, SUB:           3 gas
- MUL, DIV:           5 gas
- SLOAD (cold):    2100 gas
- SLOAD (warm):     100 gas
- SSTORE (new):   22100 gas  ← แพงมาก!
- SSTORE (update): 5000 gas
- SSTORE (clear):  15000 gas (refund)
- CALL:             700 gas
- LOG (event):      375 + 375/topic + 8/byte
- KECCAK256:         30 + 6/word
- CREATE:         32000 gas
- TX base:        21000 gas
```

---

## 2. Storage Optimization

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// ❌ Wastes storage slots
contract UnpackedStorage {
    uint256 a;   // slot 0: 32 bytes
    uint128 b;   // slot 1: 16 bytes (wastes 16!)
    uint128 c;   // slot 2: 16 bytes (wastes 16!)
    address d;   // slot 3: 20 bytes (wastes 12!)
    bool e;      // slot 4: 1 byte (wastes 31!)
    // Total: 5 slots = 5 × 32 = 160 bytes
}

// ✅ Packed storage slots
contract PackedStorage {
    uint256 a;   // slot 0: 32 bytes
    uint128 b;   // slot 1: 16 bytes
    uint128 c;   //         16 bytes (fits in slot 1!)
    address d;   // slot 2: 20 bytes
    bool e;      //          1 byte (fits in slot 2!)
    // Total: 3 slots = 3 × 32 = 96 bytes
    // Saves 2 SSTORE operations!
}

// Struct Packing
contract StructPacking {
    
    // ❌ Bad packing: 3 slots
    struct UserBad {
        uint256 balance;     // slot 0
        address wallet;      // slot 1 (20 bytes wasted 12)
        uint256 lastActive;  // slot 2
        bool active;         // slot 3 (wasted 31!)
    }
    
    // ✅ Good packing: 2 slots  
    struct UserGood {
        uint256 balance;     // slot 0
        address wallet;      // slot 1 (20 bytes)
        uint64 lastActive;   //         8 bytes (fits!)
        bool active;         //         1 byte (fits!)
        // 20+8+1 = 29 bytes, fits in 32!
    }
    
    // ✅ Maximum packing example
    struct NFTData {
        address owner;       // 20 bytes
        uint32 tokenId;      //  4 bytes
        uint32 mintTime;     //  4 bytes
        uint16 royalty;      //  2 bytes
        uint8 category;      //  1 byte
        bool listed;         //  1 byte
        // Total: 32 bytes = 1 slot!
    }
    
    mapping(uint256 => NFTData) public nftData;
    
    // Reading packed struct: 1 SLOAD vs 4 SLOADs
    function getOwner(uint256 tokenId) external view returns (address) {
        return nftData[tokenId].owner; // 1 SLOAD only
    }
}

// Immutable vs Constant vs Storage
contract ImmutableOptimization {
    
    // ✅ constant: replaced at compile time (0 gas!)
    uint256 public constant MAX_SUPPLY = 10_000;
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    
    // ✅ immutable: set in constructor, stored in bytecode
    address public immutable owner;  // ~2100 gas vs ~2200 SLOAD
    uint256 public immutable deployTime;
    
    // ❌ storage variable (expensive)
    address public mutableOwner;  // SLOAD each read
    
    constructor() {
        owner = msg.sender;
        deployTime = block.timestamp;
        mutableOwner = msg.sender;
    }
    
    // Gas comparison:
    // constant: ~21000 (base tx only)
    // immutable: ~21100
    // storage: ~23200 (2100 SLOAD cold + overhead)
}
```

---

## 3. Computation Optimization

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract ComputationOptimization {
    
    // ✅ Use unchecked for safe arithmetic
    function sumArray(uint256[] calldata arr) external pure returns (uint256 sum) {
        uint256 len = arr.length;
        for (uint256 i = 0; i < len;) {
            unchecked {
                sum += arr[i];
                ++i;  // ++i cheaper than i++ (no temp var)
            }
        }
    }
    
    // ✅ Cache storage reads
    function processUser(address user) external returns (uint256) {
        // ❌ Bad: reads storage 3 times
        // if (balances[user] > 0) {
        //     return balances[user] * 2 + balances[user];
        // }
        
        // ✅ Good: reads storage once
        uint256 balance = balances[user];
        if (balance > 0) {
            return balance * 3; // uses cached value
        }
        return 0;
    }
    
    mapping(address => uint256) public balances;
    
    // ✅ Short-circuit evaluation
    function checkConditions(uint256 x, uint256 y) external pure returns (bool) {
        // Put cheapest/most likely to fail first
        return x > 0 && y > 0 && expensiveCheck(x, y);
    }
    
    function expensiveCheck(uint256 x, uint256 y) internal pure returns (bool) {
        return keccak256(abi.encodePacked(x, y)) != bytes32(0);
    }
    
    // ✅ Use mapping over array for O(1) lookup
    mapping(address => bool) public whitelist;    // O(1)
    address[] public whitelistArray;              // O(n) to search!
    
    // ✅ Bit operations over division
    function divideBy2(uint256 x) external pure returns (uint256) {
        return x >> 1; // cheaper than x / 2
    }
    
    function multiplyBy8(uint256 x) external pure returns (uint256) {
        return x << 3; // cheaper than x * 8
    }
    
    function isEven(uint256 x) external pure returns (bool) {
        return x & 1 == 0; // cheaper than x % 2 == 0
    }
    
    // ✅ Batch operations
    function batchTransfer(
        address[] calldata recipients,
        uint256[] calldata amounts
    ) external {
        require(recipients.length == amounts.length);
        
        // Better: 1 tx vs N txs
        for (uint256 i; i < recipients.length;) {
            balances[recipients[i]] += amounts[i];
            unchecked { ++i; }
        }
    }
}
```

---

## 4. Calldata Optimization

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract CalldataOptimization {
    
    // ❌ memory: copies array (expensive)
    function sumMemory(uint256[] memory arr) external pure returns (uint256 sum) {
        for (uint256 i; i < arr.length;) {
            unchecked { sum += arr[i]; ++i; }
        }
    }
    
    // ✅ calldata: no copy (cheaper for read-only)
    function sumCalldata(uint256[] calldata arr) external pure returns (uint256 sum) {
        for (uint256 i; i < arr.length;) {
            unchecked { sum += arr[i]; ++i; }
        }
    }
    
    // ✅ string calldata vs memory
    function hashStringCalldata(string calldata s) external pure returns (bytes32) {
        return keccak256(bytes(s)); // no copy
    }
    
    // ✅ bytes calldata for binary data
    function processData(bytes calldata data) external pure returns (bytes32) {
        return keccak256(data);
    }
    
    // Internal function: use memory (calldata can't be modified)
    function _processInternal(uint256[] memory arr) internal pure returns (uint256 sum) {
        for (uint256 i; i < arr.length;) {
            arr[i] *= 2; // can modify memory
            unchecked { sum += arr[i]; ++i; }
        }
    }
}
```

---

## 5. Storage Patterns

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract StoragePatterns {
    
    // ✅ Bitmap: 256 booleans in 1 slot
    uint256 private _bitmap;
    
    function setFlag(uint8 index, bool value) external {
        if (value) {
            _bitmap |= (1 << index);
        } else {
            _bitmap &= ~(1 << index);
        }
    }
    
    function getFlag(uint8 index) external view returns (bool) {
        return (_bitmap >> index) & 1 == 1;
    }
    
    // ✅ Pack multiple values into uint256
    // Example: tokenId (uint32) + amount (uint128) + timestamp (uint64) + flags (uint32)
    
    function packValues(
        uint32 tokenId,
        uint128 amount,
        uint64 timestamp,
        uint32 flags
    ) public pure returns (uint256 packed) {
        // tokenId: bits 0-31
        // amount: bits 32-159
        // timestamp: bits 160-223
        // flags: bits 224-255
        packed = uint256(tokenId)
            | (uint256(amount) << 32)
            | (uint256(timestamp) << 160)
            | (uint256(flags) << 224);
    }
    
    function unpackValues(uint256 packed) public pure returns (
        uint32 tokenId,
        uint128 amount,
        uint64 timestamp,
        uint32 flags
    ) {
        tokenId = uint32(packed);
        amount = uint128(packed >> 32);
        timestamp = uint64(packed >> 160);
        flags = uint32(packed >> 224);
    }
    
    // ✅ Delete storage for gas refund
    mapping(address => uint256) public stakes;
    
    function unstake(address user) external {
        uint256 amount = stakes[user];
        require(amount > 0);
        
        delete stakes[user]; // Refund 15000 gas!
        // stakes[user] = 0 would only get 100 gas refund
        
        payable(user).transfer(amount);
    }
    
    // ✅ Use events instead of storage for historical data
    event PriceUpdated(uint256 indexed timestamp, uint256 price);
    
    // ❌ Storing price history in storage is expensive
    // uint256[] public priceHistory;
    
    // ✅ Emit events - retrievable off-chain
    function updatePrice(uint256 price) external {
        emit PriceUpdated(block.timestamp, price); // 375 gas vs 22100 gas SSTORE!
    }
}
```

---

## 6. Workshop: Gas-Optimized Token

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title OptimizedToken
 * @dev Gas-optimized ERC-20 token
 * Techniques: struct packing, immutables, unchecked arithmetic
 */
contract OptimizedToken {
    
    // ✅ Pack name+symbol info to avoid multiple SLOAD
    // Use bytes32 instead of string for short values
    bytes32 private immutable _name;
    bytes32 private immutable _symbol;
    uint8 public immutable decimals;
    
    // ✅ Immutables
    address public immutable owner;
    uint256 public immutable maxSupply;
    
    // ✅ Total supply packed with other data if possible
    uint256 public totalSupply;
    
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    // ✅ Pack multiple per-address data into single slot
    // bits 0-191: balance (uint192 = ~6.3 × 10^57, more than enough)
    // bits 192-223: lastTransfer timestamp (uint32, valid until 2106)
    // bits 224-255: flags (uint32)
    // Using uint256 mapping is simpler but shown here for illustration
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    error InsufficientBalance();
    error InsufficientAllowance();
    error ZeroAddress();
    error MaxSupplyExceeded();
    
    constructor(
        string memory name_,
        string memory symbol_,
        uint8 decimals_,
        uint256 maxSupply_
    ) {
        // Convert to bytes32 for cheaper reads
        _name = _toBytes32(name_);
        _symbol = _toBytes32(symbol_);
        decimals = decimals_;
        owner = msg.sender;
        maxSupply = maxSupply_;
    }
    
    function name() external view returns (string memory) {
        return _fromBytes32(_name);
    }
    
    function symbol() external view returns (string memory) {
        return _fromBytes32(_symbol);
    }
    
    function transfer(address to, uint256 amount) external returns (bool) {
        if (to == address(0)) revert ZeroAddress();
        
        uint256 senderBalance = balanceOf[msg.sender];
        if (senderBalance < amount) revert InsufficientBalance();
        
        // ✅ unchecked: balance already checked above
        unchecked {
            balanceOf[msg.sender] = senderBalance - amount;
            balanceOf[to] += amount;
        }
        
        emit Transfer(msg.sender, to, amount);
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        if (to == address(0)) revert ZeroAddress();
        
        uint256 currentAllowance = allowance[from][msg.sender];
        
        if (currentAllowance != type(uint256).max) {
            if (currentAllowance < amount) revert InsufficientAllowance();
            unchecked { allowance[from][msg.sender] = currentAllowance - amount; }
        }
        
        uint256 senderBalance = balanceOf[from];
        if (senderBalance < amount) revert InsufficientBalance();
        
        unchecked {
            balanceOf[from] = senderBalance - amount;
            balanceOf[to] += amount;
        }
        
        emit Transfer(from, to, amount);
        return true;
    }
    
    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }
    
    function mint(address to, uint256 amount) external {
        require(msg.sender == owner, "Not owner");
        
        unchecked {
            uint256 newSupply = totalSupply + amount;
            if (newSupply > maxSupply) revert MaxSupplyExceeded();
            totalSupply = newSupply;
            balanceOf[to] += amount;
        }
        
        emit Transfer(address(0), to, amount);
    }
    
    function _toBytes32(string memory s) private pure returns (bytes32 result) {
        bytes memory b = bytes(s);
        require(b.length <= 32, "Too long");
        assembly { result := mload(add(b, 32)) }
    }
    
    function _fromBytes32(bytes32 b) private pure returns (string memory s) {
        // Count non-zero bytes
        uint256 len;
        for (uint256 i = 0; i < 32; ++i) {
            if (b[i] == 0) break;
            ++len;
        }
        s = new string(len);
        assembly { mstore(add(s, 32), b) }
    }
}
```

---

## 7. Gas Profiling Script

```typescript
// scripts/gas-profile.ts
import { ethers } from "hardhat";

async function main() {
    const [deployer] = await ethers.getSigners();
    
    // Deploy contracts
    const Standard = await ethers.getContractFactory("ERC20");
    const Optimized = await ethers.getContractFactory("OptimizedToken");
    
    const std = await Standard.deploy("Standard", "STD");
    const opt = await Optimized.deploy("Optimized", "OPT", 18, ethers.parseEther("1000000"));
    
    // Measure gas
    const recipient = "0x1234567890123456789012345678901234567890";
    const amount = ethers.parseEther("100");
    
    // Mint
    const mintStd = await std.getDeployTransaction();
    const mintOpt = await opt.mint(deployer.address, amount);
    
    const stdMintReceipt = await mintOpt.wait();
    
    console.log("=== Gas Comparison ===");
    console.log(`Optimized Mint: ${stdMintReceipt?.gasUsed} gas`);
    
    // Transfer
    const transferStd = await std.transfer(recipient, amount);
    const transferOpt = await opt.transfer(recipient, amount);
    
    const stdTxReceipt = await transferStd.wait();
    const optTxReceipt = await transferOpt.wait();
    
    console.log(`\nTransfer Gas:`);
    console.log(`Standard ERC-20: ${stdTxReceipt?.gasUsed}`);
    console.log(`Optimized Token: ${optTxReceipt?.gasUsed}`);
    console.log(`Savings: ${Number(stdTxReceipt?.gasUsed) - Number(optTxReceipt?.gasUsed)} gas`);
}

main();
```

---

## Gas Optimization Checklist

```
Storage:
[ ] Pack related variables into same slot
[ ] Use immutable for constructor-set values
[ ] Use constant for compile-time values
[ ] Use bytes32 instead of string for short strings
[ ] Delete unused storage (gas refund)
[ ] Use events instead of storage for history

Computation:
[ ] Use unchecked{} for safe arithmetic
[ ] Use ++i instead of i++
[ ] Cache array length before loop
[ ] Cache storage reads in local variables
[ ] Short-circuit conditions (cheapest first)
[ ] Use bit operations (>>, <<, &, |)

Function Design:
[ ] Use calldata instead of memory for read-only params
[ ] Use external instead of public when possible
[ ] Batch operations in single function
[ ] Use custom errors (cheaper than strings)
[ ] Avoid returning large arrays

Architecture:
[ ] Use mappings instead of arrays for lookups
[ ] Off-chain computation + on-chain verification
[ ] Use events for indexable historical data
[ ] Consider L2 for frequent operations
```

---

## สรุป Part 16

Gas Optimization ที่เรียนรู้:
- ✅ EVM gas costs
- ✅ Storage slot packing
- ✅ Immutables และ constants
- ✅ Unchecked arithmetic
- ✅ Calldata optimization
- ✅ Bitmap patterns
- ✅ Gas-optimized token

## Quiz

1. ทำไม `SSTORE` ถึงแพงกว่า `SLOAD` มาก?
2. `constant` vs `immutable` ต่างกันอย่างไร?
3. ทำไม `delete` ถึง refund gas มากกว่า `= 0`?
4. เมื่อไหร่ควรใช้ `unchecked`?

---

## Next: Part 17 - Upgradeable Contracts
