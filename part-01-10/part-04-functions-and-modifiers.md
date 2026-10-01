# Part 04: Functions และ Modifiers

## สารบัญ
1. Function Syntax และ Visibility
2. Function Mutability
3. Return Values
4. Modifiers
5. Function Overloading
6. Fallback และ Receive Functions
7. External vs Internal Calls
8. Function Selectors และ ABI
9. Virtual Functions และ Override
10. Workshop: Access Control System

---

## 1. Function Syntax และ Visibility

### โครงสร้าง Function

```
function <name>(<parameters>) <visibility> <mutability> <modifiers> returns (<types>) {
    // body
}
```

### Visibility Levels

```
Visibility:
┌─────────────────────────────────────────────────────────────────┐
│  public    │ เรียกได้จากทุกที่ (ภายใน + ภายนอก + contracts อื่น) │
│  external  │ เรียกได้จากภายนอกเท่านั้น (ไม่ใช่ภายใน)           │
│  internal  │ ภายใน contract + child contracts เท่านั้น           │
│  private   │ เฉพาะ contract นี้เท่านั้น                          │
└─────────────────────────────────────────────────────────────────┘
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract VisibilityDemo {
    
    uint256 private _secretValue = 42;
    uint256 internal _sharedValue = 100;
    uint256 public publicValue = 200;
    
    // public: เรียกได้จากทุกที่
    function publicFunc() public pure returns (string memory) {
        return "I'm public";
    }
    
    // external: เรียกได้จากภายนอกเท่านั้น
    // ถูกกว่า public สำหรับ large data (ใช้ calldata)
    function externalFunc() external pure returns (string memory) {
        return "I'm external";
    }
    
    // internal: ภายใน + child contracts
    function internalFunc() internal pure returns (string memory) {
        return "I'm internal";
    }
    
    // private: เฉพาะ contract นี้เท่านั้น
    function privateFunc() private pure returns (string memory) {
        return "I'm private";
    }
    
    // เรียก internal/private จากภายใน
    function callInternal() public pure returns (string memory) {
        return internalFunc(); // ✅
        // return externalFunc(); // ❌ ต้อง this.externalFunc()
    }
    
    // เรียก external ด้วย this.
    function callExternal() public view returns (string memory) {
        return this.externalFunc(); // ✅ แต่แพงกว่า (external call)
    }
    
    // Gas Comparison:
    // external function รับ large arrays ถูกกว่า public
    // เพราะ calldata ไม่ต้อง copy ไปยัง memory
    
    function processPublic(uint256[] memory data) public pure returns (uint256) {
        uint256 sum = 0;
        for (uint i = 0; i < data.length; i++) sum += data[i];
        return sum;
    }
    
    function processExternal(uint256[] calldata data) external pure returns (uint256) {
        uint256 sum = 0;
        for (uint i = 0; i < data.length; i++) sum += data[i];
        return sum; // ถูกกว่า processPublic!
    }
}

// Child contract สามารถเข้าถึง internal ได้
contract ChildContract is VisibilityDemo {
    
    function useParent() public pure returns (string memory) {
        return internalFunc(); // ✅ เข้าถึง internal ได้
        // return privateFunc(); // ❌ ไม่ได้ เป็น private
    }
    
    function readShared() public view returns (uint256) {
        return _sharedValue; // ✅ internal state variable
        // return _secretValue; // ❌ private state variable
    }
}
```

---

## 2. Function Mutability

```
Mutability:
┌──────────────────────────────────────────────────────────────────┐
│  (none)   │ ปกติ: อ่านและเขียน state ได้                        │
│  view     │ อ่านอย่างเดียว: ไม่เขียน state                       │
│  pure     │ ไม่อ่านและไม่เขียน state                            │
│  payable  │ รับ ETH ได้                                          │
└──────────────────────────────────────────────────────────────────┘
```

