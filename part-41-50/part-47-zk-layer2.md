# Part 47: ZK Rollups และ zkEVM

## สารบัญ
1. ZK Rollup Architecture
2. zkSync Era Development
3. Starknet (Cairo Basics)
4. Proof Systems Overview
5. Workshop: Cross-L2 Protocol

---

## 1. ZK Rollup Architecture

```
ZK Rollup คืออะไร:
- Execute transactions off-chain
- Generate cryptographic proof (ZK Proof)
- Submit proof + state root to L1
- Anyone can verify proof instantly

ต่างจาก Optimistic Rollup:
Optimistic:
- Assume valid unless challenged
- 7-day withdrawal period
- Gas: cheap for execution
- Security: economic (fraud proof)

ZK:
- Prove validity mathematically
- Instant finality on L1
- Gas: expensive to prove, but decreasing
- Security: cryptographic

ZK Proof Types:
1. zkSNARK: Succinct, fast verify, trusted setup
   - Used by: zkSync Lite, Loopring
2. zkSTARK: No trusted setup, quantum-resistant, bigger proof
   - Used by: Starknet
3. PLONK: Universal trusted setup, flexible
   - Used by: Aztec, Polygon zkEVM

zkEVM Types (EVM compatibility):
- Type 1: Fully EVM equivalent (Scroll)
- Type 2: EVM equivalent, different storage/hashes (zkSync)
- Type 3: Almost EVM equivalent
- Type 4: High-level language equiv (StarkNet/Cairo)

Current zkEVMs:
- zkSync Era: EVM-compatible, PLONK-based
- Polygon zkEVM: Near type 2
- Scroll: Type 1 (closest to EVM)
- StarkNet: Type 4 (Cairo, not EVM)
- Linea: Prover by ConsenSys
```

---

## 2. zkSync Era

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * zkSync Era ต่างจาก Ethereum:
 * 
 * 1. Native Account Abstraction (ทุก account เป็น smart contract)
 * 2. Paymaster (สามารถจ่าย gas ด้วย ERC-20)
 * 3. ECDSA + other signature types
 * 4. Some precompiles ต่างกัน
 * 5. msg.sender ใน constructor = address(0) ไม่ใช่ deployer!
 */

/**
 * zkSync Native AA - IAccount Interface
 * ทุก account ต้อง implement นี้
 */
interface IAccount {
    function validateTransaction(
        bytes32 txHash,
        bytes32 suggestedSignedHash,
        Transaction calldata transaction
    ) external payable returns (bytes4 magic);

    function executeTransaction(
        bytes32 txHash,
        bytes32 suggestedSignedHash,
        Transaction calldata transaction
    ) external payable;

    function executeTransactionFromOutside(
        Transaction calldata transaction
    ) external payable;

    function payForTransaction(
        bytes32 txHash,
        bytes32 suggestedSignedHash,
        Transaction calldata transaction
    ) external payable;

    function prepareForPaymaster(
        bytes32 txHash,
        bytes32 possibleSignedHash,
        Transaction calldata transaction
    ) external payable;
}

struct Transaction {
    uint256 txType;
    uint256 from;
    uint256 to;
    uint256 gasLimit;
    uint256 gasPerPubdataByteLimit;
    uint256 maxFeePerGas;
    uint256 maxPriorityFeePerGas;
    uint256 paymaster;
    uint256 nonce;
    uint256 value;
    uint256[4] reserved;
    bytes data;
    bytes signature;
    bytes32[] factoryDeps;
    bytes paymasterInput;
    bytes reservedDynamic;
}

/**
 * zkSync Paymaster:
 * ให้ users จ่าย gas ด้วย ERC-20 (เช่น USDC)
 */
interface IPaymaster {
    function validateAndPayForPaymasterTransaction(
        bytes32 txHash,
        bytes32 suggestedSignedHash,
        Transaction calldata transaction
    ) external payable returns (bytes4 magic, bytes memory context);
    
    function postTransaction(
        bytes calldata context,
        Transaction calldata transaction,
        bytes32 txHash,
        bytes32 suggestedSignedHash,
        ExecutionResult txResult,
        uint256 maxRefundedGas
    ) external payable;
}

enum ExecutionResult { Revert, Success }

/**
 * ERC-20 Paymaster for zkSync
 */
