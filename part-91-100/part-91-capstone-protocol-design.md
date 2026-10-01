# Part 91: Capstone - การออกแบบ Full DeFi Protocol (OmniYield)

## บทนำ

ยินดีต้อนรับสู่ส่วน **Capstone** ของหลักสูตร Solidity นี้! ในส่วนนี้เราจะนำทุกอย่างที่เรียนมาตลอดหลักสูตรมารวมกัน และสร้าง **OmniYield** — DeFi Protocol ที่สมบูรณ์แบบ ซึ่งประกอบด้วย:

- **Multi-strategy yield optimizer** สำหรับเพิ่มผลตอบแทนสูงสุด
- **On-chain governance** พร้อม tokenomics ที่ครบวงจร
- **Security patterns** ระดับ production
- **Frontend integration** ด้วย React + Wagmi V2

Part 91 จะเน้นที่การ **ออกแบบ** Protocol ก่อนเขียนโค้ดจริง เพราะการออกแบบที่ดีคือหัวใจของ Protocol ที่ปลอดภัยและขยายได้

---

## 1. ภาพรวมของ OmniYield

### 1.1 วิสัยทัศน์ (Vision)

OmniYield คือ **yield aggregator** ที่:
1. รับ deposits จาก users ในรูป ERC-4626 Vault
2. จัดสรร assets ไปยัง strategies หลายตัว (Aave, Uniswap V3, Compound ฯลฯ)
3. Harvest yield อัตโนมัติและ compound กลับ
4. ให้ governance token (OYT) แก่ผู้ใช้
5. บริหารผ่าน on-chain governance โดย holders ของ veOYT

### 1.2 Core Components

```
OmniYield Protocol
├── Core Layer
│   ├── OmniYieldVault (ERC-4626)     - รับ deposits, ออก shares
│   ├── OmniRegistry                   - จัดการ contract addresses
│   └── FeeController                  - คำนวณและจัดเก็บค่าธรรมเนียม
│
├── Strategy Layer
│   ├── StrategyBase (abstract)        - base class สำหรับ strategies
│   ├── AaveStrategy                   - ฝากเงินใน Aave V3
│   └── UniswapV3Strategy              - provide liquidity ใน Uniswap V3
│
├── Governance Layer
│   ├── OYT Token (ERC-20Votes)        - governance token
│   ├── veOYT                          - vote-escrowed OYT
│   └── OmniGovernor                   - on-chain governance
│
└── Infrastructure
    ├── TimelockController             - delay สำหรับ governance actions
    ├── EmergencyAdmin                 - emergency pause/shutdown
    └── MultiSigWallet                 - admin multisig
```

---

## 2. Architecture Diagram (Text-Based)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        OmniYield Ecosystem                          │
└─────────────────────────────────────────────────────────────────────┘

  Users / Frontend
       │
       │ deposit(assets) / withdraw(shares)
       ▼
┌──────────────────────────────────────────────────────────────────┐
│                     OmniYieldVault (ERC-4626)                    │
│  ┌────────────────┐  ┌─────────────────┐  ┌──────────────────┐  │
│  │  ERC-4626 Core │  │ Strategy Manager│  │  Harvest Engine  │  │
│  │  - deposit()   │  │ - allocate()    │  │  - harvest()     │  │
│  │  - withdraw()  │  │ - rebalance()   │  │  - compound()    │  │
│  │  - redeem()    │  │ - removeStrat() │  │  - reportGain()  │  │
│  └────────────────┘  └────────┬────────┘  └──────────────────┘  │
└───────────────────────────────┼──────────────────────────────────┘
                                │ IStrategy interface
           ┌────────────────────┼────────────────────┐
           │                    │                    │
           ▼                    ▼                    ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│   AaveStrategy   │  │ UniswapV3Strategy│  │  FutureStrategy  │
│  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │
│  │ deposit()  │  │  │  │ deposit()  │  │  │  │ deposit()  │  │
│  │ withdraw() │  │  │  │ withdraw() │  │  │  │ withdraw() │  │
│  │ harvest()  │  │  │  │ harvest()  │  │  │  │ harvest()  │  │
│  │ totalAssets│  │  │  │ totalAssets│  │  │  │ totalAssets│  │
│  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │
│       │          │  │        │         │  │                   │
│       ▼          │  │        ▼         │  └──────────────────┘
│  ┌─────────┐     │  │  ┌──────────┐   │
│  │ Aave V3 │     │  │  │Uniswap V3│   │
│  │ Pool    │     │  │  │ Position │   │
│  └─────────┘     │  │  └──────────┘   │
└──────────────────┘  └──────────────────┘

           ┌─────────────────────────────┐
           │         OmniRegistry        │
           │  - strategyRegistry[]       │
           │  - contractAddresses{}      │
           │  - version control          │
           └──────────────┬──────────────┘
                          │
                          │ reads/writes
                          ▼
┌─────────────────────────────────────────────────────┐
│                  Governance Layer                    │
│                                                     │
│  ┌─────────────┐    ┌──────────────┐               │
│  │  OYT Token  │───▶│   veOYT      │               │
│  │ (ERC20Votes)│    │ (lock OYT)   │               │
│  └─────────────┘    └──────┬───────┘               │
│                             │ voting power           │
│                             ▼                        │
│              ┌──────────────────────────┐           │
│              │      OmniGovernor        │           │
│              │  - propose()             │           │
│              │  - castVote()            │           │
│              │  - execute()             │           │
│              └──────────┬───────────────┘           │
│                         │ timelock delay              │
│                         ▼                             │
│              ┌──────────────────────────┐           │
│              │   TimelockController     │           │
│              │  - queue()               │           │
│              │  - execute()             │           │
│              └──────────────────────────┘           │
└─────────────────────────────────────────────────────┘

           ┌─────────────────────────────┐
           │      EmergencyAdmin         │
           │  - pause all contracts      │
           │  - emergency withdraw       │
           │  - 3/5 multisig required    │
           └─────────────────────────────┘
```

### 2.1 Data Flow

```
DEPOSIT FLOW:
User ──deposit(USDC)──▶ Vault ──allocate()──▶ Strategies
                         │
                         └──mint shares──▶ User