```solidity
contract MutabilityDemo {
    
    uint256 public counter = 0;
    mapping(address => uint256) public balances;
    
    // (none): state-changing function
    function increment() public {
        counter++;  // เขียน state
    }
    
    function deposit() public payable {
        balances[msg.sender] += msg.value;  // เขียน state + รับ ETH
    }
    
    // view: อ่าน state แต่ไม่เขียน
    // ไม่เสีย gas ถ้าเรียกนอก blockchain (off-chain)
    function getCounter() public view returns (uint256) {
        return counter;  // อ่าน state
    }
    
    function getBalance(address addr) public view returns (uint256) {
        return balances[addr];  // อ่าน state
    }
    
    // pure: ไม่อ่านและไม่เขียน state
    // ใช้แค่ parameters และ constants
    function add(uint256 a, uint256 b) public pure returns (uint256) {
        return a + b;  // ไม่ยุ่งกับ state
    }
    
    function hashData(bytes memory data) public pure returns (bytes32) {
        return keccak256(data);  // ไม่อ่าน state
    }
    
    // payable: รับ ETH ได้
    // ถ้าไม่ระบุ payable และมีคนส่ง ETH มา → revert!
    function receiveETH() public payable {
        // msg.value มีค่า
        require(msg.value > 0, "No ETH sent");
        balances[msg.sender] += msg.value;
    }
    
    // ตัวอย่าง: function ที่ไม่ใช่ payable
    function nonPayable() public pure returns (string memory) {
        // ถ้าส่ง ETH มาพร้อม transaction นี้ → revert!
        return "No ETH accepted";
    }
    
    // view function เรียกฟรี (off-chain)
    // แต่ถ้าเรียกจาก contract อื่น มี gas cost
    
    // What view/pure CANNOT do:
    // - เขียน state variables
    // - emit events
    // - สร้าง contracts ใหม่
    // - ส่ง ETH
    // - เรียก non-view/pure functions
}
```

---

## 3. Return Values

```solidity
contract ReturnValues {
    
    // Single return value
    function singleReturn() public pure returns (uint256) {
        return 42;
    }
    
    // Multiple return values
    function multipleReturns() public pure returns (
        uint256 a,
        string memory b,
        bool c,
        address d
    ) {
        return (100, "Hello", true, address(0));
    }
    
    // Named return values
    function namedReturns() public pure returns (
        uint256 result,
        bool success
    ) {
        result = 100;    // implicit return
        success = true;  // implicit return
        // ไม่ต้องมี return statement!
    }
    
    // Mixed named returns
    function mixedReturns(uint256 x) public pure returns (
        uint256 doubled,
        bool isEven
    ) {
        doubled = x * 2;
        isEven = x % 2 == 0;
        // return doubled, isEven; // ก็ได้
    }
    
    // Return array
    function returnArray() public pure returns (uint256[] memory) {
        uint256[] memory arr = new uint256[](3);
        arr[0] = 10;
        arr[1] = 20;
        arr[2] = 30;
        return arr;
    }
    
    // Return struct
    struct Point {
        uint256 x;
        uint256 y;
    }
    
    function returnStruct() public pure returns (Point memory) {
        return Point({x: 5, y: 10});
    }
    
    // Destructure return values
    function useMultipleReturns() public view returns (uint256) {
        (uint256 a, string memory b, bool c, address d) = multipleReturns();
        
        // ถ้าไม่ต้องการบางค่า ใช้ _
        (uint256 result, ) = mixedReturns(10);
        
        return result;
    }
    
    // Early return
    function earlyReturn(uint256 x) public pure returns (string memory) {
        if (x == 0) return "zero";
        if (x < 10) return "small";
        if (x < 100) return "medium";
        return "large";
    }
}
```

---

## 4. Modifiers

Modifiers คือ **reusable code** ที่รันก่อน/หลัง function

