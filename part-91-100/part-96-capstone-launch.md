# Part 96: Capstone - Protocol Launch & Go-Live

## บทนำ

การเปิดตัว DeFi Protocol สู่สาธารณะเป็นช่วงเวลาที่ตื่นเต้นและน่ากลัวในเวลาเดียวกัน Protocol ที่ดีที่สุดในโลกอาจล้มเหลวได้หากขาดการวางแผนการเปิดตัวที่ดี ในบทนี้เราจะเรียนรู้ทุกขั้นตอนตั้งแต่การ deploy บน testnet จนถึงการสร้างชุมชนและการตอบสนองต่อเหตุการณ์ฉุกเฉิน

## Launch Checklist ฉบับสมบูรณ์

### Phase 1: Pre-Testnet (สัปดาห์ 1-2)

```
✅ Pre-Launch Checklist
├── Code Freeze
│   ├── All features implemented
│   ├── No WIP branches merged
│   └── Version tagged (v1.0.0-rc1)
├── Internal Security Review
│   ├── Manual code review by 2+ engineers
│   ├── Automated tool scans (Slither, Mythril)
│   └── Economic model verification
├── Documentation
│   ├── Technical specification complete
│   ├── User documentation drafted
│   └── Emergency procedure documented
└── Testing
    ├── Unit test coverage > 95%
    ├── Integration tests passing
    ├── Fork tests on mainnet state
    └── Gas optimization pass
```

### Phase 2: Testnet Deployment

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title DeploymentVerifier
 * @notice สัญญาสำหรับตรวจสอบการ deploy บน testnet
 * @dev ใช้เพื่อยืนยันว่า deployment ทำงานถูกต้องก่อน mainnet
 */
contract DeploymentVerifier {
    struct DeploymentRecord {
        address deployer;
        uint256 timestamp;
        bytes32 codeHash;
        string network;
        bool verified;
    }

    mapping(address => DeploymentRecord) public deployments;
    address public owner;

    event ContractDeployed(
        address indexed contractAddress,
        string network,
        bytes32 codeHash
    );

    event DeploymentVerified(
        address indexed contractAddress,
        address indexed verifier
    );

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    function recordDeployment(
        address contractAddress,
        string calldata network
    ) external onlyOwner {
        bytes32 codeHash = contractAddress.codehash;
        require(codeHash != bytes32(0), "Contract not deployed");

        deployments[contractAddress] = DeploymentRecord({
            deployer: msg.sender,
            timestamp: block.timestamp,
            codeHash: codeHash,
            network: network,
            verified: false
        });

        emit ContractDeployed(contractAddress, network, codeHash);
    }

    function verifyDeployment(address contractAddress) external onlyOwner {
        DeploymentRecord storage record = deployments[contractAddress];
        require(record.deployer != address(0), "Deployment not recorded");
        require(!record.verified, "Already verified");

        // ตรวจสอบว่า code hash ยังตรงกัน (ไม่มีการเปลี่ยนแปลง)
        require(
            contractAddress.codehash == record.codeHash,
            "Code hash mismatch"
        );

        record.verified = true;
        emit DeploymentVerified(contractAddress, msg.sender);
    }

    function isVerified(address contractAddress) external view returns (bool) {
        return deployments[contractAddress].verified;
    }
}
```

### Testnet Deployment Script

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";
import "../src/OmniYieldVault.sol";
import "../src/OmniYieldGovernance.sol";
import "../src/OmniYieldToken.sol";

/**
 * @title DeployTestnet
 * @notice Script สำหรับ deploy OmniYield บน testnet
 */
contract DeployTestnet is Script {
    // Testnet addresses
    address constant TESTNET_USDC = 0x94a9D9AC8a22534E3FaCa9F4e7F2E2cf85d5E4C8;
    address constant TESTNET_WETH = 0xfFf9976782d46CC05630D1f6eBAb18b2324d6B14;

    struct DeployedContracts {
        address token;
        address vault;
        address governance;
        address timelock;
    }

    function run() external returns (DeployedContracts memory deployed) {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        address deployer = vm.addr(deployerPrivateKey);

        console.log("=== OmniYield Testnet Deployment ===");
        console.log("Deployer:", deployer);
        console.log("Network: Sepolia Testnet");
        console.log("Block:", block.number);

        vm.startBroadcast(deployerPrivateKey);

        // 1. Deploy Token
        console.log("\n[1/4] Deploying OmniYield Token...");
        OmniYieldToken token = new OmniYieldToken(
            "OmniYield Token",
            "OYT",
            deployer
        );
        deployed.token = address(token);
        console.log("Token deployed at:", deployed.token);

        // 2. Deploy Timelock
        console.log("\n[2/4] Deploying Timelock...");
        address[] memory proposers = new address[](1);
        address[] memory executors = new address[](1);
        proposers[0] = deployer;
        executors[0] = address(0); // anyone can execute

        // TimelockController ในรูปแบบ simplified
        OmniYieldTimelock timelock = new OmniYieldTimelock(
            2 days,
            proposers,
            executors,
            deployer
        );
        deployed.timelock = address(timelock);
        console.log("Timelock deployed at:", deployed.timelock);

        // 3. Deploy Vault
        console.log("\n[3/4] Deploying OmniYield Vault...");
        OmniYieldVault vault = new OmniYieldVault(
            TESTNET_USDC,
            deployed.token,
            deployed.timelock
        );
        deployed.vault = address(vault);
        console.log("Vault deployed at:", deployed.vault);

        // 4. Deploy Governance
        console.log("\n[4/4] Deploying Governance...");
        OmniYieldGovernance governance = new OmniYieldGovernance(
            token,
            timelock,
            "OmniYield Governor"
        );
        deployed.governance = address(governance);
        console.log("Governance deployed at:", deployed.governance);

        // Post-deployment setup
        _setupPermissions(token, vault, governance, timelock, deployer);

        vm.stopBroadcast();

        _printDeploymentSummary(deployed);
        _saveDeploymentAddresses(deployed);

        return deployed;
    }

    function _setupPermissions(
        OmniYieldToken token,
        OmniYieldVault vault,
        OmniYieldGovernance governance,
        OmniYieldTimelock timelock,
        address deployer
    ) internal {
        console.log("\n=== Setting up permissions ===");

        // Grant vault minter role on token
        bytes32 MINTER_ROLE = token.MINTER_ROLE();
        token.grantRole(MINTER_ROLE, address(vault));
        console.log("Vault granted MINTER_ROLE");

        // Grant governance proposer role on timelock
        bytes32 PROPOSER_ROLE = timelock.PROPOSER_ROLE();
        timelock.grantRole(PROPOSER_ROLE, address(governance));
        console.log("Governance granted PROPOSER_ROLE");

        // Renounce admin role (ให้ timelock เป็น admin)
        bytes32 TIMELOCK_ADMIN_ROLE = timelock.TIMELOCK_ADMIN_ROLE();
        timelock.grantRole(TIMELOCK_ADMIN_ROLE, address(timelock));
        timelock.revokeRole(TIMELOCK_ADMIN_ROLE, deployer);
        console.log("Admin role transferred to timelock");
    }

    function _printDeploymentSummary(DeployedContracts memory deployed) internal view {
        console.log("\n==========================================");
        console.log("      DEPLOYMENT SUMMARY - TESTNET");
        console.log("==========================================");
        console.log("Token:      ", deployed.token);
        console.log("Vault:      ", deployed.vault);
        console.log("Governance: ", deployed.governance);
        console.log("Timelock:   ", deployed.timelock);
        console.log("==========================================");
        console.log("\nNext steps:");
        console.log("1. Verify contracts on Etherscan");
        console.log("2. Run testnet integration tests");
        console.log("3. Submit for audit");
    }

    function _saveDeploymentAddresses(DeployedContracts memory deployed) internal {
        string memory json = string.concat(
            '{"network":"sepolia","token":"',
            vm.toString(deployed.token),
            '","vault":"',
            vm.toString(deployed.vault),
            '","governance":"',
            vm.toString(deployed.governance),
            '","timelock":"',
            vm.toString(deployed.timelock),
            '","deployedAt":',
            vm.toString(block.timestamp),
            "}"
        );

        vm.writeFile("./deployments/sepolia-latest.json", json);
        console.log("\nDeployment addresses saved to ./deployments/sepolia-latest.json");
    }
}
```

