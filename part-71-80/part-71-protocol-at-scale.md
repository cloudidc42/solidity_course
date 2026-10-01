# Part 71: Protocol Architecture at Scale

## บทนำ

เมื่อ Protocol เติบโตจากโปรเจกต์เล็กๆ ไปสู่ระบบขนาดใหญ่ที่มีผู้ใช้หลักล้านคนและ TVL หลายพันล้านดอลลาร์ สถาปัตยกรรมของ Smart Contract ต้องถูกออกแบบอย่างรอบคอบ ในส่วนนี้เราจะเรียนรู้วิธีจัดระเบียบ Protocol ขนาดใหญ่ตั้งแต่โครงสร้าง Monorepo ไปจนถึงการจัดการ Upgrade แบบ DAO-controlled

---

## 71.1 Monorepo Structure สำหรับ Large Protocols

### ทำไมต้องใช้ Monorepo?

Protocol ขนาดใหญ่อย่าง Uniswap, Aave, Compound ต่างใช้ Monorepo เพราะ:
- **Code Sharing**: ใช้ TypeScript types, ABIs, และ constants ร่วมกัน
- **Atomic Commits**: เปลี่ยน contract + frontend + SDK พร้อมกันใน commit เดียว
- **Consistent Versioning**: ทุก package ใช้ dependency version เดียวกัน
- **Simplified CI/CD**: test pipeline เดียวครอบคลุมทุก package

### โครงสร้าง Monorepo มาตรฐาน

```
my-protocol/
├── packages/
│   ├── contracts/          # Solidity smart contracts
│   │   ├── src/
│   │   │   ├── core/
│   │   │   ├── periphery/
│   │   │   ├── libraries/
│   │   │   └── interfaces/
│   │   ├── test/
│   │   ├── script/
│   │   ├── foundry.toml
│   │   └── package.json
│   │
│   ├── frontend/           # React/Next.js dApp
│   │   ├── src/
│   │   │   ├── components/
│   │   │   ├── hooks/      # wagmi/viem hooks
│   │   │   ├── pages/
│   │   │   └── utils/
│   │   ├── public/
│   │   └── package.json
│   │
│   ├── subgraph/           # The Graph indexing
│   │   ├── src/
│   │   │   ├── mappings/
│   │   │   └── utils/
│   │   ├── schema.graphql
│   │   ├── subgraph.yaml
│   │   └── package.json
│   │
│   └── sdk/                # TypeScript SDK
│       ├── src/
│       │   ├── contracts/  # Contract instances
│       │   ├── entities/   # Business logic
│       │   └── utils/
│       ├── dist/
│       └── package.json
│
├── .github/
│   └── workflows/
│       ├── contracts-ci.yml
│       ├── frontend-ci.yml
│       └── subgraph-ci.yml
│
├── turbo.json              # Turborepo config
├── pnpm-workspace.yaml
└── package.json
```

### turbo.json สำหรับ Pipeline

```json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", "out/**", "artifacts/**"]
    },
    "test": {
      "dependsOn": ["build"],
      "cache": false
    },
    "lint": {
      "outputs": []
    },
    "contracts#test": {
      "cache": false,
      "env": ["MAINNET_RPC_URL", "FORK_BLOCK_NUMBER"]
    }
  }
}
```

### pnpm-workspace.yaml

```yaml
packages:
  - "packages/*"
```

---

## 71.2 Contract Size Limits: EIP-170 และการแก้ปัญหา

### ปัญหา Contract Too Large

