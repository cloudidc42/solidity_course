# Part 50: Production Deployment Checklist

## สารบัญ
1. Pre-Deployment Security Checklist
2. Deployment Script Best Practices
3. Verification and Monitoring Setup
4. Post-Deployment Validation
5. Workshop: Complete Production Deploy

---

## 1. Pre-Deployment Checklist

```
Security Review:
□ Unit tests cover > 90% of lines
□ Integration tests for all user flows
□ Fuzz tests for core math functions
□ Invariant tests pass
□ Static analysis (Slither) - 0 high/critical issues
□ Internal security review complete
□ External audit complete (for >$1M TVL protocols)
□ Bug bounty program live
□ No TODO/FIXME in production code

Access Control:
□ All admin functions protected
□ No tx.origin usage
□ Owner/admin is multisig (not EOA)
□ Timelock on parameter changes
□ Emergency pause mechanism tested

Oracle Security:
□ Price feed staleness checks
□ Multiple oracle sources
□ Circuit breaker tested

Upgradability:
□ Storage layout verified (no collision)
□ Initialize function called once only
□ Upgrade tested on fork

Gas:
□ Reasonable gas limits estimated
□ Gas optimization for critical paths
□ L2 calldata costs considered

Documentation:
□ All public functions documented
□ Deployment addresses recorded
□ Admin procedures documented
□ Emergency procedures documented
```

---

## 2. Production Deploy Script

```typescript
// scripts/production-deploy.ts
import { ethers } from "hardhat";
import { verify } from "./verify";
import fs from "fs";
import path from "path";

interface DeploymentRecord {
  network: string;
  chainId: number;
  timestamp: string;
  contracts: {
    [name: string]: {
      address: string;
      txHash: string;
      blockNumber: number;
      args: any[];
    };
  };
}

async function main() {
  const [deployer] = await ethers.getSigners();
  const network = await ethers.provider.getNetwork();
  
  console.log("=".repeat(50));
  console.log("Production Deployment");
  console.log("=".repeat(50));
  console.log(`Network: ${network.name} (${network.chainId})`);
  console.log(`Deployer: ${deployer.address}`);
  
  const balance = await deployer.provider.getBalance(deployer.address);
  console.log(`Balance: ${ethers.formatEther(balance)} ETH`);
  
  if (balance < ethers.parseEther("0.1")) {
    throw new Error("Insufficient ETH for deployment");
  }
  
  const record: DeploymentRecord = {
    network: network.name,
    chainId: Number(network.chainId),
    timestamp: new Date().toISOString(),
    contracts: {},
  };
  
  // ===== Step 1: Deploy Implementation =====
  console.log("\nDeploying implementation...");
  const Impl = await ethers.getContractFactory("VaultV1");
  const impl = await Impl.deploy();
  await impl.waitForDeployment();
  const implAddress = await impl.getAddress();
  const implTx = impl.deploymentTransaction()!;
  
  record.contracts["VaultV1"] = {
    address: implAddress,
    txHash: implTx.hash,
    blockNumber: (await implTx.wait())!.blockNumber,
    args: [],
  };
  console.log(`Implementation: ${implAddress}`);
  
  // ===== Step 2: Deploy Proxy =====
  console.log("\nDeploying proxy...");
  const MULTISIG = process.env.MULTISIG_ADDRESS!;
  const initData = Impl.interface.encodeFunctionData("initialize", [MULTISIG]);
  
  const Proxy = await ethers.getContractFactory("ERC1967Proxy");
  const proxy = await Proxy.deploy(implAddress, initData);
  await proxy.waitForDeployment();
  const proxyAddress = await proxy.getAddress();
  const proxyTx = proxy.deploymentTransaction()!;
  
  record.contracts["VaultProxy"] = {
    address: proxyAddress,
    txHash: proxyTx.hash,
    blockNumber: (await proxyTx.wait())!.blockNumber,
    args: [implAddress, initData],
  };
  console.log(`Proxy: ${proxyAddress}`);
  
  // ===== Step 3: Deploy Supporting Contracts =====
  console.log("\nDeploying rate limiter...");
  const RateLimiter = await ethers.getContractFactory("RateLimiter");
  const rateLimiter = await RateLimiter.deploy(proxyAddress);
  await rateLimiter.waitForDeployment();
  
  record.contracts["RateLimiter"] = {
    address: await rateLimiter.getAddress(),
    txHash: rateLimiter.deploymentTransaction()!.hash,
    blockNumber: 0,
    args: [proxyAddress],
  };
  
  // ===== Step 4: Configure =====
  console.log("\nConfiguring contracts...");
  const vault = await ethers.getContractAt("VaultV1", proxyAddress, deployer);
  
  await (await vault.setRateLimiter(await rateLimiter.getAddress())).wait();
  console.log("Rate limiter set");
  
  await (await vault.setFee(30)).wait(); // 0.3%
  console.log("Fee set");
  
  // ===== Step 5: Transfer Admin to Multisig =====
  console.log("\nTransferring admin to multisig...");
  await (await vault.transferOwnership(MULTISIG)).wait();
  console.log(`Admin transferred to: ${MULTISIG}`);
  
  // ===== Step 6: Verify on Etherscan =====
  if (process.env.ETHERSCAN_API_KEY) {
    console.log("\nVerifying contracts...");
    
    for (const [name, info] of Object.entries(record.contracts)) {
      try {
        await verify(info.address, info.args);
        console.log(`${name} verified`);
      } catch (err: any) {
        if (err.message.includes("Already Verified")) {
          console.log(`${name} already verified`);
        } else {
          console.error(`Failed to verify ${name}:`, err.message);
        }
      }
    }
  }
  
  // ===== Step 7: Save Record =====
  const recordPath = path.join(
    "deployments",
    `${network.name}-${Date.now()}.json`
  );
  
  fs.mkdirSync("deployments", { recursive: true });
  fs.writeFileSync(recordPath, JSON.stringify(record, null, 2));
  
  console.log("\n=".repeat(50));
  console.log("Deployment Complete!");
  console.log(`Record saved: ${recordPath}`);
  console.log("\nContract Addresses:");
  
  for (const [name, info] of Object.entries(record.contracts)) {
    console.log(`  ${name}: ${info.address}`);
  }
}

main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

---

## 3. Post-Deployment Validation

```typescript
// scripts/validate-deployment.ts
import { ethers } from "hardhat";