HARVEST FLOW:
Keeper ──harvest()──▶ Vault ──harvest()──▶ Each Strategy
                                              │
                                              └── collect yield ──▶ Vault
                                                                      │
                                                   compound ◀─────────┘
                                                   (buy more assets)
                                                        │
                                                        └──▶ Strategies (re-allocate)

GOVERNANCE FLOW:
Holder ──lock OYT──▶ veOYT ──▶ voting power
Proposer ──propose()──▶ Governor ──vote period──▶ queue()──▶ Timelock ──execute()──▶ Vault/Registry
```

---

## 3. Interface Design

### 3.1 IVault Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC4626} from "@openzeppelin/contracts/interfaces/IERC4626.sol";

/// @title IVault - Interface สำหรับ OmniYield Vault
/// @notice กำหนด functions เพิ่มเติมนอกเหนือจาก ERC-4626 standard
interface IVault is IERC4626 {
    // ============================================================
    // Structs
    // ============================================================

    /// @notice ข้อมูลของแต่ละ strategy
    struct StrategyParams {
        address strategy;        // address ของ strategy contract
        uint256 allocation;      // สัดส่วน allocation (basis points, 10000 = 100%)
        uint256 lastHarvest;     // timestamp ของ harvest ล่าสุด
        uint256 totalDebt;       // assets ที่อยู่ใน strategy นี้
        uint256 totalGain;       // cumulative gain จาก strategy นี้
        uint256 totalLoss;       // cumulative loss จาก strategy นี้
        bool active;             // strategy ยังใช้งานอยู่หรือไม่
    }

    // ============================================================
    // Events
    // ============================================================

    event StrategyAdded(address indexed strategy, uint256 allocation);
    event StrategyRemoved(address indexed strategy);
    event StrategyRebalanced(address indexed strategy, uint256 newAllocation);
    event Harvested(address indexed strategy, uint256 gain, uint256 loss, uint256 debtPayment);
    event EmergencyShutdown(address indexed caller);
    event FeeUpdated(uint256 performanceFee, uint256 managementFee);

    // ============================================================
    // View Functions
    // ============================================================

    /// @notice คืน list ของ strategies ทั้งหมด
    function getStrategies() external view returns (address[] memory);

    /// @notice คืน params ของ strategy ที่ระบุ
    function getStrategyParams(address strategy) external view returns (StrategyParams memory);

    /// @notice คืน total assets ที่อยู่ใน vault (รวม strategies)
    function totalDebt() external view returns (uint256);

    /// @notice คืน assets ที่อยู่ใน vault เอง (idle funds)
    function idleAssets() external view returns (uint256);

    /// @notice คืน APY ประมาณการ (annualized, basis points)
    function estimatedAPY() external view returns (uint256);

    /// @notice vault อยู่ใน emergency shutdown หรือไม่
    function emergencyShutdown() external view returns (bool);

    // ============================================================
    // Strategy Management (Governance only)
    // ============================================================

    /// @notice เพิ่ม strategy ใหม่
    function addStrategy(address strategy, uint256 allocation) external;

    /// @notice ลบ strategy
    function removeStrategy(address strategy) external;

    /// @notice ปรับ allocation ของ strategies
    function rebalance(address[] calldata strategies, uint256[] calldata allocations) external;

    // ============================================================
    // Harvest (Keeper only)
    // ============================================================

    /// @notice harvest yield จาก strategy ที่ระบุ
    function harvest(address strategy) external returns (uint256 gain, uint256 loss);

    /// @notice harvest ทุก strategies
    function harvestAll() external;

    // ============================================================
    // Emergency (Emergency Admin only)
    // ============================================================

    /// @notice เปิด emergency shutdown
    function setEmergencyShutdown(bool shutdown) external;

    /// @notice ถอน assets ทั้งหมดออกจาก strategy เข้า vault (emergency)
    function emergencyWithdrawFromStrategy(address strategy) external;
}
```

### 3.2 IStrategy Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title IStrategy - Interface สำหรับ OmniYield Strategies
/// @notice ทุก strategy ต้อง implement interface นี้
interface IStrategy {
    // ============================================================
    // Events
    // ============================================================

    event Deposited(uint256 amount);
    event Withdrawn(uint256 amount);
    event Harvested(uint256 profit, uint256 loss);
    event EmergencyExit(uint256 amount);

    // ============================================================
    // View Functions
    // ============================================================

    /// @notice ชื่อของ strategy (human-readable)
    function name() external view returns (string memory);

    /// @notice address ของ vault ที่ strategy นี้ทำงานให้
    function vault() external view returns (address);

    /// @notice address ของ underlying asset
    function asset() external view returns (address);

    /// @notice total assets ที่ strategy นี้ manage อยู่ (รวม yield ที่ยังไม่ harvest)
    function totalAssets() external view returns (uint256);

    /// @notice estimated gain ที่ยังไม่ harvest
    function estimatedTotalAssets() external view returns (uint256);

    /// @notice strategy อยู่ใน emergency exit หรือไม่
    function emergencyExit() external view returns (bool);

    /// @notice APY ของ strategy นี้ (annualized, basis points)
    function apr() external view returns (uint256);

    // ============================================================
    // Core Operations (Vault only)
    // ============================================================

    /// @notice ฝาก assets เข้า strategy
    /// @param amount จำนวน assets ที่จะฝาก
    function deposit(uint256 amount) external;

    /// @notice ถอน assets ออกจาก strategy
    /// @param amount จำนวนที่ต้องการถอน
    /// @return withdrawn จำนวนที่ถอนได้จริง
    function withdraw(uint256 amount) external returns (uint256 withdrawn);

    /// @notice harvest yield และส่งกลับ vault
    /// @return profit กำไรที่ได้
    /// @return loss ขาดทุน (ถ้ามี)
    function harvest() external returns (uint256 profit, uint256 loss);

    // ============================================================
    // Emergency
    // ============================================================

    /// @notice ถอนทุกอย่างออกและส่งกลับ vault (emergency)
    function emergencyWithdraw() external;