EIP-170 กำหนดขนาด bytecode สูงสุดที่ **24,576 bytes** (~24KB) ถ้า contract ใหญ่เกินนี้จะ deploy ไม่ได้

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// ตัวอย่าง: Contract ที่ใหญ่เกินไป - จะ deploy ไม่ได้!
// ถ้า compiled bytecode > 24,576 bytes
contract MassiveContract {
    // ฟังก์ชันร้อยฟังก์ชัน...
    // ตรรกะซับซ้อนมาก...
}
```

### วิธีที่ 1: แยกด้วย Libraries

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// ====================================
// Library สำหรับ Math Operations
// ====================================
library MathLib {
    uint256 constant WAD = 1e18;
    uint256 constant RAY = 1e27;

    function wadMul(uint256 x, uint256 y) internal pure returns (uint256) {
        return (x * y + WAD / 2) / WAD;
    }

    function wadDiv(uint256 x, uint256 y) internal pure returns (uint256) {
        return (x * WAD + y / 2) / y;
    }

    function rayMul(uint256 x, uint256 y) internal pure returns (uint256) {
        return (x * y + RAY / 2) / RAY;
    }

    function rayDiv(uint256 x, uint256 y) internal pure returns (uint256) {
        return (x * RAY + y / 2) / y;
    }

    function rayToWad(uint256 x) internal pure returns (uint256) {
        return x / 1e9;
    }

    function wadToRay(uint256 x) internal pure returns (uint256) {
        return x * 1e9;
    }

    /// @notice คำนวณ compound interest
    /// @param rate อัตราดอกเบี้ยต่อวินาที (ray)
    /// @param lastUpdateTimestamp timestamp ล่าสุด
    function calculateCompoundedInterest(
        uint256 rate,
        uint40 lastUpdateTimestamp
    ) internal view returns (uint256) {
        uint256 exp = block.timestamp - uint256(lastUpdateTimestamp);
        if (exp == 0) return RAY;

        uint256 expMinusOne;
        uint256 expMinusTwo;
        uint256 basePowerTwo;
        uint256 basePowerThree;

        unchecked {
            expMinusOne = exp - 1;
            expMinusTwo = exp > 2 ? exp - 2 : 0;
            basePowerTwo = rayMul(rate, rate) / (365 days * 365 days);
            basePowerThree = rayMul(basePowerTwo, rate) / 365 days;
        }

        uint256 secondTerm = exp * expMinusOne * basePowerTwo;
        unchecked {
            secondTerm /= 2;
        }
        uint256 thirdTerm = exp * expMinusOne * expMinusTwo * basePowerThree;
        unchecked {
            thirdTerm /= 6;
        }

        return RAY + rate * exp / 365 days + secondTerm + thirdTerm;
    }
}

// ====================================
// Library สำหรับ Validation
// ====================================
library ValidationLib {
    error ZeroAddress();
    error ZeroAmount();
    error InvalidRange(uint256 min, uint256 max, uint256 value);
    error Unauthorized(address caller, address expected);

    function requireNotZeroAddress(address addr) internal pure {
        if (addr == address(0)) revert ZeroAddress();
    }

    function requireNotZeroAmount(uint256 amount) internal pure {
        if (amount == 0) revert ZeroAmount();
    }

    function requireInRange(uint256 value, uint256 min, uint256 max) internal pure {
        if (value < min || value > max) revert InvalidRange(min, max, value);
    }

    function requireCaller(address caller, address expected) internal pure {
        if (caller != expected) revert Unauthorized(caller, expected);
    }
}

// ====================================
// Library สำหรับ Access Control Logic
// ====================================
library AccessLib {
    bytes32 constant ADMIN_ROLE = keccak256("ADMIN_ROLE");
    bytes32 constant OPERATOR_ROLE = keccak256("OPERATOR_ROLE");
    bytes32 constant GUARDIAN_ROLE = keccak256("GUARDIAN_ROLE");

    struct RoleData {
        mapping(address => bool) members;
        bytes32 adminRole;
    }

    struct AccessStorage {
        mapping(bytes32 => RoleData) roles;
    }

    function hasRole(
        AccessStorage storage self,
        bytes32 role,
        address account
    ) internal view returns (bool) {
        return self.roles[role].members[account];
    }

    function grantRole(
        AccessStorage storage self,
        bytes32 role,
        address account
    ) internal {
        self.roles[role].members[account] = true;
    }

    function revokeRole(
        AccessStorage storage self,
        bytes32 role,
        address account
    ) internal {
        self.roles[role].members[account] = false;
    }
}

// ====================================
// Main Contract ใช้ Libraries
// ====================================
contract LendingProtocol {
    using MathLib for uint256;
    using ValidationLib for address;
    using ValidationLib for uint256;

    AccessLib.AccessStorage private _access;

    struct Market {
        address asset;
        uint256 totalSupply;
        uint256 totalBorrow;
        uint256 borrowRate;        // ray per second
        uint40 lastUpdateTimestamp;
        uint256 supplyIndex;       // ray
        uint256 borrowIndex;       // ray
    }

    mapping(address => Market) public markets;
    mapping(address => mapping(address => uint256)) public userSupply; // market => user => shares
    mapping(address => mapping(address => uint256)) public userBorrow; // market => user => scaled

    event MarketCreated(address indexed asset);
    event Supply(address indexed market, address indexed user, uint256 amount, uint256 shares);
    event Borrow(address indexed market, address indexed user, uint256 amount);

    constructor() {
        AccessLib.grantRole(_access, AccessLib.ADMIN_ROLE, msg.sender);
    }

    modifier onlyAdmin() {
        require(AccessLib.hasRole(_access, AccessLib.ADMIN_ROLE, msg.sender), "Not admin");
        _;
    }

    function createMarket(address asset, uint256 initialBorrowRate) external onlyAdmin {
        asset.requireNotZeroAddress();
        initialBorrowRate.requireInRange(1e18, 1e27, 1e18); // 1 wei to 1e27 range check

        markets[asset] = Market({
            asset: asset,
            totalSupply: 0,
            totalBorrow: 0,
            borrowRate: initialBorrowRate,
            lastUpdateTimestamp: uint40(block.timestamp),
            supplyIndex: MathLib.RAY,
            borrowIndex: MathLib.RAY
        });

        emit MarketCreated(asset);
    }

    function _accrueInterest(Market storage market) internal {
        uint256 newIndex = MathLib.calculateCompoundedInterest(
            market.borrowRate,
            market.lastUpdateTimestamp
        );
        market.borrowIndex = MathLib.rayMul(market.borrowIndex, newIndex);
        market.supplyIndex = MathLib.rayMul(market.supplyIndex, newIndex);
        market.lastUpdateTimestamp = uint40(block.timestamp);
    }

    function supply(address asset, uint256 amount) external {
        amount.requireNotZeroAmount();
        Market storage market = markets[asset];

        _accrueInterest(market);

        uint256 shares = market.supplyIndex > 0
            ? MathLib.rayDiv(amount * MathLib.RAY, market.supplyIndex)
            : amount;

        market.totalSupply += shares;
        userSupply[asset][msg.sender] += shares;

        emit Supply(asset, msg.sender, amount, shares);
    }
}
```