```solidity
contract ModifierDemo {
    
    address public owner;
    bool public paused;
    mapping(address => bool) public whitelist;
    
    constructor() {
        owner = msg.sender;
    }
    
    // === Basic Modifiers ===
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _; // ← ตรงนี้คือ body ของ function
    }
    
    modifier notPaused() {
        require(!paused, "Contract is paused");
        _;
    }
    
    modifier onlyWhitelisted() {
        require(whitelist[msg.sender], "Not whitelisted");
        _;
    }
    
    // === Modifiers ที่รับ Parameters ===
    
    modifier validAmount(uint256 amount) {
        require(amount > 0, "Amount must be > 0");
        require(amount <= type(uint256).max / 2, "Amount too large");
        _;
    }
    
    modifier onlyAfter(uint256 timestamp) {
        require(block.timestamp >= timestamp, "Too early");
        _;
    }
    
    modifier onlyBefore(uint256 timestamp) {
        require(block.timestamp < timestamp, "Too late");
        _;
    }
    
    // === Modifier Order ===
    
    // รัน modifier จากซ้ายไปขวา
    function sensitiveFunction() 
        public 
        onlyOwner       // 1: ตรวจสอบ owner
        notPaused       // 2: ตรวจสอบ pause
        returns (string memory) 
    {
        return "Executed!";
    }
    
    // === Code Before และ After ===
    
    modifier withLogging() {
        // Code ก่อน function
        emit FunctionCalled(msg.sender, "before");
        
        _; // ← function body
        
        // Code หลัง function  
        emit FunctionCalled(msg.sender, "after");
    }
    
    event FunctionCalled(address caller, string when);
    
    function loggedFunction() public withLogging returns (uint256) {
        return 42;
    }
    
    // === Reentrancy Guard Modifier ===
    
    bool private _locked;
    
    modifier nonReentrant() {
        require(!_locked, "ReentrancyGuard: reentrant call");
        _locked = true;
        _;
        _locked = false;
    }
    
    uint256 public contractBalance;
    
    function deposit() public payable {
        contractBalance += msg.value;
    }
    
    function withdraw(uint256 amount) public nonReentrant {
        require(contractBalance >= amount, "Insufficient balance");
        
        contractBalance -= amount; // State change ก่อน transfer!
        
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
    
    // === Admin Functions ===
    
    function setPaused(bool _paused) public onlyOwner {
        paused = _paused;
    }
    
    function addToWhitelist(address addr) public onlyOwner {
        whitelist[addr] = true;
    }
    
    function transferOwnership(address newOwner) public onlyOwner {
        require(newOwner != address(0), "Zero address");
        owner = newOwner;
    }
    
    // === Multiple Modifiers ===
    
    uint256 public constant MIN_DEPOSIT = 0.01 ether;
    
    function adminDeposit(uint256 amount) 
        public 
        onlyOwner 
        notPaused 
        validAmount(amount) 
    {
        contractBalance += amount;
    }
}
```

---

## 5. Function Overloading

```solidity
contract Overloading {
    
    // Solidity อนุญาต Overloading (functions ชื่อเดียวกัน parameter ต่างกัน)
    
    function process(uint256 x) public pure returns (uint256) {
        return x * 2;
    }
    
    function process(uint256 x, uint256 y) public pure returns (uint256) {
        return x + y;
    }
    
    function process(string memory s) public pure returns (uint256) {
        return bytes(s).length;
    }
    
    // Solidity เลือก function ที่ match parameter types
    
    function testOverload() public pure returns (uint256, uint256, uint256) {
        uint256 r1 = process(10);        // calls process(uint256)
        uint256 r2 = process(10, 20);    // calls process(uint256, uint256)
        uint256 r3 = process("Hello");   // calls process(string)
        
        return (r1, r2, r3); // 20, 30, 5
    }
    
    // ⚠️ Overloading ต้อง parameter types ต่างกัน
    // ชนิด return value เดียวกันก็ได้
    
    // ❌ ทำไม่ได้! Return types ต่างกันไม่นับ
    // function duplicate(uint256 x) public pure returns (uint256) { return x; }
    // function duplicate(uint256 x) public pure returns (string memory) { return "x"; }
}
```

