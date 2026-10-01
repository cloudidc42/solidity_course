# Part 02: การติดตั้งเครื่องมือพัฒนา

## สารบัญ
1. Overview เครื่องมือที่ต้องใช้
2. ติดตั้ง Node.js
3. ติดตั้ง VS Code + Extensions
4. ติดตั้ง Hardhat
5. ติดตั้ง Foundry (Alternative)
6. โครงสร้างโปรเจกต์ Hardhat
7. เขียนและ Deploy Contract แรก
8. Remix IDE (Online Alternative)
9. Configuration Files
10. Workshop: Hello World Contract

---

## 1. Overview เครื่องมือที่ต้องใช้

```
Development Stack:

┌─────────────────────────────────────────────────────────────────┐
│                    Solidity Dev Stack                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Editor:                                                         │
│  ┌─────────────────┐                                            │
│  │   VS Code       │ ← IDE หลัก                                 │
│  │   + Solidity    │ ← Syntax highlighting, IntelliSense        │
│  │     Extension   │                                            │
│  └─────────────────┘                                            │
│                                                                  │
│  Framework (เลือก 1):                                            │
│  ┌─────────────────┐  ┌─────────────────┐                       │
│  │   Hardhat       │  │    Foundry      │                       │
│  │   (JavaScript)  │  │   (Rust-based)  │                       │
│  └─────────────────┘  └─────────────────┘                       │
│                                                                  │
│  Runtime:                                                        │
│  ┌─────────────────┐                                            │
│  │   Node.js 18+   │ ← ต้องการสำหรับ Hardhat                    │
│  └─────────────────┘                                            │
│                                                                  │
│  Wallet:                                                         │
│  ┌─────────────────┐                                            │
│  │   MetaMask      │ ← Browser extension                        │
│  └─────────────────┘                                            │
│                                                                  │
│  Optional:                                                       │
│  ┌─────────────────┐  ┌─────────────────┐                       │
│  │   Remix IDE     │  │   Tenderly      │                       │
│  │   (Online)      │  │   (Debugging)   │                       │
│  └─────────────────┘  └─────────────────┘                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. ติดตั้ง Node.js

### ตรวจสอบว่ามี Node.js อยู่แล้วหรือยัง

```bash
node --version    # ควรเป็น v18 ขึ้นไป
npm --version     # ควรเป็น v8 ขึ้นไป
```

### ติดตั้ง Node.js ผ่าน NVM (แนะนำ)

```bash
# ติดตั้ง NVM (Node Version Manager)
# macOS/Linux:
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# หรือ Homebrew (macOS):
brew install nvm

# Restart terminal หรือ:
source ~/.bashrc  # หรือ ~/.zshrc

# ติดตั้ง Node.js เวอร์ชันล่าสุด LTS
nvm install --lts
nvm use --lts

# ตรวจสอบ
node --version   # v20.x.x
npm --version    # 10.x.x
```

### ติดตั้ง Node.js บน Windows

```powershell
# วิธีที่ 1: ดาวน์โหลดจาก nodejs.org
# ไปที่ https://nodejs.org/en/download/
# ดาวน์โหลด LTS version
# ติดตั้งด้วย installer

# วิธีที่ 2: Chocolatey
choco install nodejs-lts

# วิธีที่ 3: winget
winget install OpenJS.NodeJS.LTS

# ตรวจสอบ
node --version
npm --version
```

### เกี่ยวกับ pnpm (แนะนำแทน npm)

```bash
# ติดตั้ง pnpm (เร็วกว่าและ efficient กว่า npm)
npm install -g pnpm

# ใช้งาน pnpm
pnpm install         # แทน npm install
pnpm add <package>   # แทน npm install <package>
pnpm run <script>    # แทน npm run <script>
```

---

## 3. ติดตั้ง VS Code + Extensions

### ดาวน์โหลด VS Code

```
ไปที่: https://code.visualstudio.com/
ดาวน์โหลดตาม OS ของคุณ
```

### Extensions ที่จำเป็น

```bash
# ติดตั้งผ่าน command line:
code --install-extension JuanBlanco.solidity
code --install-extension tintinweb.solidity-visual-developer
code --install-extension dbaeumer.vscode-eslint
code --install-extension esbenp.prettier-vscode
code --install-extension ms-vscode.vscode-typescript-next
code --install-extension GitHub.copilot
```

### Extensions List

```
Essential:
1. Solidity (Juan Blanco) - JuanBlanco.solidity
   - Syntax highlighting
   - Error detection
   - Code completion