### วิธีที่ 2: Diamond Pattern (EIP-2535)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// ====================================
// Diamond Storage
// ====================================
library DiamondStorageLib {
    bytes32 constant DIAMOND_STORAGE_POSITION = keccak256("diamond.standard.diamond.storage");

    struct FacetAddressAndPosition {
        address facetAddress;
        uint96 functionSelectorPosition; // position in facetFunctionSelectors.functionSelectors array
    }

    struct FacetFunctionSelectors {
        bytes4[] functionSelectors;
        uint256 facetAddressPosition; // position of facetAddress in facetAddresses array
    }

    struct DiamondStorage {
        // maps function selector to the facet address and
        // the position of the selector in the facetFunctionSelectors.selectors array
        mapping(bytes4 => FacetAddressAndPosition) selectorToFacetAndPosition;
        // maps facet addresses to function selectors
        mapping(address => FacetFunctionSelectors) facetFunctionSelectors;
        // facet addresses
        address[] facetAddresses;
        // Used to query if a contract implements an interface.
        // Used to implement ERC-165.
        mapping(bytes4 => bool) supportedInterfaces;
        // owner of the contract
        address contractOwner;
    }

    function diamondStorage() internal pure returns (DiamondStorage storage ds) {
        bytes32 position = DIAMOND_STORAGE_POSITION;
        assembly {
            ds.slot := position
        }
    }

    function enforceIsContractOwner() internal view {
        require(msg.sender == diamondStorage().contractOwner, "Must be contract owner");
    }
}

// ====================================
// Diamond Cut Facet
// ====================================
interface IDiamondCut {
    enum FacetCutAction {
        Add,
        Replace,
        Remove
    }

    struct FacetCut {
        address facetAddress;
        FacetCutAction action;
        bytes4[] functionSelectors;
    }

    function diamondCut(
        FacetCut[] calldata _diamondCut,
        address _init,
        bytes calldata _calldata
    ) external;

    event DiamondCut(FacetCut[] _diamondCut, address _init, bytes _calldata);
}

contract DiamondCutFacet is IDiamondCut {
    event DiamondCut(FacetCut[] _diamondCut, address _init, bytes _calldata);

    function diamondCut(
        FacetCut[] calldata _diamondCut,
        address _init,
        bytes calldata _calldata
    ) external override {
        DiamondStorageLib.enforceIsContractOwner();
        LibDiamondCut.diamondCut(_diamondCut, _init, _calldata);
        emit DiamondCut(_diamondCut, _init, _calldata);
    }
}