## Audit Process

### Pre-Audit Preparation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title AuditPreparation
 * @notice รายการสิ่งที่ต้องเตรียมก่อนส่ง audit
 * @dev checklist สำหรับทีม security
 */

// =============================================================
// AUDIT SCOPE DEFINITION
// =============================================================
//
// In-Scope Contracts:
// ├── src/OmniYieldVault.sol        - Core vault logic
// ├── src/OmniYieldToken.sol        - ERC20 governance token
// ├── src/OmniYieldGovernance.sol   - Governor contract
// ├── src/OmniYieldTimelock.sol     - Timelock controller
// ├── src/strategies/               - Yield strategies (3 files)
// └── src/libraries/               - Shared libraries (2 files)
//
// Out-of-Scope:
// ├── test/                         - Test files
// ├── script/                       - Deploy scripts
// └── External protocols (Aave, Compound) - Assume secure
//
// Total lines of code: ~2,500 SLOC
// Estimated audit duration: 4-6 weeks
// =============================================================

contract AuditReadinessChecker {
    // Known issues (documented เพื่อ auditor รับทราบ)
    struct KnownIssue {
        string description;
        string severity; // "informational", "low", "medium", "high"
        string mitigation;
        bool accepted; // false = กำลังแก้, true = ยอมรับ risk
    }

    KnownIssue[] public knownIssues;

    constructor() {
        // รายการปัญหาที่ทราบแล้ว ต้องแจ้งให้ auditor รู้
        knownIssues.push(KnownIssue({
            description: "Oracle price feed can be up to 1 hour stale in extreme conditions",
            severity: "low",
            mitigation: "Emergency pause mechanism available, monitoring in place",
            accepted: true
        }));

        knownIssues.push(KnownIssue({
            description: "Governance can change fee parameters with 2-day timelock",
            severity: "informational",
            mitigation: "By design, community controls fees through governance",
            accepted: true
        }));
    }

    function getKnownIssuesCount() external view returns (uint256) {
        return knownIssues.length;
    }

    function getKnownIssue(uint256 index) external view returns (KnownIssue memory) {
        require(index < knownIssues.length, "Invalid index");
        return knownIssues[index];
    }
}
```

### Audit Communication Template

```
## Security Audit Request - OmniYield Protocol v1.0

**Contact:** security@omniyield.io
**Codebase:** https://github.com/omniyield/contracts (tag: v1.0.0-audit)
**Timeline:** 4-6 weeks preferred, hard deadline: [DATE]
**Budget:** $150,000 - $200,000

### Protocol Overview
OmniYield is a multi-strategy yield aggregator that automatically
allocates user deposits across Aave, Compound, and Yearn to maximize
APY while maintaining security.