2. Solidity Visual Developer - tintinweb.solidity-visual-developer
   - UML diagrams
   - Graph visualization
   - Enhanced IntelliSense

Recommended:
3. ESLint - dbaeumer.vscode-eslint
   - JavaScript/TypeScript linting

4. Prettier - esbenp.prettier-vscode
   - Code formatting

5. GitLens - eamodio.gitlens
   - Enhanced Git features

6. Error Lens - usernamehw.errorlens
   - Inline error display

7. Thunder Client - rangav.vscode-thunder-client
   - API testing (แทน Postman)
```

### VS Code Settings สำหรับ Solidity

สร้างไฟล์ `.vscode/settings.json` ในโปรเจกต์:

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "[solidity]": {
    "editor.defaultFormatter": "JuanBlanco.solidity"
  },
  "solidity.compileUsingRemoteVersion": "latest",
  "solidity.defaultCompiler": "remote",
  "editor.tabSize": 4,
  "editor.insertSpaces": true,
  "files.eol": "\n",
  "editor.rulers": [120],
  "solidity.linter": "solhint",
  "editor.inlineSuggest.enabled": true
}
```

---

## 4. ติดตั้ง Hardhat

Hardhat คือ **Development Environment** สำหรับ Ethereum ที่นิยมมากที่สุด

### สร้างโปรเจกต์ใหม่

```bash
# สร้างโฟลเดอร์โปรเจกต์
mkdir my-solidity-project
cd my-solidity-project

# Initialize npm project
npm init -y

# ติดตั้ง Hardhat
npm install --save-dev hardhat

# สร้าง Hardhat project
npx hardhat init
```

### เลือก Project Type

```
What do you want to do? …
❯ Create a JavaScript project
  Create a TypeScript project
  Create a TypeScript project (with Viem)
  Create an empty hardhat.config.js
  Quit
```

เลือก **Create a TypeScript project** (แนะนำ)

### ติดตั้ง Dependencies เพิ่มเติม

```bash
# สำหรับ TypeScript project
npm install --save-dev @nomicfoundation/hardhat-toolbox
npm install --save-dev @nomicfoundation/hardhat-ethers
npm install --save-dev ethers

# สำหรับ Testing
npm install --save-dev @nomicfoundation/hardhat-chai-matchers
npm install --save-dev @types/chai
npm install --save-dev @types/mocha

# OpenZeppelin Contracts (ใช้บ่อยมาก)
npm install @openzeppelin/contracts

# dotenv สำหรับ Environment Variables
npm install --save-dev dotenv
```

### package.json ที่สมบูรณ์

```json
{
  "name": "my-solidity-project",
  "version": "1.0.0",
  "description": "Solidity Smart Contracts Project",
  "scripts": {
    "compile": "hardhat compile",
    "test": "hardhat test",
    "test:coverage": "hardhat coverage",
    "deploy:local": "hardhat run scripts/deploy.ts --network localhost",
    "deploy:sepolia": "hardhat run scripts/deploy.ts --network sepolia",
    "node": "hardhat node",
    "clean": "hardhat clean",
    "typechain": "hardhat typechain"
  },
  "devDependencies": {
    "@nomicfoundation/hardhat-toolbox": "^4.0.0",
    "@types/chai": "^4.3.11",
    "@types/mocha": "^10.0.6",
    "@types/node": "^20.11.0",
    "hardhat": "^2.19.4",
    "ts-node": "^10.9.2",
    "typescript": "^5.3.3",
    "dotenv": "^16.4.0"
  },
  "dependencies": {
    "@openzeppelin/contracts": "^5.0.1",
    "ethers": "^6.9.2"
  }
}
```