library LibDiamondCut {
    bytes32 constant CLEAR_ADDRESS_MASK = bytes32(uint256(0xffffffffffffffffffffffff));
    bytes32 constant CLEAR_SELECTOR_MASK = bytes32(uint256(0xffffffff << 224));

    function diamondCut(
        IDiamondCut.FacetCut[] memory _diamondCut,
        address _init,
        bytes memory _calldata
    ) internal {
        for (uint256 facetIndex; facetIndex < _diamondCut.length; ) {
            IDiamondCut.FacetCutAction action = _diamondCut[facetIndex].action;
            if (action == IDiamondCut.FacetCutAction.Add) {
                addFunctions(
                    _diamondCut[facetIndex].facetAddress,
                    _diamondCut[facetIndex].functionSelectors
                );
            } else if (action == IDiamondCut.FacetCutAction.Replace) {
                replaceFunctions(
                    _diamondCut[facetIndex].facetAddress,
                    _diamondCut[facetIndex].functionSelectors
                );
            } else if (action == IDiamondCut.FacetCutAction.Remove) {
                removeFunctions(
                    _diamondCut[facetIndex].facetAddress,
                    _diamondCut[facetIndex].functionSelectors
                );
            } else {
                revert("LibDiamondCut: Incorrect FacetCutAction");
            }
            unchecked {
                facetIndex++;
            }
        }
        emit IDiamondCut.DiamondCut(_diamondCut, _init, _calldata);
        initializeDiamondCut(_init, _calldata);
    }

    function addFunctions(address _facetAddress, bytes4[] memory _functionSelectors) internal {
        require(_functionSelectors.length > 0, "LibDiamondCut: No selectors in facet to cut");
        DiamondStorageLib.DiamondStorage storage ds = DiamondStorageLib.diamondStorage();
        require(_facetAddress != address(0), "LibDiamondCut: Add facet can't be address(0)");
        uint96 selectorPosition = uint96(
            ds.facetFunctionSelectors[_facetAddress].functionSelectors.length
        );
        if (selectorPosition == 0) {
            addFacet(ds, _facetAddress);
        }
        for (uint256 selectorIndex; selectorIndex < _functionSelectors.length; ) {
            bytes4 selector = _functionSelectors[selectorIndex];
            address oldFacetAddress = ds.selectorToFacetAndPosition[selector].facetAddress;
            require(
                oldFacetAddress == address(0),
                "LibDiamondCut: Can't add function that already exists"
            );
            addFunction(ds, selector, selectorPosition, _facetAddress);
            selectorPosition++;
            unchecked {
                selectorIndex++;
            }
        }
    }

    function replaceFunctions(address _facetAddress, bytes4[] memory _functionSelectors) internal {
        require(_functionSelectors.length > 0, "LibDiamondCut: No selectors in facet to cut");
        DiamondStorageLib.DiamondStorage storage ds = DiamondStorageLib.diamondStorage();
        require(_facetAddress != address(0), "LibDiamondCut: Replace facet can't be address(0)");
        uint96 selectorPosition = uint96(
            ds.facetFunctionSelectors[_facetAddress].functionSelectors.length
        );
        if (selectorPosition == 0) {
            addFacet(ds, _facetAddress);
        }
        for (uint256 selectorIndex; selectorIndex < _functionSelectors.length; ) {
            bytes4 selector = _functionSelectors[selectorIndex];
            address oldFacetAddress = ds.selectorToFacetAndPosition[selector].facetAddress;
            require(
                oldFacetAddress != _facetAddress,
                "LibDiamondCut: Can't replace function with same function"
            );
            removeFunction(ds, oldFacetAddress, selector);
            addFunction(ds, selector, selectorPosition, _facetAddress);
            selectorPosition++;
            unchecked {
                selectorIndex++;
            }
        }
    }

    function removeFunctions(address _facetAddress, bytes4[] memory _functionSelectors) internal {
        require(_functionSelectors.length > 0, "LibDiamondCut: No selectors in facet to cut");
        DiamondStorageLib.DiamondStorage storage ds = DiamondStorageLib.diamondStorage();
        require(_facetAddress == address(0), "LibDiamondCut: Remove facet address must be address(0)");
        for (uint256 selectorIndex; selectorIndex < _functionSelectors.length; ) {
            bytes4 selector = _functionSelectors[selectorIndex];
            address oldFacetAddress = ds.selectorToFacetAndPosition[selector].facetAddress;
            removeFunction(ds, oldFacetAddress, selector);
            unchecked {
                selectorIndex++;
            }
        }
    }

    function addFacet(
        DiamondStorageLib.DiamondStorage storage ds,
        address _facetAddress
    ) internal {
        enforceHasContractCode(_facetAddress, "LibDiamondCut: New facet has no code");
        ds.facetFunctionSelectors[_facetAddress].facetAddressPosition = ds.facetAddresses.length;
        ds.facetAddresses.push(_facetAddress);
    }

    function addFunction(
        DiamondStorageLib.DiamondStorage storage ds,
        bytes4 _selector,
        uint96 _selectorPosition,
        address _facetAddress
    ) internal {
        ds.selectorToFacetAndPosition[_selector].functionSelectorPosition = _selectorPosition;
        ds.facetFunctionSelectors[_facetAddress].functionSelectors.push(_selector);
        ds.selectorToFacetAndPosition[_selector].facetAddress = _facetAddress;
    }

    function removeFunction(
        DiamondStorageLib.DiamondStorage storage ds,
        address _facetAddress,
        bytes4 _selector
    ) internal {
        require(_facetAddress != address(0), "LibDiamondCut: Can't remove function that doesn't exist");
        require(_facetAddress != address(this), "LibDiamondCut: Can't remove immutable function");
        uint256 selectorPosition = ds
            .selectorToFacetAndPosition[_selector]
            .functionSelectorPosition;
        uint256 lastSelectorPosition = ds
            .facetFunctionSelectors[_facetAddress]
            .functionSelectors
            .length - 1;
        if (selectorPosition != lastSelectorPosition) {
            bytes4 lastSelector = ds.facetFunctionSelectors[_facetAddress].functionSelectors[
                lastSelectorPosition
            ];
            ds.facetFunctionSelectors[_facetAddress].functionSelectors[
                selectorPosition
            ] = lastSelector;
            ds.selectorToFacetAndPosition[lastSelector].functionSelectorPosition = uint96(
                selectorPosition
            );
        }
        ds.facetFunctionSelectors[_facetAddress].functionSelectors.pop();
        delete ds.selectorToFacetAndPosition[_selector];
        if (lastSelectorPosition == 0) {
            uint256 lastFacetAddressPosition = ds.facetAddresses.length - 1;
            uint256 facetAddressPosition = ds
                .facetFunctionSelectors[_facetAddress]
                .facetAddressPosition;
            if (facetAddressPosition != lastFacetAddressPosition) {
                address lastFacetAddress = ds.facetAddresses[lastFacetAddressPosition];
                ds.facetAddresses[facetAddressPosition] = lastFacetAddress;
                ds.facetFunctionSelectors[lastFacetAddress].facetAddressPosition = facetAddressPosition;
            }
            ds.facetAddresses.pop();
            delete ds.facetFunctionSelectors[_facetAddress].facetAddressPosition;
        }
    }

    function initializeDiamondCut(address _init, bytes memory _calldata) internal {
        if (_init == address(0)) {
            return;
        }
        enforceHasContractCode(_init, "LibDiamondCut: _init address has no code");
        (bool success, bytes memory error) = _init.delegatecall(_calldata);
        if (!success) {
            if (error.length > 0) {
                assembly {
                    let returndata_size := mload(error)
                    revert(add(32, error), returndata_size)
                }
            } else {
                revert("LibDiamondCut: _init function reverted");
            }
        }
    }

    function enforceHasContractCode(address _contract, string memory _errorMessage) internal view {
        uint256 contractSize;
        assembly {
            contractSize := extcodesize(_contract)
        }
        require(contractSize > 0, _errorMessage);
    }
}
```

---

## 71.3 Deployment Orchestration: CREATE2 และ Multi-Chain Consistency

### CREATE2 ให้ Deterministic Addresses

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// ====================================
// Factory Contract ที่ใช้ CREATE2
// ====================================
contract ProtocolDeployer {
    event Deployed(address indexed contractAddress, bytes32 indexed salt, string contractName);

    /// @notice คำนวณ address ก่อน deploy
    /// @param bytecode bytecode ของ contract ที่จะ deploy
    /// @param salt salt สำหรับ CREATE2
    /// @return predicted address ที่จะ deploy
    function computeAddress(
        bytes memory bytecode,
        bytes32 salt
    ) public view returns (address) {
        bytes32 hash = keccak256(
            abi.encodePacked(
                bytes1(0xff),
                address(this),
                salt,
                keccak256(bytecode)
            )
        );
        return address(uint160(uint256(hash)));
    }

    /// @notice Deploy contract ด้วย CREATE2
    function deploy(
        bytes memory bytecode,
        bytes32 salt,
        string memory contractName
    ) external returns (address addr) {
        assembly {
            addr := create2(0, add(bytecode, 0x20), mload(bytecode), salt)
            if iszero(extcodesize(addr)) {
                revert(0, 0)
            }
        }
        emit Deployed(addr, salt, contractName);
    }

    /// @notice Deploy และ initialize ในทีเดียว
    function deployAndInit(
        bytes memory bytecode,
        bytes32 salt,
        bytes memory initData,
        string memory contractName
    ) external returns (address addr) {
        assembly {
            addr := create2(0, add(bytecode, 0x20), mload(bytecode), salt)
            if iszero(extcodesize(addr)) {
                revert(0, 0)
            }
        }

        if (initData.length > 0) {
            (bool success, bytes memory result) = addr.call(initData);
            if (!success) {
                assembly {
                    revert(add(result, 32), mload(result))
                }
            }
        }

        emit Deployed(addr, salt, contractName);
    }
}

// ====================================
// Salt Generator สำหรับ Consistent Deployment
// ====================================
library SaltLib {
    /// @notice สร้าง salt จากชื่อ contract และ version
    function makeSalt(
        string memory contractName,
        uint256 version,
        uint256 chainId
    ) internal pure returns (bytes32) {
        return keccak256(abi.encodePacked(contractName, version, chainId));
    }

    /// @notice สร้าง salt ที่เหมือนกันทุก chain
    function makeChainAgnosticSalt(
        string memory contractName,
        uint256 version
    ) internal pure returns (bytes32) {
        return keccak256(abi.encodePacked(contractName, version));
    }
}

// ====================================
// Multi-Chain Deployment Orchestrator
// ====================================
contract MultiChainDeploymentOrchestrator {
    using SaltLib for string;

    struct DeploymentRecord {
        address contractAddress;
        uint256 chainId;
        uint256 version;
        uint256 deployedAt;
        bool isActive;
    }

    // contractName => version => chainId => deployment
    mapping(bytes32 => mapping(uint256 => mapping(uint256 => DeploymentRecord))) public deployments;
    mapping(bytes32 => uint256) public latestVersions;

    address public immutable deployer;
    ProtocolDeployer public immutable factory;

    event ContractDeployed(
        string indexed contractName,
        uint256 version,
        uint256 chainId,
        address contractAddress
    );

    constructor(address _factory) {
        deployer = msg.sender;
        factory = ProtocolDeployer(_factory);
    }

    modifier onlyDeployer() {
        require(msg.sender == deployer, "Only deployer");
        _;
    }

    function deployContract(
        string memory contractName,
        uint256 version,
        bytes memory bytecode,
        bytes memory initData
    ) external onlyDeployer returns (address addr) {
        bytes32 nameHash = keccak256(bytes(contractName));
        bytes32 salt = SaltLib.makeChainAgnosticSalt(contractName, version);

        addr = factory.deployAndInit(bytecode, salt, initData, contractName);

        deployments[nameHash][version][block.chainid] = DeploymentRecord({
            contractAddress: addr,
            chainId: block.chainid,
            version: version,
            deployedAt: block.timestamp,
            isActive: true
        });

        if (version > latestVersions[nameHash]) {
            latestVersions[nameHash] = version;
        }

        emit ContractDeployed(contractName, version, block.chainid, addr);
    }

    function getLatestDeployment(
        string memory contractName
    ) external view returns (DeploymentRecord memory) {
        bytes32 nameHash = keccak256(bytes(contractName));
        uint256 latestVersion = latestVersions[nameHash];
        return deployments[nameHash][latestVersion][block.chainid];
    }
}
```