### Key Security Concerns
1. Reentrancy in vault deposit/withdraw flow
2. Oracle manipulation resistance
3. Governance attack vectors (flash loan voting)
4. Strategy migration safety
5. Fee calculation edge cases

### What We've Done
- Internal review by 2 senior engineers
- Slither: 0 high/critical findings
- Mythril: 0 high/critical findings
- 100% line coverage, 95%+ branch coverage
- Economic model formal verification

### What We Need
- Full security audit
- Economic/game theory review
- Deployment review

Please provide:
- Team composition
- Similar protocol experience
- Preliminary timeline
- Fixed price quote
```

## Mainnet Deployment

### Final Deployment Checklist

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";

/**
 * @title MainnetDeployment
 * @notice Script สำหรับ deploy production บน mainnet
 * @dev ต้องมี multisig สำหรับทุก transaction
 */
contract MainnetDeployment is Script {
    // ⚠️ MAINNET ADDRESSES - ตรวจสอบทุกครั้งก่อน run
    address constant MAINNET_USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address constant MAINNET_WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address constant SAFE_MULTISIG = 0x...; // Gnosis Safe address

    // Timelock delay: 48 hours สำหรับ mainnet
    uint256 constant TIMELOCK_DELAY = 48 hours;

    function run() external {
        // ⚠️ ต้องใช้ hardware wallet สำหรับ mainnet
        // vm.startBroadcast() จะ prompt สำหรับ signature

        _preflightChecks();

        uint256 deployerKey = vm.envUint("MAINNET_PRIVATE_KEY");
        address deployer = vm.addr(deployerKey);

        // Verify deployer ถูกต้อง
        require(
            deployer == 0x..., // expected deployer address
            "Wrong deployer"
        );

        console.log("WARNING: MAINNET DEPLOYMENT");
        console.log("Deployer:", deployer);
        console.log("Block:", block.number);
        console.log("Timestamp:", block.timestamp);

        // Pause 10 seconds เพื่อให้มีเวลา abort ถ้าต้องการ
        // (ใน real script ให้ใช้ manual confirmation)

        vm.startBroadcast(deployerKey);
        _deploy();
        vm.stopBroadcast();
    }

    function _preflightChecks() internal view {
        // ตรวจสอบ network
        require(block.chainid == 1, "Must be mainnet");

        // ตรวจสอบ gas price ไม่แพงเกิน
        require(tx.gasprice <= 50 gwei, "Gas price too high");

        console.log("Preflight checks passed");
    }

    function _deploy() internal {
        // Actual deployment logic
        // ทุก contract ต้องถูก verify บน Etherscan ทันที
    }
}
```

### Post-Deployment Verification

```bash
#!/bin/bash
# verify-deployment.sh - รัน script นี้หลัง deploy ทุกครั้ง

echo "=== Post-Deployment Verification ==="

NETWORK="mainnet"
TOKEN_ADDRESS="0x..."
VAULT_ADDRESS="0x..."

# 1. Verify source code on Etherscan
forge verify-contract \
    --chain $NETWORK \
    --etherscan-api-key $ETHERSCAN_KEY \
    --constructor-args $(cast abi-encode "constructor(address,address)" $USDC $TOKEN_ADDRESS) \
    $VAULT_ADDRESS \
    src/OmniYieldVault.sol:OmniYieldVault

echo "✅ Source code verified"

# 2. Check ownership
OWNER=$(cast call $VAULT_ADDRESS "owner()" --rpc-url $RPC_URL)
echo "Owner: $OWNER"

# 3. Check initial state
PAUSED=$(cast call $VAULT_ADDRESS "paused()" --rpc-url $RPC_URL)
echo "Paused: $PAUSED"

TVL=$(cast call $VAULT_ADDRESS "totalAssets()" --rpc-url $RPC_URL)
echo "Initial TVL: $TVL"

# 4. Verify permissions
MINTER_ROLE=$(cast call $TOKEN_ADDRESS "MINTER_ROLE()" --rpc-url $RPC_URL)
HAS_MINTER=$(cast call $TOKEN_ADDRESS "hasRole(bytes32,address)" $MINTER_ROLE $VAULT_ADDRESS --rpc-url $RPC_URL)
echo "Vault has MINTER_ROLE: $HAS_MINTER"

echo "=== Verification Complete ==="
```

## Liquidity Bootstrap

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title LiquidityBootstrap
 * @notice สัญญาสำหรับ bootstrap initial liquidity ผ่าน LBP (Liquidity Bootstrap Pool)
 * @dev ใช้สำหรับการกระจาย token อย่างยุติธรรมในช่วงแรก
 */