---

## 5. โครงสร้างโปรเจกต์ Hardhat

```
my-solidity-project/
├── contracts/              ← Smart Contracts (.sol files)
│   ├── Token.sol
│   ├── NFT.sol
│   └── interfaces/
│       └── IToken.sol
├── scripts/                ← Deploy & Interaction Scripts
│   ├── deploy.ts
│   └── interact.ts
├── test/                   ← Test Files
│   ├── Token.test.ts
│   └── NFT.test.ts
├── ignition/               ← Hardhat Ignition (Deployment)
│   └── modules/
│       └── Token.ts
├── artifacts/              ← Compiled Contracts (auto-generated)
├── cache/                  ← Compilation Cache (auto-generated)
├── typechain-types/        ← TypeScript Types (auto-generated)
├── .env                    ← Environment Variables (อย่า commit!)
├── .env.example            ← Template for .env
├── .gitignore
├── hardhat.config.ts       ← Hardhat Configuration
├── package.json
└── tsconfig.json
```

### hardhat.config.ts

```typescript
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
import * as dotenv from "dotenv";

dotenv.config();

const PRIVATE_KEY = process.env.PRIVATE_KEY || "0x" + "0".repeat(64);
const SEPOLIA_RPC_URL = process.env.SEPOLIA_RPC_URL || "";
const ETHERSCAN_API_KEY = process.env.ETHERSCAN_API_KEY || "";
const ALCHEMY_API_KEY = process.env.ALCHEMY_API_KEY || "";

const config: HardhatUserConfig = {
  solidity: {
    version: "0.8.24",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200,
      },
      viaIR: true,  // เปิด IR-based compilation (ดีสำหรับ optimization)
    },
  },
  
  networks: {
    // Local development network
    hardhat: {
      chainId: 31337,
    },
    
    // Local node (hardhat node command)
    localhost: {
      url: "http://127.0.0.1:8545",
      chainId: 31337,
    },
    
    // Sepolia testnet
    sepolia: {
      url: `https://eth-sepolia.g.alchemy.com/v2/${ALCHEMY_API_KEY}`,
      accounts: [PRIVATE_KEY],
      chainId: 11155111,
    },
    
    // Ethereum mainnet (ระวัง!)
    mainnet: {
      url: `https://eth-mainnet.g.alchemy.com/v2/${ALCHEMY_API_KEY}`,
      accounts: [PRIVATE_KEY],
      chainId: 1,
    },
    
    // Polygon
    polygon: {
      url: `https://polygon-mainnet.g.alchemy.com/v2/${ALCHEMY_API_KEY}`,
      accounts: [PRIVATE_KEY],
      chainId: 137,
    },
    
    // Arbitrum
    arbitrum: {
      url: "https://arb1.arbitrum.io/rpc",
      accounts: [PRIVATE_KEY],
      chainId: 42161,
    },
  },
  
  etherscan: {
    apiKey: {
      mainnet: ETHERSCAN_API_KEY,
      sepolia: ETHERSCAN_API_KEY,
      polygon: process.env.POLYGONSCAN_API_KEY || "",
      arbitrumOne: process.env.ARBISCAN_API_KEY || "",
    },
  },
  
  gasReporter: {
    enabled: process.env.REPORT_GAS !== undefined,
    currency: "USD",
    gasPrice: 21,
  },
  
  paths: {
    sources: "./contracts",
    tests: "./test",
    cache: "./cache",
    artifacts: "./artifacts",
  },
};

