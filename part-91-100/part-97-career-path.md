# Part 97: Career Path in Solidity Development

## บทนำ

การเป็น Solidity Developer ไม่ได้หมายความว่าแค่เขียน code ได้ แต่ต้องเข้าใจระบบนิเวศ DeFi, security threats, economics, และ community building ในบทนี้เราจะ map เส้นทางอาชีพตั้งแต่ Junior จนถึง Principal Engineer และแนะนำวิธีสร้าง portfolio ที่แข็งแกร่ง

## Career Levels ใน Solidity Development

### ภาพรวม Career Ladder

```
Junior Developer (0-2 ปี)
    ↓ เขียน Solidity ได้, ทำ testing พื้นฐาน
Mid-Level Developer (2-4 ปี)
    ↓ design patterns, security basics, DeFi protocols
Senior Developer (4-7 ปี)
    ↓ architecture, advanced security, economic modeling
Lead Engineer (7-10 ปี)
    ↓ team leadership, protocol design, audit experience
Principal Engineer (10+ ปี)
    ↓ ecosystem influence, novel research, protocol governance
```

### Skills Matrix

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title SkillsMatrix
 * @notice ตารางทักษะที่ต้องมีในแต่ละ level
 * @dev ใช้เป็น self-assessment tool
 */
contract SkillsMatrix {
    enum Level { None, Basic, Intermediate, Advanced, Expert }

    struct SkillSet {
        // Solidity & Smart Contracts
        Level solidityFundamentals;
        Level designPatterns;
        Level gasOptimization;
        Level upgradeableContracts;
        Level crossChainDev;

        // Testing & Quality
        Level unitTesting;
        Level fuzzTesting;
        Level formalVerification;
        Level testCoverage;

        // Security
        Level vulnerabilityKnowledge;
        Level securityAuditing;
        Level threatModeling;
        Level economicSecurity;

        // Architecture
        Level systemDesign;
        Level protocolDesign;
        Level tokenomics;
        Level defiMechanisms;

        // Business & Soft Skills
        Level technicalWriting;
        Level codeReview;
        Level mentoring;
        Level communityBuilding;
    }

    // Target skills สำหรับแต่ละ level
    mapping(string => SkillSet) public targetSkills;

    function initializeTargets() external {
        // Junior Level
        targetSkills["junior"] = SkillSet({
            solidityFundamentals: Level.Intermediate,
            designPatterns: Level.Basic,
            gasOptimization: Level.Basic,
            upgradeableContracts: Level.None,
            crossChainDev: Level.None,

            unitTesting: Level.Intermediate,
            fuzzTesting: Level.None,
            formalVerification: Level.None,
            testCoverage: Level.Basic,

            vulnerabilityKnowledge: Level.Basic,
            securityAuditing: Level.None,
            threatModeling: Level.None,
            economicSecurity: Level.None,

            systemDesign: Level.Basic,
            protocolDesign: Level.None,
            tokenomics: Level.None,
            defiMechanisms: Level.Basic,

            technicalWriting: Level.Basic,
            codeReview: Level.Basic,
            mentoring: Level.None,
            communityBuilding: Level.None
        });

        // Senior Level
        targetSkills["senior"] = SkillSet({
            solidityFundamentals: Level.Expert,
            designPatterns: Level.Advanced,
            gasOptimization: Level.Advanced,
            upgradeableContracts: Level.Advanced,
            crossChainDev: Level.Intermediate,

            unitTesting: Level.Expert,
            fuzzTesting: Level.Advanced,
            formalVerification: Level.Intermediate,
            testCoverage: Level.Expert,

            vulnerabilityKnowledge: Level.Advanced,
            securityAuditing: Level.Intermediate,
            threatModeling: Level.Intermediate,
            economicSecurity: Level.Intermediate,

            systemDesign: Level.Advanced,
            protocolDesign: Level.Intermediate,
            tokenomics: Level.Intermediate,
            defiMechanisms: Level.Advanced,

            technicalWriting: Level.Advanced,
            codeReview: Level.Advanced,
            mentoring: Level.Basic,
            communityBuilding: Level.Basic
        });
    }
}
```

## Junior Developer (0-2 ปี)

### ทักษะที่จำเป็น

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title JuniorSkillsDemo
 * @notice ตัวอย่าง code ที่ Junior Developer ควรทำได้
 * @dev สร้าง simple token พร้อม basic safety checks
 */
contract JuniorSkillsDemo {
    // ✅ Junior ต้องรู้: Basic types, mappings, events
    mapping(address => uint256) public balances;
    uint256 public totalSupply;
    string public name;
    string public symbol;

    event Transfer(address indexed from, address indexed to, uint256 amount);

    // ✅ Junior ต้องรู้: Constructor, modifiers
    constructor(string memory _name, string memory _symbol, uint256 initialSupply) {
        name = _name;
        symbol = _symbol;
        totalSupply = initialSupply;
        balances[msg.sender] = initialSupply;
    }

    modifier validAmount(uint256 amount) {
        require(amount > 0, "Amount must be positive");
        _;
    }

    modifier hasSufficientBalance(address account, uint256 amount) {
        require(balances[account] >= amount, "Insufficient balance");
        _;
    }

    // ✅ Junior ต้องรู้: Basic functions, require statements
    function transfer(address to, uint256 amount)
        external
        validAmount(amount)
        hasSufficientBalance(msg.sender, amount)
        returns (bool)
    {
        require(to != address(0), "Transfer to zero address");

        balances[msg.sender] -= amount;
        balances[to] += amount;

        emit Transfer(msg.sender, to, amount);
        return true;
    }

    // ✅ Junior ต้องรู้: View functions
    function balanceOf(address account) external view returns (uint256) {
        return balances[account];
    }
}
```