contract LiquidityBootstrap is Ownable, ReentrancyGuard {
    IERC20 public immutable oyt; // OmniYield Token
    IERC20 public immutable usdc;

    uint256 public constant BOOTSTRAP_DURATION = 72 hours;
    uint256 public constant MIN_CONTRIBUTION = 100e6; // 100 USDC
    uint256 public constant MAX_CONTRIBUTION = 10_000e6; // 10,000 USDC per address
    uint256 public constant TOTAL_OYT_FOR_BOOTSTRAP = 10_000_000e18; // 10M OYT

    uint256 public startTime;
    uint256 public totalRaised;
    uint256 public participantCount;

    mapping(address => uint256) public contributions;
    mapping(address => bool) public claimed;

    bool public bootstrapStarted;
    bool public bootstrapFinalized;

    event BootstrapStarted(uint256 startTime, uint256 endTime);
    event Contributed(address indexed user, uint256 amount, uint256 totalRaised);
    event TokensClaimed(address indexed user, uint256 oytAmount);
    event BootstrapFinalized(uint256 totalRaised, uint256 participantCount);

    error BootstrapNotStarted();
    error BootstrapEnded();
    error BootstrapNotEnded();
    error BootstrapAlreadyFinalized();
    error ContributionTooSmall(uint256 minimum);
    error ContributionTooLarge(uint256 maximum, uint256 current);
    error AlreadyClaimed();
    error NothingToClaim();

    constructor(
        address _oyt,
        address _usdc,
        address _owner
    ) Ownable(_owner) {
        oyt = IERC20(_oyt);
        usdc = IERC20(_usdc);
    }

    /**
     * @notice เริ่ม bootstrap phase
     */
    function startBootstrap() external onlyOwner {
        require(!bootstrapStarted, "Already started");
        require(
            oyt.balanceOf(address(this)) >= TOTAL_OYT_FOR_BOOTSTRAP,
            "Insufficient OYT"
        );

        bootstrapStarted = true;
        startTime = block.timestamp;

        emit BootstrapStarted(startTime, startTime + BOOTSTRAP_DURATION);
    }

    /**
     * @notice ร่วม bootstrap โดยการ contribute USDC
     * @param amount จำนวน USDC ที่ต้องการ contribute
     */
    function contribute(uint256 amount) external nonReentrant {
        if (!bootstrapStarted) revert BootstrapNotStarted();
        if (block.timestamp > startTime + BOOTSTRAP_DURATION) revert BootstrapEnded();

        if (amount < MIN_CONTRIBUTION) revert ContributionTooSmall(MIN_CONTRIBUTION);

        uint256 newTotal = contributions[msg.sender] + amount;
        if (newTotal > MAX_CONTRIBUTION) {
            revert ContributionTooLarge(MAX_CONTRIBUTION, newTotal);
        }

        if (contributions[msg.sender] == 0) {
            participantCount++;
        }

        contributions[msg.sender] = newTotal;
        totalRaised += amount;

        usdc.transferFrom(msg.sender, address(this), amount);

        emit Contributed(msg.sender, amount, totalRaised);
    }

    /**
     * @notice สรุปผล bootstrap และเปิดให้ claim tokens
     */
    function finalizeBootstrap() external onlyOwner {
        if (!bootstrapStarted) revert BootstrapNotStarted();
        if (block.timestamp <= startTime + BOOTSTRAP_DURATION) revert BootstrapNotEnded();
        if (bootstrapFinalized) revert BootstrapAlreadyFinalized();

        bootstrapFinalized = true;

        emit BootstrapFinalized(totalRaised, participantCount);
    }

    /**
     * @notice Claim OYT tokens ตาม proportion ของ contribution
     */
    function claimTokens() external nonReentrant {
        if (!bootstrapFinalized) revert BootstrapNotEnded();
        if (claimed[msg.sender]) revert AlreadyClaimed();

        uint256 userContribution = contributions[msg.sender];
        if (userContribution == 0) revert NothingToClaim();

        // คำนวณ OYT ที่ได้รับตามสัดส่วน
        uint256 oytAmount = (userContribution * TOTAL_OYT_FOR_BOOTSTRAP) / totalRaised;

        claimed[msg.sender] = true;
        oyt.transfer(msg.sender, oytAmount);

        emit TokensClaimed(msg.sender, oytAmount);
    }

    /**
     * @notice คำนวณ OYT ที่จะได้รับถ้า claim ตอนนี้
     * @param user ที่อยู่ที่ต้องการตรวจสอบ
     * @return อัตราส่วน OYT ที่จะได้รับ
     */
    function estimatedAllocation(address user) external view returns (uint256) {
        if (totalRaised == 0) return 0;
        return (contributions[user] * TOTAL_OYT_FOR_BOOTSTRAP) / totalRaised;
    }

    /**
     * @notice ดึงเงิน USDC ที่ระดมทุนได้ (หลัง finalize)
     */
    function withdrawRaisedFunds(address recipient) external onlyOwner {
        require(bootstrapFinalized, "Not finalized");
        uint256 balance = usdc.balanceOf(address(this));
        usdc.transfer(recipient, balance);
    }

    /**
     * @notice ข้อมูล bootstrap สำหรับ UI
     */
    function getBootstrapInfo() external view returns (
        bool started,
        bool finalized,
        uint256 endTime,
        uint256 raised,
        uint256 participants,
        uint256 timeRemaining
    ) {
        started = bootstrapStarted;
        finalized = bootstrapFinalized;
        endTime = bootstrapStarted ? startTime + BOOTSTRAP_DURATION : 0;
        raised = totalRaised;
        participants = participantCount;

        if (bootstrapStarted && block.timestamp < endTime) {
            timeRemaining = endTime - block.timestamp;
        }
    }
}
```

## Bug Bounty Program

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

/**
 * @title BugBountyManager
 * @notice ระบบจัดการ Bug Bounty สำหรับ OmniYield Protocol
 * @dev จัดการการ submit, review, และ payout bug bounties
 */
contract BugBountyManager is Ownable {
    // Severity levels ตาม Immunefi standard
    enum Severity {
        Critical,   // $50,000 - $250,000
        High,       // $10,000 - $50,000
        Medium,     // $1,000 - $10,000
        Low,        // $100 - $1,000
        Informational // ไม่มี reward
    }

    struct BugReport {
        address reporter;
        bytes32 reportHash; // hash ของรายงาน (เพื่อ privacy)
        Severity severity;
        uint256 reward;
        bool paid;
        bool rejected;
        string rejectionReason;
        uint256 submittedAt;
        uint256 resolvedAt;
    }

    mapping(uint256 => BugReport) public reports;
    uint256 public reportCount;

    // Reward tiers (in USD equivalent, paid in USDC)
    mapping(Severity => uint256) public maxRewards;
    mapping(Severity => uint256) public minRewards;

    IERC20 public immutable usdc;
    uint256 public totalPaid;

    event ReportSubmitted(
        uint256 indexed reportId,
        address indexed reporter,
        bytes32 reportHash
    );

    event ReportResolved(
        uint256 indexed reportId,
        Severity severity,
        uint256 reward,
        bool accepted
    );

    event BountyPaid(
        uint256 indexed reportId,
        address indexed reporter,
        uint256 amount
    );

    error InsufficientBountyFunds(uint256 required, uint256 available);
    error ReportAlreadyResolved(uint256 reportId);
    error ReportNotResolved(uint256 reportId);
    error InvalidReward(uint256 min, uint256 max, uint256 provided);

    constructor(address _usdc, address _owner) Ownable(_owner) {
        usdc = IERC20(_usdc);
        _initializeRewardTiers();
    }

    function _initializeRewardTiers() internal {
        // Critical: ช่องโหว่ที่ทำให้สูญเสียเงินผู้ใช้ทั้งหมด
        maxRewards[Severity.Critical] = 250_000e6; // $250,000
        minRewards[Severity.Critical] = 50_000e6;  // $50,000

        // High: สูญเสียเงินบางส่วน หรือ governance ถูก manipulate
        maxRewards[Severity.High] = 50_000e6;    // $50,000
        minRewards[Severity.High] = 10_000e6;    // $10,000

        // Medium: ผลกระทบปานกลาง เช่น DoS ชั่วคราว
        maxRewards[Severity.Medium] = 10_000e6;  // $10,000
        minRewards[Severity.Medium] = 1_000e6;   // $1,000

        // Low: ผลกระทบน้อย
        maxRewards[Severity.Low] = 1_000e6;      // $1,000
        minRewards[Severity.Low] = 100e6;        // $100
    }

    /**
     * @notice Submit bug report (เก็บเป็น hash เพื่อ privacy)
     * @param reportHash keccak256 hash ของ report content
     */
    function submitReport(bytes32 reportHash) external returns (uint256 reportId) {
        reportId = reportCount++;

        reports[reportId] = BugReport({
            reporter: msg.sender,
            reportHash: reportHash,
            severity: Severity.Informational,
            reward: 0,
            paid: false,
            rejected: false,
            rejectionReason: "",
            submittedAt: block.timestamp,
            resolvedAt: 0
        });

        emit ReportSubmitted(reportId, msg.sender, reportHash);
    }

    /**
     * @notice ทีม security review และ resolve report
     */
    function resolveReport(
        uint256 reportId,
        Severity severity,
        uint256 reward,
        bool accept,
        string calldata reason
    ) external onlyOwner {
        BugReport storage report = reports[reportId];
        require(report.reporter != address(0), "Report not found");

        if (report.resolvedAt != 0) revert ReportAlreadyResolved(reportId);

        if (accept) {
            // ตรวจสอบ reward อยู่ใน range
            if (severity != Severity.Informational) {
                if (reward < minRewards[severity] || reward > maxRewards[severity]) {
                    revert InvalidReward(minRewards[severity], maxRewards[severity], reward);
                }
            }

            // ตรวจสอบมีเงินพอ
            if (usdc.balanceOf(address(this)) < reward) {
                revert InsufficientBountyFunds(reward, usdc.balanceOf(address(this)));
            }

            report.severity = severity;
            report.reward = reward;
        } else {
            report.rejected = true;
            report.rejectionReason = reason;
        }

        report.resolvedAt = block.timestamp;

        emit ReportResolved(reportId, severity, reward, accept);
    }

    /**
     * @notice จ่าย reward ให้ reporter
     */
    function payBounty(uint256 reportId) external onlyOwner {
        BugReport storage report = reports[reportId];

        if (report.resolvedAt == 0) revert ReportNotResolved(reportId);
        require(!report.paid, "Already paid");
        require(!report.rejected, "Report was rejected");
        require(report.reward > 0, "No reward to pay");

        report.paid = true;
        totalPaid += report.reward;

        usdc.transfer(report.reporter, report.reward);

        emit BountyPaid(reportId, report.reporter, report.reward);
    }

    /**
     * @notice ดู statistics ของ bounty program
     */
    function getBountyStats() external view returns (
        uint256 totalReports,
        uint256 totalPaidAmount,
        uint256 availableFunds,
        uint256 pendingReports
    ) {
        totalReports = reportCount;
        totalPaidAmount = totalPaid;
        availableFunds = usdc.balanceOf(address(this));

        for (uint256 i = 0; i < reportCount; i++) {
            if (reports[i].resolvedAt == 0) {
                pendingReports++;
            }
        }
    }

    // Funding function
    receive() external payable {}

    function fundBountyProgram(uint256 amount) external {
        usdc.transferFrom(msg.sender, address(this), amount);
    }
}
```

