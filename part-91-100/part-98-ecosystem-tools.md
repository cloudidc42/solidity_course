# Part 98: Solidity Ecosystem Tooling Reference

## บทนำ

การเลือกเครื่องมือที่ถูกต้องสำหรับงานแต่ละประเภทเป็นทักษะที่สำคัญของ Solidity Developer ในบทนี้เราจะสำรวจเครื่องมือทุกตัวในระบบนิเวศ Solidity อย่างละเอียด ตั้งแต่ Development Frameworks ไปจนถึง Security Tools และ Monitoring Solutions

## Development Frameworks: Feature Matrix

```
╔══════════════════════╦══════════╦══════════╦══════════╗
║ Feature              ║ Hardhat  ║ Foundry  ║ Truffle  ║
╠══════════════════════╬══════════╬══════════╬══════════╣
║ Language             ║ JS/TS    ║ Solidity ║ JS/TS    ║
║ Test Speed           ║ Medium   ║ Fast     ║ Slow     ║
║ Fuzz Testing         ║ Plugin   ║ Built-in ║ No       ║
║ Forking              ║ Yes      ║ Yes      ║ Yes      ║
║ Stack Traces         ║ Excellent║ Good     ║ Basic    ║
║ Gas Reports          ║ Plugin   ║ Built-in ║ Plugin   ║
║ Coverage             ║ Plugin   ║ Built-in ║ Plugin   ║
║ Debugging (console)  ║ Yes      ║ Yes      ║ Yes      ║
║ Plugin Ecosystem     ║ Large    ║ Growing  ║ Medium   ║
║ CI/CD Integration    ║ Easy     ║ Easy     ║ Medium   ║
║ Learning Curve       ║ Medium   ║ Medium   ║ Easy     ║
║ Script Language      ║ JS/TS    ║ Solidity ║ JS/TS    ║
║ Maintenance Status   ║ Active   ║ Active   ║ Slower   ║
╚══════════════════════╩══════════╩══════════╩══════════╝

Recommendation:
- New projects: Foundry (speed, built-in tools)
- JS-heavy teams: Hardhat (familiar ecosystem)
- Legacy projects: maintain what works
```

## Hardhat: Advanced Configuration

```javascript
// hardhat.config.ts - Professional configuration
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
import "@nomicfoundation/hardhat-verify";
import "@openzeppelin/hardhat-upgrades";
import "hardhat-gas-reporter";
import "hardhat-contract-sizer";
import "solidity-coverage";
import "hardhat-deploy";
import * as dotenv from "dotenv";

dotenv.config();

const PRIVATE_KEY = process.env.PRIVATE_KEY || "0x" + "0".repeat(64);
const ETHERSCAN_KEY = process.env.ETHERSCAN_KEY || "";
const INFURA_KEY = process.env.INFURA_KEY || "";
const ALCHEMY_KEY = process.env.ALCHEMY_KEY || "";

const config: HardhatUserConfig = {
    solidity: {
        compilers: [
            {
                version: "0.8.24",
                settings: {
                    optimizer: {
                        enabled: true,
                        runs: 200,
                        details: {
                            yul: true,
                            yulDetails: {
                                stackAllocation: true,
                                optimizerSteps: "dhfoDgvulfnTUtnIf",
                            },
                        },
                    },
                    viaIR: true, // enable Yul IR optimization
                    metadata: {
                        bytecodeHash: "none", // deterministic builds
                    },
                },
            },
        ],
    },

    networks: {
        hardhat: {
            chainId: 31337,
            forking: {
                url: `https://eth-mainnet.g.alchemy.com/v2/${ALCHEMY_KEY}`,
                blockNumber: 20_000_000, // pin block สำหรับ reproducible tests
                enabled: process.env.FORK === "true",
            },
            accounts: {
                count: 20,
                accountsBalance: "10000000000000000000000", // 10,000 ETH
            },
            gas: "auto",
            gasPrice: "auto",
            loggingEnabled: false,
        },

        sepolia: {
            url: `https://sepolia.infura.io/v3/${INFURA_KEY}`,
            accounts: [PRIVATE_KEY],
            chainId: 11155111,
            gasMultiplier: 1.2,
        },

        mainnet: {
            url: `https://eth-mainnet.g.alchemy.com/v2/${ALCHEMY_KEY}`,
            accounts: [PRIVATE_KEY],
            chainId: 1,
            gasMultiplier: 1.1,
            timeout: 120000,
        },
    },

    etherscan: {
        apiKey: {
            mainnet: ETHERSCAN_KEY,
            sepolia: ETHERSCAN_KEY,
            polygon: process.env.POLYGONSCAN_KEY || "",
            arbitrumOne: process.env.ARBISCAN_KEY || "",
        },
    },

    gasReporter: {
        enabled: process.env.REPORT_GAS === "true",
        currency: "USD",
        coinmarketcap: process.env.CMC_KEY,
        outputFile: "./gas-report.txt",
        noColors: false,
        excludeContracts: ["Mock"],
    },

    contractSizer: {
        alphaSort: true,
        runOnCompile: process.env.SIZE === "true",
        disambiguatePaths: false,
    },

    mocha: {
        timeout: 120000,
        reporter: "spec",
    },
};