### Junior Portfolio Projects

```
Portfolio สำหรับ Junior Developer:

1. ERC20 Token with Vesting (สมบูรณ์)
   - Token with time-based vesting schedule
   - Cliff period support
   - Multiple beneficiaries

2. Simple NFT Collection
   - ERC721 with IPFS metadata
   - Reveal mechanism
   - Royalty support (ERC2981)

3. Multi-Signature Wallet
   - Require M-of-N signatures
   - Transaction queue
   - Cancel mechanism

4. Basic AMM (Automated Market Maker)
   - Constant product formula (x*y=k)
   - Add/remove liquidity
   - Swap tokens

Skills ที่ต้องโชว์:
- Clean, readable code
- Comprehensive tests (>90% coverage)
- NatSpec documentation
- Gas-conscious implementation
```

## Mid-Level Developer (2-4 ปี)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/utils/Pausable.sol";

/**
 * @title MidLevelSkillsDemo
 * @notice ตัวอย่าง code ที่แสดงทักษะ Mid-Level Developer
 * @dev Yield vault with advanced patterns
 */
contract MidLevelSkillsDemo is ERC20, Ownable, ReentrancyGuard, Pausable {
    // ✅ Mid ต้องรู้: Inheritance, Design Patterns
    IERC20 public immutable asset;
    uint256 public totalAssets;

    // ✅ Mid ต้องรู้: Custom errors (gas efficient)
    error InsufficientShares(uint256 requested, uint256 available);
    error ZeroAmount();
    error MaxDepositExceeded(uint256 max, uint256 requested);

    // ✅ Mid ต้องรู้: Events with indexed params
    event Deposit(
        address indexed caller,
        address indexed owner,
        uint256 assets,
        uint256 shares
    );

    event Withdraw(
        address indexed caller,
        address indexed receiver,
        address indexed owner,
        uint256 assets,
        uint256 shares
    );

    // ✅ Mid ต้องรู้: Storage layout optimization
    uint128 public maxDeposit;
    uint64 public lastHarvestTimestamp;
    uint64 public harvestInterval;

    constructor(
        address _asset,
        address _owner
    ) ERC20("Yield Vault Share", "YVS") Ownable(_owner) {
        asset = IERC20(_asset);
        maxDeposit = type(uint128).max;
        harvestInterval = 1 days;
    }

    // ✅ Mid ต้องรู้: ERC4626 pattern
    function deposit(uint256 assets, address receiver)
        external
        nonReentrant
        whenNotPaused
        returns (uint256 shares)
    {
        if (assets == 0) revert ZeroAmount();
        if (assets > maxDeposit) revert MaxDepositExceeded(maxDeposit, assets);

        shares = _convertToShares(assets);
        if (shares == 0) revert ZeroAmount();

        totalAssets += assets;
        _mint(receiver, shares);

        asset.transferFrom(msg.sender, address(this), assets);

        emit Deposit(msg.sender, receiver, assets, shares);
    }

    function withdraw(uint256 assets, address receiver, address owner)
        external
        nonReentrant
        returns (uint256 shares)
    {
        shares = _convertToShares(assets);
        if (balanceOf(owner) < shares) {
            revert InsufficientShares(shares, balanceOf(owner));
        }

        if (msg.sender != owner) {
            uint256 allowed = allowance(owner, msg.sender);
            if (allowed != type(uint256).max) {
                _approve(owner, msg.sender, allowed - shares);
            }
        }

        totalAssets -= assets;
        _burn(owner, shares);
        asset.transfer(receiver, assets);

        emit Withdraw(msg.sender, receiver, owner, assets, shares);
    }

    // ✅ Mid ต้องรู้: Fixed-point math
    function _convertToShares(uint256 assets) internal view returns (uint256) {
        uint256 supply = totalSupply();
        if (supply == 0 || totalAssets == 0) return assets;
        return (assets * supply) / totalAssets;
    }

    function _convertToAssets(uint256 shares) internal view returns (uint256) {
        uint256 supply = totalSupply();
        if (supply == 0) return shares;
        return (shares * totalAssets) / supply;
    }

    // ✅ Mid ต้องรู้: Emergency functions
    function pause() external onlyOwner { _pause(); }
    function unpause() external onlyOwner { _unpause(); }
}
```

### Mid-Level Portfolio Projects

```
Portfolio สำหรับ Mid-Level Developer:

1. Full DeFi Protocol (Lending หรือ DEX)
   - Borrow/Supply with interest rate model
   - Liquidation mechanism
   - Collateral management
   - Comprehensive Foundry test suite

2. Governance System
   - ERC20 governance token
   - On-chain voting (Governor Bravo style)
   - Timelock controller
   - Delegation support

3. Cross-chain Bridge (Basic)
   - Lock & Mint pattern
   - Message passing
   - Security considerations documented

4. Security Audit Report
   - Audit ของ open-source project
   - หรือ CTF writeup (Ethernaut/DamnVulnerableDeFi)

Skills ที่ต้องโชว์:
- Architecture decisions documented
- Gas optimization techniques applied
- Security vulnerabilities addressed
- Integration tests with forked mainnet
```

## Senior Developer (4-7 ปี)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title SeniorSkillsDemo
 * @notice ตัวอย่าง code ที่แสดงทักษะ Senior Developer
 * @dev Advanced AMM with concentrated liquidity concepts
 */
contract SeniorSkillsDemo {
    // ✅ Senior ต้องรู้: Assembly optimization
    function efficientSqrt(uint256 x) internal pure returns (uint256 y) {
        if (x == 0) return 0;
        assembly {
            // Babylonian method in assembly
            let z := add(div(x, 2), 1)
            y := x
            for {} lt(z, y) {} {
                y := z
                z := div(add(div(x, z), z), 2)
            }
        }
    }

    // ✅ Senior ต้องรู้: Bitwise operations for efficiency
    struct PackedTick {
        // pack multiple values into single slot
        uint128 liquidityGross; // 128 bits
        int128 liquidityNet;    // 128 bits
    }

    // ✅ Senior ต้องรู้: Tick bitmap (Uniswap V3 style)
    mapping(int16 => uint256) public tickBitmap;

    function flipTick(int24 tick, int24 tickSpacing) internal {
        require(tick % tickSpacing == 0, "Tick not aligned");
        (int16 wordPos, uint8 bitPos) = position(tick / tickSpacing);
        uint256 mask = 1 << bitPos;
        tickBitmap[wordPos] ^= mask;
    }

    function position(int24 tick) private pure returns (int16 wordPos, uint8 bitPos) {
        wordPos = int16(tick >> 8);
        bitPos = uint8(uint24(tick % 256));
    }

    // ✅ Senior ต้องรู้: Q64.96 fixed point math
    uint160 constant MIN_SQRT_RATIO = 4295128739;
    uint160 constant MAX_SQRT_RATIO = 1461446703485210103287273052203988822378723970342;

    /**
     * @notice คำนวณ amount0 จาก liquidity และ price range
     * @dev ใช้ Q64.96 format สำหรับ sqrtPrice
     */
    function getAmount0Delta(
        uint160 sqrtRatioA,
        uint160 sqrtRatioB,
        uint128 liquidity,
        bool roundUp
    ) internal pure returns (uint256 amount0) {
        if (sqrtRatioA > sqrtRatioB) (sqrtRatioA, sqrtRatioB) = (sqrtRatioB, sqrtRatioA);

        uint256 numerator1 = uint256(liquidity) << 96;
        uint256 numerator2 = sqrtRatioB - sqrtRatioA;

        require(sqrtRatioA > 0);

        amount0 = roundUp
            ? _divRoundingUp(
                _mulDivRoundingUp(numerator1, numerator2, sqrtRatioB),
                sqrtRatioA
            )
            : (numerator1 * numerator2) / sqrtRatioB / sqrtRatioA;
    }

    function _mulDivRoundingUp(
        uint256 a,
        uint256 b,
        uint256 denominator
    ) internal pure returns (uint256 result) {
        result = (a * b) / denominator;
        if ((a * b) % denominator > 0) result++;
    }

    function _divRoundingUp(uint256 a, uint256 b) internal pure returns (uint256) {
        return (a + b - 1) / b;
    }

    // ✅ Senior ต้องรู้: Formal verification preparation
    // ทุก function ต้องมี invariants documented
    /**
     * @notice Invariant: totalLiquidity >= 0 เสมอ
     * @notice Invariant: sum(positions) == totalLiquidity
     * @notice Invariant: protocol fees <= total fees collected
     */
    mapping(bytes32 => uint256) public positions;
    uint256 public totalLiquidity;

    function addLiquidity(bytes32 positionKey, uint256 amount) external {
        positions[positionKey] += amount;
        totalLiquidity += amount;
        // Invariant preserved: totalLiquidity increases by same amount as position
    }
}
```

