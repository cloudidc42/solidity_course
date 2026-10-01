# Part 23: Layer 2 Solutions

## สารบัญ
1. L2 Overview
2. Optimistic Rollups (Arbitrum/Optimism)
3. ZK Rollups (zkSync/Polygon zkEVM)
4. Cross-L2 Differences
5. Workshop: L2-Optimized Contract

---

## 1. L2 Overview

```
ปัญหา Ethereum L1:
- High gas fees (~$5-50 ต่อ transaction)
- Low throughput (~15-30 TPS)
- ไม่ suitable สำหรับ high-frequency apps

Layer 2 Solutions:
- Batch transactions → compress → submit to L1
- Security ยังคงมาจาก Ethereum (L1)
- ลด cost 10-100x, เพิ่ม TPS ขึ้นมาก

Types of L2:

1. Optimistic Rollups:
   - ส่ง state root ขึ้น L1 ทันที
   - "Optimistic": assume transactions are valid
   - มี 7-day challenge period
   - Fraud proofs: ถ้าพบ invalid state → challenge
   - Examples: Arbitrum, Optimism, Base, Blast

2. ZK Rollups:
   - ส่ง validity proof (ZK proof) ขึ้น L1
   - ไม่ต้อง challenge period
   - Instant finality
   - Computationally expensive to prove
   - Examples: zkSync Era, Starknet, Polygon zkEVM, Scroll

3. State Channels: สองฝ่ายคุยกันนอก chain (Lightning Network style)
4. Plasma: child chain ส่ง Merkle root ขึ้น L1 (deprecated)
5. Validium: ZK proof + off-chain data (faster, less secure)
```

---

## 2. Solidity บน Optimistic Rollups

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Arbitrum-specific features
 * 
 * Arbitrum Nitro:
 * - EVM compatible (ส่วนใหญ่ใช้ code เดิมได้)
 * - Precompiles เพิ่มเติมที่ address 0x64-0x70
 * - block.number = L2 block (เร็วกว่า L1)
 * - ArbGas: gas model ต่างจาก L1
 * 
 * ความแตกต่างสำคัญ:
 * - block.number เป็น L2 block (~250ms)
 * - L1 block number: ArbSys.arbBlockNumber() vs block.number
 * - tx.gasprice อาจแตกต่าง
 */

interface ArbSys {
    function arbBlockNumber() external view returns (uint256);
    function arbBlockHash(uint256 arbBlockNum) external view returns (bytes32);
    function arbChainID() external view returns (uint256);
}

contract ArbitrumAware {
    
    ArbSys constant ARB_SYS = ArbSys(address(0x64));
    
    // L2 block number (Arbitrum-specific)
    function getL2BlockNumber() public view returns (uint256) {
        return ARB_SYS.arbBlockNumber();
    }
    
    // Chain ID check
    function isArbitrum() public view returns (bool) {
        return ARB_SYS.arbChainID() == 42161; // Arbitrum One
    }
    
    // Regular block.number ใน Arbitrum = L2 block
    // ไม่เหมาะสำหรับ time-based logic ที่ต้อง sync กับ L1
    function getTimestamp() public view returns (uint256) {
        return block.timestamp; // L1-synced, safe to use
    }
}

/**
 * Optimism / Base specific
 * 
 * OP Stack (Optimism, Base, Blast):
 * - Bedrock upgrade: EVM compatible
 * - L1Block precompile: ข้อมูล L1 block ล่าสุด
 */
interface IOptimismL1Block {
    function number() external view returns (uint64);
    function timestamp() external view returns (uint64);
    function basefee() external view returns (uint256);
    function hash() external view returns (bytes32);
    function sequenceNumber() external view returns (uint64);
}