export default config;
```

## Foundry: Complete Configuration

```toml
# foundry.toml - Production configuration
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
test = "test"
script = "script"

# Compiler
solc_version = "0.8.24"
optimizer = true
optimizer_runs = 200
via_ir = true

# Testing
fuzz = { runs = 1000, seed = "0x1" }
invariant = { runs = 256, depth = 50, fail_on_revert = true }

# Gas
gas_reports = ["*"]
gas_limit = 9000000

# Environment
allow_paths = ["../lib"]

# Remappings
remappings = [
    "@openzeppelin/=lib/openzeppelin-contracts/",
    "@forge-std/=lib/forge-std/src/",
    "@chainlink/=lib/chainlink/contracts/src/v0.8/",
]

# RPC endpoints (reference by name)
[rpc_endpoints]
mainnet = "${ETH_RPC_URL}"
sepolia = "${SEPOLIA_RPC_URL}"
arbitrum = "${ARB_RPC_URL}"
polygon = "${POLYGON_RPC_URL}"

# Etherscan API keys
[etherscan]
mainnet = { key = "${ETHERSCAN_KEY}" }
sepolia = { key = "${ETHERSCAN_KEY}" }
arbitrum = { key = "${ARBISCAN_KEY}", url = "https://api.arbiscan.io/api" }
polygon = { key = "${POLYGONSCAN_KEY}", url = "https://api.polygonscan.com/api" }

[profile.ci]
# Faster CI settings
fuzz = { runs = 256 }
optimizer_runs = 10