---

## 6. Fallback และ Receive Functions

```solidity
contract FallbackAndReceive {
    
    event Received(address sender, uint256 amount, bytes data);
    event FallbackCalled(address sender, uint256 amount, bytes data);
    
    // receive(): เรียกเมื่อมี plain ETH transfer (ไม่มี data)
    receive() external payable {
        emit Received(msg.sender, msg.value, "");
    }
    
    // fallback(): เรียกเมื่อ function ไม่พบ หรือมี ETH + data
    fallback() external payable {
        emit FallbackCalled(msg.sender, msg.value, msg.data);
    }
    
    // การทำงาน:
    // ส่ง ETH ไม่มี data     → receive()
    // เรียก function ไม่มี   → fallback()
    // ส่ง ETH + data         → fallback() (ถ้าไม่มี receive)
    
    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
}

// Proxy Contract ใช้ fallback
contract SimpleProxy {
    
    address public implementation;
    
    constructor(address _implementation) {
        implementation = _implementation;
    }
    
    // Delegate calls ทั้งหมดไปยัง implementation
    fallback() external payable {
        address impl = implementation;
        assembly {
            // Copy calldata
            calldatacopy(0, 0, calldatasize())
            
            // Delegate call
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            
            // Copy return data
            returndatacopy(0, 0, returndatasize())
            
            // Return or revert
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
    
    receive() external payable {}
}
```

---

## 7. External vs Internal Calls

```solidity
interface ICalculator {
    function add(uint256 a, uint256 b) external pure returns (uint256);
    function multiply(uint256 a, uint256 b) external pure returns (uint256);
}

contract Calculator is ICalculator {
    
    function add(uint256 a, uint256 b) external pure returns (uint256) {
        return a + b;
    }
    
    function multiply(uint256 a, uint256 b) external pure returns (uint256) {
        return a * b;
    }
}

contract CallerContract {
    
    ICalculator public calculator;
    
    constructor(address _calculator) {
        calculator = ICalculator(_calculator);
    }
    
    // External Call: เรียก contract อื่น
    function externalCallDemo() public view returns (uint256) {
        return calculator.add(5, 3); // External call → Gas: ~2600
    }
    
    // Low-level External Call: ใช้ call()
    function lowLevelCall(address target, bytes memory data) public returns (bool, bytes memory) {
        (bool success, bytes memory result) = target.call(data);
        return (success, result);
    }
    
    // Static Call (read-only)
    function staticCallDemo(address target) public view returns (bytes memory) {
        (bool success, bytes memory result) = target.staticcall(
            abi.encodeWithSignature("getBalance()")
        );
        require(success, "Static call failed");
        return result;
    }
    
    // Delegate Call: รัน code ของ contract อื่นใน context ของ contract นี้
    function delegateCallDemo(address target, bytes memory data) public returns (bytes memory) {
        (bool success, bytes memory result) = target.delegatecall(data);
        require(success, "Delegate call failed");
        return result;
    }
}
```

---

## 8. Function Selectors และ ABI