    /// @notice migrate strategy ไปยัง strategy ใหม่
    function migrate(address newStrategy) external;
}
```

### 3.3 IGovernor Interface (Custom Extension)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title IOmniGovernor - Custom governor interface สำหรับ OmniYield
interface IOmniGovernor {
    // ============================================================
    // Enums
    // ============================================================

    /// @notice ประเภทของ proposal
    enum ProposalType {
        GENERAL,            // ทั่วไป
        STRATEGY_ADD,       // เพิ่ม strategy ใหม่
        STRATEGY_REMOVE,    // ลบ strategy
        FEE_CHANGE,         // เปลี่ยนค่าธรรมเนียม
        EMERGENCY_ACTION,   // emergency action
        PROTOCOL_UPGRADE    // upgrade contracts
    }

    /// @notice สถานะของ proposal
    enum ProposalState {
        Pending,     // รอ voting period เริ่ม
        Active,      // กำลัง voting
        Canceled,    // ถูก cancel
        Defeated,    // ไม่ผ่าน
        Succeeded,   // ผ่าน (รอ queue)
        Queued,      // อยู่ใน timelock queue
        Expired,     // หมดเวลา execute
        Executed     // executed แล้ว
    }

    // ============================================================
    // Structs
    // ============================================================

    struct ProposalMeta {
        ProposalType proposalType;
        string description;
        address proposer;
        uint256 startBlock;
        uint256 endBlock;
        bool executed;
        bool canceled;
    }

    struct VoteResult {
        uint256 forVotes;
        uint256 againstVotes;
        uint256 abstainVotes;
    }

    // ============================================================
    // Events
    // ============================================================

    event ProposalCreated(
        uint256 indexed proposalId,
        address proposer,
        ProposalType proposalType,
        string description
    );

    event VoteCast(
        address indexed voter,
        uint256 indexed proposalId,
        uint8 support,
        uint256 weight,
        string reason
    );

    event ProposalQueued(uint256 indexed proposalId, uint256 eta);
    event ProposalExecuted(uint256 indexed proposalId);
    event ProposalCanceled(uint256 indexed proposalId);

    // ============================================================
    // View Functions
    // ============================================================

    function proposalMeta(uint256 proposalId) external view returns (ProposalMeta memory);
    function proposalVotes(uint256 proposalId) external view returns (VoteResult memory);
    function hasVoted(uint256 proposalId, address account) external view returns (bool);
    function quorumForProposal(uint256 proposalId) external view returns (uint256);

    // ============================================================
    // Actions
    // ============================================================

    function propose(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        string memory description,
        ProposalType proposalType
    ) external returns (uint256 proposalId);

    function castVote(uint256 proposalId, uint8 support) external returns (uint256 weight);

    function castVoteWithReason(
        uint256 proposalId,
        uint8 support,
        string calldata reason
    ) external returns (uint256 weight);

    function queue(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        bytes32 descriptionHash
    ) external returns (uint256 proposalId);

    function execute(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        bytes32 descriptionHash
    ) external payable returns (uint256 proposalId);

    function cancel(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        bytes32 descriptionHash
    ) external returns (uint256 proposalId);
}
```

### 3.4 IRegistry Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title IRegistry - Interface สำหรับ OmniYield Registry
/// @notice เก็บ addresses ของ contracts ทั้งหมดใน protocol
interface IRegistry {
    // ============================================================
    // Structs
    // ============================================================

    struct ContractRecord {
        address contractAddress;
        uint256 version;
        uint256 deployedAt;
        bool deprecated;
        string description;
    }

    // ============================================================
    // Events
    // ============================================================

    event ContractRegistered(bytes32 indexed key, address indexed contractAddress, uint256 version);
    event ContractDeprecated(bytes32 indexed key, address indexed contractAddress);
    event ContractUpgraded(bytes32 indexed key, address indexed oldAddress, address indexed newAddress);

    // ============================================================
    // View Functions
    // ============================================================

    /// @notice คืน address ปัจจุบันของ contract ที่ระบุ
    function getAddress(bytes32 key) external view returns (address);

    /// @notice คืน record เต็มของ contract
    function getRecord(bytes32 key) external view returns (ContractRecord memory);

    /// @notice คืน history ของ contract (เพื่อ audit trail)
    function getHistory(bytes32 key) external view returns (ContractRecord[] memory);

    /// @notice ตรวจว่า contract นี้ถูก register อยู่หรือไม่
    function isRegistered(address contractAddress) external view returns (bool);

    /// @notice keys ทั้งหมดที่ register อยู่
    function getAllKeys() external view returns (bytes32[] memory);

    // ============================================================
    // Management (Governance only)
    // ============================================================

    /// @notice register contract ใหม่
    function register(
        bytes32 key,
        address contractAddress,
        string calldata description
    ) external;

    /// @notice upgrade contract (เพิ่ม version ใหม่)
    function upgrade(bytes32 key, address newAddress) external;

    /// @notice deprecate contract
    function deprecate(bytes32 key) external;

    // ============================================================
    // Registry Keys (constants)
    // ============================================================

    // ใช้ bytes32 constants สำหรับ keys เพื่อป้องกัน typos
    // bytes32 constant VAULT = keccak256("VAULT");
    // bytes32 constant REGISTRY = keccak256("REGISTRY");
    // bytes32 constant GOVERNOR = keccak256("GOVERNOR");
    // bytes32 constant TIMELOCK = keccak256("TIMELOCK");
    // bytes32 constant OYT_TOKEN = keccak256("OYT_TOKEN");
    // bytes32 constant VE_OYT = keccak256("VE_OYT");
    // bytes32 constant FEE_CONTROLLER = keccak256("FEE_CONTROLLER");
    // bytes32 constant EMERGENCY_ADMIN = keccak256("EMERGENCY_ADMIN");
}
```

---

## 4. Storage Layout Planning (EIP-7201)

### 4.1 ทำไมต้องใช้ EIP-7201?

EIP-7201 (Namespaced Storage Layout) แก้ปัญหา storage collision ใน upgradeable contracts โดยใช้ **diamond storage pattern** ที่กำหนด storage slot ด้วย hash แทนตำแหน่งตายตัว

```
ปัญหาปกติ:
contract A {
    uint256 x;  // slot 0
    address y;  // slot 1
}

