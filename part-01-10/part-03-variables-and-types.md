# Part 03: Solidity Variables และ Types

## สารบัญ
1. ภาพรวม Type System ของ Solidity
2. Value Types
3. Reference Types
4. Special Global Variables
5. Constants และ Immutables
6. State Variables vs Local Variables vs Function Parameters
7. Type Conversions
8. Packed Encoding และ Storage Layout
9. Workshop: ระบบจัดการข้อมูลนักศึกษา
10. Gas Optimization Tips

---

## 1. ภาพรวม Type System ของ Solidity

```
Solidity Type System:

Types
├── Value Types (เก็บค่าโดยตรง)
│   ├── Integer Types
│   │   ├── uint (0 ถึง 2^256-1)
│   │   ├── int (-2^255 ถึง 2^255-1)
│   │   └── uint8, uint16, ..., uint256
│   ├── Boolean (true/false)
│   ├── Address
│   │   ├── address (plain)
│   │   └── address payable (รับ ETH ได้)
│   ├── Bytes Fixed
│   │   └── bytes1, bytes2, ..., bytes32
│   └── Enum
│
└── Reference Types (เก็บ reference/pointer)
    ├── Arrays
    │   ├── Dynamic arrays (uint[])
    │   └── Fixed arrays (uint[5])
    ├── Bytes Dynamic (bytes)
    ├── String
    ├── Mappings (mapping(key => value))
    └── Structs
```

---

## 2. Value Types