```solidity
contract FunctionSelectors {
    
    // Function Selector = keccak256("functionName(paramTypes)") แรก 4 bytes
    
    function getSelector(string memory sig) public pure returns (bytes4) {
        return bytes4(keccak256(bytes(sig)));
    }
    
    // ตัวอย่าง:
    // transfer(address,uint256) → 0xa9059cbb
    // balanceOf(address)        → 0x70a08231
    // approve(address,uint256)  → 0x095ea7b3
    
    function transferSelector() public pure returns (bytes4) {
        return bytes4(keccak256("transfer(address,uint256)"));
        // = 0xa9059cbb
    }
    
    // Encode function call
    function encodeCall(address to, uint256 amount) public pure returns (bytes memory) {
        return abi.encodeWithSignature("transfer(address,uint256)", to, amount);
        // = 0xa9059cbb + abi.encode(to, amount)
    }
    
    // Alternative encoding methods
    function encodeMethods(address to, uint256 amount) public pure returns (
        bytes memory method1,
        bytes memory method2,
        bytes memory method3
    ) {
        // Method 1: encodeWithSignature
        method1 = abi.encodeWithSignature("transfer(address,uint256)", to, amount);
        
        // Method 2: encodeWithSelector
        bytes4 selector = bytes4(keccak256("transfer(address,uint256)"));
        method2 = abi.encodeWithSelector(selector, to, amount);
        
        // Method 3: encodeCall (type-safe)
        // method3 = abi.encodeCall(IERC20.transfer, (to, amount));
        
        method3 = method1; // สำหรับตัวอย่างนี้
    }
    
    // Decode return data
    function decodeReturn(bytes memory returnData) public pure returns (bool) {
        return abi.decode(returnData, (bool));
    }
    
    // Decode multiple values
    function decodeMultiple(bytes memory data) public pure returns (address, uint256) {
        return abi.decode(data, (address, uint256));
    }
}
```

---

## 9. Virtual Functions และ Override

```solidity
// Base Contract
contract Animal {
    
    string public name;
    
    constructor(string memory _name) {
        name = _name;
    }
    
    // virtual: สามารถ override ได้
    function speak() public virtual returns (string memory) {
        return "...";
    }
    
    // virtual: ต้องให้ child implement
    function move() public virtual returns (string memory) {
        return "moving";
    }
    
    // non-virtual: ไม่สามารถ override
    function breathe() public pure returns (string memory) {
        return "breathing";
    }
}

contract Dog is Animal {
    
    constructor() Animal("Dog") {}
    
    // override: เขียนทับ parent function
    function speak() public override returns (string memory) {
        return "Woof!";
    }
    
    function move() public override returns (string memory) {
        return "running";
    }
}

contract Cat is Animal {
    
    constructor() Animal("Cat") {}
    
    function speak() public override returns (string memory) {
        return "Meow!";
    }
    
    // ใช้ super. เพื่อเรียก parent
    function describe() public view returns (string memory) {
        return string.concat(name, " says: Meow!");
    }
}

// Multiple Inheritance
contract Pet is Animal {
    
    string public petName;
    
    constructor(string memory _name, string memory _petName) 
        Animal(_name) 
    {
        petName = _petName;
    }
}

contract GuideDog is Dog, Pet {
    
    constructor() Dog() Pet("Dog", "Buddy") {}
    
    // ต้อง override เพราะ Dog และ Pet ต่างมี speak()
    function speak() public override(Dog, Pet) returns (string memory) {
        return "Woof! (Guide Dog)";
    }
}

// Abstract Contract
abstract contract Shape {
    
    // abstract function: ไม่มี implementation
    function area() public virtual returns (uint256);
    function perimeter() public virtual returns (uint256);
    
    // concrete function
    function describe() public virtual returns (string memory) {
        return string.concat(
            "Area: ",
            uint2str(area()),
            ", Perimeter: ",
            uint2str(perimeter())
        );
    }
    
    function uint2str(uint256 n) internal pure returns (string memory) {
        if (n == 0) return "0";
        uint256 j = n;
        uint256 len;
        while (j != 0) {
            len++;
            j /= 10;
        }
        bytes memory bstr = new bytes(len);
        uint256 k = len;
        while (n != 0) {
            k = k - 1;
            uint8 temp = (48 + uint8(n - n / 10 * 10));
            bytes1 b1 = bytes1(temp);
            bstr[k] = b1;
            n /= 10;
        }
        return string(bstr);
    }
}

contract Circle is Shape {
    
    uint256 public radius;
    uint256 private constant PI_SCALED = 314159; // π * 100000
    
    constructor(uint256 _radius) {
        radius = _radius;
    }
    
    function area() public override view returns (uint256) {
        return (PI_SCALED * radius * radius) / 100000;
    }
    
    function perimeter() public override view returns (uint256) {
        return (2 * PI_SCALED * radius) / 100000;
    }
}

contract Rectangle is Shape {
    
    uint256 public width;
    uint256 public height;
    
    constructor(uint256 _width, uint256 _height) {
        width = _width;
        height = _height;
    }
    
    function area() public override view returns (uint256) {
        return width * height;
    }
    
    function perimeter() public override view returns (uint256) {
        return 2 * (width + height);
    }
}
```

