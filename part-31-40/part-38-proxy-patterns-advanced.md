# Part 38: Proxy Patterns Advanced

## สารบัญ
1. Proxy Pattern Review
2. UUPS vs Transparent Proxy
3. Diamond (EIP-2535) Pattern
4. Beacon Proxy
5. Workshop: Upgradeable Protocol

---

## 1. Proxy Pattern Review

```
Proxy Pattern คืออะไร:

User → Proxy → Implementation
         ↑          ↑
    State (storage)  Logic (code)

ทำไมต้องใช้:
1. Upgradeable: เปลี่ยน logic โดยไม่เปลี่ยน address
2. Code sharing: หลาย proxies ใช้ logic เดียวกัน
3. Gas savings: factory pattern

Storage Collision Problem:
- Proxy มี storage ของตัวเอง (admin, implementation)
- Logic contract มี storage ของตัวเอง
- ถ้า slot เดียวกัน → collision!
- Solution: EIP-1967 Standard Proxy Storage Slots

EIP-1967 Slots:
- IMPLEMENTATION_SLOT: keccak256("eip1967.proxy.implementation") - 1
- ADMIN_SLOT: keccak256("eip1967.proxy.admin") - 1
- BEACON_SLOT: keccak256("eip1967.proxy.beacon") - 1
```

---

## 2. UUPS Proxy

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * UUPS (Universal Upgradeable Proxy Standard) EIP-1822
 * 
 * ต่างจาก Transparent:
 * - Upgrade logic อยู่ใน Implementation (ไม่ใช่ Proxy)
 * - Proxy เล็กกว่า ประหยัด gas มากกว่า
 * - ต้องมี upgrade function ใน implementation ทุก version
 * 
 * Risk: ถ้า implementation ใหม่ไม่มี upgrade function
 *        → locked forever!
 */
abstract contract UUPSUpgradeable {
    
    bytes32 private constant _IMPLEMENTATION_SLOT = 
        bytes32(uint256(keccak256("eip1967.proxy.implementation")) - 1);
    
    event Upgraded(address indexed implementation);
    
    function _getImplementation() internal view returns (address impl) {
        assembly {
            impl := sload(_IMPLEMENTATION_SLOT)
        }
    }
    
    function _setImplementation(address newImpl) private {
        require(newImpl.code.length > 0, "Not a contract");
        assembly {
            sstore(_IMPLEMENTATION_SLOT, newImpl)
        }
    }
    
    // Must be overridden to add access control!
    function _authorizeUpgrade(address newImplementation) internal virtual;
    
    function upgradeTo(address newImplementation) external {
        _authorizeUpgrade(newImplementation);
        _setImplementation(newImplementation);
        emit Upgraded(newImplementation);
    }
    
    function upgradeToAndCall(address newImplementation, bytes calldata data) external payable {
        _authorizeUpgrade(newImplementation);
        _setImplementation(newImplementation);
        emit Upgraded(newImplementation);
        
        if (data.length > 0) {
            (bool success,) = newImplementation.delegatecall(data);
            require(success, "Upgrade call failed");
        }
    }
}

/**
 * UUPS Proxy Contract
 * Delegates everything to implementation
 */