[profile.intense]
# Comprehensive testing
fuzz = { runs = 10000 }
invariant = { runs = 1000, depth = 100 }
```

### Foundry Test Patterns

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "forge-std/StdCheats.sol";
import "forge-std/console2.sol";
import "../src/OmniYieldVault.sol";

/**
 * @title VaultTest
 * @notice Comprehensive Foundry test patterns
 */
contract VaultTest is Test {
    OmniYieldVault vault;
    MockERC20 usdc;

    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    address attacker = makeAddr("attacker");

    // ════════════════════════════════════════
    // Setup
    // ════════════════════════════════════════

    function setUp() public {
        usdc = new MockERC20("USD Coin", "USDC", 6);
        vault = new OmniYieldVault(address(usdc), address(this));

        // Fund test accounts
        deal(address(usdc), alice, 1_000_000e6);
        deal(address(usdc), bob, 1_000_000e6);

        // Approve vault
        vm.prank(alice);
        usdc.approve(address(vault), type(uint256).max);

        vm.prank(bob);
        usdc.approve(address(vault), type(uint256).max);
    }

    // ════════════════════════════════════════
    // Unit Tests
    // ════════════════════════════════════════

    function test_Deposit_Basic() public {
        uint256 depositAmount = 1000e6;

        vm.prank(alice);
        uint256 shares = vault.deposit(depositAmount, alice);

        assertEq(vault.balanceOf(alice), shares);
        assertEq(vault.totalAssets(), depositAmount);
        assertGt(shares, 0);
    }

    function test_Deposit_RevertOnZero() public {
        vm.expectRevert(OmniYieldVault.ZeroAmount.selector);
        vm.prank(alice);
        vault.deposit(0, alice);
    }

    function test_Withdraw_FullAmount() public {
        uint256 depositAmount = 1000e6;

        vm.prank(alice);
        uint256 shares = vault.deposit(depositAmount, alice);

        uint256 balanceBefore = usdc.balanceOf(alice);

        vm.prank(alice);
        uint256 withdrawn = vault.redeem(shares, alice, alice);

        assertEq(usdc.balanceOf(alice) - balanceBefore, withdrawn);
        assertEq(vault.balanceOf(alice), 0);
    }

    // ════════════════════════════════════════
    // Fuzz Tests
    // ════════════════════════════════════════

    function testFuzz_Deposit(uint256 amount) public {
        // Bound amount ให้อยู่ใน reasonable range
        amount = bound(amount, 1e6, 1_000_000e6);

        deal(address(usdc), alice, amount);

        vm.prank(alice);
        uint256 shares = vault.deposit(amount, alice);

        // Properties ที่ต้องเป็นจริงเสมอ
        assertGt(shares, 0, "Shares must be positive");
        assertEq(vault.totalAssets(), amount, "Total assets must equal deposit");
        assertEq(vault.balanceOf(alice), shares, "Alice shares must match");
    }

    function testFuzz_DepositWithdraw_Roundtrip(uint256 amount) public {
        amount = bound(amount, 1e6, 1_000_000e6);
        deal(address(usdc), alice, amount);

        vm.prank(alice);
        uint256 shares = vault.deposit(amount, alice);

        vm.prank(alice);
        uint256 withdrawn = vault.redeem(shares, alice, alice);

        // ต้องได้รับ assets กลับมาอย่างน้อย 99.9% (accounting for rounding)
        assertGe(withdrawn * 1000, amount * 999, "Withdraw should be close to deposit");
    }

    // ════════════════════════════════════════
    // Invariant Tests
    // ════════════════════════════════════════

    function invariant_TotalSharesMatchAccounts() public view {
        // totalSupply() == sum of all balances (ERC20 invariant)
        // ใน real invariant test จะต้องมี actors ที่ทำ random actions
        assertEq(
            vault.totalSupply(),
            vault.balanceOf(alice) + vault.balanceOf(bob),
            "Total shares must match sum of balances"
        );
    }

    function invariant_AssetsAlwaysPositive() public view {
        assertGe(vault.totalAssets(), 0, "Total assets cannot be negative");
    }

    // ════════════════════════════════════════
    // Fork Tests
    // ════════════════════════════════════════

    function test_Fork_IntegrationWithAave() public {
        // เรียกใช้เฉพาะเมื่อ fork mainnet
        // vm.createSelectFork("mainnet", 20_000_000);

        // ทดสอบ integration กับ Aave
        // address aavePool = 0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2;
        // ...
    }

    // ════════════════════════════════════════
    // Snapshot Tests
    // ════════════════════════════════════════

    function test_StateSnapshot() public {
        vm.prank(alice);
        vault.deposit(1000e6, alice);

        uint256 snapshot = vm.snapshot();

        // ทำ state changes
        vm.prank(bob);
        usdc.approve(address(vault), 500e6);
        deal(address(usdc), bob, 500e6);
        vm.prank(bob);
        vault.deposit(500e6, bob);

        assertEq(vault.totalAssets(), 1500e6);

        // Revert to snapshot
        vm.revertTo(snapshot);

        assertEq(vault.totalAssets(), 1000e6);
    }
}

contract MockERC20 {
    string public name;
    string public symbol;
    uint8 public decimals;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    uint256 public totalSupply;

    constructor(string memory _name, string memory _symbol, uint8 _decimals) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
    }

    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount;
        totalSupply += amount;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount);
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(balanceOf[from] >= amount);
        require(allowance[from][msg.sender] >= amount);
        allowance[from][msg.sender] -= amount;
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        return true;
    }
}
```

## Security Tools Deep Dive

### Slither

```bash
#!/bin/bash
# run-slither.sh - Comprehensive Slither analysis

echo "=== Running Slither Security Analysis ==="

# Basic run
slither . \
    --filter-paths "lib/,test/,script/" \
    --exclude-dependencies \
    --detect all \
    --json slither-report.json

# Specific detectors สำหรับ DeFi
slither . \
    --detect reentrancy-eth,reentrancy-no-eth,reentrancy-benign \
    --detect arbitrary-send-eth \
    --detect delegatecall-loop \
    --detect msg-value-loop \
    --detect tautology \
    --detect boolean-equality \
    --detect locked-ether \
    --detect tx-origin \
    --detect uninitialized-local \
    --detect uninitialized-state \
    --detect unprotected-upgrade

# Generate inheritance graph
slither . --print inheritance-graph
dot -Tpng inheritance-graph.dot -o inheritance-graph.png

# Generate call graph
slither . --print call-graph
dot -Tpng call-graph.dot -o call-graph.png

# Check ERC compliance
slither-check-erc src/OmniYieldToken.sol OmniYieldToken --erc ERC20
slither-check-erc src/OmniYieldVault.sol OmniYieldVault --erc ERC4626

echo "=== Slither Analysis Complete ==="
echo "Report saved to: slither-report.json"
```

### Mythril