contract B is A {
    uint256 z;  // slot 2
}

ปัญหา: ถ้า upgrade เพิ่ม variable ใน A จะ collision กับ B

EIP-7201 แก้ด้วย:
storage.A อยู่ที่ keccak256("omniyield.storage.vault") - 1
storage.B อยู่ที่ keccak256("omniyield.storage.strategy") - 1
ไม่มีทางชนกัน!
```

### 4.2 Vault Storage Layout

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title VaultStorage - Storage layout สำหรับ OmniYieldVault
/// @dev ใช้ EIP-7201 namespaced storage เพื่อป้องกัน collision
library VaultStorage {
    // ============================================================
    // Storage Namespace
    // ============================================================

    // keccak256(abi.encode(uint256(keccak256("omniyield.storage.vault")) - 1))
    // & ~bytes32(uint256(0xff))
    bytes32 private constant VAULT_STORAGE_LOCATION =
        0x3d2d0f5f3d5c5b0e5f5c5b0e5f5c5b0e5f5c5b0e5f5c5b0e5f5c5b0e5f5c00;

    // ============================================================
    // Storage Struct
    // ============================================================

    struct Layout {
        // ─── Asset & Shares ───────────────────────────────────────
        address asset;              // underlying asset (e.g., USDC)
        uint256 totalDebt;          // total assets deployed to strategies
        uint256 lastReport;         // timestamp ของ harvest ล่าสุด

        // ─── Strategies ──────────────────────────────────────────
        address[] strategies;       // ordered list ของ strategies
        mapping(address => StrategyParams) strategyParams;

        // ─── Fee Configuration ───────────────────────────────────
        uint256 performanceFee;     // basis points (e.g., 2000 = 20%)
        uint256 managementFee;      // basis points per year (e.g., 200 = 2%)
        address feeRecipient;       // ผู้รับค่าธรรมเนียม

        // ─── Access Control ──────────────────────────────────────
        address governance;         // address ของ governor/timelock
        address keeper;             // address ของ keeper (สำหรับ harvest)
        address emergencyAdmin;     // address ของ emergency admin

        // ─── Emergency ──────────────────────────────────────────
        bool emergencyShutdown;     // หยุดทุก operations

        // ─── Deposit Limits ──────────────────────────────────────
        uint256 depositLimit;       // ไม่เกิน limit นี้
        bool depositsPaused;        // pause deposits ชั่วคราว
    }

    struct StrategyParams {
        uint256 allocation;         // basis points
        uint256 lastHarvest;        // timestamp
        uint256 totalDebt;          // assets ใน strategy
        uint256 totalGain;          // cumulative gain
        uint256 totalLoss;          // cumulative loss
        bool active;
    }

    // ============================================================
    // Storage Accessor
    // ============================================================

    function layout() internal pure returns (Layout storage $) {
        assembly {
            $.slot := VAULT_STORAGE_LOCATION
        }
    }
}
```

### 4.3 Strategy Storage Layout

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title StrategyStorage - Storage layout สำหรับ Strategy contracts
library StrategyStorage {
    // keccak256(abi.encode(uint256(keccak256("omniyield.storage.strategy")) - 1))
    // & ~bytes32(uint256(0xff))
    bytes32 private constant STRATEGY_STORAGE_LOCATION =
        0x7a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f00;

    struct Layout {
        // ─── Identity ────────────────────────────────────────────
        string strategyName;
        address vault;              // vault ที่ strategy ทำงานให้
        address asset;              // underlying asset

        // ─── Position Tracking ───────────────────────────────────
        uint256 totalDeposited;     // total ที่ vault deposit มา
        uint256 totalWithdrawn;     // total ที่ withdraw กลับไป
        uint256 lastHarvest;        // timestamp ของ harvest ล่าสุด

        // ─── External Protocol State ─────────────────────────────
        // ใช้ storage ตามที่แต่ละ strategy ต้องการ
        // เช่น Aave: aTokenAddress, Uniswap: tokenId

        // ─── Emergency ──────────────────────────────────────────
        bool emergencyExit;
    }

    function layout() internal pure returns (Layout storage $) {
        bytes32 location = STRATEGY_STORAGE_LOCATION;
        assembly {
            $.slot := location
        }
    }
}
```

### 4.4 Governor Storage Layout

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title GovernorStorage - Storage layout สำหรับ OmniGovernor
library GovernorStorage {
    // keccak256(abi.encode(uint256(keccak256("omniyield.storage.governor")) - 1))
    // & ~bytes32(uint256(0xff))
    bytes32 private constant GOVERNOR_STORAGE_LOCATION =
        0x9b8c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c3d2e1f0a00;

    struct Layout {
        // ─── Timing ──────────────────────────────────────────────
        uint256 votingDelay;        // blocks ก่อนเริ่ม vote
        uint256 votingPeriod;       // blocks ของ voting period
        uint256 proposalThreshold;  // min votes to propose

        // ─── Quorum ──────────────────────────────────────────────
        uint256 quorumNumerator;    // % ของ total supply ที่ต้องมา vote

        // ─── Proposals ───────────────────────────────────────────
        mapping(uint256 => ProposalCore) proposals;
        mapping(uint256 => mapping(address => bool)) hasVoted;
        mapping(uint256 => ProposalVote) proposalVotes;

        // ─── Guardian ────────────────────────────────────────────
        address guardian;           // สามารถ cancel proposals ได้
    }

    struct ProposalCore {
        address proposer;
        uint256 voteStart;
        uint256 voteEnd;
        bool executed;
        bool canceled;
        uint8 proposalType;         // ProposalType enum
    }

    struct ProposalVote {
        uint256 againstVotes;
        uint256 forVotes;
        uint256 abstainVotes;
    }

    function layout() internal pure returns (Layout storage $) {
        bytes32 location = GOVERNOR_STORAGE_LOCATION;
        assembly {
            $.slot := location
        }
    }
}
```