---

## 71.4 Upgrade Governance at Scale: DAO-Controlled Upgrades with Mandatory Timelock

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/governance/Governor.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorTimelockControl.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorSettings.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorCountingSimple.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorVotes.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorVotesQuorumFraction.sol";
import "@openzeppelin/contracts/governance/TimelockController.sol";

// ====================================
// Protocol Governance Token
// ====================================
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Votes.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";

contract GovernanceToken is ERC20, ERC20Permit, ERC20Votes {
    constructor(
        string memory name,
        string memory symbol,
        address initialHolder
    ) ERC20(name, symbol) ERC20Permit(name) {
        _mint(initialHolder, 100_000_000 * 1e18); // 100M tokens
    }

    function _afterTokenTransfer(
        address from,
        address to,
        uint256 amount
    ) internal override(ERC20, ERC20Votes) {
        super._afterTokenTransfer(from, to, amount);
    }

    function _mint(address to, uint256 amount) internal override(ERC20, ERC20Votes) {
        super._mint(to, amount);
    }

    function _burn(address account, uint256 amount) internal override(ERC20, ERC20Votes) {
        super._burn(account, amount);
    }
}

// ====================================
// Protocol Governor
// ====================================
contract ProtocolGovernor is
    Governor,
    GovernorSettings,
    GovernorCountingSimple,
    GovernorVotes,
    GovernorVotesQuorumFraction,
    GovernorTimelockControl
{
    constructor(
        IVotes _token,
        TimelockController _timelock
    )
        Governor("Protocol Governor")
        GovernorSettings(
            1 days,     // voting delay: 1 day
            1 weeks,    // voting period: 1 week
            100_000e18  // proposal threshold: 100K tokens
        )
        GovernorVotes(_token)
        GovernorVotesQuorumFraction(4) // 4% quorum
        GovernorTimelockControl(_timelock)
    {}

    // ====================================
    // Required overrides
    // ====================================
    function votingDelay()
        public
        view
        override(IGovernor, GovernorSettings)
        returns (uint256)
    {
        return super.votingDelay();
    }

    function votingPeriod()
        public
        view
        override(IGovernor, GovernorSettings)
        returns (uint256)
    {
        return super.votingPeriod();
    }

    function quorum(uint256 blockNumber)
        public
        view
        override(IGovernor, GovernorVotesQuorumFraction)
        returns (uint256)
    {
        return super.quorum(blockNumber);
    }

    function state(uint256 proposalId)
        public
        view
        override(Governor, GovernorTimelockControl)
        returns (ProposalState)
    {
        return super.state(proposalId);
    }

    function propose(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        string memory description
    ) public override(Governor, IGovernor) returns (uint256) {
        return super.propose(targets, values, calldatas, description);
    }

    function proposalThreshold()
        public
        view
        override(Governor, GovernorSettings)
        returns (uint256)
    {
        return super.proposalThreshold();
    }

    function _execute(
        uint256 proposalId,
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        bytes32 descriptionHash
    ) internal override(Governor, GovernorTimelockControl) {
        super._execute(proposalId, targets, values, calldatas, descriptionHash);
    }

    function _cancel(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        bytes32 descriptionHash
    ) internal override(Governor, GovernorTimelockControl) returns (uint256) {
        return super._cancel(targets, values, calldatas, descriptionHash);
    }

    function _executor()
        internal
        view
        override(Governor, GovernorTimelockControl)
        returns (address)
    {
        return super._executor();
    }

    function supportsInterface(bytes4 interfaceId)
        public
        view
        override(Governor, GovernorTimelockControl)
        returns (bool)
    {
        return super.supportsInterface(interfaceId);
    }
}