contract OptimismAware {
    
    IOptimismL1Block constant L1_BLOCK = IOptimismL1Block(0x4200000000000000000000000000000000000015);
    
    function getL1BlockNumber() public view returns (uint64) {
        return L1_BLOCK.number();
    }
    
    function getL1Timestamp() public view returns (uint64) {
        return L1_BLOCK.timestamp();
    }
    
    function getL1BaseFee() public view returns (uint256) {
        return L1_BLOCK.basefee();
    }
}
```

---

## 3. L1 → L2 Message Passing

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Arbitrum Inbox: ส่ง message จาก L1 → L2
 * 
 * createRetryableTicket: สร้าง retryable ticket
 * - ถ้า auto-redeem fail → user redeem เองได้ภายใน 7 วัน
 * - เหมาะสำหรับ bridge tokens
 */
interface IInbox {
    function createRetryableTicket(
        address to,                  // L2 destination
        uint256 l2CallValue,         // ETH value ใน L2
        uint256 maxSubmissionCost,   // L1 submission cost
        address excessFeeRefundAddress,
        address callValueRefundAddress,
        uint256 gasLimit,
        uint256 maxFeePerGas,
        bytes calldata data
    ) external payable returns (uint256);
}

contract L1ToL2Bridge {
    
    IInbox public immutable inbox;
    address public l2Target; // contract ใน L2 ที่จะรับ message
    
    event MessageSent(
        address indexed sender,
        address indexed l2Target,
        uint256 ticketId
    );
    
    constructor(address _inbox, address _l2Target) {
        inbox = IInbox(_inbox);
        l2Target = _l2Target;
    }
    
    function sendMessage(
        bytes calldata data,
        uint256 maxSubmissionCost,
        uint256 gasLimit,
        uint256 maxFeePerGas
    ) external payable returns (uint256 ticketId) {
        ticketId = inbox.createRetryableTicket{value: msg.value}(
            l2Target,
            0, // no ETH value in L2 call
            maxSubmissionCost,
            msg.sender, // refund to sender if excess
            msg.sender,
            gasLimit,
            maxFeePerGas,
            data
        );
        
        emit MessageSent(msg.sender, l2Target, ticketId);
    }
}

/**
 * L2 → L1 Message (Withdrawals)
 * 
 * Optimistic Rollup:
 * 1. ส่ง withdraw message ใน L2
 * 2. รอ 7 วัน challenge period
 * 3. Prove + Finalize ใน L1
 */
interface IL2CrossDomainMessenger {
    function sendMessage(
        address _target,
        bytes calldata _message,
        uint32 _minGasLimit
    ) external payable;
}

contract L2Messenger {
    
    IL2CrossDomainMessenger public immutable messenger;
    
    constructor(address _messenger) {
        messenger = IL2CrossDomainMessenger(_messenger);
    }
    
    // ส่ง message กลับ L1
    function withdrawToL1(
        address l1Contract,
        bytes calldata message,
        uint32 gasLimit
    ) external {
        messenger.sendMessage(l1Contract, message, gasLimit);
    }
}
```

---

## 4. Gas Optimization บน L2

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * L2 Gas ต่างจาก L1:
 * 
 * Total Cost = L2 Execution Gas + L1 Data Fee
 * 
 * L1 Data Fee = calldata bytes × L1 base price × overhead
 * 
 * ดังนั้น optimization บน L2:
 * 1. ลด calldata bytes (สำคัญมาก!)
 * 2. Calldata compression (EIP-2028: 0 bytes = 4 gas, non-zero = 16 gas)
 * 3. Batch operations
 * 4. Use events (cheap) แทน storage (แพง)
 */