## Lead Engineer (7-10 ปี)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title LeadEngineerSkillsDemo
 * @notice ตัวอย่าง Architecture Decision Record (ADR)
 * @dev Lead ต้องสร้างและ document architectural decisions
 *
 * ADR-001: Vault Architecture - ERC4626 vs Custom Implementation
 *
 * Status: Accepted
 * Date: 2024-01-15
 * Context: เราต้องการ vault ที่ compatible กับ ecosystem tools
 *
 * Decision: ใช้ ERC4626 เป็น base
 *
 * Consequences:
 * (+) Compatible กับ DeFi aggregators (Yearn, Beefy)
 * (+) Well-audited standard
 * (-) อาจมี overhead จาก standard interface
 * (-) ต้องระวัง inflation attack บน empty vault
 *
 * Mitigation: ใช้ virtual shares offset เพื่อป้องกัน inflation attack
 */
abstract contract ERC4626WithInflationProtection {
    // ✅ Lead ต้องรู้: Inflation attack prevention
    // ใส่ virtual offset เพื่อทำให้ first deposit ราคาสูงขึ้น
    uint256 private constant VIRTUAL_SHARES = 1e3;
    uint256 private constant VIRTUAL_ASSETS = 1;

    uint256 internal _totalAssets;

    function _convertToShares(uint256 assets, bool roundUp) internal view returns (uint256) {
        uint256 supply = _totalShares() + VIRTUAL_SHARES;
        uint256 totalAssets_ = _totalAssets + VIRTUAL_ASSETS;

        if (roundUp) {
            return (assets * supply + totalAssets_ - 1) / totalAssets_;
        }
        return (assets * supply) / totalAssets_;
    }

    function _totalShares() internal view virtual returns (uint256);

    // ✅ Lead ต้องรู้: Protocol integration testing
    /**
     * @notice Integration test framework สำหรับ fork testing
     * @dev Lead ต้องวาง architecture ของ test suite
     */
}