// ====================================
// Emergency Guardian - สำหรับ security incidents
// ====================================
contract EmergencyGuardian {
    address public immutable timelock;
    address public immutable multisig; // Guardian multisig (3/5 signers)

    mapping(address => bool) public paused;
    bool public globalPause;

    event ContractPaused(address indexed target, address indexed guardian);
    event GlobalPauseActivated(address indexed guardian);
    event GlobalPauseDeactivated(address indexed caller);

    error NotGuardian();
    error NotTimelock();

    constructor(address _timelock, address _multisig) {
        timelock = _timelock;
        multisig = _multisig;
    }

    modifier onlyGuardian() {
        if (msg.sender != multisig) revert NotGuardian();
        _;
    }

    modifier onlyTimelock() {
        if (msg.sender != timelock) revert NotTimelock();
        _;
    }

    /// @notice Guardian สามารถ pause ทันทีในกรณีฉุกเฉิน
    function pauseContract(address target) external onlyGuardian {
        paused[target] = true;
        emit ContractPaused(target, msg.sender);
    }

    function activateGlobalPause() external onlyGuardian {
        globalPause = true;
        emit GlobalPauseActivated(msg.sender);
    }

    /// @notice เฉพาะ Timelock (DAO vote) เท่านั้นที่ unpause ได้
    function deactivateGlobalPause() external onlyTimelock {
        globalPause = false;
        emit GlobalPauseDeactivated(msg.sender);
    }

    function unpauseContract(address target) external onlyTimelock {
        paused[target] = false;
    }
}
```

---

## 71.5 On-Chain Registry: ProtocolRegistry

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// ====================================
// Protocol Registry - ติดตาม contracts ทั้งหมด
// ====================================
contract ProtocolRegistry {
    // ====================================
    // Types
    // ====================================
    struct ContractInfo {
        address contractAddress;
        string contractType;    // "CORE", "PERIPHERY", "LIBRARY", "PROXY"
        string version;         // semver: "1.0.0"
        uint256 deployedAt;
        uint256 chainId;
        bool isActive;
        bytes32 codeHash;      // keccak256 of deployed bytecode
        address deployer;
        string description;
    }

    struct VersionHistory {
        address[] addresses;
        string[] versions;
        uint256[] timestamps;
    }

    // ====================================
    // Storage
    // ====================================
    address public immutable governance;

    // contractName => current info
    mapping(bytes32 => ContractInfo) public contracts;
    // contractName => version history
    mapping(bytes32 => VersionHistory) private _versionHistory;
    // address => contractName (reverse lookup)
    mapping(address => bytes32) public addressToName;
    // all registered contract names
    bytes32[] public registeredNames;

    // contractType => list of contract names
    mapping(string => bytes32[]) public contractsByType;

    // ====================================
    // Events
    // ====================================
    event ContractRegistered(
        bytes32 indexed nameHash,
        string name,
        address indexed contractAddress,
        string version,
        string contractType
    );
    event ContractUpdated(
        bytes32 indexed nameHash,
        address indexed oldAddress,
        address indexed newAddress,
        string newVersion
    );
    event ContractDeactivated(bytes32 indexed nameHash, address indexed contractAddress);

    // ====================================
    // Errors
    // ====================================
    error NotGovernance();
    error AlreadyRegistered(bytes32 nameHash);
    error NotRegistered(bytes32 nameHash);
    error ZeroAddress();
    error EmptyString();

    constructor(address _governance) {
        governance = _governance;
    }

    modifier onlyGovernance() {
        if (msg.sender != governance) revert NotGovernance();
        _;
    }

    // ====================================
    // Registration
    // ====================================

    /// @notice ลงทะเบียน contract ใหม่
    function register(
        string calldata name,
        address contractAddress,
        string calldata version,
        string calldata contractType,
        string calldata description
    ) external onlyGovernance {
        if (contractAddress == address(0)) revert ZeroAddress();
        if (bytes(name).length == 0) revert EmptyString();

        bytes32 nameHash = keccak256(bytes(name));
        if (contracts[nameHash].contractAddress != address(0)) {
            revert AlreadyRegistered(nameHash);
        }

        bytes32 codeHash;
        assembly {
            codeHash := extcodehash(contractAddress)
        }

        contracts[nameHash] = ContractInfo({
            contractAddress: contractAddress,
            contractType: contractType,
            version: version,
            deployedAt: block.timestamp,
            chainId: block.chainid,
            isActive: true,
            codeHash: codeHash,
            deployer: tx.origin,
            description: description
        });

        addressToName[contractAddress] = nameHash;
        registeredNames.push(nameHash);
        contractsByType[contractType].push(nameHash);

        // บันทึก version history
        _versionHistory[nameHash].addresses.push(contractAddress);
        _versionHistory[nameHash].versions.push(version);
        _versionHistory[nameHash].timestamps.push(block.timestamp);

        emit ContractRegistered(nameHash, name, contractAddress, version, contractType);
    }

    /// @notice อัพเดท contract address (เมื่อ upgrade)
    function update(
        string calldata name,
        address newAddress,
        string calldata newVersion
    ) external onlyGovernance {
        if (newAddress == address(0)) revert ZeroAddress();

        bytes32 nameHash = keccak256(bytes(name));
        ContractInfo storage info = contracts[nameHash];
        if (info.contractAddress == address(0)) revert NotRegistered(nameHash);

        address oldAddress = info.contractAddress;

        bytes32 newCodeHash;
        assembly {
            newCodeHash := extcodehash(newAddress)
        }

        // อัพเดท reverse lookup
        delete addressToName[oldAddress];
        addressToName[newAddress] = nameHash;

        info.contractAddress = newAddress;
        info.version = newVersion;
        info.codeHash = newCodeHash;

        // บันทึก history
        _versionHistory[nameHash].addresses.push(newAddress);
        _versionHistory[nameHash].versions.push(newVersion);
        _versionHistory[nameHash].timestamps.push(block.timestamp);

        emit ContractUpdated(nameHash, oldAddress, newAddress, newVersion);
    }

    /// @notice ปิดการใช้งาน contract
    function deactivate(string calldata name) external onlyGovernance {
        bytes32 nameHash = keccak256(bytes(name));
        ContractInfo storage info = contracts[nameHash];
        if (info.contractAddress == address(0)) revert NotRegistered(nameHash);

        info.isActive = false;
        emit ContractDeactivated(nameHash, info.contractAddress);
    }

    // ====================================
    // View Functions
    // ====================================

    /// @notice ดึง address ของ contract จากชื่อ
    function getAddress(string calldata name) external view returns (address) {
        bytes32 nameHash = keccak256(bytes(name));
        ContractInfo storage info = contracts[nameHash];
        require(info.isActive, "Contract not active");
        return info.contractAddress;
    }

    /// @notice ดึงข้อมูลทั้งหมดของ contract
    function getContractInfo(string calldata name) external view returns (ContractInfo memory) {
        return contracts[keccak256(bytes(name))];
    }

    /// @notice ดูประวัติ versions
    function getVersionHistory(
        string calldata name
    ) external view returns (
        address[] memory addresses,
        string[] memory versions,
        uint256[] memory timestamps
    ) {
        VersionHistory storage history = _versionHistory[keccak256(bytes(name))];
        return (history.addresses, history.versions, history.timestamps);
    }

    /// @notice ดึง contracts ทั้งหมดตาม type
    function getContractsByType(
        string calldata contractType
    ) external view returns (bytes32[] memory) {
        return contractsByType[contractType];
    }

    /// @notice ตรวจสอบว่า address ยังใช้ code เดิมไหม
    function isCodeIntact(string calldata name) external view returns (bool) {
        bytes32 nameHash = keccak256(bytes(name));
        ContractInfo storage info = contracts[nameHash];
        if (info.contractAddress == address(0)) return false;

        bytes32 currentCodeHash;
        assembly {
            currentCodeHash := extcodehash(info.contractAddress)
        }

        return currentCodeHash == info.codeHash;
    }

    /// @notice ดึงรายชื่อ contracts ทั้งหมด
    function getAllRegisteredNames() external view returns (bytes32[] memory) {
        return registeredNames;
    }

    /// @notice ดึงจำนวน contracts ที่ลงทะเบียน
    function registeredCount() external view returns (uint256) {
        return registeredNames.length;
    }
}
```