contract L2OptimizedToken {
    
    // Pack อย่างแน่นที่สุด
    // address = 20 bytes, uint96 = 12 bytes → 1 slot
    mapping(address => uint96) private _balances;
    mapping(address => mapping(address => uint96)) private _allowances;
    
    uint96 private _totalSupply;
    
    // ใช้ bytes32 แทน string (ประหยัด calldata)
    bytes32 public immutable name32;
    bytes32 public immutable symbol32;
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    constructor(bytes32 _name, bytes32 _symbol, uint96 initialSupply) {
        name32 = _name;
        symbol32 = _symbol;
        _balances[msg.sender] = initialSupply;
        _totalSupply = initialSupply;
    }
    
    function name() external view returns (string memory) {
        return _bytes32ToString(name32);
    }
    
    function symbol() external view returns (string memory) {
        return _bytes32ToString(symbol32);
    }
    
    function totalSupply() external view returns (uint256) {
        return _totalSupply;
    }
    
    function balanceOf(address account) external view returns (uint256) {
        return _balances[account];
    }
    
    // ใช้ uint96 internally → ประหยัด storage
    function transfer(address to, uint96 amount) external returns (bool) {
        _balances[msg.sender] -= amount;
        _balances[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }
    
    function _bytes32ToString(bytes32 b) internal pure returns (string memory) {
        uint256 len = 0;
        while (len < 32 && b[len] != 0) len++;
        bytes memory result = new bytes(len);
        for (uint256 i = 0; i < len; i++) {
            result[i] = b[i];
        }
        return string(result);
    }
    
    // Batch transfer (ลด L1 data fee รวม operations)
    function batchTransfer(
        address[] calldata recipients,
        uint96[] calldata amounts
    ) external {
        require(recipients.length == amounts.length, "Length mismatch");
        
        uint96 totalAmount = 0;
        for (uint256 i = 0; i < amounts.length; i++) {
            unchecked { totalAmount += amounts[i]; }
        }
        
        _balances[msg.sender] -= totalAmount;
        
        for (uint256 i = 0; i < recipients.length; i++) {
            unchecked { _balances[recipients[i]] += amounts[i]; }
            emit Transfer(msg.sender, recipients[i], amounts[i]);
        }
    }
}
```

---

## 5. ZK-Rollup Considerations

```
ZK-Rollup differences:
1. Account abstraction native (zkSync Era)
2. Paymaster: ช่วย sponsor gas fees
3. EIP-4337 built-in (AA)
4. Different precompiles (ECMUL, ECADD ราคาถูกกว่า)

zkSync Era specific:
- Storage pricing ต่างจาก Arbitrum
- Pubdata pricing: สำคัญมาก (calldata + state diff)
- Contract size limit ต่างกัน

ข้อควรระวัง:
- ecrecover ใช้ได้ใน zkSync แต่ต้องผ่าน AA
- PUSH0 opcode (EIP-3855) อาจไม่รองรับใน zkEVM บางตัว
- Precompiles (SHA256, RIPEMD) ต่างกัน

Best Practice:
1. ทดสอบบน L2 testnet เสมอก่อน mainnet
2. ตรวจสอบ EVM equivalence ของแต่ละ L2
3. ใช้ Hardhat + hardhat-zksync-deploy สำหรับ zkSync
4. Foundry รองรับ Arbitrum/Optimism ผ่าน --fork-url
```

---

## 6. Workshop: Multi-Chain Deploy Script

```typescript
// scripts/deploy-multichain.ts
import { ethers } from "hardhat";
import { HardhatRuntimeEnvironment } from "hardhat/types";

interface NetworkConfig {
    name: string;
    chainId: number;
    rpcUrl: string;
    isL2: boolean;
}

const NETWORKS: NetworkConfig[] = [
    { name: "Ethereum", chainId: 1, rpcUrl: process.env.ETH_RPC!, isL2: false },
    { name: "Arbitrum One", chainId: 42161, rpcUrl: process.env.ARB_RPC!, isL2: true },
    { name: "Optimism", chainId: 10, rpcUrl: process.env.OP_RPC!, isL2: true },
    { name: "Base", chainId: 8453, rpcUrl: process.env.BASE_RPC!, isL2: true },
];

async function deployToNetwork(config: NetworkConfig) {
    console.log(`\nDeploying to ${config.name} (chainId: ${config.chainId})`);
    
    const provider = new ethers.JsonRpcProvider(config.rpcUrl);
    const wallet = new ethers.Wallet(process.env.PRIVATE_KEY!, provider);
    
    const balance = await provider.getBalance(wallet.address);
    console.log(`Balance: ${ethers.formatEther(balance)} ETH`);
    
    // L2 ใช้ gas price ต่ำกว่ามาก
    const feeData = await provider.getFeeData();
    console.log(`Gas price: ${ethers.formatUnits(feeData.gasPrice || 0n, "gwei")} gwei`);
    
    // Deploy contract (same bytecode, different chain)
    const factory = new ethers.ContractFactory(
        TOKEN_ABI,
        TOKEN_BYTECODE,
        wallet
    );
    
    const contract = await factory.deploy(
        ethers.encodeBytes32String("MyToken"),
        ethers.encodeBytes32String("MTK"),
        ethers.parseEther("1000000")
    );
    
    await contract.waitForDeployment();
    
    const address = await contract.getAddress();
    console.log(`${config.name}: ${address}`);
    
    return { network: config.name, chainId: config.chainId, address };
}

async function main() {
    const deployments = [];
    
    for (const network of NETWORKS) {
        try {
            const result = await deployToNetwork(network);
            deployments.push(result);
        } catch (err) {
            console.error(`Failed to deploy to ${network.name}:`, err);
        }
    }
    
    console.log("\n=== Deployment Summary ===");
    for (const d of deployments) {
        console.log(`${d.network} (${d.chainId}): ${d.address}`);
    }
}

const TOKEN_ABI = []; // ใส่ ABI จริง
const TOKEN_BYTECODE = ""; // ใส่ bytecode จริง

main().catch(console.error);
```

---

## สรุป Part 23

Layer 2 ที่เรียนรู้:
- ✅ Optimistic vs ZK Rollup
- ✅ Arbitrum/Optimism precompiles
- ✅ L1→L2 message passing
- ✅ L2 gas optimization (calldata reduction)
- ✅ Multi-chain deployment

## Quiz

1. Optimistic Rollup vs ZK Rollup: trade-off อะไร?
2. ทำไม calldata optimization ถึงสำคัญกว่าบน L2?
3. `block.number` บน Arbitrum ต่างจาก L1 อย่างไร?
4. Challenge period 7 วันเพื่ออะไร?

---

## Next: Part 24 - Cross-chain Bridges