## Community Launch

```javascript
// community-setup.js - Script สำหรับตั้งค่า Discord bot และ community tools

const communitySetup = {
    discord: {
        channels: [
            { name: "📢-announcements", type: "GUILD_ANNOUNCEMENT", permissions: "readonly" },
            { name: "💬-general", type: "GUILD_TEXT" },
            { name: "❓-help", type: "GUILD_TEXT" },
            { name: "🔒-security", type: "GUILD_TEXT", permissions: "readonly" },
            { name: "🗳️-governance", type: "GUILD_TEXT" },
            { name: "💡-proposals", type: "GUILD_TEXT" },
            { name: "📊-protocol-stats", type: "GUILD_TEXT", permissions: "readonly" },
            { name: "🐛-bug-reports", type: "GUILD_TEXT" },
            { name: "🔧-developers", type: "GUILD_TEXT" },
            { name: "🌍-thai-community", type: "GUILD_TEXT" },
        ],
        roles: [
            { name: "Core Team", color: "#FF0000", permissions: "admin" },
            { name: "Security Researcher", color: "#FF6600" },
            { name: "Developer", color: "#0099FF" },
            { name: "OYT Holder (≥100)", color: "#9900FF" },
            { name: "OYT Whale (≥10000)", color: "#FFD700" },
            { name: "Community Member", color: "#00CC00" },
        ],
        bots: [
            "MEE6 - Moderation, leveling",
            "Carl-bot - Reaction roles, logging",
            "DeFi Pulse Bot - Live protocol stats",
            "Snapshot Bot - Governance voting alerts"
        ]
    },

    documentation: {
        platform: "GitBook",
        sections: [
            "Overview & How it works",
            "Getting Started (Deposit, Withdraw)",
            "Governance Guide",
            "Smart Contract Reference (auto-generated)",
            "Security & Audits",
            "Bug Bounty Program",
            "FAQ",
            "Changelog"
        ]
    },

    governanceForum: {
        platform: "Discourse",
        categories: [
            "General Discussion",
            "Governance Proposals (GIP)",
            "Temperature Check",
            "Protocol Research",
            "Security Disclosures",
            "Partnership Proposals"
        ],
        proposalTemplate: `