---

## 10. Workshop: Access Control System

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title AccessControl
 * @dev ระบบควบคุมการเข้าถึงแบบ Role-based
 */
contract AccessControl {
    
    // === Types ===
    
    struct RoleData {
        mapping(address => bool) members;
        bytes32 adminRole;
        uint256 memberCount;
    }
    
    // === State Variables ===
    
    mapping(bytes32 => RoleData) private _roles;
    
    // Role Constants
    bytes32 public constant DEFAULT_ADMIN_ROLE = 0x00;
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");
    bytes32 public constant OPERATOR_ROLE = keccak256("OPERATOR_ROLE");
    bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");
    
    // === Events ===
    
    event RoleGranted(bytes32 indexed role, address indexed account, address indexed sender);
    event RoleRevoked(bytes32 indexed role, address indexed account, address indexed sender);
    event RoleAdminChanged(bytes32 indexed role, bytes32 indexed previousAdminRole, bytes32 indexed newAdminRole);
    
    // === Constructor ===
    
    constructor() {
        _setupRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _setupRole(ADMIN_ROLE, msg.sender);
        
        // Set admin role for each role
        _setRoleAdmin(ADMIN_ROLE, DEFAULT_ADMIN_ROLE);
        _setRoleAdmin(OPERATOR_ROLE, ADMIN_ROLE);
        _setRoleAdmin(PAUSER_ROLE, ADMIN_ROLE);
    }
    
    // === Modifiers ===
    
    modifier onlyRole(bytes32 role) {
        _checkRole(role, msg.sender);
        _;
    }
    
    // === Core Functions ===
    
    function hasRole(bytes32 role, address account) public view returns (bool) {
        return _roles[role].members[account];
    }
    
    function getRoleAdmin(bytes32 role) public view returns (bytes32) {
        return _roles[role].adminRole;
    }
    
    function getRoleMemberCount(bytes32 role) public view returns (uint256) {
        return _roles[role].memberCount;
    }
    
    function grantRole(bytes32 role, address account) public onlyRole(getRoleAdmin(role)) {
        _grantRole(role, account);
    }
    
    function revokeRole(bytes32 role, address account) public onlyRole(getRoleAdmin(role)) {
        _revokeRole(role, account);
    }
    
    function renounceRole(bytes32 role, address account) public {
        require(account == msg.sender, "Can only renounce for self");
        _revokeRole(role, account);
    }
    
    // === Internal Functions ===
    
    function _checkRole(bytes32 role, address account) internal view {
        if (!hasRole(role, account)) {
            revert(string(abi.encodePacked(
                "AccessControl: account ",
                _toHexString(uint160(account), 20),
                " is missing role ",
                _toHexString(uint256(role), 32)
            )));
        }
    }
    
    function _setupRole(bytes32 role, address account) internal {
        _grantRole(role, account);
    }
    
    function _setRoleAdmin(bytes32 role, bytes32 adminRole) internal {
        bytes32 previousAdminRole = getRoleAdmin(role);
        _roles[role].adminRole = adminRole;
        emit RoleAdminChanged(role, previousAdminRole, adminRole);
    }
    
    function _grantRole(bytes32 role, address account) internal {
        if (!hasRole(role, account)) {
            _roles[role].members[account] = true;
            _roles[role].memberCount++;
            emit RoleGranted(role, account, msg.sender);
        }
    }
    
    function _revokeRole(bytes32 role, address account) internal {
        if (hasRole(role, account)) {
            _roles[role].members[account] = false;
            _roles[role].memberCount--;
            emit RoleRevoked(role, account, msg.sender);
        }
    }
    
    // Helper: convert to hex string
    function _toHexString(uint256 value, uint256 length) internal pure returns (string memory) {
        bytes memory buffer = new bytes(2 * length + 2);
        buffer[0] = "0";
        buffer[1] = "x";
        for (uint256 i = 2 * length + 1; i > 1; --i) {
            buffer[i] = _SYMBOLS[value & 0xf];
            value >>= 4;
        }
        require(value == 0, "Strings: hex length insufficient");
        return string(buffer);
    }
    
    bytes16 private constant _SYMBOLS = "0123456789abcdef";
}