---

## Workshop 71: สร้าง Protocol Infrastructure สมบูรณ์

### โจทย์

สร้าง Protocol Infrastructure ที่รวม:
1. ProtocolRegistry สำหรับ track deployments
2. ProtocolDeployer ด้วย CREATE2
3. TimelockController สำหรับ DAO
4. EmergencyGuardian

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/governance/TimelockController.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";

// ====================================
// Protocol Deployer + Registry Integration
// ====================================
contract ProtocolInfrastructure is AccessControl {
    bytes32 public constant DEPLOYER_ROLE = keccak256("DEPLOYER_ROLE");
    bytes32 public constant GUARDIAN_ROLE = keccak256("GUARDIAN_ROLE");

    ProtocolRegistry public immutable registry;
    TimelockController public immutable timelock;

    bool public emergencyPaused;

    // contractType constants
    string constant CORE_TYPE = "CORE";
    string constant PERIPHERY_TYPE = "PERIPHERY";
    string constant LIBRARY_TYPE = "LIBRARY";

    event EmergencyPause(address indexed guardian, uint256 timestamp);
    event EmergencyResume(address indexed caller, uint256 timestamp);
    event ProtocolDeployment(
        string indexed contractName,
        address indexed contractAddress,
        string version
    );

    constructor(
        address admin,
        address[] memory guardians,
        uint256 timelockDelay
    ) {
        // Setup Timelock
        address[] memory proposers = new address[](1);
        proposers[0] = admin;
        address[] memory executors = new address[](1);
        executors[0] = address(0); // anyone can execute after delay

        timelock = new TimelockController(
            timelockDelay,
            proposers,
            executors,
            admin
        );

        // Setup Registry (governed by timelock)
        registry = new ProtocolRegistry(address(timelock));

        // Setup roles
        _grantRole(DEFAULT_ADMIN_ROLE, address(timelock));
        _grantRole(DEPLOYER_ROLE, admin);

        for (uint i = 0; i < guardians.length; i++) {
            _grantRole(GUARDIAN_ROLE, guardians[i]);
        }
    }

    modifier whenNotPaused() {
        require(!emergencyPaused, "Protocol is paused");
        _;
    }

    function activateEmergencyPause() external onlyRole(GUARDIAN_ROLE) {
        emergencyPaused = true;
        emit EmergencyPause(msg.sender, block.timestamp);
    }

    function deactivateEmergencyPause() external onlyRole(DEFAULT_ADMIN_ROLE) {
        emergencyPaused = false;
        emit EmergencyResume(msg.sender, block.timestamp);
    }

    /// @notice Deploy contract ด้วย CREATE2 และ register อัตโนมัติ
    function deployAndRegister(
        string calldata name,
        bytes calldata bytecode,
        bytes calldata initData,
        string calldata version,
        string calldata contractType,
        string calldata description,
        bytes32 salt
    ) external onlyRole(DEPLOYER_ROLE) whenNotPaused returns (address addr) {
        // Deploy ด้วย CREATE2
        bytes memory deployBytecode = bytecode;
        assembly {
            addr := create2(0, add(deployBytecode, 0x20), mload(deployBytecode), salt)
            if iszero(extcodesize(addr)) {
                revert(0, 0)
            }
        }

        // Initialize ถ้ามี initData
        if (initData.length > 0) {
            (bool success, bytes memory result) = addr.call(initData);
            if (!success) {
                assembly {
                    revert(add(result, 32), mload(result))
                }
            }
        }

        emit ProtocolDeployment(name, addr, version);
    }

    /// @notice คำนวณ address ก่อน deploy
    function predictAddress(
        bytes calldata bytecode,
        bytes32 salt
    ) external view returns (address) {
        bytes32 hash = keccak256(
            abi.encodePacked(
                bytes1(0xff),
                address(this),
                salt,
                keccak256(bytecode)
            )
        );
        return address(uint160(uint256(hash)));
    }

    /// @notice สร้าง proposal สำหรับ upgrade contract ผ่าน Timelock
    function scheduleUpgrade(
        string calldata name,
        address newAddress,
        string calldata newVersion
    ) external onlyRole(DEFAULT_ADMIN_ROLE) returns (bytes32 operationId) {
        bytes memory data = abi.encodeCall(
            registry.update,
            (name, newAddress, newVersion)
        );

        timelock.schedule(
            address(registry),  // target
            0,                  // value
            data,               // calldata
            bytes32(0),         // predecessor
            keccak256(abi.encodePacked(name, newVersion, block.timestamp)), // salt
            timelock.getMinDelay()
        );

        return keccak256(abi.encode(address(registry), 0, data, bytes32(0)));
    }
}

