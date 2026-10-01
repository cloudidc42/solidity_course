# Part 10: Contract Deployment และ Verification

## สารบัญ
1. Deployment Basics
2. Constructor Arguments
3. Deterministic Deployment (CREATE2)
4. Factory Patterns
5. Proxy Patterns (พื้นฐาน)
6. Contract Verification
7. Hardhat Deploy Script
8. Foundry Deployment
9. Workshop: Token Factory

---

## 1. Deployment Basics

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// Contract ที่ deploy contract อื่น
contract Deployer {
    
    event ContractDeployed(address indexed deployed, bytes32 indexed salt);
    
    // Deploy ด้วย CREATE (address = keccak256(deployer, nonce))
    function deployBasic(bytes memory bytecode) external returns (address deployed) {
        assembly {
            deployed := create(0, add(bytecode, 0x20), mload(bytecode))
        }
        require(deployed != address(0), "Deploy failed");
    }
    
    // Deploy ด้วย CREATE2 (deterministic)
    // address = keccak256(0xff, deployer, salt, keccak256(bytecode))
    function deployWithSalt(bytes memory bytecode, bytes32 salt) 
        external 
        returns (address deployed) 
    {
        assembly {
            deployed := create2(0, add(bytecode, 0x20), mload(bytecode), salt)
        }
        require(deployed != address(0), "Deploy failed");
        emit ContractDeployed(deployed, salt);
    }
    
    // คำนวณ address ก่อน deploy
    function computeAddress(bytes memory bytecode, bytes32 salt) 
        public view returns (address) 
    {
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
}
```

### Transaction Lifecycle การ Deploy

```
1. สร้าง transaction:
   - to: address(0) หรือ "" (empty)
   - data: contract bytecode + constructor args
   - value: ETH ส่ง constructor (ถ้ามี)

2. EVM execute:
   - สร้าง empty account ที่ address ใหม่
   - run init code (constructor)
   - เก็บ return data เป็น contract code

3. Contract address คำนวณจาก:
   CREATE:  keccak256(rlp([sender, nonce]))[12:]
   CREATE2: keccak256(0xff + sender + salt + keccak256(bytecode))[12:]
```

---

## 2. Constructor Arguments

```solidity
// Constructor รับ arguments
contract TokenWithArgs {
    
    string public name;
    string public symbol;
    uint8 public decimals;
    uint256 public totalSupply;
    address public owner;
    
    mapping(address => uint256) public balanceOf;
    
    constructor(
        string memory _name,
        string memory _symbol,
        uint8 _decimals,
        uint256 _initialSupply,
        address _owner
    ) {
        require(_owner != address(0), "Zero owner");
        require(_initialSupply > 0, "Zero supply");
        
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
        owner = _owner;
        
        totalSupply = _initialSupply * (10 ** uint256(_decimals));
        balanceOf[_owner] = totalSupply;
    }
}

// ABI Encoding ของ constructor args
// ถ้า deploy จาก script:
// bytes memory constructorArgs = abi.encode(
//     "MyToken", "MTK", uint8(18), uint256(1000000), ownerAddress
// );
// bytes memory bytecode = abi.encodePacked(type(TokenWithArgs).creationCode, constructorArgs);
```

---

## 3. Factory Pattern

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface IToken {
    function name() external view returns (string memory);
    function symbol() external view returns (string memory);
    function totalSupply() external view returns (uint256);
    function balanceOf(address) external view returns (uint256);
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
    function approve(address, uint256) external returns (bool);
}

contract SimpleToken {
    
    string public name;
    string public symbol;
    uint8 public constant decimals = 18;
    uint256 public totalSupply;
    address public owner;
    
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    constructor(
        string memory _name,
        string memory _symbol,
        uint256 _supply,
        address _owner
    ) {
        name = _name;
        symbol = _symbol;
        owner = _owner;
        totalSupply = _supply * 1e18;
        balanceOf[_owner] = totalSupply;
        emit Transfer(address(0), _owner, totalSupply);
    }
    
    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount, "Insufficient");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }
    
    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(allowance[from][msg.sender] >= amount, "Not allowed");
        require(balanceOf[from] >= amount, "Insufficient");
        allowance[from][msg.sender] -= amount;
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        emit Transfer(from, to, amount);
        return true;
    }
}

// Factory Contract
contract TokenFactory {
    
    struct TokenInfo {
        address tokenAddress;
        string name;
        string symbol;
        uint256 totalSupply;
        address creator;
        uint256 createdAt;
    }
    
    TokenInfo[] public tokens;
    mapping(address => address[]) public creatorTokens;
    mapping(address => bool) public isFactoryToken;
    
    uint256 public deploymentFee = 0.001 ether;
    address public feeTo;
    
    event TokenCreated(
        address indexed token,
        address indexed creator,
        string name,
        string symbol,
        uint256 totalSupply
    );
    
    error InsufficientFee(uint256 required, uint256 provided);
    error ZeroAddress();
    
    constructor(address _feeTo) {
        if (_feeTo == address(0)) revert ZeroAddress();
        feeTo = _feeTo;
    }
    
    function createToken(
        string calldata name,
        string calldata symbol,
        uint256 supply
    ) external payable returns (address token) {
        if (msg.value < deploymentFee) {
            revert InsufficientFee(deploymentFee, msg.value);
        }
        
        // Deploy new token
        token = address(new SimpleToken(name, symbol, supply, msg.sender));
        
        // Record
        tokens.push(TokenInfo({
            tokenAddress: token,
            name: name,
            symbol: symbol,
            totalSupply: supply * 1e18,
            creator: msg.sender,
            createdAt: block.timestamp
        }));
        
        creatorTokens[msg.sender].push(token);
        isFactoryToken[token] = true;
        
        // Send fee
        if (deploymentFee > 0) {
            (bool sent,) = feeTo.call{value: deploymentFee}("");
            require(sent, "Fee transfer failed");
        }
        
        // Refund excess
        uint256 excess = msg.value - deploymentFee;
        if (excess > 0) {
            (bool refunded,) = msg.sender.call{value: excess}("");
            require(refunded, "Refund failed");
        }
        
        emit TokenCreated(token, msg.sender, name, symbol, supply);
    }
    
    // CREATE2 Factory - deterministic addresses
    function createTokenDeterministic(
        string calldata name,
        string calldata symbol,
        uint256 supply,
        bytes32 salt
    ) external payable returns (address token) {
        if (msg.value < deploymentFee) {
            revert InsufficientFee(deploymentFee, msg.value);
        }
        
        bytes memory bytecode = abi.encodePacked(
            type(SimpleToken).creationCode,
            abi.encode(name, symbol, supply, msg.sender)
        );
        
        assembly {
            token := create2(0, add(bytecode, 0x20), mload(bytecode), salt)
        }
        require(token != address(0), "Deploy failed");
        
        tokens.push(TokenInfo({
            tokenAddress: token,
            name: name,
            symbol: symbol,
            totalSupply: supply * 1e18,
            creator: msg.sender,
            createdAt: block.timestamp
        }));
        
        creatorTokens[msg.sender].push(token);
        isFactoryToken[token] = true;
        
        if (deploymentFee > 0) {
            (bool sent,) = feeTo.call{value: deploymentFee}("");
            require(sent, "Fee transfer failed");
        }
        
        emit TokenCreated(token, msg.sender, name, symbol, supply);
    }
    
    function computeTokenAddress(
        string calldata name,
        string calldata symbol,
        uint256 supply,
        address creator,
        bytes32 salt
    ) external view returns (address) {
        bytes memory bytecode = abi.encodePacked(
            type(SimpleToken).creationCode,
            abi.encode(name, symbol, supply, creator)
        );
        
        return address(uint160(uint256(keccak256(
            abi.encodePacked(bytes1(0xff), address(this), salt, keccak256(bytecode))
        ))));
    }
    
    function getTokenCount() external view returns (uint256) {
        return tokens.length;
    }
    
    function getCreatorTokens(address creator) external view returns (address[] memory) {
        return creatorTokens[creator];
    }
    
    function getTokenInfo(uint256 index) external view returns (TokenInfo memory) {
        require(index < tokens.length, "Out of bounds");
        return tokens[index];
    }
}
```

---

## 4. Proxy Pattern (Minimal Proxy - EIP-1167)

```solidity
// Minimal Proxy (Clone) - deploy ราคาถูก
// Bytecode ขนาด 45 bytes แทน full contract

library Clones {
    
    function clone(address implementation) internal returns (address instance) {
        assembly {
            // EIP-1167 minimal proxy bytecode
            mstore(0x00, 0x3d602d80600a3d3981f3363d3d373d3d3d363d73)
            mstore(0x14, shl(96, implementation))
            mstore(0x28, 0x5af43d82803e903d91602b57fd5bf3ff)
            instance := create(0, 0x0c, 0x37)
        }
        require(instance != address(0), "Clone failed");
    }
    
    function cloneDeterministic(address implementation, bytes32 salt) 
        internal returns (address instance) 
    {
        assembly {
            mstore(0x00, 0x3d602d80600a3d3981f3363d3d373d3d3d363d73)
            mstore(0x14, shl(96, implementation))
            mstore(0x28, 0x5af43d82803e903d91602b57fd5bf3ff)
            instance := create2(0, 0x0c, 0x37, salt)
        }
        require(instance != address(0), "Clone failed");
    }
    
    function predictDeterministicAddress(
        address implementation,
        bytes32 salt,
        address deployer
    ) internal pure returns (address predicted) {
        assembly {
            let ptr := mload(0x40)
            mstore(add(ptr, 0x38), deployer)
            mstore(add(ptr, 0x24), 0x5af43d82803e903d91602b57fd5bf3ff)
            mstore(add(ptr, 0x14), implementation)
            mstore(ptr, 0x3d602d80600a3d3981f3363d3d373d3d3d363d73)
            mstore(add(ptr, 0x58), salt)
            mstore(add(ptr, 0x78), keccak256(add(ptr, 0x0c), 0x37))
            predicted := keccak256(add(ptr, 0x43), 0x55)
        }
    }
}

// Implementation Contract (Logic)
contract TokenImplementation {
    
    // Storage slots ต้อง match กับ proxy
    string public name;
    string public symbol;
    uint256 public totalSupply;
    address public owner;
    bool private _initialized;
    
    mapping(address => uint256) public balanceOf;
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    // ใช้ initialize แทน constructor (เพราะ proxy)
    function initialize(
        string calldata _name,
        string calldata _symbol,
        uint256 _supply,
        address _owner
    ) external {
        require(!_initialized, "Already initialized");
        _initialized = true;
        
        name = _name;
        symbol = _symbol;
        owner = _owner;
        totalSupply = _supply * 1e18;
        balanceOf[_owner] = totalSupply;
        
        emit Transfer(address(0), _owner, totalSupply);
    }
    
    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount, "Insufficient");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }
    
    function mint(address to, uint256 amount) external onlyOwner {
        totalSupply += amount;
        balanceOf[to] += amount;
        emit Transfer(address(0), to, amount);
    }
}

// Factory ใช้ Clones (ถูกกว่า deploy ใหม่ ~10x)
contract CloneFactory {
    
    using Clones for address;
    
    address public immutable implementation;
    
    address[] public deployedTokens;
    
    event TokenCloned(address indexed clone, address indexed creator);
    
    constructor() {
        implementation = address(new TokenImplementation());
    }
    
    function createToken(
        string calldata name,
        string calldata symbol,
        uint256 supply
    ) external returns (address token) {
        token = implementation.clone();
        TokenImplementation(token).initialize(name, symbol, supply, msg.sender);
        
        deployedTokens.push(token);
        emit TokenCloned(token, msg.sender);
    }
    
    function createTokenDeterministic(
        string calldata name,
        string calldata symbol,
        uint256 supply,
        bytes32 salt
    ) external returns (address token) {
        token = implementation.cloneDeterministic(salt);
        TokenImplementation(token).initialize(name, symbol, supply, msg.sender);
        
        deployedTokens.push(token);
        emit TokenCloned(token, msg.sender);
    }
    
    function predictAddress(bytes32 salt) external view returns (address) {
        return Clones.predictDeterministicAddress(implementation, salt, address(this));
    }
}
```

---

## 5. Hardhat Deploy Script

```typescript
// scripts/deploy.ts
import { ethers, network } from "hardhat";
import { writeFileSync, readFileSync, existsSync } from "fs";
import path from "path";

interface DeploymentInfo {
  network: string;
  chainId: number;
  deployer: string;
  timestamp: string;
  contracts: {
    [key: string]: {
      address: string;
      txHash: string;
      blockNumber: number;
      constructorArgs?: any[];
    };
  };
}

async function main() {
  const [deployer] = await ethers.getSigners();
  const chainId = (await ethers.provider.getNetwork()).chainId;
  
  console.log(`Deploying on network: ${network.name} (chainId: ${chainId})`);
  console.log(`Deployer: ${deployer.address}`);
  
  const balance = await ethers.provider.getBalance(deployer.address);
  console.log(`Balance: ${ethers.formatEther(balance)} ETH`);
  
  // Load existing deployments (ถ้ามี)
  const deploymentFile = `deployments/${network.name}.json`;
  let deployment: DeploymentInfo = existsSync(deploymentFile)
    ? JSON.parse(readFileSync(deploymentFile, "utf8"))
    : {
        network: network.name,
        chainId: Number(chainId),
        deployer: deployer.address,
        timestamp: new Date().toISOString(),
        contracts: {},
      };
  
  // === Deploy TokenFactory ===
  console.log("\n1. Deploying TokenFactory...");
  
  const feeRecipient = deployer.address; // ปรับตามต้องการ
  
  const TokenFactory = await ethers.getContractFactory("TokenFactory");
  const tokenFactory = await TokenFactory.deploy(feeRecipient);
  await tokenFactory.waitForDeployment();
  
  const factoryAddress = await tokenFactory.getAddress();
  const factoryTx = tokenFactory.deploymentTransaction();
  
  console.log(`TokenFactory deployed at: ${factoryAddress}`);
  
  deployment.contracts.TokenFactory = {
    address: factoryAddress,
    txHash: factoryTx!.hash,
    blockNumber: factoryTx!.blockNumber!,
    constructorArgs: [feeRecipient],
  };
  
  // === Deploy CloneFactory ===
  console.log("\n2. Deploying CloneFactory...");
  
  const CloneFactory = await ethers.getContractFactory("CloneFactory");
  const cloneFactory = await CloneFactory.deploy();
  await cloneFactory.waitForDeployment();
  
  const cloneFactoryAddress = await cloneFactory.getAddress();
  const cloneFactoryTx = cloneFactory.deploymentTransaction();
  
  console.log(`CloneFactory deployed at: ${cloneFactoryAddress}`);
  
  deployment.contracts.CloneFactory = {
    address: cloneFactoryAddress,
    txHash: cloneFactoryTx!.hash,
    blockNumber: cloneFactoryTx!.blockNumber!,
  };
  
  // === Save deployments ===
  deployment.timestamp = new Date().toISOString();
  writeFileSync(deploymentFile, JSON.stringify(deployment, null, 2));
  console.log(`\nDeployment saved to ${deploymentFile}`);
  
  // === Verify contracts (mainnet/testnet) ===
  if (chainId !== 31337n) {
    console.log("\nWaiting for block confirmations before verification...");
    await tokenFactory.deploymentTransaction()?.wait(5);
    
    console.log("Verifying TokenFactory...");
    try {
      const { run } = await import("hardhat");
      await run("verify:verify", {
        address: factoryAddress,
        constructorArguments: [feeRecipient],
      });
      console.log("TokenFactory verified!");
    } catch (e: any) {
      if (e.message.includes("Already Verified")) {
        console.log("Already verified.");
      } else {
        console.error("Verification failed:", e.message);
      }
    }
  }
  
  console.log("\n=== Deployment Summary ===");
  console.log(`Network: ${network.name}`);
  console.log(`TokenFactory: ${factoryAddress}`);
  console.log(`CloneFactory: ${cloneFactoryAddress}`);
}

main().catch((error) => {
  console.error(error);
  process.exit(1);
});
```

---

## 6. Foundry Deployment Script

```solidity
// script/Deploy.s.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";
import "../src/TokenFactory.sol";
import "../src/CloneFactory.sol";

contract DeployScript is Script {
    
    function setUp() public {}
    
    function run() public {
        // ดึง private key จาก env
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        address deployer = vm.addr(deployerPrivateKey);
        
        console.log("Deploying on chain:", block.chainid);
        console.log("Deployer:", deployer);
        console.log("Balance:", deployer.balance);
        
        vm.startBroadcast(deployerPrivateKey);
        
        // Deploy TokenFactory
        TokenFactory tokenFactory = new TokenFactory(deployer);
        console.log("TokenFactory:", address(tokenFactory));
        
        // Deploy CloneFactory
        CloneFactory cloneFactory = new CloneFactory();
        console.log("CloneFactory:", address(cloneFactory));
        
        // Test: สร้าง token ตัวอย่าง
        tokenFactory.createToken{value: 0.001 ether}(
            "Test Token",
            "TEST",
            1000000
        );
        
        vm.stopBroadcast();
    }
}

// Deploy command:
// forge script script/Deploy.s.sol:DeployScript --rpc-url $RPC_URL --broadcast --verify

// Dry run (simulate):
// forge script script/Deploy.s.sol:DeployScript --rpc-url $RPC_URL

// Local:
// forge script script/Deploy.s.sol:DeployScript --rpc-url http://localhost:8545 --broadcast
```

---

## 7. Contract Verification

```bash
# Hardhat verify (Etherscan)
npx hardhat verify --network sepolia 0xYOUR_CONTRACT_ADDRESS "arg1" "arg2"

# ถ้า constructor args ซับซ้อน - ใช้ file
# verify-args.js:
# module.exports = ["My Token", "MTK", 1000000, "0x..."];
npx hardhat verify --network sepolia \
  --constructor-args verify-args.js \
  0xYOUR_CONTRACT_ADDRESS

# Foundry verify
forge verify-contract \
  --chain-id 11155111 \
  --num-of-optimizations 200 \
  --constructor-args $(cast abi-encode "constructor(address)" 0xFEE_RECIPIENT) \
  0xYOUR_CONTRACT_ADDRESS \
  src/TokenFactory.sol:TokenFactory \
  $ETHERSCAN_API_KEY

# ตรวจสอบ status
forge verify-check --chain-id 11155111 GUID $ETHERSCAN_API_KEY

# hardhat.config.ts - เพิ่ม etherscan config
import { HardhatUserConfig } from "hardhat/config";

const config: HardhatUserConfig = {
  // ...
  etherscan: {
    apiKey: {
      mainnet: process.env.ETHERSCAN_API_KEY!,
      sepolia: process.env.ETHERSCAN_API_KEY!,
      polygon: process.env.POLYGONSCAN_API_KEY!,
      arbitrumOne: process.env.ARBISCAN_API_KEY!,
      optimisticEthereum: process.env.OPTIMISM_API_KEY!,
      base: process.env.BASESCAN_API_KEY!,
    },
    customChains: [
      {
        network: "base",
        chainId: 8453,
        urls: {
          apiURL: "https://api.basescan.org/api",
          browserURL: "https://basescan.org",
        },
      },
    ],
  },
};
```

---

## 8. TypeScript Test สำหรับ Factory

```typescript
// test/TokenFactory.test.ts
import { expect } from "chai";
import { ethers } from "hardhat";
import { TokenFactory, SimpleToken } from "../typechain-types";
import { Signer } from "ethers";

describe("TokenFactory", function () {
  let factory: TokenFactory;
  let owner: Signer;
  let user1: Signer;
  let user2: Signer;
  
  const DEPLOYMENT_FEE = ethers.parseEther("0.001");
  
  beforeEach(async function () {
    [owner, user1, user2] = await ethers.getSigners();
    
    const Factory = await ethers.getContractFactory("TokenFactory");
    factory = await Factory.deploy(await owner.getAddress());
    await factory.waitForDeployment();
  });
  
  describe("createToken", function () {
    it("should create token with correct parameters", async function () {
      const tx = await factory.connect(user1).createToken(
        "My Token", "MTK", 1000000,
        { value: DEPLOYMENT_FEE }
      );
      
      const receipt = await tx.wait();
      
      // ดึง event
      const event = receipt?.logs.find(
        log => factory.interface.parseLog(log as any)?.name === "TokenCreated"
      );
      const parsed = factory.interface.parseLog(event as any);
      
      const tokenAddress = parsed?.args.token;
      expect(tokenAddress).to.not.equal(ethers.ZeroAddress);
      
      // ตรวจสอบ token
      const token = await ethers.getContractAt("SimpleToken", tokenAddress);
      expect(await token.name()).to.equal("My Token");
      expect(await token.symbol()).to.equal("MTK");
      expect(await token.owner()).to.equal(await user1.getAddress());
    });
    
    it("should revert with insufficient fee", async function () {
      await expect(
        factory.connect(user1).createToken(
          "My Token", "MTK", 1000000,
          { value: ethers.parseEther("0.0001") }
        )
      ).to.be.revertedWithCustomError(factory, "InsufficientFee");
    });
    
    it("should collect deployment fee", async function () {
      const ownerBalanceBefore = await ethers.provider.getBalance(
        await owner.getAddress()
      );
      
      await factory.connect(user1).createToken(
        "My Token", "MTK", 1000000,
        { value: DEPLOYMENT_FEE }
      );
      
      const ownerBalanceAfter = await ethers.provider.getBalance(
        await owner.getAddress()
      );
      
      expect(ownerBalanceAfter - ownerBalanceBefore).to.equal(DEPLOYMENT_FEE);
    });
    
    it("should refund excess ETH", async function () {
      const excess = ethers.parseEther("0.005");
      const sent = DEPLOYMENT_FEE + excess;
      
      const user1Before = await ethers.provider.getBalance(await user1.getAddress());
      
      const tx = await factory.connect(user1).createToken(
        "My Token", "MTK", 1000000,
        { value: sent }
      );
      const receipt = await tx.wait();
      const gasUsed = receipt!.gasUsed * receipt!.gasPrice;
      
      const user1After = await ethers.provider.getBalance(await user1.getAddress());
      
      // user1 จ่าย fee + gas เท่านั้น (ได้ excess คืน)
      expect(user1Before - user1After).to.be.closeTo(
        DEPLOYMENT_FEE + gasUsed,
        ethers.parseEther("0.0001") // tolerance
      );
    });
    
    it("should track creator tokens", async function () {
      const user1Addr = await user1.getAddress();
      
      await factory.connect(user1).createToken("T1", "T1", 1000, { value: DEPLOYMENT_FEE });
      await factory.connect(user1).createToken("T2", "T2", 2000, { value: DEPLOYMENT_FEE });
      await factory.connect(user2).createToken("T3", "T3", 3000, { value: DEPLOYMENT_FEE });
      
      const user1Tokens = await factory.getCreatorTokens(user1Addr);
      expect(user1Tokens.length).to.equal(2);
      
      const user2Tokens = await factory.getCreatorTokens(await user2.getAddress());
      expect(user2Tokens.length).to.equal(1);
      
      expect(await factory.getTokenCount()).to.equal(3);
    });
  });
  
  describe("createTokenDeterministic", function () {
    it("should deploy to predictable address", async function () {
      const salt = ethers.keccak256(ethers.toUtf8Bytes("my-unique-salt"));
      const user1Addr = await user1.getAddress();
      
      // คำนวณ address ก่อน deploy
      const predicted = await factory.computeTokenAddress(
        "DeterToken", "DTK", 1000000, user1Addr, salt
      );
      
      // Deploy
      const tx = await factory.connect(user1).createTokenDeterministic(
        "DeterToken", "DTK", 1000000, salt,
        { value: DEPLOYMENT_FEE }
      );
      const receipt = await tx.wait();
      
      // ดึง actual address จาก event
      const event = receipt?.logs.find(
        log => factory.interface.parseLog(log as any)?.name === "TokenCreated"
      );
      const parsed = factory.interface.parseLog(event as any);
      const actual = parsed?.args.token;
      
      expect(actual).to.equal(predicted);
    });
  });
});
```

---

## สรุป Part 10

Contract Deployment ที่เรียนรู้:
- ✅ CREATE vs CREATE2
- ✅ Constructor arguments
- ✅ Factory Pattern
- ✅ Clone/Minimal Proxy (EIP-1167)
- ✅ Hardhat deploy script
- ✅ Foundry deployment script
- ✅ Contract verification (Etherscan)

## Quiz

1. CREATE2 มีประโยชน์อะไรกว่า CREATE?
2. Minimal Proxy ประหยัด gas ได้อย่างไร?
3. ทำไม Proxy contracts ถึงต้องใช้ `initialize()` แทน `constructor()`?
4. `keccak256(0xff || deployer || salt || keccak256(bytecode))` คือสูตรของอะไร?

---

## Next: Part 11 - ERC-20 Token Standard