```bash
#!/bin/bash
# run-mythril.sh

echo "=== Running Mythril Analysis ==="

# Basic analysis
myth analyze src/OmniYieldVault.sol \
    --solc-json solc-config.json \
    --execution-timeout 300 \
    --max-depth 50 \
    -o jsonv2 \
    > mythril-report.json

# Specific vulnerability checks
myth analyze src/OmniYieldVault.sol \
    --execution-timeout 600 \
    --strategy dfs \
    --solver-timeout 10000

# สำหรับ contract ที่ complex มาก
myth analyze src/OmniYieldVault.sol \
    --execution-timeout 1800 \
    --create-timeout 60 \
    --parallel-solving

echo "Analysis complete. Check mythril-report.json"
```

### Echidna (Fuzzing)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "../src/OmniYieldVault.sol";
import "./mocks/MockERC20.sol";

/**
 * @title EchidnaVaultTest
 * @notice Echidna fuzzing test สำหรับ OmniYield Vault
 * @dev รัน: echidna . --contract EchidnaVaultTest --config echidna.yaml
 */
contract EchidnaVaultTest {
    OmniYieldVault vault;
    MockERC20 asset;

    address constant USER1 = address(0x1);
    address constant USER2 = address(0x2);

    constructor() {
        asset = new MockERC20("Test Token", "TEST", 18);
        vault = new OmniYieldVault(address(asset), address(this));

        // Initial setup
        asset.mint(USER1, 1_000_000e18);
        asset.mint(USER2, 1_000_000e18);
    }

    // Invariant: Total supply ต้องสอดคล้องกับ total assets
    function echidna_totalSupplyConsistency() public view returns (bool) {
        uint256 supply = vault.totalSupply();
        uint256 assets = vault.totalAssets();

        // ถ้ามี shares ต้องมี assets
        if (supply > 0) return assets > 0;
        return true;
    }

    // Invariant: ไม่มีใครถือ shares มากกว่า total supply
    function echidna_noSharesExceedSupply() public view returns (bool) {
        return vault.balanceOf(USER1) + vault.balanceOf(USER2) <= vault.totalSupply();
    }

    // Invariant: Exchange rate ต้องไม่ลดลง (เว้นแต่ strategic loss)
    uint256 previousRate;

    function echidna_exchangeRateDoesNotDecrease() public view returns (bool) {
        uint256 supply = vault.totalSupply();
        if (supply == 0) return true;

        uint256 currentRate = (vault.totalAssets() * 1e18) / supply;
        return currentRate >= previousRate;
    }

    // Actions ที่ Echidna จะ call แบบ random
    function depositForUser1(uint256 amount) public {
        amount = (amount % 1_000e18) + 1;
        asset.mint(USER1, amount);

        try vault.deposit{gas: 500000}(amount, USER1) {
            previousRate = vault.totalSupply() > 0
                ? (vault.totalAssets() * 1e18) / vault.totalSupply()
                : 1e18;
        } catch {}
    }

    function withdrawForUser1(uint256 shares) public {
        uint256 userShares = vault.balanceOf(USER1);
        if (userShares == 0) return;

        shares = (shares % userShares) + 1;
        try vault.redeem{gas: 500000}(shares, USER1, USER1) {} catch {}
    }
}
```

```yaml
# echidna.yaml - Echidna configuration
testMode: property
testLimit: 50000
seqLen: 100
shrinkLimit: 5000
fuzzer: mutSeeds
coverage: true
corpusDir: "./corpus"
coverageOutputDir: "./coverage-echidna"
outputFormat: text

balanceContract: 10000000000000000000
balanceAddr: 10000000000000000000

# Addresses to use
sender: ["0x10000", "0x20000", "0x30000"]
deployer: "0x30000"

# Slippage tolerance
swapTimeout: 60