# [GIP-XXX] ชื่อ Proposal

**Author:** @username
**Date:** YYYY-MM-DD
**Status:** Discussion / Temperature Check / Voting

## Summary
สรุปสั้นๆ ว่า proposal นี้ต้องการทำอะไร

## Motivation
ทำไมต้องทำสิ่งนี้? ปัญหาที่แก้ไขคืออะไร?

## Specification
รายละเอียดทางเทคนิคของการเปลี่ยนแปลง

## Implementation
ขั้นตอนการ implement ถ้า proposal ผ่าน

## Voting Options
- Yes - สนับสนุน proposal นี้
- No - ไม่สนับสนุน
- Abstain - งดออกเสียง

## Timeline
- Discussion period: 1 สัปดาห์
- Snapshot vote: วันที่ XXX - XXX
        `
    }
};
```

## First 30 Days Monitoring

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title ProtocolMonitor
 * @notice ระบบ monitoring สำหรับ 30 วันแรกหลัง launch
 * @dev ติดตาม KPIs หลักของ protocol
 */
contract ProtocolMonitor {
    struct DailyMetrics {
        uint256 tvl;
        uint256 uniqueDepositors;
        uint256 totalTransactions;
        uint256 totalGasSpent;
        uint256 feesGenerated;
        uint256 avgApy;
        uint256 timestamp;
    }

    // เก็บ metrics รายวัน
    mapping(uint256 => DailyMetrics) public dailyMetrics; // day => metrics
    uint256 public launchDay;
    uint256 public currentDay;

    // Alert thresholds
    uint256 public constant TVL_DROP_THRESHOLD = 20; // 20% drop triggers alert
    uint256 public constant MIN_DAILY_TRANSACTIONS = 10;

    address public protocolVault;
    address public alertRecipient;

    event MetricsRecorded(uint256 day, DailyMetrics metrics);
    event AlertTriggered(string alertType, uint256 severity, string message);

    constructor(address _vault, address _alertRecipient) {
        protocolVault = _vault;
        alertRecipient = _alertRecipient;
        launchDay = block.timestamp / 1 days;
        currentDay = launchDay;
    }

    /**
     * @notice บันทึก metrics ประจำวัน (ควรเรียกผ่าน Keeper/Automation)
     */
    function recordDailyMetrics(DailyMetrics calldata metrics) external {
        // ใน production ควรมีการตรวจสอบ authorization
        uint256 dayIndex = block.timestamp / 1 days - launchDay;

        dailyMetrics[dayIndex] = metrics;
        currentDay = block.timestamp / 1 days;

        emit MetricsRecorded(dayIndex, metrics);

        // ตรวจสอบ alerts
        _checkAlerts(dayIndex, metrics);
    }

    /**
     * @notice ตรวจสอบ conditions ที่ต้องแจ้งเตือน
     */
    function _checkAlerts(uint256 day, DailyMetrics memory metrics) internal {
        // Alert: TVL ลดลงมากกว่า 20%
        if (day > 0) {
            DailyMetrics memory prevDay = dailyMetrics[day - 1];
            if (prevDay.tvl > 0) {
                uint256 tvlChange = prevDay.tvl > metrics.tvl
                    ? ((prevDay.tvl - metrics.tvl) * 100) / prevDay.tvl
                    : 0;

                if (tvlChange >= TVL_DROP_THRESHOLD) {
                    emit AlertTriggered(
                        "TVL_DROP",
                        2, // severity: high
                        string.concat(
                            "TVL dropped ",
                            _uint2str(tvlChange),
                            "% in 24 hours"
                        )
                    );
                }
            }
        }

        // Alert: Transaction count ต่ำมาก
        if (metrics.totalTransactions < MIN_DAILY_TRANSACTIONS && day > 2) {
            emit AlertTriggered(
                "LOW_ACTIVITY",
                1, // severity: medium
                "Daily transaction count below threshold"
            );
        }
    }

    /**
     * @notice ดู trend ของ TVL ใน N วันที่ผ่านมา
     */
    function getTVLTrend(uint256 daysBack) external view returns (
        uint256[] memory tvls,
        uint256[] memory timestamps
    ) {
        uint256 today = block.timestamp / 1 days - launchDay;
        uint256 startDay = today >= daysBack ? today - daysBack : 0;
        uint256 count = today - startDay + 1;

        tvls = new uint256[](count);
        timestamps = new uint256[](count);

        for (uint256 i = 0; i < count; i++) {
            tvls[i] = dailyMetrics[startDay + i].tvl;
            timestamps[i] = dailyMetrics[startDay + i].timestamp;
        }
    }

    /**
     * @notice สรุป KPIs สำหรับ 30 วันแรก
     */
    function getLaunchSummary() external view returns (
        uint256 peakTVL,
        uint256 totalUniqueUsers,
        uint256 totalTransactions,
        uint256 totalFeesGenerated,
        uint256 avgDailyAPY
    ) {
        uint256 today = block.timestamp / 1 days - launchDay;
        uint256 apySum = 0;
        uint256 apyCount = 0;

        for (uint256 i = 0; i <= today && i < 30; i++) {
            DailyMetrics memory m = dailyMetrics[i];
            if (m.tvl > peakTVL) peakTVL = m.tvl;
            if (m.uniqueDepositors > totalUniqueUsers) totalUniqueUsers = m.uniqueDepositors;
            totalTransactions += m.totalTransactions;
            totalFeesGenerated += m.feesGenerated;
            if (m.avgApy > 0) {
                apySum += m.avgApy;
                apyCount++;
            }
        }

        if (apyCount > 0) avgDailyAPY = apySum / apyCount;
    }

    function _uint2str(uint256 value) internal pure returns (string memory) {
        if (value == 0) return "0";
        uint256 temp = value;
        uint256 digits;
        while (temp != 0) {
            digits++;
            temp /= 10;
        }
        bytes memory buffer = new bytes(digits);
        while (value != 0) {
            digits--;
            buffer[digits] = bytes1(uint8(48 + uint256(value % 10)));
            value /= 10;
        }
        return string(buffer);
    }
}
```

## Incident Response Plan

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/utils/Pausable.sol";

/**
 * @title IncidentResponse
 * @notice ระบบ Incident Response สำหรับ OmniYield Protocol
 * @dev มี circuit breaker และระบบตอบสนองต่อเหตุการณ์ฉุกเฉิน
 */
contract IncidentResponse is AccessControl, Pausable {
    bytes32 public constant GUARDIAN_ROLE = keccak256("GUARDIAN_ROLE");
    bytes32 public constant SECURITY_COUNCIL_ROLE = keccak256("SECURITY_COUNCIL_ROLE");

    struct Incident {
        uint256 id;
        string title;
        uint8 severity; // 1=Low, 2=Medium, 3=High, 4=Critical
        uint256 detectedAt;
        uint256 resolvedAt;
        address detectedBy;
        bool paused; // protocol ถูก pause ไหม
        string[] actions; // actions ที่ดำเนินการแล้ว
    }

    mapping(uint256 => Incident) public incidents;
    uint256 public incidentCount;

    // Circuit breaker: ถ้าเกิน threshold จะ auto-pause
    uint256 public maxWithdrawPerBlock = 1_000_000e6; // 1M USDC per block
    uint256 public withdrawnThisBlock;
    uint256 public lastWithdrawBlock;

    event IncidentDeclared(
        uint256 indexed incidentId,
        string title,
        uint8 severity
    );

    event IncidentResolved(uint256 indexed incidentId);
    event EmergencyPause(address indexed by, string reason);
    event CircuitBreakerTriggered(uint256 amount, uint256 threshold);

    constructor(address[] memory guardians, address[] memory securityCouncil) {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);

        for (uint256 i = 0; i < guardians.length; i++) {
            _grantRole(GUARDIAN_ROLE, guardians[i]);
        }

        for (uint256 i = 0; i < securityCouncil.length; i++) {
            _grantRole(SECURITY_COUNCIL_ROLE, securityCouncil[i]);
        }
    }

    /**
     * @notice ประกาศ incident
     * @dev ถ้า severity = 4 (Critical) จะ auto-pause protocol
     */
    function declareIncident(
        string calldata title,
        uint8 severity,
        bool pauseProtocol
    ) external onlyRole(GUARDIAN_ROLE) returns (uint256 incidentId) {
        require(severity >= 1 && severity <= 4, "Invalid severity");

        incidentId = incidentCount++;

        incidents[incidentId] = Incident({
            id: incidentId,
            title: title,
            severity: severity,
            detectedAt: block.timestamp,
            resolvedAt: 0,
            detectedBy: msg.sender,
            paused: pauseProtocol,
            actions: new string[](0)
        });

        if (pauseProtocol || severity == 4) {
            _pause();
            emit EmergencyPause(msg.sender, title);
        }

        emit IncidentDeclared(incidentId, title, severity);
    }

    /**
     * @notice บันทึก action ที่ดำเนินการในระหว่าง incident
     */
    function addIncidentAction(
        uint256 incidentId,
        string calldata action
    ) external onlyRole(GUARDIAN_ROLE) {
        require(incidents[incidentId].detectedAt > 0, "Incident not found");
        incidents[incidentId].actions.push(action);
    }

    /**
     * @notice ปิด incident และ resume protocol (ถ้าจำเป็น)
     */
    function resolveIncident(
        uint256 incidentId,
        bool resumeProtocol
    ) external onlyRole(SECURITY_COUNCIL_ROLE) {
        Incident storage incident = incidents[incidentId];
        require(incident.detectedAt > 0, "Incident not found");
        require(incident.resolvedAt == 0, "Already resolved");

        incident.resolvedAt = block.timestamp;

        if (resumeProtocol && paused()) {
            _unpause();
        }

        emit IncidentResolved(incidentId);
    }

    /**
     * @notice Circuit breaker สำหรับ withdrawal
     */
    function checkWithdrawLimit(uint256 amount) internal {
        if (block.number > lastWithdrawBlock) {
            withdrawnThisBlock = 0;
            lastWithdrawBlock = block.number;
        }

        withdrawnThisBlock += amount;

        if (withdrawnThisBlock > maxWithdrawPerBlock) {
            emit CircuitBreakerTriggered(withdrawnThisBlock, maxWithdrawPerBlock);
            _pause();
            revert("Circuit breaker triggered: withdrawal limit exceeded");
        }
    }

    /**
     * @notice ดูรายละเอียด incident
     */
    function getIncident(uint256 incidentId) external view returns (
        Incident memory
    ) {
        require(incidents[incidentId].detectedAt > 0, "Incident not found");
        return incidents[incidentId];
    }
}
```