/**
 * @title CodeReviewGuidelines
 * @dev Lead ต้อง establish code review standards
 *
 * Code Review Checklist:
 *
 * Security:
 * □ Reentrancy protection on all state-changing + external calls
 * □ Integer overflow/underflow (Solidity 0.8+ protects, but verify)
 * □ Access control on sensitive functions
 * □ Oracle manipulation resistance
 * □ Flash loan attack vectors considered
 *
 * Design:
 * □ Follows existing patterns consistently
 * □ Gas optimization reasonable
 * □ Error messages informative
 * □ Events emitted for all state changes
 *
 * Testing:
 * □ Happy path covered
 * □ Edge cases tested
 * □ Revert conditions tested
 * □ Fuzz tests for mathematical functions
 *
 * Documentation:
 * □ NatSpec on all public/external functions
 * □ Complex logic explained with inline comments
 * □ State variables documented
 */
contract CodeReviewExample {
    // BAD: ไม่มี documentation, ไม่ชัดเจน
    // function f(address a, uint256 x) external { ... }

    /**
     * @notice ถอนเงิน assets ออกจาก vault
     * @dev ตรวจสอบ shares และ transfer assets ให้ receiver
     * @param assets จำนวน assets ที่ต้องการถอน (denominated in underlying token)
     * @param receiver ที่อยู่ที่จะรับ assets
     * @return shares จำนวน vault shares ที่ถูก burn
     */
    function withdraw(
        uint256 assets,
        address receiver
    ) external returns (uint256 shares) {
        // GOOD: ชัดเจน, documented, returns value
        require(assets > 0, "Cannot withdraw zero assets");
        require(receiver != address(0), "Invalid receiver address");

        shares = 0; // placeholder
        emit Withdrawn(msg.sender, receiver, assets, shares);
    }

    event Withdrawn(
        address indexed caller,
        address indexed receiver,
        uint256 assets,
        uint256 shares
    );
}
```

## Getting Your First Audit

### Learning Path สู่ Smart Contract Auditor

```
Week 1-4: Foundation
├── อ่าน SWC Registry (ช่องโหว่ทุกตัว)
├── ทำ Ethernaut (18 levels)
├── ทำ Damn Vulnerable DeFi (18 challenges)
└── อ่าน audit reports จาก Trail of Bits, Sigma Prime

Week 5-8: Practice
├── Code4rena: เริ่มด้วย low-risk findings
├── Sherlock: อ่าน previous contest reports
├── ทำ CTFs: Paradigm CTF, Secureum
└── เขียน writeups สำหรับทุก finding

Month 3-6: Build Reputation
├── First contest findings (แม้แต่ QA/low)
├── สร้าง portfolio ของ findings
├── เขียน blog posts เกี่ยวกับ vulnerabilities
└── Network ใน Security community

Month 6-12: First Paid Engagement
├── Private audit กับ protocol ขนาดเล็ก
├── Bounty hunt บน Immunefi
└── Apply กับ audit firms

```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title AuditPracticeContract
 * @notice Contract สำหรับฝึก audit skill
 * @dev สัญญานี้มี bugs จงหาให้เจอ! (Workshop exercise)
 *
 * HINT: มีช่องโหว่อย่างน้อย 5 ตัวใน contract นี้
 * ลองหาดูก่อนอ่านเฉลย
 */
contract AuditPractice {
    mapping(address => uint256) public balances;
    address public owner;
    bool public locked;

    // BUG 1: tx.origin ไม่ควรใช้ใน authorization
    modifier onlyOwner() {
        require(tx.origin == owner, "Not owner");
        _;
    }

    // BUG 2: ไม่มี reentrancy protection
    function withdraw(uint256 amount) external {
        require(balances[msg.sender] >= amount, "Insufficient");
        // BUG 3: state update หลัง external call (CEI violation)
        (bool success,) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
        balances[msg.sender] -= amount; // ควรมาก่อน external call
    }

    // BUG 4: Integer underflow ไม่ถูก handle (ใน old Solidity จะ wrap around)
    // ใน 0.8+ จะ revert แต่ logic อาจยังผิด
    function calculateReward(uint256 amount, uint256 duration) external pure returns (uint256) {
        // BUG: ไม่ check overflow สำหรับ multiplication
        return amount * duration * 1000; // อาจ overflow ได้
    }

    // BUG 5: Unchecked return value
    function transferTokens(address token, address to, uint256 amount) external onlyOwner {
        (bool success,) = token.call(
            abi.encodeWithSignature("transfer(address,uint256)", to, amount)
        );
        // ไม่ check success!
    }
}

/*
 * ANSWERS (อย่าอ่านก่อนลองทำ!):
 *
 * 1. tx.origin Phishing: Owner ถูก trick ให้ call contract malicious
 *    → ใช้ msg.sender แทน tx.origin
 *
 * 2. Reentrancy: attacker re-enter ก่อน state update
 *    → เพิ่ม nonReentrant modifier หรือ update state ก่อน
 *
 * 3. CEI Violation: Check-Effects-Interactions violated
 *    → ย้าย balances[msg.sender] -= amount; มาก่อน call
 *
 * 4. Possible Integer Overflow: สำหรับ large inputs
 *    → ใช้ SafeMath หรือ check overflow explicitly
 *
 * 5. Unchecked Return: USDT return false instead of reverting
 *    → ใช้ SafeERC20 library หรือ check return value
 */
```