### Integer Types

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract IntegerTypes {
    
    // === Unsigned Integers (ไม่มีเครื่องหมาย ≥ 0) ===
    
    uint8   public u8  = 255;           // 0 to 2^8-1 = 255
    uint16  public u16 = 65535;         // 0 to 2^16-1 = 65535
    uint32  public u32 = 4294967295;    // 0 to 2^32-1
    uint64  public u64 = 18446744073709551615;
    uint128 public u128 = type(uint128).max;
    uint256 public u256 = type(uint256).max; // ใหญ่สุด
    uint    public u = 100; // uint = uint256 (default)
    
    // === Signed Integers (มีเครื่องหมาย) ===
    
    int8   public i8  = -128;   // -2^7 to 2^7-1 = -128 to 127
    int16  public i16 = -32768; // -2^15 to 2^15-1
    int32  public i32 = -2147483648;
    int256 public i256 = type(int256).min;
    int    public i = -100; // int = int256 (default)
    
    // === Arithmetic Operations ===
    
    function arithmeticDemo() public pure returns (
        uint addResult,
        uint subResult,
        uint mulResult,
        uint divResult,
        uint modResult,
        uint expResult
    ) {
        uint a = 100;
        uint b = 7;
        
        addResult = a + b;   // 107
        subResult = a - b;   // 93
        mulResult = a * b;   // 700
        divResult = a / b;   // 14 (integer division, rounds down)
        modResult = a % b;   // 2 (remainder)
        expResult = 2 ** 10; // 1024 (exponentiation)
    }
    
    // === Overflow Protection (Solidity 0.8+) ===
    
    function overflowDemo() public pure returns (uint8) {
        uint8 max = type(uint8).max; // 255
        // max + 1 จะ revert ใน Solidity 0.8+
        // ใน Solidity 0.7 และต่ำกว่า จะ overflow เป็น 0!
        return max; // 255
    }
    
    // ถ้าต้องการ unchecked (สำหรับ gas optimization)
    function uncheckedAdd(uint8 a, uint8 b) public pure returns (uint8) {
        unchecked {
            return a + b; // อาจ overflow ได้! ระวัง
        }
    }
    
    // === Bitwise Operations ===
    
    function bitwiseDemo() public pure returns (
        uint andResult,
        uint orResult,
        uint xorResult,
        uint notResult,
        uint shiftLeft,
        uint shiftRight
    ) {
        uint a = 0xFF; // 11111111 in binary
        uint b = 0x0F; // 00001111 in binary
        
        andResult  = a & b;  // 00001111 = 15
        orResult   = a | b;  // 11111111 = 255
        xorResult  = a ^ b;  // 11110000 = 240
        notResult  = ~b;     // ...11110000 (inverts all bits)
        shiftLeft  = 1 << 8; // 256
        shiftRight = 256 >> 4; // 16
    }
    
    // === Comparison ===
    
    function compareDemo(uint a, uint b) public pure returns (
        bool eq,
        bool neq,
        bool lt,
        bool lte,
        bool gt,
        bool gte
    ) {
        eq  = a == b;
        neq = a != b;
        lt  = a < b;
        lte = a <= b;
        gt  = a > b;
        gte = a >= b;
    }
    
    // === Type Bounds ===
    
    function typeBounds() public pure returns (
        uint256 uintMin,
        uint256 uintMax,
        int256 intMin,
        int256 intMax
    ) {
        uintMin = type(uint256).min; // 0
        uintMax = type(uint256).max; // 2^256 - 1
        intMin  = type(int256).min;  // -2^255
        intMax  = type(int256).max;  // 2^255 - 1
    }
}
```

### Boolean

```solidity
contract BooleanTypes {
    
    bool public flag1 = true;
    bool public flag2 = false;
    
    // Logical operators
    function logicalDemo(bool a, bool b) public pure returns (
        bool andResult,
        bool orResult,
        bool notResult
    ) {
        andResult = a && b;  // AND
        orResult  = a || b;  // OR
        notResult = !a;      // NOT
        
        // Short-circuit evaluation:
        // a && b → ถ้า a = false ไม่ต้องประเมิน b
        // a || b → ถ้า a = true ไม่ต้องประเมิน b
    }
}
```

### Address Types

```solidity
contract AddressTypes {
    
    // address พื้นฐาน
    address public someAddress = 0x742d35Cc6634C0532925a3b8D4C9Db96590c6F8f;
    
    // address payable - สามารถส่ง ETH ได้
    address payable public payableAddr;
    
    constructor() {
        payableAddr = payable(msg.sender);
    }
    
    // Properties ของ address
    function addressProperties(address addr) public view returns (
        uint256 balance,
        uint256 codeSize
    ) {
        balance = addr.balance; // ยอด ETH ใน Wei
        
        // ตรวจสอบว่าเป็น Contract หรือ EOA
        uint256 size;
        assembly {
            size := extcodesize(addr)
        }
        codeSize = size; // 0 = EOA, >0 = Contract
    }
    
    // ส่ง ETH ด้วยวิธีต่างๆ
    function sendETH(address payable to) public payable {
        // วิธีที่ 1: transfer (2300 gas, revert on fail) - deprecated
        // to.transfer(msg.value);
        
        // วิธีที่ 2: send (2300 gas, returns bool) - deprecated  
        // bool success = to.send(msg.value);
        
        // วิธีที่ 3: call (แนะนำ)
        (bool success, ) = to.call{value: msg.value}("");
        require(success, "Transfer failed");
    }
    
    // ตรวจสอบว่าเป็น Contract
    function isContract(address addr) public view returns (bool) {
        uint256 size;
        assembly {
            size := extcodesize(addr)
        }
        return size > 0;
    }
    
    // แปลง address เป็น uint256
    function addressToUint(address addr) public pure returns (uint256) {
        return uint256(uint160(addr));
    }
    
    // แปลง uint256 เป็น address
    function uintToAddress(uint256 num) public pure returns (address) {
        return address(uint160(num));
    }
}
```

### Bytes Types

```solidity
contract BytesTypes {
    
    // Fixed-size bytes (bytes1 ถึง bytes32)
    bytes1 public b1 = 0x41;  // 'A'
    bytes2 public b2 = 0x4142; // 'AB'
    bytes32 public b32;
    
    // Dynamic bytes
    bytes public dynamicBytes;
    
    constructor() {
        b32 = keccak256(abi.encodePacked("Hello, World!"));
        dynamicBytes = hex"48656c6c6f"; // "Hello"
    }
    
    // Bytes operations
    function bytesDemo() public pure returns (
        bytes1 firstByte,
        uint8 asUint,
        bytes32 hash
    ) {
        bytes memory data = "Hello, World!";
        
        firstByte = data[0]; // 0x48 = 'H'
        asUint = uint8(data[0]); // 72
        hash = keccak256(data);
    }
    
    // Concatenate bytes
    function concatenate(bytes memory a, bytes memory b) public pure returns (bytes memory) {
        return bytes.concat(a, b);
    }
    
    // bytes32 ← → string
    function bytes32ToString(bytes32 b) public pure returns (string memory) {
        uint8 i = 0;
        while(i < 32 && b[i] != 0) {
            i++;
        }
        bytes memory byteArray = new bytes(i);
        for (uint8 j = 0; j < i; j++) {
            byteArray[j] = b[j];
        }
        return string(byteArray);
    }
}
```

### String

```solidity
contract StringTypes {
    
    string public greeting = "Hello, World!";
    
    // String operations
    function stringDemo() public pure returns (
        string memory upper,
        uint256 length,
        bytes memory asBytes
    ) {
        string memory s = "Hello, Solidity!";
        
        // String ใน Solidity ทำอะไรได้น้อยมาก
        // ต้องแปลงเป็น bytes ก่อน
        asBytes = bytes(s);
        length = asBytes.length;
        
        // ไม่มี built-in string manipulation!
        // ต้องใช้ library หรือแปลง
        upper = s; // ตัวอย่างเท่านั้น
    }
    
    // Concatenate strings
    function concatenate(string memory a, string memory b) public pure returns (string memory) {
        return string.concat(a, b);
        // หรือ: return string(bytes.concat(bytes(a), bytes(b)));
    }
    
    // Compare strings
    function compare(string memory a, string memory b) public pure returns (bool) {
        return keccak256(bytes(a)) == keccak256(bytes(b));
    }
    
    // Check empty string
    function isEmpty(string memory s) public pure returns (bool) {
        return bytes(s).length == 0;
    }
    
    // String length (character count)
    function length(string memory s) public pure returns (uint256) {
        return bytes(s).length; // นับ bytes ไม่ใช่ characters (UTF-8)
    }
}
```

---

## 3. Reference Types

### Arrays

```solidity
contract ArrayTypes {
    
    // === Static Arrays ===
    
    uint[5] public fixedArray = [1, 2, 3, 4, 5];
    
    function fixedArrayDemo() public view returns (uint length, uint firstElement) {
        length = fixedArray.length; // 5 (constant)
        firstElement = fixedArray[0]; // 1
    }
    
    // === Dynamic Arrays ===
    
    uint[] public dynamicArray;
    address[] public addressList;
    
    function dynamicArrayDemo() public {
        // Push
        dynamicArray.push(10);
        dynamicArray.push(20);
        dynamicArray.push(30);
        
        // Pop (ลบตัวสุดท้าย)
        dynamicArray.pop(); // ลบ 30
        
        // Length
        uint len = dynamicArray.length; // 2
        
        // Update element
        dynamicArray[0] = 100;
        
        // Delete element (set to default value, ไม่ลด length)
        delete dynamicArray[0]; // dynamicArray[0] = 0
    }
    
    // ลบ element กลาง array (ไม่รักษา order)
    function removeElement(uint index) public {
        require(index < dynamicArray.length, "Index out of bounds");
        
        // ย้าย element สุดท้ายมาแทน
        dynamicArray[index] = dynamicArray[dynamicArray.length - 1];
        dynamicArray.pop();
    }
    
    // ลบ element และรักษา order (แพงกว่า)
    function removeElementOrdered(uint index) public {
        require(index < dynamicArray.length, "Index out of bounds");
        
        for (uint i = index; i < dynamicArray.length - 1; i++) {
            dynamicArray[i] = dynamicArray[i + 1];
        }
        dynamicArray.pop();
    }
    
    // === 2D Arrays ===
    
    uint[][] public matrix;
    
    function matrixDemo() public {
        uint[] memory row1 = new uint[](3);
        row1[0] = 1; row1[1] = 2; row1[2] = 3;
        
        uint[] memory row2 = new uint[](3);
        row2[0] = 4; row2[1] = 5; row2[2] = 6;
        
        matrix.push(row1);
        matrix.push(row2);
        
        // Access: matrix[row][col]
        uint element = matrix[0][1]; // 2
    }
    
    // === Memory Arrays ===
    
    function memoryArrayDemo(uint size) public pure returns (uint[] memory) {
        // Memory arrays ต้อง new และกำหนด size
        uint[] memory result = new uint[](size);
        
        for (uint i = 0; i < size; i++) {
            result[i] = i * i; // 0, 1, 4, 9, 16, ...
        }
        
        return result;
    }
    
    // Return array
    function getAll() public view returns (uint[] memory) {
        return dynamicArray;
    }
    
    // Check if element exists
    function contains(uint value) public view returns (bool) {
        for (uint i = 0; i < dynamicArray.length; i++) {
            if (dynamicArray[i] == value) {
                return true;
            }
        }
        return false;
    }
}
```

### Mappings

```solidity
contract MappingTypes {
    
    // Basic mapping
    mapping(address => uint256) public balances;
    
    // Mapping ซ้อนกัน (nested mapping)
    mapping(address => mapping(address => uint256)) public allowances;
    
    // Mapping กับ struct
    struct User {
        string name;
        uint256 score;
        bool active;
    }
    mapping(address => User) public users;
    
    // Mapping กับ array
    mapping(address => uint256[]) public userTransactions;
    
    // Enumerable mapping (ต้องเก็บ keys แยก)
    address[] public accountList;
    mapping(address => bool) private _exists;
    
    function setBalance(address addr, uint256 amount) public {
        balances[addr] = amount;
        
        // Track new address
        if (!_exists[addr]) {
            _exists[addr] = true;
            accountList.push(addr);
        }
    }
    
    function setAllowance(address owner, address spender, uint256 amount) public {
        allowances[owner][spender] = amount;
    }
    
    function getAllAccounts() public view returns (address[] memory) {
        return accountList;
    }
    
    // Mapping ไม่สามารถ iterate ได้โดยตรง!
    // ต้องเก็บ keys แยกต่างหาก
    function getTotalBalance() public view returns (uint256 total) {
        for (uint i = 0; i < accountList.length; i++) {
            total += balances[accountList[i]];
        }
    }
}
```

### Structs

```solidity
contract StructTypes {
    
    // Define struct
    struct Person {
        string name;
        uint256 age;
        address wallet;
        bool active;
    }
    
    struct Product {
        uint256 id;
        string name;
        uint256 price;
        uint256 stock;
        address seller;
    }
    
    // State variables
    Person[] public persons;
    mapping(uint256 => Product) public products;
    uint256 public productCount;
    
    // Create struct
    function addPerson(string memory name, uint256 age) public {
        // วิธีที่ 1: ใช้ named fields
        persons.push(Person({
            name: name,
            age: age,
            wallet: msg.sender,
            active: true
        }));
        
        // วิธีที่ 2: ตำแหน่ง
        // persons.push(Person(name, age, msg.sender, true));
    }
    
    function addProduct(string memory name, uint256 price, uint256 stock) public {
        productCount++;
        products[productCount] = Product({
            id: productCount,
            name: name,
            price: price,
            stock: stock,
            seller: msg.sender
        });
    }
    
    // อ่าน struct
    function getPerson(uint256 index) public view returns (
        string memory name,
        uint256 age,
        address wallet,
        bool active
    ) {
        Person memory p = persons[index];
        return (p.name, p.age, p.wallet, p.active);
    }
    
    // แก้ไข struct ใน storage
    function updatePersonAge(uint256 index, uint256 newAge) public {
        persons[index].age = newAge; // direct update
    }
    
    // ต้องระวัง! memory copy
    function wrongUpdate(uint256 index, uint256 newAge) public {
        Person memory p = persons[index]; // copy ไปยัง memory
        p.age = newAge; // แก้ memory copy เท่านั้น!
        // persons[index] ยังไม่เปลี่ยน!
    }
    
    // วิธีถูก: ใช้ storage pointer
    function correctUpdate(uint256 index, uint256 newAge) public {
        Person storage p = persons[index]; // pointer ไปยัง storage
        p.age = newAge; // แก้โดยตรงใน storage
    }
}
```

---

## 4. Special Global Variables

```solidity
contract GlobalVariables {
    
    // Variables เกี่ยวกับ Transaction/Message
    
    function msgVariables() public payable returns (
        address sender,
        uint256 value,
        bytes memory data,
        uint256 gas
    ) {
        sender = msg.sender;  // address ที่เรียก function นี้
        value  = msg.value;   // ETH ที่ส่งมา (in Wei)
        data   = msg.data;    // calldata ทั้งหมด (raw)
        gas    = gasleft();   // gas ที่เหลืออยู่
    }
    
    // Variables เกี่ยวกับ Transaction
    
    function txVariables() public view returns (
        address origin,
        uint256 gasPrice
    ) {
        origin   = tx.origin;   // ต้นทางดั้งเดิม (EOA ที่ sign)
        gasPrice = tx.gasprice; // Gas price ของ transaction นี้
    }
    
    // Variables เกี่ยวกับ Block
    
    function blockVariables() public view returns (
        uint256 blockNumber,
        uint256 timestamp,
        address miner,
        uint256 difficulty,
        uint256 gasLimit,
        bytes32 blockHash
    ) {
        blockNumber = block.number;     // หมายเลข block ปัจจุบัน
        timestamp   = block.timestamp;  // Unix timestamp (วินาที)
        miner       = block.coinbase;   // address ของ validator
        difficulty  = block.prevrandao; // (ใน PoS แทน block.difficulty)
        gasLimit    = block.gaslimit;   // gas limit ของ block
        
        // Block hash ของ block ที่ผ่านมา (เฉพาะ 256 blocks ล่าสุด)
        blockHash = blockhash(block.number - 1);
    }
    
    // ⚠️ ระวัง: block.timestamp สามารถถูก manipulate ได้เล็กน้อย
    // ไม่ควรใช้สำหรับ random number generation!
    
    // ABI Encoding Functions
    
    function abiEncodingDemo() public pure returns (
        bytes memory encoded1,
        bytes memory encoded2,
        bytes32 hash1,
        bytes32 hash2
    ) {
        uint256 a = 100;
        string memory s = "Hello";
        
        // abi.encode - เหมือน ABI encoding มาตรฐาน
        encoded1 = abi.encode(a, s);
        
        // abi.encodePacked - compact encoding ไม่มี padding
        encoded2 = abi.encodePacked(a, s);
        
        // keccak256 hash
        hash1 = keccak256(abi.encode(a, s));
        hash2 = keccak256(abi.encodePacked(a, s));
    }
    
    // Cryptographic Functions
    
    function cryptoDemo(bytes memory data) public pure returns (
        bytes32 keccakHash,
        bytes32 sha256Hash,
        bytes20 ripemd160Hash
    ) {
        keccakHash   = keccak256(data);           // Ethereum's hash function
        sha256Hash   = sha256(data);               // SHA-256
        ripemd160Hash = ripemd160(data);           // RIPEMD-160
    }
}
```

---

## 5. Constants และ Immutables

```solidity
contract ConstantsAndImmutables {
    
    // === CONSTANT ===
    // กำหนดค่าตอน compile time, ไม่สามารถเปลี่ยนได้
    // ไม่ใช้ Storage slot!
    
    uint256 public constant MAX_SUPPLY = 1_000_000;
    uint256 public constant DECIMALS = 18;
    string public constant NAME = "MyToken";
    address public constant BURN_ADDRESS = 0x000000000000000000000000000000000000dEaD;
    bytes32 public constant ROLE_ADMIN = keccak256("ADMIN_ROLE");
    
    // === IMMUTABLE ===
    // กำหนดค่าใน Constructor ครั้งเดียว, ไม่สามารถเปลี่ยนได้
    // ไม่ใช้ Storage slot! (inlined ใน bytecode)
    
    address public immutable OWNER;
    uint256 public immutable DEPLOYMENT_TIME;
    uint256 public immutable CHAIN_ID;
    
    constructor() {
        OWNER = msg.sender;
        DEPLOYMENT_TIME = block.timestamp;
        CHAIN_ID = block.chainid;
    }
    
    // === ทำไมต้องใช้ constant/immutable? ===
    // Gas Savings!
    
    // ❌ แบบนี้ใช้ SLOAD (2100 gas)
    address public owner_expensive;
    
    // ✅ แบบนี้ฟรี! (ค่าอยู่ใน bytecode)
    address public immutable owner_cheap;
    
    // การใช้งาน
    modifier onlyOwner() {
        require(msg.sender == OWNER, "Not owner");
        _;
    }
    
    function getMaxMintAmount() public pure returns (uint256) {
        return MAX_SUPPLY / 100; // 10,000
    }
}
```

---

## 6. State Variables vs Local Variables vs Function Parameters

```solidity
contract VariableLocations {
    
    // === STATE VARIABLES ===
    // เก็บใน Blockchain Storage (ถาวร, แพง)
    
    uint256 public stateVar = 100;  // storage
    
    // === DATA LOCATIONS ===
    // storage, memory, calldata
    
    uint[] public storageArray; // ใน storage
    
    function locationDemo(
        uint[] calldata calldataArray, // calldata: read-only, ถูกที่สุด
        uint   normalParam             // value type: copy
    ) public {
        
        // LOCAL VARIABLES ใน function
        uint localVar = 42; // memory (value types)
        
        // Memory array: ชั่วคราว ใช้ใน function เท่านั้น
        uint[] memory memArray = new uint[](5);
        memArray[0] = 1;
        
        // Storage reference (pointer ไปยัง storage)
        uint[] storage storageRef = storageArray;
        storageRef.push(99); // แก้ storage โดยตรง!
        
        // ✅ calldata ถูกกว่า memory สำหรับ input parameters
        // เพราะไม่ต้อง copy ข้อมูล
        
        uint sum = 0;
        for (uint i = 0; i < calldataArray.length; i++) {
            sum += calldataArray[i]; // อ่าน calldata โดยตรง
        }
    }
    
    // Calldata vs Memory สำหรับ string/bytes
    
    // ❌ ใช้ memory (ต้อง copy)
    function processMemory(string memory s) public pure returns (uint) {
        return bytes(s).length;
    }
    
    // ✅ ใช้ calldata (ไม่ต้อง copy, ถูกกว่า)
    function processCalldata(string calldata s) public pure returns (uint) {
        return bytes(s).length;
    }
    
    // Nested struct ใน storage
    struct Info {
        uint256 value;
        string name;
    }
    
    Info public info;
    
    // ❌ อ่านทั้ง struct มา memory แล้วแก้
    function updateWrong() public {
        Info memory temp = info; // copy storage → memory
        temp.value = 100;
        info = temp; // copy memory → storage (2 SSTORE)
    }
    
    // ✅ แก้โดยตรงใน storage
    function updateCorrect() public {
        info.value = 100; // 1 SSTORE
    }
}
```

---

## 7. Type Conversions

```solidity
contract TypeConversions {
    
    // === Implicit Conversions (อัตโนมัติ) ===
    // เฉพาะเมื่อไม่มีการสูญเสียข้อมูล
    
    function implicitConversion() public pure returns (uint256) {
        uint8 small = 100;
        uint256 large = small; // ✅ implicit: uint8 → uint256 (ปลอดภัย)
        return large;
    }
    
    // === Explicit Conversions (Manual Cast) ===
    
    function explicitConversion() public pure returns (
        uint8 truncated,
        int8 signed,
        uint256 fromBool,
        address fromUint
    ) {
        uint256 large = 300;
        truncated = uint8(large); // 300 % 256 = 44 (ข้อมูลสูญหาย!)
        
        uint8 u = 200;
        signed = int8(u); // 200 → -56 (overflow)
        
        bool b = true;
        // fromBool = uint256(b); // ❌ ทำไม่ได้โดยตรง
        fromBool = b ? 1 : 0; // ✅ แบบนี้
        
        uint160 addr = 0x742d35Cc6634C0532925a3b8D4C9Db96590c6F8f;
        fromUint = address(addr); // uint160 → address
    }
    
    // Integer → Bytes
    function intToBytes() public pure returns (
        bytes32 b32,
        bytes1 b1
    ) {
        uint256 n = 256;
        b32 = bytes32(n); // left-padded
        b1  = bytes1(uint8(n)); // truncate to 1 byte = 0
    }
    
    // Bytes → Integer
    function bytesToInt() public pure returns (uint256) {
        bytes32 b = 0x0000000000000000000000000000000000000000000000000000000000000064;
        return uint256(b); // 100
    }
    
    // Address Conversions
    function addressConversions() public view returns (
        uint256 asUint,
        bytes20 asBytes,
        address payable asPayable
    ) {
        address addr = msg.sender;
        asUint    = uint256(uint160(addr));
        asBytes   = bytes20(addr);
        asPayable = payable(addr);
    }
    
    // String ← → Bytes
    function stringBytesConversion(string memory s) public pure returns (
        bytes memory asBytes,
        string memory backToString
    ) {
        asBytes = bytes(s);
        backToString = string(asBytes);
    }
}
```

---

## 8. Packed Encoding และ Storage Layout

```solidity
contract StorageLayout {
    
    // Solidity เก็บ State Variables ใน Storage Slots
    // แต่ละ Slot = 32 bytes (256 bits)
    
    // ❌ Unoptimized: ใช้ 4 slots
    uint128 public a_unopt; // slot 0 (ใช้ 16 bytes แต่ครอง 32 bytes)
    uint256 public b_unopt; // slot 1 (32 bytes)
    uint128 public c_unopt; // slot 2 (ใช้ 16 bytes แต่ครอง 32 bytes)
    uint8   public d_unopt; // slot 3 (ใช้ 1 byte แต่ครอง 32 bytes)
    
    // ✅ Optimized: ใช้ 2 slots (Packing)
    uint128 public a_opt;   // slot 0 (16 bytes)
    uint128 public c_opt;   // slot 0 (รวม 32 bytes = 1 slot เต็ม)
    uint256 public b_opt;   // slot 1 (32 bytes)
    uint8   public d_opt;   // slot 2 (1 byte - ต้องอยู่ท้าย)
    
    // ตัวอย่างจริง: ERC-20 Token Packing
    struct PackedUserInfo {
        uint128 balance;      // 16 bytes ┐
        uint64  lastUpdate;   // 8 bytes  │ รวม = 32 bytes = 1 slot
        uint32  multiplier;   // 4 bytes  │
        uint32  tier;         // 4 bytes  ┘
    }
    
    // ตรวจสอบ Slot ด้วย Assembly
    function getSlot(uint256 slotNumber) public view returns (bytes32 value) {
        assembly {
            value := sload(slotNumber)
        }
    }
    
    // Slot ของแต่ละ Variable
    // a_unopt → slot 0
    // b_unopt → slot 1
    // c_unopt → slot 2
    // d_unopt → slot 3
    
    // Dynamic arrays: slot เก็บ length, data อยู่ที่ keccak256(slot)
    uint[] public dynamicArray;
    
    // Mapping: data อยู่ที่ keccak256(key . slot)
    mapping(address => uint) public balanceMap;
}
```

---

## 9. Workshop: ระบบจัดการข้อมูลนักศึกษา

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title StudentRegistry
 * @dev ระบบจัดการข้อมูลนักศึกษาบน Blockchain
 */
contract StudentRegistry {
    
    // === Types ===
    
    enum Grade { F, D, C, B, A }
    
    struct Student {
        uint256 id;
        string name;
        address wallet;
        uint8 age;
        Grade currentGrade;
        uint256[] courseScores; // คะแนนแต่ละวิชา
        bool enrolled;
        uint256 enrollmentTime;
    }
    
    struct Course {
        uint256 id;
        string name;
        uint256 maxStudents;
        address[] enrolledStudents;
        bool active;
    }
    
    // === State Variables ===
    
    address public immutable ADMIN;
    uint256 public studentCount;
    uint256 public courseCount;
    
    mapping(uint256 => Student) public students;
    mapping(address => uint256) public walletToStudentId;
    mapping(uint256 => Course) public courses;
    mapping(uint256 => mapping(uint256 => bool)) public studentInCourse; // studentId → courseId
    
    // Constants
    uint256 public constant MAX_STUDENTS = 10000;
    uint8 public constant MIN_PASSING_SCORE = 60;
    
    // === Events ===
    
    event StudentEnrolled(uint256 indexed studentId, string name, address wallet);
    event StudentGradeUpdated(uint256 indexed studentId, Grade newGrade);
    event CourseCreated(uint256 indexed courseId, string name);
    event StudentAddedToCourse(uint256 indexed studentId, uint256 indexed courseId);
    
    // === Constructor ===
    
    constructor() {
        ADMIN = msg.sender;
    }
    
    // === Modifiers ===
    
    modifier onlyAdmin() {
        require(msg.sender == ADMIN, "Only admin");
        _;
    }
    
    modifier studentExists(uint256 studentId) {
        require(studentId > 0 && studentId <= studentCount, "Student not found");
        require(students[studentId].enrolled, "Student not enrolled");
        _;
    }
    
    // === Functions ===
    
    function enrollStudent(
        string calldata name,
        uint8 age
    ) external returns (uint256 studentId) {
        require(studentCount < MAX_STUDENTS, "Max students reached");
        require(bytes(name).length > 0, "Name cannot be empty");
        require(bytes(name).length <= 100, "Name too long");
        require(age >= 15 && age <= 100, "Invalid age");
        require(walletToStudentId[msg.sender] == 0, "Already enrolled");
        
        studentCount++;
        studentId = studentCount;
        
        students[studentId] = Student({
            id: studentId,
            name: name,
            wallet: msg.sender,
            age: age,
            currentGrade: Grade.C,
            courseScores: new uint256[](0),
            enrolled: true,
            enrollmentTime: block.timestamp
        });
        
        walletToStudentId[msg.sender] = studentId;
        
        emit StudentEnrolled(studentId, name, msg.sender);
    }
    
    function createCourse(
        string calldata name,
        uint256 maxStudents
    ) external onlyAdmin returns (uint256 courseId) {
        require(bytes(name).length > 0, "Course name required");
        require(maxStudents > 0 && maxStudents <= 1000, "Invalid max students");
        
        courseCount++;
        courseId = courseCount;
        
        courses[courseId] = Course({
            id: courseId,
            name: name,
            maxStudents: maxStudents,
            enrolledStudents: new address[](0),
            active: true
        });
        
        emit CourseCreated(courseId, name);
    }
    
    function addStudentToCourse(
        uint256 studentId,
        uint256 courseId
    ) external onlyAdmin studentExists(studentId) {
        require(courseId > 0 && courseId <= courseCount, "Course not found");
        require(courses[courseId].active, "Course not active");
        require(!studentInCourse[studentId][courseId], "Already in course");
        
        Course storage course = courses[courseId];
        require(course.enrolledStudents.length < course.maxStudents, "Course full");
        
        course.enrolledStudents.push(students[studentId].wallet);
        studentInCourse[studentId][courseId] = true;
        
        emit StudentAddedToCourse(studentId, courseId);
    }
    
    function addScore(
        uint256 studentId,
        uint256 score
    ) external onlyAdmin studentExists(studentId) {
        require(score <= 100, "Score must be 0-100");
        
        students[studentId].courseScores.push(score);
        
        // Update grade based on average
        Grade newGrade = calculateGrade(studentId);
        students[studentId].currentGrade = newGrade;
        
        emit StudentGradeUpdated(studentId, newGrade);
    }
    
    function calculateGrade(uint256 studentId) public view returns (Grade) {
        uint256[] storage scores = students[studentId].courseScores;
        if (scores.length == 0) return Grade.C;
        
        uint256 total = 0;
        for (uint256 i = 0; i < scores.length; i++) {
            total += scores[i];
        }
        
        uint256 avg = total / scores.length;
        
        if (avg >= 90) return Grade.A;
        if (avg >= 80) return Grade.B;
        if (avg >= 70) return Grade.C;
        if (avg >= 60) return Grade.D;
        return Grade.F;
    }
    
    function getStudentInfo(uint256 studentId) external view 
        studentExists(studentId) 
        returns (
            uint256 id,
            string memory name,
            address wallet,
            uint8 age,
            Grade grade,
            uint256 averageScore,
            bool enrolled
        ) 
    {
        Student storage s = students[studentId];
        
        uint256 avg = 0;
        if (s.courseScores.length > 0) {
            uint256 total = 0;
            for (uint256 i = 0; i < s.courseScores.length; i++) {
                total += s.courseScores[i];
            }
            avg = total / s.courseScores.length;
        }
        
        return (s.id, s.name, s.wallet, s.age, s.currentGrade, avg, s.enrolled);
    }
    
    function getMyStudentId() external view returns (uint256) {
        uint256 id = walletToStudentId[msg.sender];
        require(id != 0, "Not enrolled");
        return id;
    }
    
    function getCourseStudents(uint256 courseId) external view returns (address[] memory) {
        require(courseId > 0 && courseId <= courseCount, "Course not found");
        return courses[courseId].enrolledStudents;
    }
    
    function isPassingGrade(Grade grade) public pure returns (bool) {
        return grade != Grade.F;
    }
    
    function gradeToString(Grade grade) public pure returns (string memory) {
        if (grade == Grade.A) return "A";
        if (grade == Grade.B) return "B";
        if (grade == Grade.C) return "C";
        if (grade == Grade.D) return "D";
        return "F";
    }
}
```