contract ERC20Paymaster is IPaymaster {
    
    address public immutable allowedToken; // USDC
    address public immutable priceOracle;
    
    uint256 public constant TOKEN_PER_ETH = 2000e6; // 2000 USDC = 1 ETH
    
    bytes4 constant PAYMASTER_VALIDATION_SUCCESS = 0x4b45faee;
    
    constructor(address _token, address _oracle) {
        allowedToken = _token;
        priceOracle = _oracle;
    }
    
    function validateAndPayForPaymasterTransaction(
        bytes32,
        bytes32,
        Transaction calldata transaction
    ) external payable override returns (bytes4 magic, bytes memory context) {
        require(msg.sender == 0x0000000000000000000000000000000000008001, "Not bootloader"); // zkSync bootloader
        
        // Extract paymaster input
        require(transaction.paymasterInput.length >= 4, "Invalid input");
        
        bytes4 paymasterInputSelector = bytes4(transaction.paymasterInput[:4]);
        require(
            paymasterInputSelector == bytes4(keccak256("approvalBased(address,uint256,bytes)")),
            "Unsupported flow"
        );
        
        (address token, uint256 minAllowance, bytes memory innerInput) = 
            abi.decode(transaction.paymasterInput[4:], (address, uint256, bytes));
        
        require(token == allowedToken, "Wrong token");
        
        // Calculate required USDC
        uint256 requiredETH = transaction.gasLimit * transaction.maxFeePerGas;
        uint256 requiredToken = requiredETH * TOKEN_PER_ETH / 1e18;
        
        require(minAllowance >= requiredToken, "Insufficient allowance");
        
        address user = address(uint160(transaction.from));
        
        // Take USDC from user
        IERC20(token).transferFrom(user, address(this), requiredToken);
        
        // Pay ETH to bootloader
        (bool success,) = payable(msg.sender).call{value: requiredETH}("");
        require(success, "Failed to pay bootloader");
        
        context = abi.encode(user, requiredToken, requiredETH);
        magic = PAYMASTER_VALIDATION_SUCCESS;
    }
    
    function postTransaction(
        bytes calldata context,
        Transaction calldata,
        bytes32, bytes32,
        ExecutionResult,
        uint256 maxRefundedGas
    ) external payable override {
        // Refund excess gas cost in USDC
        (address user,, uint256 paidETH) = abi.decode(context, (address, uint256, uint256));
        
        uint256 refundETH = maxRefundedGas * tx.gasprice;
        if (refundETH > 0) {
            uint256 refundToken = refundETH * TOKEN_PER_ETH / 1e18;
            IERC20(allowedToken).transfer(user, refundToken);
        }
    }
    
    receive() external payable {}
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
}
```

---

## 3. Starknet Basics (Cairo)

```
// Cairo (Starknet's language) - ไม่ใช่ Solidity
// แต่ concept เหมือนกัน

use starknet::ContractAddress;
use starknet::get_caller_address;

#[starknet::contract]
mod SimpleToken {
    use starknet::ContractAddress;
    use starknet::get_caller_address;
    
    #[storage]
    struct Storage {
        name: felt252,
        symbol: felt252,
        total_supply: u256,
        balances: LegacyMap::<ContractAddress, u256>,
        allowances: LegacyMap::<(ContractAddress, ContractAddress), u256>,
    }
    
    #[event]
    #[derive(Drop, starknet::Event)]
    enum Event {
        Transfer: Transfer,
        Approval: Approval,
    }
    
    #[derive(Drop, starknet::Event)]
    struct Transfer {
        from: ContractAddress,
        to: ContractAddress,
        value: u256,
    }
    
    #[constructor]
    fn constructor(ref self: ContractState, name: felt252, symbol: felt252, supply: u256) {
        self.name.write(name);
        self.symbol.write(symbol);
        self.total_supply.write(supply);
        let caller = get_caller_address();
        self.balances.write(caller, supply);
    }
    
    #[abi(embed_v0)]
    impl ITokenImpl of super::IToken<ContractState> {
        fn transfer(ref self: ContractState, to: ContractAddress, amount: u256) -> bool {
            let caller = get_caller_address();
            let from_balance = self.balances.read(caller);
            assert(from_balance >= amount, 'Insufficient balance');
            
            self.balances.write(caller, from_balance - amount);
            let to_balance = self.balances.read(to);
            self.balances.write(to, to_balance + amount);
            
            self.emit(Transfer { from: caller, to, value: amount });
            true
        }
    }
}
```

---

## 4. Workshop: L2 Deployment Script

```typescript
// scripts/deploy-multichain.ts
import { ethers } from "hardhat";
import { Provider, Wallet } from "zksync-ethers";

async function deployToZkSync() {
  const zkProvider = new Provider("https://mainnet.era.zksync.io");
  const wallet = new Wallet(process.env.PRIVATE_KEY!, zkProvider);
  
  console.log("Deploying to zkSync Era...");
  
  // zkSync requires different deployment
  const { Deployer } = await import("@matterlabs/hardhat-zksync-deploy");
  const deployer = new Deployer(hre, wallet);
  
  const artifact = await deployer.loadArtifact("MyToken");
  
  // Deploy with zkSync deployer
  const token = await deployer.deploy(artifact, ["My Token", "MTK"]);
  await token.waitForDeployment();
  
  console.log(`zkSync Token: ${await token.getAddress()}`);
}

async function deployToScroll() {
  const provider = new ethers.JsonRpcProvider("https://rpc.scroll.io");
  const wallet = new ethers.Wallet(process.env.PRIVATE_KEY!, provider);
  
  const factory = await ethers.getContractFactory("MyToken", wallet);
  const token = await factory.deploy("My Token", "MTK");
  await token.waitForDeployment();
  
  console.log(`Scroll Token: ${await token.getAddress()}`);
}

async function main() {
  await deployToZkSync();
  await deployToScroll();
}

main().catch(console.error);
```

---

## สรุป Part 47

ZK Layer 2 ที่เรียนรู้:
- ✅ ZK Rollup vs Optimistic Rollup
- ✅ zkEVM types (1-4)
- ✅ zkSync Era native AA
- ✅ ERC-20 Paymaster
- ✅ Cairo/Starknet basics
- ✅ Multi-chain deployment

## Next: Part 48 - EIP Standards Deep Dive