### Platforms สำหรับฝึก Auditing

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title AuditPlatformComparison
 * @dev เปรียบเทียบ platforms สำหรับ aspiring auditors
 *
 * ╔══════════════════╦═══════════════╦═══════════════╦═══════════════╗
 * ║ Platform         ║ Type          ║ Reward Model  ║ Entry Level   ║
 * ╠══════════════════╬═══════════════╬═══════════════╬═══════════════╣
 * ║ Code4rena        ║ Contest       ║ Competition   ║ Open          ║
 * ║ Sherlock         ║ Contest       ║ Competition   ║ Open          ║
 * ║ Immunefi         ║ Bug Bounty    ║ Fixed reward  ║ Open          ║
 * ║ Cantina          ║ Contest       ║ Competition   ║ Open          ║
 * ║ HackenProof      ║ Bug Bounty    ║ Fixed reward  ║ Open          ║
 * ╚══════════════════╩═══════════════╩═══════════════╩═══════════════╝
 *
 * Strategy สำหรับ Beginner:
 * 1. เริ่มที่ Code4rena - เยอะ contests, community ดี
 * 2. อ่าน reports จาก past contests ก่อน
 * 3. Target smaller protocols (prize pool < $100k) ก่อน
 * 4. Focus บน specific vulnerability types ที่เชี่ยวชาญ
 *
 * วิธีอ่าน Audit Report:
 * - Executive Summary: ภาพรวม findings
 * - Individual Findings: severity, description, recommendation
 * - Appendix: scope, methodology
 *
 * Useful Resources:
 * - https://solodit.xyz (searchable audit findings database)
 * - https://github.com/pcaversaccio/reentrancy-attacks
 * - https://www.immunebytes.com/blog/top-10-ethereum-smart-contract-vulnerabilities/
 */