export default config;
```

### .env ไฟล์

```bash
# .env (อย่า commit ไปที่ git!)
PRIVATE_KEY=0xYourPrivateKeyHere
SEPOLIA_RPC_URL=https://eth-sepolia.g.alchemy.com/v2/YourAlchemyKey
ALCHEMY_API_KEY=YourAlchemyAPIKey
ETHERSCAN_API_KEY=YourEtherscanAPIKey
POLYGONSCAN_API_KEY=YourPolygonscanAPIKey
ARBISCAN_API_KEY=YourArbiscanAPIKey
REPORT_GAS=true
```

```bash
# .env.example (commit นี้ได้)
PRIVATE_KEY=0x0000000000000000000000000000000000000000000000000000000000000000
SEPOLIA_RPC_URL=
ALCHEMY_API_KEY=
ETHERSCAN_API_KEY=
POLYGONSCAN_API_KEY=
ARBISCAN_API_KEY=
REPORT_GAS=
```

### .gitignore

```
node_modules/
artifacts/
cache/
.env
coverage/
coverage.json
typechain-types/
```

---

## 6. ติดตั้ง Foundry (Alternative Framework)

Foundry เป็น framework ที่เขียนด้วย Rust เร็วมากและนิยมมากขึ้นเรื่อยๆ

```bash
# ติดตั้ง Foundryup (Installer)
curl -L https://foundry.paradigm.xyz | bash

# Restart terminal แล้วรัน:
foundryup

# ตรวจสอบ
forge --version    # forge 0.2.0
cast --version     # cast 0.2.0
anvil --version    # anvil 0.2.0
chisel --version   # chisel 0.2.0
```

### เครื่องมือใน Foundry

```
Foundry Suite:

forge  → Build, test, deploy contracts
cast   → Interact with EVM (เหมือน ethers.js CLI)
anvil  → Local testnet (เหมือน hardhat node)
chisel → Solidity REPL (Interactive shell)
```

### สร้างโปรเจกต์ Foundry

```bash
# สร้างโปรเจกต์ใหม่
forge init my-foundry-project
cd my-foundry-project

# โครงสร้าง:
# src/        ← Smart Contracts
# test/       ← Tests (Solidity!)
# script/     ← Deploy Scripts
# lib/        ← Dependencies (git submodules)
# foundry.toml ← Configuration

# ติดตั้ง OpenZeppelin
forge install OpenZeppelin/openzeppelin-contracts
```

### foundry.toml

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
optimizer = true
optimizer_runs = 200
via_ir = true
solc_version = "0.8.24"

[rpc_endpoints]
sepolia = "${SEPOLIA_RPC_URL}"
mainnet = "${MAINNET_RPC_URL}"

[etherscan]
sepolia = { key = "${ETHERSCAN_API_KEY}" }
mainnet = { key = "${ETHERSCAN_API_KEY}" }

[fmt]
line_length = 120
tab_width = 4
bracket_spacing = true
```

### Hardhat vs Foundry

| Feature | Hardhat | Foundry |
|---------|---------|---------|
| Language | JavaScript/TypeScript | Rust |
| Test Language | JavaScript/TypeScript | Solidity |
| Speed | Moderate | Very Fast |
| Community | Larger | Growing |
| Fuzzing | Plugin needed | Built-in |
| Scripting | JS/TS | Solidity |
| Fork Support | Yes | Yes (faster) |
| Learning Curve | Medium | Medium |

**แนะนำ**: เรียนรู้ทั้งสอง แต่เริ่มต้นด้วย Hardhat เพราะ community ใหญ่กว่า

---

## 7. Workshop: Hello World Contract

### สร้าง Contract แรก

```bash
# สร้างโปรเจกต์ Hardhat
mkdir hello-solidity
cd hello-solidity
npm init -y
npm install --save-dev hardhat
npx hardhat init
# เลือก: Create a TypeScript project
```

### contracts/HelloWorld.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title HelloWorld
 * @dev Smart Contract แรกสำหรับการเรียนรู้
 * @author Solidity Course
 */