async function validate() {
  const vaultAddress = process.env.VAULT_ADDRESS!;
  const multisigAddress = process.env.MULTISIG_ADDRESS!;
  
  const vault = await ethers.getContractAt("VaultV1", vaultAddress);
  
  console.log("Running post-deployment validation...\n");
  
  const checks: { name: string; pass: boolean; value: string }[] = [];
  
  // 1. Ownership
  const owner = await vault.owner();
  checks.push({
    name: "Owner is multisig",
    pass: owner.toLowerCase() === multisigAddress.toLowerCase(),
    value: owner,
  });
  
  // 2. Not paused
  const paused = await vault.paused();
  checks.push({
    name: "Not paused",
    pass: !paused,
    value: paused.toString(),
  });
  
  // 3. Fee within bounds
  const fee = await vault.feeRate();
  checks.push({
    name: "Fee <= 1%",
    pass: fee <= 100n,
    value: `${fee}bps`,
  });
  
  // 4. Implementation is correct
  const implSlot = "0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc";
  const implRaw = await ethers.provider.getStorage(vaultAddress, implSlot);
  const impl = "0x" + implRaw.slice(-40);
  checks.push({
    name: "Implementation set",
    pass: impl !== "0x" + "0".repeat(40),
    value: impl,
  });
  
  // 5. Rate limiter configured
  const rateLimiter = await vault.rateLimiter();
  checks.push({
    name: "Rate limiter configured",
    pass: rateLimiter !== ethers.ZeroAddress,
    value: rateLimiter,
  });
  
  // Print results
  let allPassed = true;
  for (const check of checks) {
    const status = check.pass ? "✅" : "❌";
    console.log(`${status} ${check.name}: ${check.value}`);
    if (!check.pass) allPassed = false;
  }
  
  console.log("\n" + (allPassed ? "All checks passed!" : "SOME CHECKS FAILED!"));
  
  if (!allPassed) process.exit(1);
}

validate().catch(console.error);
```

---

## 4. Workshop: Full CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1
      
      - name: Run Forge Tests
        run: forge test -vvv --gas-report
      
      - name: Run Slither
        run: |
          pip install slither-analyzer
          slither . --config slither.config.json
      
      - name: Check Coverage
        run: |
          forge coverage --report summary
          # Fail if coverage < 90%

  audit-check:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Mythril
        run: |
          docker run mythril/myth analyze src/Vault.sol \
            --solv 0.8.24 \
            --execution-timeout 90
  
  deploy-staging:
    name: Deploy to Testnet
    needs: [test, audit-check]
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Sepolia
        env:
          PRIVATE_KEY: ${{ secrets.STAGING_PRIVATE_KEY }}
          SEPOLIA_RPC: ${{ secrets.SEPOLIA_RPC }}
        run: npx hardhat run scripts/production-deploy.ts --network sepolia
      
      - name: Validate Deployment
        run: npx hardhat run scripts/validate-deployment.ts --network sepolia
  
  deploy-production:
    name: Deploy to Mainnet
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production  # Requires manual approval
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Mainnet
        env:
          PRIVATE_KEY: ${{ secrets.PROD_PRIVATE_KEY }}
          MAINNET_RPC: ${{ secrets.MAINNET_RPC }}
          ETHERSCAN_API_KEY: ${{ secrets.ETHERSCAN_API_KEY }}
        run: npx hardhat run scripts/production-deploy.ts --network mainnet
      
      - name: Validate Mainnet Deployment
        run: npx hardhat run scripts/validate-deployment.ts --network mainnet
      
      - name: Start Monitoring
        run: npx ts-node monitoring/watcher.ts &
```

---

## สรุป Part 50

Production Deployment ที่เรียนรู้:
- ✅ Pre-deployment security checklist
- ✅ Deployment script with records
- ✅ Post-deployment validation
- ✅ CI/CD pipeline
- ✅ Etherscan verification

## ยินดีด้วย! จบ Section 5 (Parts 41-50)

สรุป Section ที่ผ่านมา:
- Section 1 (1-10): Blockchain Basics
- Section 2 (11-20): Intermediate Solidity
- Section 3 (21-30): Advanced DeFi
- Section 4 (31-40): Professional Patterns
- Section 5 (41-50): Full-Stack & Production

Section ถัดไป: Parts 51-60 - Protocol Architecture Patterns