---

## 10. Gas Optimization Tips สำหรับ Types

```solidity
contract GasOptimizationTips {
    
    // ✅ TIP 1: ใช้ uint256 แทน uint8/uint16
    // Solidity ต้อง pad uint8 เป็น 256 bits ก่อนคำนวณ
    // ทำให้บางกรณี uint256 ถูกกว่า uint8
    
    uint8 public u8_var;    // อาจแพงกว่า!
    uint256 public u256_var; // บ่อยครั้งถูกกว่า
    
    // ✅ TIP 2: Pack Variables ใน Struct
    
    // ❌ ไม่ดี: 3 storage slots
    struct Unpacked {
        uint128 a; // slot 0 (ครอง 32 bytes แต่ใช้ 16)
        uint256 b; // slot 1 (32 bytes)
        uint128 c; // slot 2 (ครอง 32 bytes แต่ใช้ 16)
    }
    
    // ✅ ดี: 2 storage slots
    struct Packed {
        uint128 a; // slot 0 (16 bytes) ┐
        uint128 c; // slot 0 (16 bytes) ┘ = 1 slot
        uint256 b; // slot 1 (32 bytes)
    }
    
    // ✅ TIP 3: ใช้ calldata แทน memory สำหรับ input
    
    function expensiveInput(uint256[] memory arr) public pure returns (uint256) {
        // memory: copy calldata → memory
        uint256 sum = 0;
        for (uint256 i = 0; i < arr.length; i++) sum += arr[i];
        return sum;
    }
    
    function cheapInput(uint256[] calldata arr) public pure returns (uint256) {
        // calldata: อ่านโดยตรง ไม่ต้อง copy
        uint256 sum = 0;
        for (uint256 i = 0; i < arr.length; i++) sum += arr[i];
        return sum;
    }
    
    // ✅ TIP 4: Cache array length ใน loop
    
    uint256[] public bigArray;
    
    function inefficientLoop() public view returns (uint256) {
        uint256 sum = 0;
        for (uint256 i = 0; i < bigArray.length; i++) { // SLOAD ทุก iteration!
            sum += bigArray[i];
        }
        return sum;
    }
    
    function efficientLoop() public view returns (uint256) {
        uint256 len = bigArray.length; // SLOAD ครั้งเดียว
        uint256 sum = 0;
        for (uint256 i = 0; i < len; i++) {
            sum += bigArray[i];
        }
        return sum;
    }
    
    // ✅ TIP 5: ใช้ constant สำหรับค่าที่ไม่เปลี่ยน
    
    uint256 public maxSupply = 1000000; // ❌ SLOAD ทุกครั้ง
    uint256 public constant MAX_SUPPLY = 1000000; // ✅ compile-time constant
    
    // ✅ TIP 6: short-circuit evaluation
    
    function checkConditions(address addr, uint256 amount) public pure returns (bool) {
        // ✅ ตรวจสอบเงื่อนไขง่ายก่อน (ถ้า false ไม่ต้องทำอย่างอื่น)
        return addr != address(0) && amount > 0 && amount < type(uint256).max;
    }
}
```