contract AuditLearningPath {
    struct Resource {
        string name;
        string url;
        string category;
        uint8 difficulty; // 1-5
    }

    Resource[] public resources;

    function initializeResources() external {
        resources.push(Resource("Ethernaut", "ethernaut.openzeppelin.com", "CTF", 1));
        resources.push(Resource("Damn Vulnerable DeFi", "damnvulnerabledefi.xyz", "CTF", 2));
        resources.push(Resource("Paradigm CTF", "ctf.paradigm.xyz", "CTF", 4));
        resources.push(Resource("SWC Registry", "swcregistry.io", "Reference", 2));
        resources.push(Resource("Secureum", "secureum.xyz", "Course", 2));
        resources.push(Resource("Code4rena", "code4rena.com", "Contest", 2));
        resources.push(Resource("Sherlock", "sherlock.xyz", "Contest", 3));
        resources.push(Resource("Immunefi", "immunefi.com", "Bounty", 3));
    }
}
```

## Building in Public

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title BuildingInPublic
 * @notice Strategy สำหรับสร้าง reputation ใน Solidity community
 *
 * 1. Open Source Contributions
 * ─────────────────────────────
 * Target repositories:
 * - OpenZeppelin Contracts (ง่ายสำหรับ first PR: documentation, tests)
 * - Foundry (Rust based, แต่ tests/docs accessible)
 * - Uniswap V4 hooks ecosystem
 * - Chainlink (adapters, tests)
 *
 * Process:
 * 1. หา "good first issue" label
 * 2. อ่าน CONTRIBUTING.md
 * 3. Join Discord ก่อน เพื่อ discuss approach
 * 4. เขียน clean PR กับ tests
 *
 * 2. Technical Writing
 * ─────────────────────────────
 * Platforms:
 * - Mirror.xyz (Web3 blogging)
 * - Substack (email newsletter)
 * - Medium / Dev.to
 * - Personal blog (GitHub Pages)
 *
 * Content ideas:
 * - "Dissecting [Protocol Name]" series
 * - Vulnerability deep-dives
 * - EIP explanations (ให้คนทั่วไปเข้าใจ)
 * - Gas optimization tricks
 * - Code review walkthrough
 *
 * 3. Conference Speaking
 * ─────────────────────────────
 * Entry level:
 * - Local Ethereum meetups (EthBangkok, EthSingapore)
 * - ETHGlobal side events
 * - Online YouTube sessions
 *
 * Mid level:
 * - ETHDenver, ETHParis workshops
 * - Devcon lightning talks
 *
 * Senior level:
 * - Devcon main stage
 * - ETHCC keynote
 * - Academic conferences (Financial Cryptography)
 *
 * 4. Community Presence
 * ─────────────────────────────
 * - Twitter/X: share findings, code snippets, EIP breakdowns
 * - Ethereum Research Forum: deep technical posts
 * - Ethereum Magicians: protocol improvement discussion
 * - Reddit r/ethdev: help beginners
 */
contract CareerProgressTracker {
    struct Achievement {
        string category;
        string description;
        uint256 points;
        bool achieved;
        uint256 achievedAt;
    }

    mapping(address => Achievement[]) public achievements;

    function addAchievement(
        address developer,
        string calldata category,
        string calldata description,
        uint256 points
    ) external {
        achievements[developer].push(Achievement({
            category: category,
            description: description,
            points: points,
            achieved: false,
            achievedAt: 0
        }));
    }

    function markAchieved(address developer, uint256 index) external {
        Achievement storage achievement = achievements[developer][index];
        require(!achievement.achieved, "Already achieved");
        achievement.achieved = true;
        achievement.achievedAt = block.timestamp;
    }

    function getTotalPoints(address developer) external view returns (uint256 total) {
        Achievement[] memory devAchievements = achievements[developer];
        for (uint256 i = 0; i < devAchievements.length; i++) {
            if (devAchievements[i].achieved) {
                total += devAchievements[i].points;
            }
        }
    }
}
```

## Workshop: สร้าง Career Development Plan

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title CareerDevelopmentPlan
 * @notice Workshop: สร้าง 12-month development plan
 *
 * แบบฝึกหัด: เติม plan ของคุณเองใน comments
 */