## Workshop: เตรียม Launch Protocol ของคุณ

### แบบฝึกหัดที่ 1: สร้าง Launch Checklist

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title LaunchChecklist
 * @notice Workshop: สร้าง checklist สำหรับ protocol ของคุณ
 *
 * TODO:
 * 1. เพิ่ม items ใน checklist ที่เหมาะกับ protocol ของคุณ
 * 2. เพิ่ม roles ที่รับผิดชอบแต่ละ item
 * 3. เพิ่มระบบ deadline tracking
 */
contract LaunchChecklist {
    enum Status { Pending, InProgress, Complete, Blocked }

    struct ChecklistItem {
        string description;
        Status status;
        address assignee;
        uint256 deadline;
        string notes;
    }

    mapping(uint256 => ChecklistItem) public items;
    uint256 public itemCount;

    event ItemAdded(uint256 indexed itemId, string description);
    event ItemUpdated(uint256 indexed itemId, Status status);

    function addItem(
        string calldata description,
        address assignee,
        uint256 deadline
    ) external returns (uint256 itemId) {
        itemId = itemCount++;
        items[itemId] = ChecklistItem({
            description: description,
            status: Status.Pending,
            assignee: assignee,
            deadline: deadline,
            notes: ""
        });
        emit ItemAdded(itemId, description);
    }

    function updateItemStatus(
        uint256 itemId,
        Status status,
        string calldata notes
    ) external {
        require(items[itemId].assignee == msg.sender, "Not assignee");
        items[itemId].status = status;
        items[itemId].notes = notes;
        emit ItemUpdated(itemId, status);
    }

    function getCompletionRate() external view returns (uint256 rate) {
        if (itemCount == 0) return 0;
        uint256 completed = 0;
        for (uint256 i = 0; i < itemCount; i++) {
            if (items[i].status == Status.Complete) completed++;
        }
        return (completed * 100) / itemCount;
    }
}
```

### แบบฝึกหัดที่ 2: วางแผน Bug Bounty Scope

```
## Bug Bounty Scope Template