# Logging
quiet: false
```

### Certora Formal Verification

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title VaultSpec.spec
 * @notice Certora Verification Language (CVL) specs สำหรับ Vault
 * @dev รัน: certoraRun src/OmniYieldVault.sol --verify OmniYieldVault:specs/VaultSpec.spec
 */

/*
// ========================================
// vault_spec.spec (CVL syntax)
// ========================================

methods {
    function totalAssets() external returns (uint256) envfree;
    function totalSupply() external returns (uint256) envfree;
    function balanceOf(address) external returns (uint256) envfree;
    function deposit(uint256, address) external returns (uint256);
    function withdraw(uint256, address, address) external returns (uint256);
    function redeem(uint256, address, address) external returns (uint256);
    function convertToShares(uint256) external returns (uint256) envfree;
    function convertToAssets(uint256) external returns (uint256) envfree;
}

// Invariant: totalSupply reflects sum of all shares
invariant totalSupplyIsConsistent()
    totalSupply() >= 0
    {
        preserved with (env e) {
            require e.msg.sender != 0;
        }
    }

// Rule: deposit increases totalAssets
rule depositIncreasesAssets(env e) {
    uint256 assetsBefore = totalAssets();
    uint256 amount;
    address receiver;

    require amount > 0;

    deposit(e, amount, receiver);

    uint256 assetsAfter = totalAssets();
    assert assetsAfter == assetsBefore + amount,
        "Deposit must increase totalAssets by exact amount";
}

// Rule: ไม่มีใครสามารถ mint shares โดยไม่ deposit
rule noFreeShares(env e, method f) {
    uint256 sharesBefore = totalSupply();
    uint256 assetsBefore = totalAssets();

    calldataarg args;
    f(e, args);

    uint256 sharesAfter = totalSupply();
    uint256 assetsAfter = totalAssets();

    assert sharesAfter > sharesBefore => assetsAfter > assetsBefore,
        "Cannot mint shares without depositing assets";
}

// Rule: Withdraw ต้องลด totalAssets
rule withdrawDecreasesAssets(env e) {
    uint256 assetsBefore = totalAssets();
    uint256 shares;
    address receiver;
    address owner;

    require balanceOf(owner) >= shares;
    require shares > 0;

    uint256 withdrawn = redeem(e, shares, receiver, owner);

    uint256 assetsAfter = totalAssets();
    assert assetsAfter == assetsBefore - withdrawn,
        "Redeem must decrease totalAssets";
}
*/

// Solidity implementation สำหรับทดสอบ properties เดียวกัน
contract VaultSpecTest {
    // Replicate CVL rules as Solidity tests
    function rule_depositIncreasesAssets(
        address vault,
        address asset,
        uint256 amount
    ) external returns (bool) {
        // Simplified test - ใน practice ใช้ Certora CLI
        return true;
    }
}
```

### Halmos (Symbolic Execution)

```bash
#!/bin/bash
# run-halmos.sh - Symbolic execution tests

echo "=== Running Halmos Symbolic Execution ==="

# Run halmos (ต้อง install ก่อน: pip install halmos)
halmos \
    --contract VaultTest \
    --function "check_" \
    --solver-timeout 30 \
    --loop 3

# Test specific invariants
halmos \
    --contract InvariantTest \
    --function "prove_totalAssetsConsistency" \
    --solver z3 \
    --verbose

echo "=== Halmos Analysis Complete ==="
```

## Deployment Tools

### OpenZeppelin Ignition

```typescript
// ignition/modules/OmniYield.ts
import { buildModule } from "@nomicfoundation/hardhat-ignition/modules";
import { parseEther, parseUnits } from "viem";

const OmniYieldModule = buildModule("OmniYieldModule", (m) => {
    // Parameters (overridable at deploy time)
    const admin = m.getParameter("admin", "0x0000..."); // multisig
    const usdc = m.getParameter("usdc", "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48");
    const timelockDelay = m.getParameter("timelockDelay", 48 * 60 * 60); // 48 hours

    // Deploy Token
    const token = m.contract("OmniYieldToken", [
        "OmniYield Token",
        "OYT",
        admin,
    ]);

    // Deploy Timelock
    const timelock = m.contract("OmniYieldTimelock", [
        timelockDelay,
        [admin], // proposers
        ["0x0000000000000000000000000000000000000000"], // executors (anyone)
        admin,
    ]);

    // Deploy Vault (depends on token and timelock)
    const vault = m.contract("OmniYieldVault", [
        usdc,
        token,
        timelock,
    ]);

    // Deploy Governance (depends on token and timelock)
    const governance = m.contract("OmniYieldGovernance", [
        token,
        timelock,
        "OmniYield Governor",
    ]);

    // Post-deployment calls
    m.call(token, "grantRole", [
        m.staticCall(token, "MINTER_ROLE"),
        vault,
    ]);

    m.call(timelock, "grantRole", [
        m.staticCall(timelock, "PROPOSER_ROLE"),
        governance,
    ]);

    return { token, vault, governance, timelock };
});

export default OmniYieldModule;
```

```bash
# Deploy ด้วย Ignition
npx hardhat ignition deploy ./ignition/modules/OmniYield.ts \
    --network sepolia \
    --verify \
    --parameters '{"admin":"0x...", "timelockDelay": 3600}'

# Verify ทุก contracts
npx hardhat ignition verify ./ignition/deployments/chain-11155111/
```

### Safe Deploy Scripts