contract UUPSProxy {
    
    bytes32 private constant _IMPLEMENTATION_SLOT = 
        bytes32(uint256(keccak256("eip1967.proxy.implementation")) - 1);
    
    constructor(address implementation, bytes memory data) payable {
        assembly {
            sstore(_IMPLEMENTATION_SLOT, implementation)
        }
        
        if (data.length > 0) {
            (bool success,) = implementation.delegatecall(data);
            require(success, "Init failed");
        }
    }
    
    fallback() external payable {
        address impl;
        assembly {
            impl := sload(_IMPLEMENTATION_SLOT)
        }
        
        assembly {
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

/**
 * Example: Upgradeable ERC-20 with UUPS
 */
contract MyTokenV1 is UUPSUpgradeable {
    
    address public owner;
    bool private initialized;
    
    string public name;
    string public symbol;
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    
    function initialize(string memory _name, string memory _symbol) external {
        require(!initialized, "Already initialized");
        initialized = true;
        owner = msg.sender;
        name = _name;
        symbol = _symbol;
    }
    
    function _authorizeUpgrade(address) internal override {
        require(msg.sender == owner, "Not owner");
    }
    
    function mint(address to, uint256 amount) external {
        require(msg.sender == owner, "Not owner");
        totalSupply += amount;
        balanceOf[to] += amount;
    }
    
    function transfer(address to, uint256 amount) external returns (bool) {
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }
}

/**
 * V2: add new feature (pause)
 */
contract MyTokenV2 is UUPSUpgradeable {
    
    address public owner;
    bool private initialized;
    
    string public name;
    string public symbol;
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    
    // NEW: pause feature
    bool public paused;
    
    function _authorizeUpgrade(address) internal override {
        require(msg.sender == owner, "Not owner");
    }
    
    // Same existing functions...
    
    // NEW function
    function setPaused(bool _paused) external {
        require(msg.sender == owner);
        paused = _paused;
    }
    
    function transfer(address to, uint256 amount) external returns (bool) {
        require(!paused, "Paused");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }
    
    function mint(address to, uint256 amount) external {
        require(msg.sender == owner, "Not owner");
        totalSupply += amount;
        balanceOf[to] += amount;
    }
}
```

---

## 3. Diamond Pattern (EIP-2535)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Diamond Pattern (EIP-2535)
 * 
 * แก้ปัญหา:
 * - Contract size limit (24KB)
 * - Logic แยกเป็น "Facets"
 * - Upgrade แค่ facet ที่ต้องการ
 * 
 * Concepts:
 * - Diamond: proxy contract
 * - Facet: implementation contract (ส่วนหนึ่ง)
 * - DiamondCut: ฟังก์ชัน upgrade
 * - DiamondLoupe: ฟังก์ชัน view current structure
 * - AppStorage: shared storage pattern
 */

struct FacetCut {
    address facetAddress;
    FacetCutAction action;
    bytes4[] functionSelectors;
}

enum FacetCutAction { Add, Replace, Remove }

interface IDiamondCut {
    event DiamondCut(FacetCut[] _diamondCut, address _init, bytes _calldata);
    
    function diamondCut(
        FacetCut[] calldata _diamondCut,
        address _init,
        bytes calldata _calldata
    ) external;
}

/**
 * DiamondStorage: ทุก facet share storage เดียวกัน
 */
library DiamondStorage {
    bytes32 constant DIAMOND_STORAGE_POSITION = keccak256("diamond.standard.storage");
    
    struct Layout {
        // Function selector → facet address
        mapping(bytes4 => address) facets;
        // All selectors (for enumeration)
        bytes4[] selectors;
        // Admin
        address contractOwner;
    }
    
    function layout() internal pure returns (Layout storage l) {
        bytes32 position = DIAMOND_STORAGE_POSITION;
        assembly {
            l.slot := position
        }
    }
}

/**
 * Diamond Proxy
 */
contract Diamond {
    
    constructor(address owner, FacetCut[] memory cuts) {
        DiamondStorage.Layout storage l = DiamondStorage.layout();
        l.contractOwner = owner;
        
        // Add initial facets
        for (uint256 i; i < cuts.length; i++) {
            _addFunctions(cuts[i].facetAddress, cuts[i].functionSelectors);
        }
    }
    
    function _addFunctions(address facet, bytes4[] memory selectors) internal {
        DiamondStorage.Layout storage l = DiamondStorage.layout();
        
        for (uint256 i; i < selectors.length; i++) {
            l.facets[selectors[i]] = facet;
            l.selectors.push(selectors[i]);
        }
    }
    
    fallback() external payable {
        DiamondStorage.Layout storage l = DiamondStorage.layout();
        address facet = l.facets[msg.sig];
        require(facet != address(0), "Function not found");
        
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
    
    receive() external payable {}
}

/**
 * AppStorage Pattern: type-safe storage ใน Diamond
 */
struct AppStorage {
    // Token state
    string name;
    string symbol;
    uint256 totalSupply;
    mapping(address => uint256) balances;
    
    // DeFi state
    uint256 exchangeRate;
    bool paused;
    
    // Governance
    address owner;
    uint256 votingPeriod;
}

/**
 * Facet ใช้ AppStorage ผ่าน layout pattern
 */
contract TokenFacet {
    
    AppStorage internal s;
    
    function name() external view returns (string memory) { return s.name; }
    function symbol() external view returns (string memory) { return s.symbol; }
    function totalSupply() external view returns (uint256) { return s.totalSupply; }
    
    function transfer(address to, uint256 amount) external returns (bool) {
        require(!s.paused, "Paused");
        s.balances[msg.sender] -= amount;
        s.balances[to] += amount;
        return true;
    }
}

contract AdminFacet {
    
    AppStorage internal s;
    
    modifier onlyOwner() {
        require(msg.sender == s.owner, "Not owner");
        _;
    }
    
    function setPaused(bool paused) external onlyOwner {
        s.paused = paused;
    }
    
    function transferOwnership(address newOwner) external onlyOwner {
        s.owner = newOwner;
    }
}
```

---

## 4. Beacon Proxy

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Beacon Proxy:
 * หลาย proxies ชี้ไปที่ beacon เดียวกัน
 * Upgrade ครั้งเดียว → ทุก proxies อัปเดต
 * 
 * Use case: Token factory (หลาย tokens ใช้ logic เดียวกัน)
 * 
 * Beacon → Implementation
 *   ↑          ↑
 * Proxy1    (ทุก proxy อ่าน implementation จาก beacon)
 * Proxy2
 * Proxy3
 */
contract UpgradeableBeacon {
    
    address private _implementation;
    address public owner;
    
    event Upgraded(address indexed implementation);
    
    constructor(address implementation_, address owner_) {
        _implementation = implementation_;
        owner = owner_;
    }
    
    function implementation() external view returns (address) {
        return _implementation;
    }
    
    function upgradeTo(address newImplementation) external {
        require(msg.sender == owner, "Not owner");
        require(newImplementation.code.length > 0, "Not a contract");
        _implementation = newImplementation;
        emit Upgraded(newImplementation);
    }
}

contract BeaconProxy {
    
    bytes32 private constant _BEACON_SLOT = 
        bytes32(uint256(keccak256("eip1967.proxy.beacon")) - 1);
    
    constructor(address beacon, bytes memory data) payable {
        assembly {
            sstore(_BEACON_SLOT, beacon)
        }
        
        if (data.length > 0) {
            address impl = IBeacon(beacon).implementation();
            (bool success,) = impl.delegatecall(data);
            require(success, "Init failed");
        }
    }
    
    function _getImplementation() internal view returns (address) {
        address beacon;
        assembly {
            beacon := sload(_BEACON_SLOT)
        }
        return IBeacon(beacon).implementation();
    }
    
    fallback() external payable {
        address impl = _getImplementation();
        
        assembly {
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

interface IBeacon {
    function implementation() external view returns (address);
}

/**
 * Token Factory ใช้ Beacon
 */
contract TokenFactory {
    
    UpgradeableBeacon public immutable beacon;
    address[] public tokens;
    
    event TokenCreated(address indexed token, string name, string symbol);
    
    constructor(address implementation_) {
        beacon = new UpgradeableBeacon(implementation_, msg.sender);
    }
    
    function createToken(
        string memory name,
        string memory symbol,
        uint256 initialSupply
    ) external returns (address token) {
        bytes memory initData = abi.encodeWithSignature(
            "initialize(string,string,uint256,address)",
            name, symbol, initialSupply, msg.sender
        );
        
        token = address(new BeaconProxy(address(beacon), initData));
        tokens.push(token);
        
        emit TokenCreated(token, name, symbol);
    }
    
    // Upgrade all tokens at once
    function upgradeAll(address newImplementation) external {
        beacon.upgradeTo(newImplementation);
    }
}
```

---

## สรุป Part 38

Proxy Patterns ที่เรียนรู้:
- ✅ EIP-1967 storage slots
- ✅ UUPS proxy (upgrade logic in implementation)
- ✅ Diamond pattern (EIP-2535, multi-facet)
- ✅ Beacon proxy (shared implementation)
- ✅ Token factory with beacon

## Quiz

1. UUPS ต่างจาก Transparent Proxy อย่างไร?
2. Diamond pattern แก้ปัญหา contract size limit อย่างไร?
3. Beacon proxy เหมาะกับ use case ไหน?
4. Storage collision คืออะไร และ EIP-1967 แก้อย่างไร?

---

## Next: Part 39 - Flash Loans Advanced