/**
 * @title SecureVault
 * @dev ตัวอย่างการใช้ Access Control กับ Vault
 */
contract SecureVault is AccessControl {
    
    // === State ===
    
    bool public paused;
    mapping(address => uint256) public deposits;
    uint256 public totalDeposited;
    uint256 public constant MAX_DEPOSIT = 100 ether;
    
    // === Events ===
    
    event Deposited(address indexed user, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);
    event EmergencyWithdraw(address indexed operator, uint256 amount);
    event Paused(address indexed pauser);
    event Unpaused(address indexed pauser);
    
    // === Modifiers ===
    
    modifier whenNotPaused() {
        require(!paused, "Vault is paused");
        _;
    }
    
    modifier whenPaused() {
        require(paused, "Vault is not paused");
        _;
    }
    
    // === User Functions ===
    
    function deposit() external payable whenNotPaused {
        require(msg.value > 0, "No ETH sent");
        require(deposits[msg.sender] + msg.value <= MAX_DEPOSIT, "Exceeds max deposit");
        
        deposits[msg.sender] += msg.value;
        totalDeposited += msg.value;
        
        emit Deposited(msg.sender, msg.value);
    }
    
    function withdraw(uint256 amount) external whenNotPaused {
        require(deposits[msg.sender] >= amount, "Insufficient balance");
        
        deposits[msg.sender] -= amount;
        totalDeposited -= amount;
        
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
        
        emit Withdrawn(msg.sender, amount);
    }
    
    // === Operator Functions ===
    
    function emergencyWithdraw(uint256 amount) external onlyRole(OPERATOR_ROLE) {
        require(paused, "Must be paused for emergency withdraw");
        require(address(this).balance >= amount, "Insufficient vault balance");
        
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
        
        emit EmergencyWithdraw(msg.sender, amount);
    }
    
    // === Pauser Functions ===
    
    function pause() external onlyRole(PAUSER_ROLE) whenNotPaused {
        paused = true;
        emit Paused(msg.sender);
    }
    
    function unpause() external onlyRole(PAUSER_ROLE) whenPaused {
        paused = false;
        emit Unpaused(msg.sender);
    }
    
    // === Admin Functions ===
    
    function addOperator(address operator) external onlyRole(ADMIN_ROLE) {
        grantRole(OPERATOR_ROLE, operator);
    }
    
    function removeOperator(address operator) external onlyRole(ADMIN_ROLE) {
        revokeRole(OPERATOR_ROLE, operator);
    }
    
    function addPauser(address pauser) external onlyRole(ADMIN_ROLE) {
        grantRole(PAUSER_ROLE, pauser);
    }
    
    // === View Functions ===
    
    function getMyBalance() external view returns (uint256) {
        return deposits[msg.sender];
    }
    
    function getVaultBalance() external view returns (uint256) {
        return address(this).balance;
    }
    
    receive() external payable {
        // รับ ETH โดยตรง
        deposits[msg.sender] += msg.value;
        totalDeposited += msg.value;
        emit Deposited(msg.sender, msg.value);
    }
}
```

### Tests สำหรับ Workshop

```typescript
// test/SecureVault.test.ts
import { expect } from "chai";
import { ethers } from "hardhat";
import { SecureVault } from "../typechain-types";