contract CareerDevelopmentPlan {
    /*
     * MY CAREER DEVELOPMENT PLAN
     * ════════════════════════════════════════════
     *
     * Current Level: [Junior/Mid/Senior]
     * Target Level: [Mid/Senior/Lead] in 12 months
     *
     * Month 1-3: Foundation Strengthening
     * ├── Complete: [...]
     * ├── Learn: [...]
     * └── Build: [...]
     *
     * Month 4-6: Specialization
     * ├── Focus area: [Security/DeFi/Infrastructure]
     * ├── Complete CTFs: [Ethernaut, DamnVulnerableDeFi]
     * └── First audit: [Code4rena contest]
     *
     * Month 7-9: Portfolio Building
     * ├── Project 1: [...]
     * ├── Project 2: [...]
     * └── Open source: [contribute to ...]
     *
     * Month 10-12: Visibility
     * ├── Blog posts: [3 posts minimum]
     * ├── Conference: [speak at / attend ...]
     * └── Job: [apply to / freelance for ...]
     *
     * Metrics to track:
     * - GitHub contributions per week
     * - Audit findings count
     * - Blog readers
     * - Network connections
     * ════════════════════════════════════════════
     */

    struct MonthlyGoal {
        uint256 month;
        string[] objectives;
        bool[] completed;
    }

    mapping(address => MonthlyGoal[]) public plans;

    function setPlan(address developer, MonthlyGoal[] calldata monthlyGoals) external {
        delete plans[developer];
        for (uint256 i = 0; i < monthlyGoals.length; i++) {
            plans[developer].push(monthlyGoals[i]);
        }
    }

    function completeObjective(
        address developer,
        uint256 month,
        uint256 objectiveIndex
    ) external {
        MonthlyGoal storage goal = plans[developer][month];
        require(objectiveIndex < goal.completed.length, "Invalid index");
        goal.completed[objectiveIndex] = true;
    }
}
```

## Principal Engineer Responsibilities

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title PrincipalEngineerRole
 * @notice ความรับผิดชอบของ Principal Engineer
 *
 * Technical Leadership:
 * ─────────────────────
 * 1. Define technical direction ของ ecosystem
 * 2. Write EIPs/ERCs (เช่น ERC-4626, ERC-6551)
 * 3. Review and influence protocol standards
 * 4. Mentor senior developers
 * 5. Represent company ใน Ethereum governance
 *
 * Research & Innovation:
 * ─────────────────────
 * 1. Publish research papers
 * 2. สร้าง novel DeFi mechanisms
 * 3. อ่านและ apply academic research
 * 4. ทำ formal verification
 *
 * Community:
 * ─────────────────────
 * 1. Ethereum Research Forum contributions
 * 2. Core developer calls
 * 3. ACD (All Core Devs) participation
 * 4. Ethereum Foundation grants
 *
 * Notable Principal Engineers ที่ควรติดตาม:
 * - Hayden Adams (Uniswap)
 * - Robert Leshner (Compound)
 * - Stani Kulechov (Aave)
 * - Vitalik Buterin (Ethereum)
 * - Dan Robinson (Paradigm)
 * - Georgios Konstantopoulos (Paradigm/Reth)
 */
contract PrincipalEngineerDemo {
    // Principal Engineers สร้าง standards ที่คนอื่น implement

    // ตัวอย่าง: ERC-4626 (Tokenized Vault Standard) - สร้างโดย Joey Santoro, t11s
    // ทุก DeFi vault ตอนนี้ implement มาตรฐานนี้

    /**
     * @notice Interface ที่ Principal Engineer ออกแบบ
     * @dev มาตรฐานนี้ถูกใช้โดย 100+ protocols
     */
    interface IErc4626Like {
        function totalAssets() external view returns (uint256);
        function convertToShares(uint256 assets) external view returns (uint256);
        function convertToAssets(uint256 shares) external view returns (uint256);
        function maxDeposit(address receiver) external view returns (uint256);
        function previewDeposit(uint256 assets) external view returns (uint256);
        function deposit(uint256 assets, address receiver) external returns (uint256 shares);
        function maxWithdraw(address owner) external view returns (uint256);
        function previewWithdraw(uint256 assets) external view returns (uint256);
        function withdraw(uint256 assets, address receiver, address owner) external returns (uint256 shares);
    }
}
```

## สรุป Part 97

- **Career Levels**: Junior → Mid → Senior → Lead → Principal มีทักษะที่ชัดเจนในแต่ละระดับ
- **Skills Matrix**: วัดตัวเองด้วย Solidity, Testing, Security, Architecture, Business skills
- **Portfolio**: แต่ละ level ต้องมี projects ที่โชว์ความสามารถที่เหมาะสม
- **Auditing Path**: Ethernaut → DamnVulnerableDeFi → Code4rena → Private audits
- **Building in Public**: Open source, blogging, speaking เป็นวิธีสร้าง reputation ที่ดีที่สุด
- **Principal Engineer**: ผู้นำระดับ ecosystem ที่สร้าง standards และ influence protocol design

## Next: Part 98 - Solidity Ecosystem Tooling Reference

ในบทถัดไปเราจะทำ comprehensive review ของเครื่องมือทั้งหมดใน Solidity ecosystem ตั้งแต่ Hardhat vs Foundry, testing frameworks, security tools ไปจนถึง deployment และ monitoring tools