// ====================================
// Test สำหรับ Protocol Infrastructure
// ====================================
contract ProtocolInfrastructureTest {
    ProtocolInfrastructure public infra;

    event TestPassed(string testName);
    event TestFailed(string testName, string reason);

    constructor() {
        address[] memory guardians = new address[](1);
        guardians[0] = address(this);

        infra = new ProtocolInfrastructure(
            address(this),
            guardians,
            2 days // timelock delay
        );
    }

    function testEmergencyPause() external {
        // Guardian สามารถ pause ได้
        infra.activateEmergencyPause();
        assert(infra.emergencyPaused());
        emit TestPassed("testEmergencyPause");
    }

    function testPredictAddress() external {
        bytes memory bytecode = type(ProtocolRegistry).creationCode;
        bytes memory constructorArgs = abi.encode(address(this));
        bytes memory fullBytecode = abi.encodePacked(bytecode, constructorArgs);
        bytes32 salt = keccak256("test-registry-v1");

        address predicted = infra.predictAddress(fullBytecode, salt);
        assert(predicted != address(0));
        emit TestPassed("testPredictAddress");
    }
}
```

### การ run tests

```bash
# ติดตั้ง dependencies
cd packages/contracts
forge install OpenZeppelin/openzeppelin-contracts

# Run tests
forge test --match-path "test/infrastructure/*" -vvv

# ตรวจสอบขนาด contracts
forge build --sizes

# Deploy ด้วย script
forge script script/DeployProtocol.s.sol --rpc-url $RPC_URL --broadcast
```

---

## สรุป Part 71

- **Monorepo Structure**: จัดระเบียบ Protocol ด้วย packages/contracts, packages/frontend, packages/subgraph, packages/sdk ใช้ Turborepo และ pnpm workspaces
- **Contract Size Limits**: EIP-170 จำกัดที่ 24,576 bytes แก้ด้วย External Libraries และ Diamond Pattern (EIP-2535)
- **CREATE2 Deployments**: ใช้สำหรับ deterministic addresses ที่เหมือนกันทุก chain ทำให้ cross-chain protocol ง่ายขึ้น
- **DAO-Controlled Upgrades**: ใช้ Governor + TimelockController ทำให้ community ควบคุม upgrade ได้ พร้อม Emergency Guardian สำหรับกรณีฉุกเฉิน
- **ProtocolRegistry**: บันทึกประวัติ deployments ทั้งหมด ตรวจสอบ code integrity และ lookup address จากชื่อ

## Next: Part 72 - Advanced Testing Strategies