### 4.5 Storage Slot Verification Script

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @notice Script สำหรับ verify storage slots ก่อน deploy
/// @dev รัน: forge script script/VerifyStorage.s.sol
contract VerifyStorageSlots {
    function run() external pure returns (
        bytes32 vaultSlot,
        bytes32 strategySlot,
        bytes32 governorSlot
    ) {
        // Vault
        vaultSlot = bytes32(
            uint256(keccak256("omniyield.storage.vault")) - 1
        ) & ~bytes32(uint256(0xff));

        // Strategy
        strategySlot = bytes32(
            uint256(keccak256("omniyield.storage.strategy")) - 1
        ) & ~bytes32(uint256(0xff));

        // Governor
        governorSlot = bytes32(
            uint256(keccak256("omniyield.storage.governor")) - 1
        ) & ~bytes32(uint256(0xff));

        // Log ด้วย event หรือ return เพื่อ verify
    }

    /// @notice ตรวจสอบว่า slots ไม่ collision กัน
    function assertNoCollision() external pure returns (bool) {
        bytes32 vault = bytes32(uint256(keccak256("omniyield.storage.vault")) - 1) & ~bytes32(uint256(0xff));
        bytes32 strategy = bytes32(uint256(keccak256("omniyield.storage.strategy")) - 1) & ~bytes32(uint256(0xff));
        bytes32 governor = bytes32(uint256(keccak256("omniyield.storage.governor")) - 1) & ~bytes32(uint256(0xff));

        require(vault != strategy, "VAULT-STRATEGY collision!");
        require(vault != governor, "VAULT-GOVERNOR collision!");
        require(strategy != governor, "STRATEGY-GOVERNOR collision!");

        return true;
    }
}
```

---

## 5. Deployment Plan

### 5.1 Pre-Deployment Checklist

```
PRE-DEPLOYMENT CHECKLIST
══════════════════════════════════════════════════════════════════

[ ] Audit เสร็จสิ้น (อย่างน้อย 1 firm + 1 contest)
[ ] Invariant tests ผ่านทุกข้อ (>= 10,000 runs)
[ ] Slither ไม่มี HIGH/MEDIUM ที่ยังไม่แก้
[ ] Gas benchmarks ทำแล้ว และยอมรับได้
[ ] Emergency procedures ได้รับการ test บน testnet
[ ] Multisig setup แล้ว (Gnosis Safe 3/5)
[ ] Deployment scripts test บน fork แล้ว
[ ] Frontend พร้อม connect
[ ] Monitoring/alerting setup แล้ว
[ ] Documentation เสร็จสิ้น
```

### 5.2 Deployment Order

```
DEPLOYMENT SEQUENCE
══════════════════════════════════════════════════════════════════

Phase 1: Infrastructure
────────────────────────────────────────────────────────────────
Step 1:  Deploy MultisigWallet (Gnosis Safe 3/5)
         - Owners: deployer + 4 core team members
         - ยังไม่ transfer ownership

Step 2:  Deploy OmniRegistry
         - Constructor: (deployer as initial admin)
         - Verify on Etherscan

Step 3:  Deploy TimelockController
         - minDelay: 48 hours (initial)
         - proposers: [governor] (เพิ่มทีหลัง)
         - executors: [address(0)] = anyone can execute
         - admin: deployer (จะ revoke ทีหลัง)

Phase 2: Tokens
────────────────────────────────────────────────────────────────
Step 4:  Deploy OYT Token
         - Constructor: (name, symbol, initialSupply, treasury)
         - Register ใน Registry: registry.register("OYT_TOKEN", oytAddress, ...)

Step 5:  Deploy veOYT
         - Constructor: (oytToken, "veOYT", "veOYT")
         - Register ใน Registry

Phase 3: Governance
────────────────────────────────────────────────────────────────
Step 6:  Deploy OmniGovernor
         - Constructor: (veOYT, timelock, params...)
         - Register ใน Registry

Step 7:  Setup Timelock Roles
         - timelock.grantRole(PROPOSER_ROLE, governor)
         - timelock.grantRole(CANCELLER_ROLE, governor)
         - timelock.revokeRole(TIMELOCK_ADMIN_ROLE, deployer)

Phase 4: Core Protocol
────────────────────────────────────────────────────────────────
Step 8:  Deploy FeeController
         - Constructor: (registry, performanceFee=2000, managementFee=200)
         - Register ใน Registry

Step 9:  Deploy OmniYieldVault
         - Constructor: (asset=USDC, registry, name, symbol)
         - Register ใน Registry
         - Set: vault.setGovernance(timelock)
         - Set: vault.setKeeper(keeperAddress)
         - Set: vault.setEmergencyAdmin(multisig)

Phase 5: Strategies
────────────────────────────────────────────────────────────────
Step 10: Deploy AaveStrategy
         - Constructor: (vault, aavePool, aToken, asset)
         - vault.addStrategy(aaveStrategy, 5000) // 50% allocation

Step 11: Deploy UniswapV3Strategy
         - Constructor: (vault, positionManager, asset, quoteToken, fee)
         - vault.addStrategy(uniStrategy, 3000) // 30% allocation

Note: 20% allocation สำหรับ idle buffer (safety)

Phase 6: Handoff to Governance
────────────────────────────────────────────────────────────────
Step 12: Transfer ownership ทั้งหมดไปยัง Timelock/Multisig
         - registry.transferOwnership(timelock)
         - feeController.transferOwnership(timelock)
         - vault ได้ set governance ไปแล้วใน Step 9

Step 13: Revoke deployer privileges
         - ตรวจสอบว่าไม่มี deployer address ใน roles ใด ๆ
         - สร้าง verification script และรัน