contract HelloWorld {
    // State Variables (เก็บบน Blockchain)
    string public greeting;
    address public owner;
    uint256 public greetingCount;
    
    // Events
    event GreetingChanged(
        address indexed changedBy,
        string oldGreeting,
        string newGreeting,
        uint256 timestamp
    );
    
    // Constructor - รันครั้งเดียวตอน Deploy
    constructor(string memory _initialGreeting) {
        greeting = _initialGreeting;
        owner = msg.sender;
        greetingCount = 0;
    }
    
    // Functions
    
    /**
     * @dev เปลี่ยน Greeting
     * @param _newGreeting ข้อความ Greeting ใหม่
     */
    function setGreeting(string memory _newGreeting) public {
        require(bytes(_newGreeting).length > 0, "Greeting cannot be empty");
        
        string memory oldGreeting = greeting;
        greeting = _newGreeting;
        greetingCount++;
        
        emit GreetingChanged(msg.sender, oldGreeting, _newGreeting, block.timestamp);
    }
    
    /**
     * @dev ดึง Greeting ปัจจุบัน
     * @return ข้อความ Greeting
     */
    function getGreeting() public view returns (string memory) {
        return greeting;
    }
    
    /**
     * @dev ดึงข้อมูลสรุป
     * @return _greeting, _owner, _count
     */
    function getInfo() public view returns (
        string memory _greeting,
        address _owner,
        uint256 _count
    ) {
        return (greeting, owner, greetingCount);
    }
}
```

### scripts/deploy.ts

```typescript
import { ethers } from "hardhat";

async function main() {
    console.log("🚀 Starting deployment...");
    
    // รับ Signer (Account ที่จะ Deploy)
    const [deployer] = await ethers.getSigners();
    console.log("📝 Deploying with account:", deployer.address);
    console.log("💰 Account balance:", ethers.formatEther(await deployer.provider.getBalance(deployer.address)));
    
    // Deploy Contract
    console.log("\n📦 Deploying HelloWorld...");
    const HelloWorld = await ethers.getContractFactory("HelloWorld");
    const helloWorld = await HelloWorld.deploy("Hello, Solidity World!");
    
    await helloWorld.waitForDeployment();
    
    const address = await helloWorld.getAddress();
    console.log("✅ HelloWorld deployed to:", address);
    
    // ทดสอบ Contract
    console.log("\n🧪 Testing contract...");
    
    const greeting = await helloWorld.getGreeting();
    console.log("📢 Initial greeting:", greeting);
    
    // เปลี่ยน Greeting
    const tx = await helloWorld.setGreeting("สวัสดี โลก Blockchain!");
    await tx.wait();
    console.log("✏️ Changed greeting successfully");
    
    const [newGreeting, owner, count] = await helloWorld.getInfo();
    console.log("📢 New greeting:", newGreeting);
    console.log("👤 Owner:", owner);
    console.log("🔢 Greeting count:", count.toString());
    
    console.log("\n🎉 Deployment complete!");
}

main()
    .then(() => process.exit(0))
    .catch((error) => {
        console.error(error);
        process.exit(1);
    });
```

### test/HelloWorld.test.ts

```typescript
import { expect } from "chai";
import { ethers } from "hardhat";
import { HelloWorld } from "../typechain-types";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";