---

## สรุป Part 03

Types ที่เรียนรู้:
- ✅ Integer types (uint, int, variants)
- ✅ Boolean
- ✅ Address (plain และ payable)
- ✅ Bytes (fixed และ dynamic)
- ✅ String
- ✅ Arrays (static, dynamic, 2D)
- ✅ Mappings
- ✅ Structs
- ✅ Enums
- ✅ Constants และ Immutables
- ✅ Data Locations (storage, memory, calldata)
- ✅ Type Conversions
- ✅ Storage Layout

## Quiz

1. ความแตกต่างระหว่าง `storage`, `memory`, และ `calldata` คืออะไร?
2. ทำไม Mapping ถึง iterate ไม่ได้?
3. `uint8 x = 255; x++;` จะเกิดอะไร?
4. Struct packing ช่วย optimize gas อย่างไร?

## แบบฝึกหัด

1. สร้าง Contract `BankAccount` ที่มี:
   - `deposit()` รับ ETH
   - `withdraw(uint amount)` ถอน ETH
   - `getBalance()` ดูยอด
   - `getHistory()` ดูประวัติการทำรายการ (array ของ structs)

2. สร้าง Contract `Inventory` สำหรับร้านค้า:
   - Struct `Item` (id, name, price, stock)
   - เพิ่ม/ลด stock
   - ค้นหา item ด้วย id
   - Enumerable list ของ items

---

## Next: Part 04 - Functions และ Modifiers