```

### 5.3 Deployment Script (Foundry)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Script, console2} from "forge-std/Script.sol";
import {OmniRegistry} from "../src/OmniRegistry.sol";
import {TimelockController} from "@openzeppelin/contracts/governance/TimelockController.sol";
import {OYTToken} from "../src/governance/OYTToken.sol";
import {VeOYT} from "../src/governance/VeOYT.sol";
import {OmniGovernor} from "../src/governance/OmniGovernor.sol";
import {OmniYieldVault} from "../src/OmniYieldVault.sol";
import {AaveStrategy} from "../src/strategies/AaveStrategy.sol";
import {UniswapV3Strategy} from "../src/strategies/UniswapV3Strategy.sol";

/// @title DeployOmniYield - Deployment script หลัก
/// @dev รัน: forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
contract DeployOmniYield is Script {
    // ============================================================
    // Configuration
    // ============================================================

    // Mainnet addresses (ปรับตาม network)
    address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address constant AAVE_POOL = 0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2;
    address constant AAVE_AUSDC = 0x98C23E9d8f34FEFb1B7BD6a91B7CF122b5ee8F1;
    address constant UNISWAP_POSITION_MANAGER = 0xC36442b4a4522E871399CD717aBDD847Ab11FE88;
    address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;

    // Protocol parameters
    uint256 constant INITIAL_OYT_SUPPLY = 100_000_000e18; // 100M tokens
    uint256 constant TIMELOCK_MIN_DELAY = 48 hours;
    uint256 constant PERFORMANCE_FEE = 2000; // 20%
    uint256 constant MANAGEMENT_FEE = 200;   // 2% per year
    uint256 constant AAVE_ALLOCATION = 5000;  // 50%
    uint256 constant UNIV3_ALLOCATION = 3000; // 30%

    // Multisig owners
    address[] MULTISIG_OWNERS = [
        0x1111111111111111111111111111111111111111, // Owner 1
        0x2222222222222222222222222222222222222222, // Owner 2
        0x3333333333333333333333333333333333333333, // Owner 3
        0x4444444444444444444444444444444444444444, // Owner 4
        0x5555555555555555555555555555555555555555  // Owner 5
    ];

    // Deployed contracts
    OmniRegistry registry;
    TimelockController timelock;
    OYTToken oyt;
    VeOYT veOYT;
    OmniGovernor governor;
    OmniYieldVault vault;
    AaveStrategy aaveStrategy;
    UniswapV3Strategy uniStrategy;

    // ============================================================
    // Main Deploy Function
    // ============================================================

    function run() external {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        address deployer = vm.addr(deployerPrivateKey);

        console2.log("=================================================");
        console2.log("Deploying OmniYield Protocol");
        console2.log("Deployer:", deployer);
        console2.log("Chain ID:", block.chainid);
        console2.log("=================================================");

        vm.startBroadcast(deployerPrivateKey);

        // Phase 1: Infrastructure
        _deployInfrastructure(deployer);

        // Phase 2: Tokens
        _deployTokens(deployer);

        // Phase 3: Governance
        _deployGovernance();

        // Phase 4: Core Protocol
        _deployCoreProtocol();

        // Phase 5: Strategies
        _deployStrategies();

        // Phase 6: Handoff
        _handoffToGovernance();

        vm.stopBroadcast();

        // Print summary
        _printDeploymentSummary();
    }

    function _deployInfrastructure(address deployer) internal {
        console2.log("\n--- Phase 1: Infrastructure ---");

        // Deploy Registry
        registry = new OmniRegistry(deployer);
        console2.log("Registry:", address(registry));

        // Deploy Timelock (proposers will be set later)
        address[] memory proposers = new address[](0);
        address[] memory executors = new address[](1);
        executors[0] = address(0); // anyone can execute after delay

        timelock = new TimelockController(
            TIMELOCK_MIN_DELAY,
            proposers,
            executors,
            deployer // initial admin
        );
        console2.log("Timelock:", address(timelock));

        // Register in Registry
        registry.register(keccak256("REGISTRY"), address(registry), "OmniRegistry v1");
        registry.register(keccak256("TIMELOCK"), address(timelock), "TimelockController v1");
    }

    function _deployTokens(address deployer) internal {
        console2.log("\n--- Phase 2: Tokens ---");

        // Deploy OYT
        oyt = new OYTToken(
            "OmniYield Token",
            "OYT",
            INITIAL_OYT_SUPPLY,
            deployer // initial mint to deployer (will distribute later)
        );
        console2.log("OYT Token:", address(oyt));

        // Deploy veOYT
        veOYT = new VeOYT(address(oyt));
        console2.log("veOYT:", address(veOYT));

        // Register
        registry.register(keccak256("OYT_TOKEN"), address(oyt), "OYT Token v1");
        registry.register(keccak256("VE_OYT"), address(veOYT), "veOYT v1");
    }

    function _deployGovernance() internal {
        console2.log("\n--- Phase 3: Governance ---");

        // Deploy Governor
        governor = new OmniGovernor(
            address(veOYT),
            address(timelock)
        );
        console2.log("Governor:", address(governor));

        // Setup Timelock roles
        bytes32 PROPOSER_ROLE = timelock.PROPOSER_ROLE();
        bytes32 CANCELLER_ROLE = timelock.CANCELLER_ROLE();
        bytes32 TIMELOCK_ADMIN_ROLE = timelock.DEFAULT_ADMIN_ROLE();

        timelock.grantRole(PROPOSER_ROLE, address(governor));
        timelock.grantRole(CANCELLER_ROLE, address(governor));
        // Revoke deployer's admin role (governor is self-administered)
        timelock.revokeRole(TIMELOCK_ADMIN_ROLE, msg.sender);

        // Register
        registry.register(keccak256("GOVERNOR"), address(governor), "OmniGovernor v1");
    }

    function _deployCoreProtocol() internal {
        console2.log("\n--- Phase 4: Core Protocol ---");

        // Deploy Vault
        vault = new OmniYieldVault(
            USDC,
            address(registry),
            "OmniYield USDC Vault",
            "omUSDC"
        );
        console2.log("Vault:", address(vault));

        // Configure Vault
        vault.setGovernance(address(timelock));
        // vault.setKeeper(keeperAddress); // set keeper จาก env
        vault.setDepositLimit(type(uint256).max); // no limit initially

        // Register
        registry.register(keccak256("VAULT"), address(vault), "OmniYieldVault v1");
    }

    function _deployStrategies() internal {
        console2.log("\n--- Phase 5: Strategies ---");

        // Deploy AaveStrategy
        aaveStrategy = new AaveStrategy(
            address(vault),
            AAVE_POOL,
            AAVE_AUSDC,
            USDC
        );
        console2.log("AaveStrategy:", address(aaveStrategy));

        // Deploy UniswapV3Strategy
        uniStrategy = new UniswapV3Strategy(
            address(vault),
            UNISWAP_POSITION_MANAGER,
            USDC,
            WETH,
            500 // 0.05% fee tier
        );
        console2.log("UniswapV3Strategy:", address(uniStrategy));

        // Add strategies to vault
        vault.addStrategy(address(aaveStrategy), AAVE_ALLOCATION);
        vault.addStrategy(address(uniStrategy), UNIV3_ALLOCATION);

        // Register
        registry.register(keccak256("STRATEGY_AAVE"), address(aaveStrategy), "AaveStrategy v1");
        registry.register(keccak256("STRATEGY_UNIV3"), address(uniStrategy), "UniswapV3Strategy v1");
    }

    function _handoffToGovernance() internal {
        console2.log("\n--- Phase 6: Handoff to Governance ---");

        // Transfer registry ownership to timelock
        registry.transferOwnership(address(timelock));
        console2.log("Registry ownership -> Timelock");

        // Vault governance already set to timelock

        console2.log("Handoff complete! Protocol is now governed by OmniGovernor.");
    }

    function _printDeploymentSummary() internal view {
        console2.log("\n=================================================");
        console2.log("DEPLOYMENT SUMMARY");
        console2.log("=================================================");
        console2.log("Registry:        ", address(registry));
        console2.log("Timelock:        ", address(timelock));
        console2.log("OYT Token:       ", address(oyt));
        console2.log("veOYT:           ", address(veOYT));
        console2.log("Governor:        ", address(governor));
        console2.log("Vault:           ", address(vault));
        console2.log("AaveStrategy:    ", address(aaveStrategy));
        console2.log("UniswapStrategy: ", address(uniStrategy));
        console2.log("=================================================");
        console2.log("Save these addresses in your config!");
    }
}
```