### In-Scope Assets
| Asset | Type | Max Reward |
|-------|------|-----------|
| CoreVault.sol | Smart Contract | $250,000 |
| GovernanceToken.sol | Smart Contract | $100,000 |
| Governor.sol | Smart Contract | $100,000 |
| Web UI (funds theft only) | Website | $50,000 |

### Out-of-Scope
- Test files
- Documentation
- Issues in third-party dependencies
- Gas optimization (unless causes DoS)
- Issues requiring compromised admin key

### Severity Classification
| Level | Criteria | Reward Range |
|-------|----------|-------------|
| Critical | Direct theft of user funds, permanent protocol shutdown | $50,000 - $250,000 |
| High | Temporary DoS, significant fund loss risk | $10,000 - $50,000 |
| Medium | Indirect loss, temporary freeze | $1,000 - $10,000 |
| Low | Limited impact issues | $100 - $1,000 |

### Disclosure Process
1. Email: security@yourprotocol.io with encrypted report
2. 24-hour acknowledgement
3. 7-day response with preliminary assessment
4. 30-day fix window (may extend for complex issues)
5. Responsible disclosure after fix + 7 days
```

## สรุป Part 96

- **Launch Process**: testnet → audit → mainnet เป็นขั้นตอนที่ขาดไม่ได้ ห้ามข้ามขั้นตอน
- **Audit Preparation**: เตรียมเอกสาร, known issues list, และ test coverage สูงๆ ก่อนส่ง audit
- **Liquidity Bootstrap**: LBP ช่วยกระจาย token อย่างยุติธรรมและสร้าง initial liquidity
- **Bug Bounty**: ตั้งค่า scope, severity, และ reward ที่ชัดเจน เพื่อดึงดูด security researcher
- **Community**: Discord, Documentation, และ Governance Forum เป็นพื้นฐานของชุมชนที่แข็งแกร่ง
- **Monitoring**: ติดตาม TVL, users, transactions ทุกวันใน 30 วันแรก
- **Incident Response**: มี playbook และ circuit breaker ก่อนที่ปัญหาจะเกิดขึ้น

## Next: Part 97 - Career Path in Solidity Development

ในบทถัดไปเราจะเรียนรู้เส้นทางอาชีพในการเป็น Solidity Developer ตั้งแต่ Junior จนถึง Principal Engineer, skills matrix, portfolio projects, และวิธีเริ่มต้นเส้นทางการเป็น smart contract auditor