```typescript
// scripts/safe-deploy.ts - Deploy ผ่าน Gnosis Safe
import { ethers } from "hardhat";
import { SafeFactory, Safe } from "@safe-global/protocol-kit";
import { MetaTransactionData } from "@safe-global/safe-core-sdk-types";

async function deployViaSafe() {
    const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
    const signer = new ethers.Wallet(process.env.PRIVATE_KEY!, provider);

    // Connect to existing Safe
    const safe = await Safe.create({
        ethAdapter: new EthersAdapter({ ethers, signerOrProvider: signer }),
        safeAddress: process.env.SAFE_ADDRESS!,
    });

    // Prepare deployment transactions
    const vaultFactory = await ethers.getContractFactory("OmniYieldVault");
    const deployTx = await vaultFactory.getDeployTransaction(
        process.env.USDC_ADDRESS!,
        process.env.TOKEN_ADDRESS!,
        process.env.TIMELOCK_ADDRESS!
    );

    // Create Safe transaction
    const safeTransactions: MetaTransactionData[] = [
        {
            to: ethers.ZeroAddress, // deploy = to zero address
            data: deployTx.data!,
            value: "0",
            operation: 0, // CALL
        }
    ];

    const safeTransaction = await safe.createTransaction({
        transactions: safeTransactions,
    });

    // Sign with this signer
    const signedTx = await safe.signTransaction(safeTransaction);

    console.log("Transaction signed. Share with other signers:");
    console.log(JSON.stringify(signedTx));

    // Execute ถ้ามี threshold แล้ว
    const txResponse = await safe.executeTransaction(signedTx);
    await txResponse.transactionResponse?.wait();

    console.log("Deployment executed via Safe!");
}
```

## Monitoring Tools

### OpenZeppelin Defender

```typescript
// defender-monitor.ts - Setup Defender monitoring
import { Defender } from "@openzeppelin/defender-sdk";

const client = new Defender({
    apiKey: process.env.DEFENDER_API_KEY!,
    apiSecret: process.env.DEFENDER_API_SECRET!,
});

async function setupMonitoring() {
    // Create Sentinel (Monitor)
    const sentinel = await client.monitor.create({
        type: "BLOCK",
        name: "OmniYield Vault Monitor",
        network: "mainnet",
        addresses: [process.env.VAULT_ADDRESS!],
        abi: JSON.stringify(VaultABI),

        // Alert conditions
        conditions: {
            event: [
                {
                    signature: "Withdraw(address,address,address,uint256,uint256)",
                    expression: "assets > 1000000000000", // > 1M USDC
                },
                {
                    signature: "EmergencyPause(address,string)",
                },
            ],
            function: [
                {
                    signature: "pause()",
                }
            ],
        },

        // Notification channels
        notificationChannels: [
            "email-channel-id",
            "slack-channel-id",
            "pagerduty-channel-id",
        ],

        // Alert throttling
        alertThreshold: {
            amount: 5,
            windowSeconds: 3600,
        },
    });

    console.log("Sentinel created:", sentinel.monitorId);

    // Create Relayer (สำหรับ automated responses)
    const relayer = await client.relay.create({
        name: "OmniYield Emergency Relayer",
        network: "mainnet",
        minBalance: BigInt("1000000000000000000"), // 1 ETH
    });

    // Create Autotask (automated response)
    await client.autotask.create({
        name: "Emergency Pause Autotask",
        encodedZippedCode: await getEncodedZippedCode("./autotasks/emergencyPause.js"),
        relayerId: relayer.relayerId,
        trigger: {
            type: "sentinel",
            sentinelId: sentinel.monitorId,
        },
        paused: false,
    });

    console.log("Monitoring setup complete!");
}

// autotasks/emergencyPause.js - Autotask code
const autotaskCode = `
const { ethers } = require("ethers");