### 5.4 Post-Deployment Verification

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Script, console2} from "forge-std/Script.sol";

/// @title VerifyDeployment - ตรวจสอบหลัง deploy
contract VerifyDeployment is Script {
    // Paste deployed addresses here
    address constant REGISTRY = address(0);  // TODO: fill in
    address constant TIMELOCK = address(0);  // TODO: fill in
    address constant GOVERNOR = address(0);  // TODO: fill in
    address constant VAULT = address(0);     // TODO: fill in

    function run() external view {
        console2.log("Verifying deployment...");

        // 1. ตรวจ registry
        IRegistry reg = IRegistry(REGISTRY);
        require(reg.getAddress(keccak256("VAULT")) == VAULT, "VAULT not registered");
        require(reg.getAddress(keccak256("GOVERNOR")) == GOVERNOR, "GOVERNOR not registered");
        require(reg.getAddress(keccak256("TIMELOCK")) == TIMELOCK, "TIMELOCK not registered");

        // 2. ตรวจ vault governance
        IVault v = IVault(VAULT);
        // require(v.governance() == TIMELOCK, "Vault governance not set to timelock");

        // 3. ตรวจ timelock proposer
        ITimelockController tl = ITimelockController(TIMELOCK);
        require(tl.hasRole(tl.PROPOSER_ROLE(), GOVERNOR), "Governor is not proposer");
        require(!tl.hasRole(tl.DEFAULT_ADMIN_ROLE(), msg.sender), "Deployer still has admin!");

        console2.log("All checks passed!");
    }
}
```

---

## 6. Workshop: ออกแบบ Protocol ของคุณเอง

### Workshop 6.1: Interface Design Exercise

**โจทย์**: เพิ่ม `IFeeController` interface ให้สมบูรณ์

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title IFeeController - TODO: เพิ่ม functions ให้ครบ
interface IFeeController {
    // TODO: เพิ่ม structs, events, functions สำหรับ:
    // 1. คำนวณ performance fee จาก profit
    // 2. คำนวณ management fee จาก AUM (assets under management)
    // 3. อัปเดต fee configuration (ผ่าน governance)
    // 4. Distribute fees ไปยัง recipients
    // 5. Query accumulated fees

    // เขียนคำตอบของคุณที่นี่...
}
```

**เฉลย**:

```solidity
interface IFeeController {
    struct FeeConfig {
        uint256 performanceFee;   // basis points
        uint256 managementFee;    // basis points per year
        address performanceFeeRecipient;
        address managementFeeRecipient;
        address protocolTreasury;
        uint256 protocolCut;      // % ของ fees ที่ไปโปรโตคอล
    }

    event FeeConfigUpdated(
        uint256 newPerformanceFee,
        uint256 newManagementFee
    );
    event FeesCollected(
        address indexed recipient,
        uint256 amount
    );

    // View
    function getFeeConfig() external view returns (FeeConfig memory);
    function calculatePerformanceFee(uint256 profit) external view returns (uint256 fee);
    function calculateManagementFee(uint256 aum, uint256 timeSinceLastFee) external view returns (uint256 fee);
    function pendingFees(address recipient) external view returns (uint256);

    // Actions
    function setFeeConfig(FeeConfig calldata config) external;
    function collectFees(address vault) external returns (uint256 totalFees);
    function claimFees() external;
}
```

### Workshop 6.2: Storage Layout Design

**โจทย์**: เขียน `FeeControllerStorage` ด้วย EIP-7201

```solidity
// TODO: เขียน FeeControllerStorage library
// ต้องมี:
// 1. bytes32 storage location ที่ถูกต้อง
// 2. Layout struct ที่ครบถ้วน
// 3. layout() accessor function

library FeeControllerStorage {
    // เขียนคำตอบของคุณที่นี่...
}
```

### Workshop 6.3: Deployment Order Analysis