describe("SecureVault", function () {
    let vault: SecureVault;
    let owner: any, admin: any, operator: any, pauser: any, user1: any;
    
    const ADMIN_ROLE = ethers.id("ADMIN_ROLE");
    const OPERATOR_ROLE = ethers.id("OPERATOR_ROLE");
    const PAUSER_ROLE = ethers.id("PAUSER_ROLE");
    
    beforeEach(async function () {
        [owner, admin, operator, pauser, user1] = await ethers.getSigners();
        
        const VaultFactory = await ethers.getContractFactory("SecureVault");
        vault = await VaultFactory.deploy();
        
        // Setup roles
        await vault.grantRole(PAUSER_ROLE, pauser.address);
        await vault.addOperator(operator.address);
    });
    
    describe("Deposit & Withdraw", function () {
        it("Should deposit ETH", async function () {
            const amount = ethers.parseEther("1");
            
            await expect(vault.connect(user1).deposit({ value: amount }))
                .to.emit(vault, "Deposited")
                .withArgs(user1.address, amount);
            
            expect(await vault.deposits(user1.address)).to.equal(amount);
        });
        
        it("Should withdraw ETH", async function () {
            const amount = ethers.parseEther("1");
            
            await vault.connect(user1).deposit({ value: amount });
            
            const balanceBefore = await ethers.provider.getBalance(user1.address);
            const tx = await vault.connect(user1).withdraw(amount);
            const receipt = await tx.wait();
            const gasUsed = receipt!.gasUsed * receipt!.gasPrice;
            const balanceAfter = await ethers.provider.getBalance(user1.address);
            
            expect(balanceAfter).to.equal(balanceBefore + amount - gasUsed);
        });
        
        it("Should not deposit when paused", async function () {
            await vault.connect(pauser).pause();
            
            await expect(
                vault.connect(user1).deposit({ value: ethers.parseEther("1") })
            ).to.be.revertedWith("Vault is paused");
        });
    });
    
    describe("Roles", function () {
        it("Pauser can pause/unpause", async function () {
            await vault.connect(pauser).pause();
            expect(await vault.paused()).to.be.true;
            
            await vault.connect(pauser).unpause();
            expect(await vault.paused()).to.be.false;
        });
        
        it("Non-pauser cannot pause", async function () {
            await expect(
                vault.connect(user1).pause()
            ).to.be.reverted;
        });
        
        it("Operator can emergency withdraw when paused", async function () {
            await vault.connect(user1).deposit({ value: ethers.parseEther("5") });
            await vault.connect(pauser).pause();
            
            await expect(
                vault.connect(operator).emergencyWithdraw(ethers.parseEther("1"))
            ).to.emit(vault, "EmergencyWithdraw");
        });
    });
});
```

---

## สรุป Part 04

Functions ที่เรียนรู้:
- ✅ Visibility (public, external, internal, private)
- ✅ Mutability (view, pure, payable)
- ✅ Return values (single, multiple, named)
- ✅ Modifiers (basic, parameterized, chained)
- ✅ Fallback และ receive
- ✅ External vs Internal calls
- ✅ Function selectors
- ✅ Virtual/Override
- ✅ Access Control System

## Quiz

1. ทำไม `external` ถูกกว่า `public` สำหรับ large arrays?
2. ตำแหน่ง `_` ใน modifier สำคัญอย่างไร?
3. ต่างกันอย่างไรระหว่าง `call()`, `staticcall()`, และ `delegatecall()`?
4. ทำไม `nonReentrant` modifier ต้อง set `_locked = false` หลัง `_`?

---

## Next: Part 05 - Control Flow และ Loops