exports.handler = async function(credentials) {
    const provider = new ethers.providers.JsonRpcProvider(credentials.httpsUrl);
    const signer = credentials.relayer.getSigner(provider);

    const vault = new ethers.Contract(
        process.env.VAULT_ADDRESS,
        ["function pause() external"],
        signer
    );

    // ตรวจสอบ conditions ก่อน pause
    const tvl = await vault.totalAssets();
    if (tvl < ethers.utils.parseUnits("1000000", 6)) {
        console.log("TVL too low, skipping auto-pause");
        return;
    }

    const tx = await vault.pause();
    await tx.wait();

    console.log("Emergency pause executed:", tx.hash);

    // Notify team
    await fetch(process.env.SLACK_WEBHOOK, {
        method: "POST",
        body: JSON.stringify({
            text: \`🚨 OmniYield Vault PAUSED. Tx: \${tx.hash}\`
        })
    });
}
`;
```

### Tenderly Monitoring

```javascript
// tenderly-setup.js - Setup Tenderly monitoring
const { Tenderly } = require("@tenderly/sdk");

const tenderly = new Tenderly({
    accessKey: process.env.TENDERLY_ACCESS_KEY,
    accountName: "omniyield",
    projectName: "protocol",
    network: 1, // mainnet
});

async function setupTenderly() {
    // Add contracts for monitoring
    await tenderly.contracts.add({
        address: process.env.VAULT_ADDRESS,
        displayName: "OmniYield Vault",
    });

    // Create alert for large withdrawals
    await tenderly.alerts.create({
        name: "Large Withdrawal Alert",
        description: "Alert when withdrawal > 500K USDC",
        conditions: [
            {
                type: "transaction",
                params: {
                    contract_address: process.env.VAULT_ADDRESS,
                    event_id: "Withdraw",
                    parameter_conditions: [
                        {
                            parameter: "assets",
                            type: "greater_than",
                            value: "500000000000", // 500K USDC (6 decimals)
                        }
                    ]
                }
            }
        ],
        channels: [
            { type: "email", sendTo: "security@omniyield.io" },
            { type: "slack", webhookUrl: process.env.SLACK_WEBHOOK }
        ]
    });

    // Setup transaction simulations
    const simResult = await tenderly.simulator.simulateTransaction({
        network_id: "1",
        from: "0x...",
        to: process.env.VAULT_ADDRESS,
        input: "0x...", // encoded function call
        gas: 500000,
        gas_price: "20000000000",
        value: "0",
        save: true,
        save_if_fails: true,
    });

    console.log("Simulation result:", simResult);
}
```

### Dune Analytics Queries

```sql
-- dune-queries/omniyield-tvl.sql
-- Total Value Locked over time

WITH vault_events AS (
    SELECT
        DATE_TRUNC('day', evt_block_time) as day,
        SUM(CASE WHEN type = 'deposit' THEN assets ELSE -assets END) as net_flow,
        COUNT(DISTINCT caller) as unique_users,
        COUNT(*) as tx_count
    FROM (
        -- Deposits
        SELECT
            evt_block_time,
            'deposit' as type,
            assets / 1e6 as assets,  -- USDC 6 decimals
            caller
        FROM omniyield_vault_evt_Deposit

        UNION ALL

        -- Withdrawals
        SELECT
            evt_block_time,
            'withdraw' as type,
            assets / 1e6 as assets,
            caller
        FROM omniyield_vault_evt_Withdraw
    ) events
    GROUP BY 1
),

cumulative_tvl AS (
    SELECT
        day,
        net_flow,
        unique_users,
        tx_count,
        SUM(net_flow) OVER (ORDER BY day ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) as tvl
    FROM vault_events
)

SELECT
    day,
    tvl,
    unique_users,
    tx_count,
    net_flow
FROM cumulative_tvl
ORDER BY day DESC
LIMIT 90  -- Last 90 days
;

-- dune-queries/omniyield-user-metrics.sql
-- User growth and retention

WITH user_first_deposit AS (
    SELECT
        caller as user_address,
        MIN(DATE_TRUNC('week', evt_block_time)) as first_deposit_week,
        COUNT(*) as total_deposits,
        SUM(assets / 1e6) as total_deposited_usdc
    FROM omniyield_vault_evt_Deposit
    GROUP BY 1
)

SELECT
    first_deposit_week,
    COUNT(DISTINCT user_address) as new_users,
    SUM(COUNT(DISTINCT user_address)) OVER (
        ORDER BY first_deposit_week
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) as cumulative_users,
    AVG(total_deposited_usdc) as avg_deposit_size
FROM user_first_deposit
GROUP BY 1
ORDER BY 1
;
```

### Forta Bot

```typescript
// forta-bot/src/agent.ts - Forta monitoring bot
import {
    Finding,
    HandleTransaction,
    TransactionEvent,
    FindingSeverity,
    FindingType,
    getEthersProvider,
} from "forta-agent";
import { BigNumber } from "ethers";

const VAULT_ADDRESS = process.env.VAULT_ADDRESS!.toLowerCase();
const LARGE_WITHDRAW_THRESHOLD = BigNumber.from("500000000000"); // 500K USDC

// Track cumulative withdrawals per block
const withdrawalsByBlock: Map<number, BigNumber> = new Map();

const handleTransaction: HandleTransaction = async (txEvent: TransactionEvent) => {
    const findings: Finding[] = [];

    // ตรวจสอบ withdraw events
    const withdrawEvents = txEvent.filterLog(
        "Withdraw(address,address,address,uint256,uint256)",
        VAULT_ADDRESS
    );

    for (const event of withdrawEvents) {
        const assets = BigNumber.from(event.args.assets);

        // Large single withdrawal
        if (assets.gt(LARGE_WITHDRAW_THRESHOLD)) {
            findings.push(
                Finding.fromObject({
                    name: "Large Withdrawal Detected",
                    description: `Withdrawal of ${assets.div(1e6).toString()} USDC`,
                    alertId: "OMNIYIELD-1",
                    severity: FindingSeverity.High,
                    type: FindingType.Suspicious,
                    metadata: {
                        amount: assets.toString(),
                        caller: event.args.caller,
                        receiver: event.args.receiver,
                    },
                })
            );
        }
    }

    // ตรวจสอบ emergency pause
    const pauseEvents = txEvent.filterLog(
        "Paused(address)",
        VAULT_ADDRESS
    );

    if (pauseEvents.length > 0) {
        findings.push(
            Finding.fromObject({
                name: "Protocol Emergency Pause",
                description: "OmniYield Vault was paused",
                alertId: "OMNIYIELD-2",
                severity: FindingSeverity.Critical,
                type: FindingType.Exploit,
                metadata: {
                    account: pauseEvents[0].args.account,
                    txHash: txEvent.hash,
                },
            })
        );
    }

    return findings;
};

export default {
    handleTransaction,
};
```

## Workshop: Setup Monitoring Stack

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title MonitoringWorkshop
 * @notice Workshop: ตั้งค่า monitoring stack สำหรับ protocol ของคุณ
 *
 * Task 1: Slither Analysis
 * ─────────────────────────
 * 1. Install: pip install slither-analyzer
 * 2. Run: slither . --filter-paths "lib/"
 * 3. Fix findings ระดับ High และ Medium ทั้งหมด
 *
 * Task 2: Echidna Fuzzing
 * ─────────────────────────
 * 1. Install: docker pull trailofbits/echidna
 * 2. เขียน invariant tests สำหรับ vault ของคุณ
 * 3. Run 10,000 iterations
 * 4. Document ทุก violation ที่เจอ
 *
 * Task 3: Defender Setup
 * ─────────────────────────
 * 1. Create OZ Defender account
 * 2. Add contracts to monitoring
 * 3. Create sentinel สำหรับ:
 *    - Large withdrawals (>$100K)
 *    - Emergency pause
 *    - Role changes
 * 4. Connect Slack webhook
 *
 * Task 4: Dune Dashboard
 * ─────────────────────────
 * 1. Create Dune account
 * 2. เขียน SQL สำหรับ:
 *    - Daily TVL
 *    - Unique users
 *    - Fee revenue
 * 3. Create public dashboard
 */
contract MonitoringChecklist {
    struct MonitoringLayer {
        string name;
        string purpose;
        string tool;
        bool implemented;
    }

    MonitoringLayer[] public layers;

    constructor() {
        layers.push(MonitoringLayer(
            "Static Analysis",
            "Find bugs before deployment",
            "Slither + Mythril",
            false
        ));
        layers.push(MonitoringLayer(
            "Fuzzing",
            "Find edge case violations",
            "Echidna + Forge",
            false
        ));
        layers.push(MonitoringLayer(
            "Formal Verification",
            "Prove critical properties",
            "Certora + Halmos",
            false
        ));
        layers.push(MonitoringLayer(
            "Runtime Monitoring",
            "Alert on suspicious activity",
            "OZ Defender + Tenderly",
            false
        ));
        layers.push(MonitoringLayer(
            "Analytics",
            "Track protocol metrics",
            "Dune + The Graph",
            false
        ));
    }
}
```

## สรุป Part 98

- **Hardhat**: เหมาะสำหรับ JS/TS teams, plugin ecosystem ดี, stack traces excellent
- **Foundry**: เร็วกว่า, fuzz testing built-in, เขียน tests ด้วย Solidity
- **Slither**: Static analysis ที่ต้องรันทุกครั้งก่อน deploy, ฟรีและมีประสิทธิภาพ
- **Echidna**: Fuzzing สำหรับหา invariant violations, เหมาะกับ mathematical contracts
- **Certora + Halmos**: Formal verification สำหรับ prove critical properties
- **OZ Defender**: Monitoring, autotasks, และ automated response สำหรับ mainnet
- **Tenderly**: Transaction simulation และ debugging tool ที่ดีที่สุด
- **Dune Analytics**: Analytics dashboard สำหรับ protocol metrics

## Next: Part 99 - Future of Solidity & Ethereum

ในบทถัดไปเราจะดูอนาคตของ Solidity และ Ethereum ecosystem รวมถึง upcoming EIPs, Solidity language features ใหม่, upgrade roadmap, และ alternative languages