**โจทย์**: ระบุว่า contracts ใดต้อง deploy ก่อน/หลัง และทำไม

```
Contracts:
A. OmniYieldVault
B. OmniRegistry  
C. AaveStrategy
D. OmniGovernor
E. TimelockController
F. OYT Token
G. veOYT

คำถาม:
1. Contract ใดต้อง deploy เป็นอันดับแรก?
2. Contract ใดสามารถ deploy ได้พร้อมกัน (parallel)?
3. มี circular dependency ไหม? ถ้ามีแก้อย่างไร?
4. Constructor argument ไหนต้องรู้ address ของ contract อื่น?
```

---

## 7. Security Considerations ในการออกแบบ

### 7.1 Principle of Least Privilege

```
Role Matrix สำหรับ OmniYield:
══════════════════════════════════════════════════════════════

Function                │ Governance │ Keeper │ EmergAdmin │ Anyone
────────────────────────┼────────────┼────────┼────────────┼────────
addStrategy()           │     ✓      │        │            │
removeStrategy()        │     ✓      │        │            │
rebalance()             │     ✓      │        │            │
harvest()               │            │   ✓    │            │
setDepositLimit()       │     ✓      │        │            │
setFees()               │     ✓      │        │            │
setEmergencyShutdown()  │            │        │     ✓      │
emergencyWithdraw()     │            │        │     ✓      │
deposit()               │            │        │            │   ✓
withdraw()              │            │        │            │   ✓
```

### 7.2 Upgrade Safety

```
Upgrade Decision Matrix:
══════════════════════════════════════════════════════════════

Component           │ Upgradeable? │ Reason
────────────────────┼──────────────┼───────────────────────
OmniYieldVault      │     Yes      │ UUPS (EIP-1822)
Strategies          │     No       │ Replace pattern แทน
OYT Token           │     No       │ Token ไม่ควร upgradeable
veOYT               │     No       │ Same as token
OmniGovernor        │     No       │ Deploy ใหม่ + migrate
OmniRegistry        │     Yes      │ UUPS
TimelockController  │     No       │ OZ standard
```

### 7.3 Emergency Circuit Breakers

```
Emergency Levels:
══════════════════════════════════════════════════════════════

Level 1 - Pause Deposits (1/3 multisig)
  → หยุดรับ deposits ใหม่
  → withdrawals ยังทำได้
  → harvest ยังทำได้

Level 2 - Full Pause (2/3 multisig)  
  → หยุดทุก user interactions
  → keeper operations ยังทำได้

Level 3 - Emergency Shutdown (3/5 multisig)
  → ถอน assets ทั้งหมดออกจาก strategies
  → users สามารถ redeem shares ได้
  → ไม่มี deposits/harvests

Level 4 - Protocol Migration (governance vote)
  → migrate ทุกอย่างไปยัง protocol ใหม่
  → requires 2-week timelock
```

---

## 8. Project Structure

```
omniyield/
├── src/
│   ├── OmniYieldVault.sol          # Main vault (ERC-4626)
│   ├── OmniRegistry.sol            # Contract registry
│   ├── FeeController.sol           # Fee management
│   ├── storage/
│   │   ├── VaultStorage.sol        # EIP-7201 vault storage
│   │   ├── StrategyStorage.sol     # EIP-7201 strategy storage
│   │   └── GovernorStorage.sol     # EIP-7201 governor storage
│   ├── strategies/
│   │   ├── StrategyBase.sol        # Abstract base
│   │   ├── AaveStrategy.sol        # Aave V3 strategy
│   │   └── UniswapV3Strategy.sol   # Uniswap V3 strategy
│   ├── governance/
│   │   ├── OYTToken.sol            # Governance token
│   │   ├── VeOYT.sol               # Vote-escrowed token
│   │   └── OmniGovernor.sol        # Governor contract
│   └── interfaces/
│       ├── IVault.sol
│       ├── IStrategy.sol
│       ├── IGovernor.sol
│       └── IRegistry.sol
├── test/
│   ├── unit/
│   ├── integration/
│   ├── invariant/
│   └── fork/
├── script/
│   ├── Deploy.s.sol
│   ├── VerifyDeployment.s.sol
│   └── helpers/
├── docs/
│   ├── architecture.md
│   ├── security.md
│   └── deployment.md
└── foundry.toml
```

### 8.1 foundry.toml Configuration

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
optimizer = true
optimizer_runs = 200
via_ir = true

[profile.default.fuzz]
runs = 10000
max_test_rejects = 65536

[profile.default.invariant]
runs = 1000
depth = 100
fail_on_revert = true

[rpc_endpoints]
mainnet = "${MAINNET_RPC_URL}"
sepolia = "${SEPOLIA_RPC_URL}"
anvil = "http://localhost:8545"

[etherscan]
mainnet = { key = "${ETHERSCAN_API_KEY}" }
sepolia = { key = "${ETHERSCAN_API_KEY}" }

[fmt]
line_length = 120
tab_width = 4
bracket_spacing = true
```

---

## สรุป Part 91

- **OmniYield** คือ multi-strategy yield optimizer ที่ประกอบด้วย Vault, Strategies, Governance และ Infrastructure
- **Architecture** ออกแบบให้ modular โดยใช้ interfaces เพื่อให้ extend ได้ง่าย
- **Interfaces** (IVault, IStrategy, IGovernor, IRegistry) กำหนด contract ก่อนเขียน implementation
- **EIP-7201 namespaced storage** ป้องกัน storage collision ใน upgradeable contracts
- **Deployment order** สำคัญมาก ต้อง deploy dependencies ก่อน
- **Multi-sig handoff** ทำให้ protocol ถูก govern โดย community ไม่ใช่ deployer
- **Security by design**: least privilege, circuit breakers, upgrade safety

## Next: Part 92 - Capstone Core Contract Implementation

ใน Part 92 เราจะ implement contracts จริงทั้งหมด:
- `OmniYieldVault`: ERC-4626 + multi-strategy
- `StrategyBase`: abstract base class
- `AaveStrategy` + `UniswapV3Strategy`
- `OmniRegistry`
- Full Foundry test suite