describe("HelloWorld", function () {
    let helloWorld: HelloWorld;
    let owner: HardhatEthersSigner;
    let addr1: HardhatEthersSigner;
    
    // Deploy Contract ก่อนแต่ละ Test
    beforeEach(async function () {
        [owner, addr1] = await ethers.getSigners();
        
        const HelloWorldFactory = await ethers.getContractFactory("HelloWorld");
        helloWorld = await HelloWorldFactory.deploy("Hello, World!");
    });
    
    describe("Deployment", function () {
        it("Should set the initial greeting", async function () {
            expect(await helloWorld.getGreeting()).to.equal("Hello, World!");
        });
        
        it("Should set the correct owner", async function () {
            expect(await helloWorld.owner()).to.equal(owner.address);
        });
        
        it("Should start with 0 greeting count", async function () {
            expect(await helloWorld.greetingCount()).to.equal(0);
        });
    });
    
    describe("setGreeting", function () {
        it("Should update the greeting", async function () {
            await helloWorld.setGreeting("New Greeting");
            expect(await helloWorld.getGreeting()).to.equal("New Greeting");
        });
        
        it("Should increment greeting count", async function () {
            await helloWorld.setGreeting("New Greeting");
            expect(await helloWorld.greetingCount()).to.equal(1);
            
            await helloWorld.setGreeting("Another Greeting");
            expect(await helloWorld.greetingCount()).to.equal(2);
        });
        
        it("Should allow anyone to set greeting", async function () {
            await helloWorld.connect(addr1).setGreeting("From addr1");
            expect(await helloWorld.getGreeting()).to.equal("From addr1");
        });
        
        it("Should revert with empty greeting", async function () {
            await expect(
                helloWorld.setGreeting("")
            ).to.be.revertedWith("Greeting cannot be empty");
        });
        
        it("Should emit GreetingChanged event", async function () {
            await expect(helloWorld.setGreeting("New Greeting"))
                .to.emit(helloWorld, "GreetingChanged")
                .withArgs(
                    owner.address,
                    "Hello, World!",
                    "New Greeting",
                    // Timestamp - ใช้ anyValue เพราะไม่รู้ล่วงหน้า
                    await ethers.provider.getBlock("latest").then(b => b!.timestamp + 1)
                );
        });
    });
    
    describe("getInfo", function () {
        it("Should return correct info", async function () {
            const [greeting, contractOwner, count] = await helloWorld.getInfo();
            
            expect(greeting).to.equal("Hello, World!");
            expect(contractOwner).to.equal(owner.address);
            expect(count).to.equal(0);
        });
    });
});
```

### รันทดสอบ

```bash
# Compile
npx hardhat compile

# Run tests
npx hardhat test

# Run tests with gas report
REPORT_GAS=true npx hardhat test

# Run specific test file
npx hardhat test test/HelloWorld.test.ts

# Test coverage
npx hardhat coverage
```

### ผลลัพธ์ที่คาดหวัง

```
  HelloWorld
    Deployment
      ✔ Should set the initial greeting
      ✔ Should set the correct owner
      ✔ Should start with 0 greeting count
    setGreeting
      ✔ Should update the greeting
      ✔ Should increment greeting count
      ✔ Should allow anyone to set greeting
      ✔ Should revert with empty greeting
      ✔ Should emit GreetingChanged event
    getInfo
      ✔ Should return correct info

  9 passing (2s)
```

### Deploy บน Local Network

```bash
# Terminal 1: Start local node
npx hardhat node

# Terminal 2: Deploy
npx hardhat run scripts/deploy.ts --network localhost

# ผลลัพธ์:
# 🚀 Starting deployment...
# 📝 Deploying with account: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
# 💰 Account balance: 10000.0
# 📦 Deploying HelloWorld...
# ✅ HelloWorld deployed to: 0x5FbDB2315678afecb367f032d93F642f64180aa3
# 🧪 Testing contract...
# 📢 Initial greeting: Hello, Solidity World!
# ✏️ Changed greeting successfully
# 📢 New greeting: สวัสดี โลก Blockchain!
# 👤 Owner: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
# 🔢 Greeting count: 1
# 🎉 Deployment complete!
```

---

## 8. Remix IDE (Online Alternative)

Remix เป็น Online IDE ที่ไม่ต้องติดตั้งอะไร เหมาะสำหรับเริ่มต้น

```
เข้าถึงได้ที่: https://remix.ethereum.org
```

### วิธีใช้ Remix

```
1. เปิด https://remix.ethereum.org
2. กดที่ไอคอน "File Explorer" (ซ้ายบน)
3. คลิก "Create new file"
4. ตั้งชื่อ: HelloWorld.sol
5. Paste โค้ด HelloWorld.sol
6. กดที่ไอคอน "Solidity Compiler" (เครื่องหมาย ✓)
7. คลิก "Compile HelloWorld.sol"
8. กดที่ไอคอน "Deploy & Run Transactions"
9. Environment: "Remix VM (Cancun)"
10. คลิก "Deploy"
11. ทดสอบ Functions ใน Interface ที่ปรากฏ
```

### Remix Shortcuts

```
Ctrl+S    → Save & Compile
Ctrl+Z    → Undo
F5        → Compile
F10       → Deploy (ถ้า configured)
```

---

## 9. Configuration Files ทั้งหมด

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "moduleResolution": "node",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "outDir": "dist",
    "declaration": true,
    "skipLibCheck": true,
    "resolveJsonModule": true
  },
  "include": [
    "scripts/**/*",
    "test/**/*",
    "hardhat.config.ts"
  ],
  "files": [],
  "exclude": ["node_modules"]
}
```

### .solhint.json (Solidity Linter)

```bash
# ติดตั้ง solhint
npm install --save-dev solhint

# สร้าง config
npx solhint --init
```

```json
{
  "extends": "solhint:recommended",
  "plugins": [],
  "rules": {
    "avoid-suicide": "error",
    "avoid-sha3": "warn",
    "no-empty-blocks": "warn",
    "func-visibility": ["warn", {"ignoreConstructors": true}],
    "state-visibility": "warn",
    "var-name-mixedcase": "warn",
    "func-name-mixedcase": "warn",
    "reason-string": ["warn", {"maxLength": 64}],
    "no-unused-vars": "warn",
    "ordering": "warn",
    "compiler-version": ["error", "^0.8.0"],
    "max-line-length": ["warn", 120]
  }
}
```

### .prettierrc (Code Formatter)

```json
{
  "singleQuote": false,
  "semi": true,
  "tabWidth": 4,
  "printWidth": 120,
  "trailingComma": "es5",
  "overrides": [
    {
      "files": "*.sol",
      "options": {
        "printWidth": 120,
        "tabWidth": 4,
        "singleQuote": false,
        "bracketSpacing": false
      }
    }
  ]
}
```

---

## 10. RPC Providers - ไปต่อยังไง

### Alchemy (แนะนำ)

```
1. ไปที่ https://www.alchemy.com/
2. สมัครบัญชี (ฟรี)
3. Create App → Ethereum → Sepolia
4. Copy API Key
5. ใส่ใน .env
```

### Infura

```
1. ไปที่ https://infura.io/
2. สมัครบัญชี (ฟรี)
3. Create Project
4. Copy Project ID
5. RPC URL: https://sepolia.infura.io/v3/YOUR_PROJECT_ID
```

### Hardhat Network Built-in

สำหรับ Development ไม่ต้องใช้ RPC Provider ภายนอก:

```bash
# Start local node (มี 20 accounts พร้อม 10,000 ETH แต่ละอัน)
npx hardhat node

# Hardhat จะแสดง:
# Started HTTP and WebSocket JSON-RPC server at http://127.0.0.1:8545/
#
# Accounts
# ========
#
# WARNING: These accounts, and their private keys, are publicly known.
# Any funds sent to them on Mainnet or any other live network WILL BE LOST.
#
# Account #0: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 (10000 ETH)
# Private Key: 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
#
# ...
```

---

## สรุป Part 02

เครื่องมือที่ติดตั้งแล้ว:
- ✅ Node.js 18+ (ผ่าน NVM)
- ✅ VS Code + Solidity Extensions
- ✅ Hardhat Framework
- ✅ TypeScript Configuration
- ✅ OpenZeppelin Contracts
- ✅ Foundry (Alternative)

## Checklist

- [ ] ติดตั้ง Node.js v18+
- [ ] ติดตั้ง VS Code + Extensions
- [ ] สร้างโปรเจกต์ Hardhat
- [ ] เขียน HelloWorld.sol
- [ ] รัน Tests ผ่าน
- [ ] Deploy บน Local Network
- [ ] ทดสอบใน Remix

## แบบฝึกหัด

1. สร้าง Contract `SimpleStorage` ที่เก็บ number และมี `get()` และ `set()` functions
2. เพิ่ม Event ที่ emit ทุกครั้งที่ค่าเปลี่ยน
3. เขียน Test ครอบคลุม 100%
4. Deploy บน Sepolia testnet

---

## Next: Part 03 - Solidity Variables และ Types

ใน Part ถัดไป เราจะเรียนรู้:
- Value Types (uint, int, bool, address, bytes)
- Reference Types (arrays, mappings, structs)
- Special Variables (msg.sender, block.timestamp)
- Type Conversions
