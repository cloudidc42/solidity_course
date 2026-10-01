# Part 01: พื้นฐาน Blockchain และ Ethereum

## สารบัญ
1. Blockchain คืออะไร?
2. Bitcoin vs Ethereum
3. Ethereum Virtual Machine (EVM)
4. Smart Contracts คืออะไร?
5. Gas และ Transaction Fees
6. Accounts ใน Ethereum
7. Transaction Lifecycle
8. Consensus Mechanisms
9. Testnets และ Mainnets
10. Workshop: ใช้งาน MetaMask ครั้งแรก

---

## 1. Blockchain คืออะไร?

Blockchain คือ **ฐานข้อมูลแบบกระจายศูนย์** (Distributed Database) ที่จัดเก็บข้อมูลในรูปแบบของ "บล็อก" ที่เชื่อมต่อกันเป็นสายโซ่

### คุณสมบัติหลักของ Blockchain

```
┌─────────────────────────────────────────────────────────────────┐
│                    BLOCKCHAIN PROPERTIES                         │
├─────────────────┬───────────────────────────────────────────────┤
│ Decentralized   │ ไม่มีศูนย์กลาง ทุกคนมีสำเนาข้อมูล          │
│ Immutable       │ เปลี่ยนแปลงข้อมูลเก่าไม่ได้                  │
│ Transparent     │ ทุกคนสามารถตรวจสอบได้                        │
│ Trustless       │ ไม่ต้องเชื่อใจกัน ใช้ Code แทน              │
│ Censorship-free │ ไม่มีใครสั่งปิดได้                           │
└─────────────────┴───────────────────────────────────────────────┘
```

### โครงสร้างของ Block

```
┌─────────────────────────────────────────────┐
│                   BLOCK #100                 │
├─────────────────────────────────────────────┤
│  Previous Hash: 0x1a2b3c4d...               │
│  Timestamp: 2024-01-15 10:30:00             │
│  Nonce: 48291                               │
├─────────────────────────────────────────────┤
│  TRANSACTIONS:                              │
│  ┌─────────────────────────────────────┐   │
│  │ TX1: Alice → Bob: 1.5 ETH           │   │
│  │ TX2: Bob → Carol: 0.5 ETH           │   │
│  │ TX3: Deploy Contract 0xABCD...      │   │
│  └─────────────────────────────────────┘   │
├─────────────────────────────────────────────┤
│  Hash: 0x5e6f7g8h...                        │
└─────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────┐
│                   BLOCK #101                 │
│  Previous Hash: 0x5e6f7g8h...  ←── ชี้ไป   │
│  ...                                        │
└─────────────────────────────────────────────┘
```

### ทำไม Hash สำคัญมาก?

```python
# ตัวอย่างการทำงานของ Hash (แนวคิด)
block_100 = {
    "transactions": ["Alice→Bob: 1 ETH", "Bob→Carol: 0.5 ETH"],
    "prev_hash": "0xABC123...",
    "nonce": 48291
}

# ถ้าใครเปลี่ยน Transaction เก่า
block_100_tampered = {
    "transactions": ["Alice→Bob: 100 ETH"],  # แก้ไข
    "prev_hash": "0xABC123...",
    "nonce": 48291
}

# Hash จะเปลี่ยนทันที และทำให้ Block ถัดไปทั้งหมดไม่ valid!
# นี่คือเหตุผลที่ Blockchain ปลอดภัย
```

---

## 2. Bitcoin vs Ethereum

### เปรียบเทียบ

| ฟีเจอร์ | Bitcoin | Ethereum |
|---------|---------|----------|
| เปิดตัว | 2009 | 2015 |
| ผู้สร้าง | Satoshi Nakamoto | Vitalik Buterin |
| วัตถุประสงค์หลัก | Digital Gold / Store of Value | World Computer |
| Smart Contracts | ไม่มี (Script จำกัด) | มี (Turing Complete) |
| Block Time | ~10 นาที | ~12 วินาที |
| Consensus | Proof of Work | Proof of Stake (หลัง Merge) |
| Supply | จำกัด 21 ล้าน BTC | ไม่จำกัดแต่มี issuance rate |
| Programming | Bitcoin Script | Solidity, Vyper |

### Ethereum's Vision

```
Ethereum = World Computer

Traditional Computer:          Ethereum:
┌─────────────────────┐        ┌─────────────────────────────┐
│   Your Computer     │        │     Ethereum Network        │
│  ┌───────────────┐  │        │  ┌─────────────────────┐    │
│  │   Programs    │  │   →    │  │    Smart Contracts  │    │
│  │   (Local)     │  │        │  │    (Global, Always  │    │
│  └───────────────┘  │        │  │     Available)      │    │
│  ┌───────────────┐  │        │  └─────────────────────┘    │
│  │   Database    │  │        │  ┌─────────────────────┐    │
│  │   (Local)     │  │        │  │  Blockchain State   │    │
│  └───────────────┘  │        │  │  (Shared, Global)   │    │
└─────────────────────┘        │  └─────────────────────┘    │
                                └─────────────────────────────┘
```

---

## 3. Ethereum Virtual Machine (EVM)

EVM คือ **virtual machine** ที่รัน Smart Contract บน Ethereum

### EVM ทำงานอย่างไร?

```
Source Code (Solidity):
    contract Counter {
        uint public count = 0;
        function increment() public {
            count++;
        }
    }
          │
          ▼ (Solidity Compiler)
          
EVM Bytecode:
    6080604052348015600f57600080fd5b5060af8061001e6000396000f3fe...
          │
          ▼ (EVM Executes on All Nodes)
          
Nodes:
    ┌────────────┐  ┌────────────┐  ┌────────────┐
    │  Node 1    │  │  Node 2    │  │  Node 3    │
    │  EVM runs  │  │  EVM runs  │  │  EVM runs  │
    │  bytecode  │  │  bytecode  │  │  bytecode  │
    └────────────┘  └────────────┘  └────────────┘
    
    All nodes get the SAME result!
```

### EVM Stack Machine

EVM ใช้ **Stack-based architecture**:

```
Stack Operations:
PUSH1 0x05     →  Stack: [5]
PUSH1 0x03     →  Stack: [3, 5]
ADD            →  Stack: [8]        (3 + 5 = 8)
PUSH1 0x02     →  Stack: [2, 8]
MUL            →  Stack: [16]       (8 * 2 = 16)
```

### EVM Opcodes ที่สำคัญ

```
Category     | Opcode  | Description
─────────────┼─────────┼──────────────────────
Arithmetic   | ADD     | a + b
             | SUB     | a - b
             | MUL     | a * b
             | DIV     | a / b
─────────────┼─────────┼──────────────────────
Comparison   | LT      | a < b
             | GT      | a > b
             | EQ      | a == b
─────────────┼─────────┼──────────────────────
Storage      | SLOAD   | Load from storage
             | SSTORE  | Save to storage
─────────────┼─────────┼──────────────────────
Control      | JUMP    | Unconditional jump
             | JUMPI   | Conditional jump
─────────────┼─────────┼──────────────────────
System       | CALL    | Call another contract
             | RETURN  | Return data
             | REVERT  | Revert transaction
```

---

## 4. Smart Contracts คืออะไร?

### คำนิยาม

Smart Contract คือ **โปรแกรมที่รันบน Blockchain** โดยอัตโนมัติตามเงื่อนไขที่กำหนดไว้ล่วงหน้า

```
Traditional Contract:           Smart Contract:
┌─────────────────────┐         ┌─────────────────────────────┐
│  Written in Text    │         │  Written in Code (Solidity) │
│  Enforced by Law    │         │  Enforced by EVM            │
│  Need Lawyers       │         │  No Intermediaries          │
│  Can be Disputed    │         │  Code is Law                │
│  Slow/Expensive     │         │  Fast/Cheap(ish)            │
└─────────────────────┘         └─────────────────────────────┘
```

### ตัวอย่าง Smart Contract แรก

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

// นี่คือ Smart Contract ตัวแรกของเรา
// Simple Escrow: ระบบฝากเงินง่ายๆ

contract SimpleEscrow {
    address public buyer;    // ผู้ซื้อ
    address public seller;   // ผู้ขาย
    address public arbiter;  // คนกลาง
    uint public amount;      // จำนวนเงิน
    bool public released;    // เงินถูกปล่อยแล้วหรือยัง
    
    // Event เพื่อแจ้งเตือน
    event FundsDeposited(uint amount);
    event FundsReleased(address to, uint amount);
    
    constructor(address _seller, address _arbiter) {
        buyer = msg.sender;
        seller = _seller;
        arbiter = _arbiter;
    }
    
    // ผู้ซื้อฝากเงิน
    function deposit() external payable {
        require(msg.sender == buyer, "Only buyer can deposit");
        require(msg.value > 0, "Must deposit some ETH");
        amount = msg.value;
        emit FundsDeposited(msg.value);
    }
    
    // ปล่อยเงินให้ผู้ขาย
    function release() external {
        require(
            msg.sender == buyer || msg.sender == arbiter,
            "Not authorized"
        );
        require(!released, "Already released");
        require(amount > 0, "No funds to release");
        
        released = true;
        uint toRelease = amount;
        amount = 0;
        
        (bool success,) = seller.call{value: toRelease}("");
        require(success, "Transfer failed");
        
        emit FundsReleased(seller, toRelease);
    }
}
```

### ทำไม Smart Contract เปลี่ยนโลก?

```
Use Cases ของ Smart Contracts:

1. Finance (DeFi)
   - Lending & Borrowing (Aave, Compound)
   - Decentralized Exchanges (Uniswap)
   - Stablecoins (DAI)
   - Yield Farming

2. NFTs & Digital Ownership
   - Digital Art (Bored Ape, CryptoPunks)
   - Gaming Items
   - Music & Media Rights

3. Governance
   - DAO Voting
   - Protocol Upgrades
   - Treasury Management

4. Real World Assets
   - Property Tokenization
   - Invoice Financing
   - Commodity Trading

5. Infrastructure
   - Cross-chain Bridges
   - Identity Systems
   - Insurance
```

---

## 5. Gas และ Transaction Fees

### Gas คืออะไร?

Gas คือ **หน่วยวัดปริมาณการคำนวณ** ที่ใช้ในการรัน Operation บน EVM

```
Gas = Fuel for Ethereum

เหมือนรถยนต์:
- รถ = Transaction / Smart Contract
- น้ำมัน = Gas
- ถ้าน้ำมันหมดกลางทาง = Transaction Revert
```

### Gas Costs ของ Operations ต่างๆ

```
Operation          | Gas Cost  | Description
───────────────────┼───────────┼──────────────────────────
ADD                | 3         | 1 + 1
MUL                | 5         | 1 * 1
SLOAD              | 2,100     | Read from storage
SSTORE (new)       | 20,000    | Write new value to storage
SSTORE (existing)  | 2,900     | Update existing storage
Transfer ETH       | 21,000    | Basic ETH transfer
Deploy Contract    | ~100,000+ | Deploy new contract
```

### การคำนวณ Transaction Fee

```
Transaction Fee = Gas Used × Gas Price

ตัวอย่าง:
Gas Used = 21,000 (simple ETH transfer)
Gas Price = 20 Gwei (20 × 10⁻⁹ ETH)

Fee = 21,000 × 20 Gwei
    = 21,000 × 20 × 10⁻⁹ ETH
    = 0.00042 ETH
    ≈ $1.5 (ถ้า ETH = $3,500)
```

### EIP-1559: London Hard Fork (2021)

หลังจาก EIP-1559 โครงสร้าง Gas เปลี่ยนเป็น:

```
Total Fee = (Base Fee + Priority Fee) × Gas Used

Base Fee:
- กำหนดโดย Protocol อัตโนมัติ
- ถูก BURN ไม่ได้ให้ Miner/Validator

Priority Fee (Tip):
- ผู้ใช้กำหนดเอง
- ให้ Validator เป็น Incentive

Max Fee:
- ราคาสูงสุดที่ยอมจ่าย
- ถ้า Base Fee ต่ำกว่า → คืนส่วนต่าง
```

```javascript
// ตัวอย่างใน ethers.js
const tx = await contract.transfer(to, amount, {
    maxFeePerGas: ethers.parseUnits("100", "gwei"),  // max ที่ยอมจ่าย
    maxPriorityFeePerGas: ethers.parseUnits("2", "gwei"),  // tip
    gasLimit: 21000  // limit
});
```

---

## 6. Accounts ใน Ethereum

มี 2 ประเภทของ Account:

### Externally Owned Account (EOA)

```
EOA:
┌─────────────────────────────────────────────┐
│  Address: 0x742d35Cc6634C0532925a3b8D4C9...  │
│  ┌──────────────────────────────────────┐   │
│  │  Public Key  ←→  Private Key         │   │
│  │  (Address)        (กุญแจลับ)         │   │
│  └──────────────────────────────────────┘   │
│  Balance: 1.5 ETH                           │
│  Nonce: 42                                  │
└─────────────────────────────────────────────┘

- ควบคุมโดย Private Key
- สามารถส่ง Transaction ได้
- ไม่มี Code
- ตัวอย่าง: กระเป๋าเงิน MetaMask ของคุณ
```

### Contract Account

```
Contract Account:
┌─────────────────────────────────────────────┐
│  Address: 0xA0b86991c6218b36c1d19D4a2e9E...  │
│  ┌──────────────────────────────────────┐   │
│  │  Code (Bytecode)                     │   │
│  │  Storage (State Variables)           │   │
│  └──────────────────────────────────────┘   │
│  Balance: 1,000 ETH                         │
│  Nonce: 1 (deploy count)                    │
└─────────────────────────────────────────────┘

- ควบคุมโดย Code
- ไม่สามารถเริ่ม Transaction เองได้
- มี Code และ Storage
- ตัวอย่าง: Uniswap, USDC
```

### Account State

```
Account State ใน Ethereum:

{
    nonce: 42,              // จำนวน transactions ที่ส่งไป
    balance: 1500000000000000000,  // Wei (1.5 ETH)
    storageRoot: "0x...",  // Hash ของ Storage (Contract เท่านั้น)
    codeHash: "0x..."      // Hash ของ Bytecode (Contract เท่านั้น)
}
```

---

## 7. Transaction Lifecycle

### จาก User ถึง Blockchain

```
1. User สร้าง Transaction
   ┌────────────────────────────────────────┐
   │  from: 0xYourAddress                   │
   │  to: 0xContractAddress                 │
   │  value: 0 ETH                          │
   │  data: 0xa9059cbb...  (function call)  │
   │  gasLimit: 100000                      │
   │  maxFeePerGas: 50 gwei                 │
   │  nonce: 42                             │
   └────────────────────────────────────────┘
          │
          ▼ Sign ด้วย Private Key
          
2. Broadcast ไปยัง Network
   ┌────────────────────────────────────────┐
   │  Signed Transaction (with signature)   │
   │  r: 0x...                              │
   │  s: 0x...                              │
   │  v: 27 or 28                           │
   └────────────────────────────────────────┘
          │
          ▼ Propagate to Nodes
          
3. Mempool (รอคิว)
   ┌────────────────────────────────────────┐
   │  Pending Transactions Pool             │
   │  [TX1, TX2, TX3, YOUR_TX, TX5, ...]    │
   │  Validator เลือก TX ที่ Fee สูงสุดก่อน │
   └────────────────────────────────────────┘
          │
          ▼ Selected by Validator
          
4. Block Inclusion
   ┌────────────────────────────────────────┐
   │  BLOCK #19,234,521                     │
   │  [TX1, TX2, YOUR_TX, TX4]              │
   └────────────────────────────────────────┘
          │
          ▼ Finalized
          
5. Confirmation
   ✅ Transaction Confirmed!
   Hash: 0x8f3a...
   Block: #19,234,521
   Gas Used: 85,432
```

### Transaction Receipt

```javascript
// Transaction Receipt ตัวอย่าง
{
    transactionHash: "0x8f3a...",
    blockNumber: 19234521,
    blockHash: "0x1234...",
    from: "0xYourAddress",
    to: "0xContractAddress",
    gasUsed: 85432,
    effectiveGasPrice: 25000000000,  // 25 gwei
    status: 1,  // 1 = success, 0 = failed
    logs: [
        {
            address: "0xContractAddress",
            topics: ["0xTransfer(address,address,uint256)"],
            data: "0x..."
        }
    ]
}
```

---

## 8. Consensus Mechanisms

### Proof of Work (PoW) - Bitcoin ยังใช้

```
PoW Process:
1. Miners ต้องหาค่า Nonce ที่ทำให้ Hash ≤ Target
2. ต้องใช้พลังงานไฟฟ้ามหาศาล
3. ยิ่งแก้เร็ว ยิ่งมีโอกาสได้ Reward

Hash Puzzle:
SHA256(Block Header + Nonce) ≤ Target

ตัวอย่าง:
Target:   0x00000000FFFF...
Hash Try: 0x000000001234... ✅ (น้อยกว่า Target!)
```

### Proof of Stake (PoS) - Ethereum ใช้ตั้งแต่ The Merge 2022

```
PoS Process:
1. Validators stake 32 ETH เพื่อเข้าร่วม
2. ถูกสุ่มเลือกตาม Stake amount
3. Propose และ Attest Blocks
4. ถ้าโกง → Stake ถูก Slash (ถูกยึด)

Advantages over PoW:
- ใช้พลังงาน 99.95% น้อยลง
- Faster finality
- More decentralized potential
```

### Ethereum's Proof of Stake

```
Validator Lifecycle:

1. Deposit 32 ETH → Beacon Chain
   ┌─────────────────────────────┐
   │  Deposit Contract           │
   │  stake 32 ETH               │
   └─────────────────────────────┘
          │
          ▼ Activation Queue
          
2. Active Validator
   - รับ Block Proposals
   - Attest Blocks ของคนอื่น
   - รับ ~4-5% APY

3. Exit / Withdrawal
   - Voluntary Exit
   - Slash (if dishonest)
```

---

## 9. Testnets และ Mainnets

### ประเภท Networks

```
Ethereum Networks:

1. Mainnet
   Chain ID: 1
   ใช้งานจริง มีมูลค่า
   ETH มีราคาจริง

2. Testnets (สำหรับ Development):

   Sepolia:
   Chain ID: 11155111
   Most recommended testnet
   PoS consensus
   
   Holesky:
   Chain ID: 17000
   Large validator testing
   
   Goerli: (deprecated)
   Chain ID: 5
   

3. Local Development:
   
   Hardhat Network:
   Chain ID: 31337
   Instant mining
   Fork mainnet capability
   
   Anvil (Foundry):
   Chain ID: 31337 (default)
   Very fast
```

### Faucets - รับ Test ETH ฟรี

```
Testnet Faucets:

Sepolia:
- https://sepoliafaucet.com
- https://faucet.sepolia.dev
- Alchemy Sepolia Faucet

วิธีใช้:
1. เปิด MetaMask
2. Switch network → Sepolia
3. Copy address ของคุณ
4. ไปที่ Faucet website
5. Paste address
6. รอรับ Test ETH (0.1-0.5 ETH)
```

---

## 10. Workshop: ใช้งาน MetaMask ครั้งแรก

### ขั้นตอนที่ 1: ติดตั้ง MetaMask

1. ไปที่ https://metamask.io/download/
2. ติดตั้ง Extension สำหรับ Browser ของคุณ
3. คลิก "Create New Wallet"
4. ตั้ง Password ที่แข็งแรง

### ขั้นตอนที่ 2: Backup Secret Recovery Phrase

```
⚠️ สำคัญมาก! ⚠️

Secret Recovery Phrase (12-24 คำ):
"word1 word2 word3 word4 word5 word6
 word7 word8 word9 word10 word11 word12"

กฎที่ต้องปฏิบัติ:
✅ เขียนลงกระดาษ เก็บในที่ปลอดภัย
✅ เก็บหลายสำเนาในที่ต่างๆ
❌ อย่าบอกใคร
❌ อย่าถ่ายภาพหน้าจอ
❌ อย่าส่งทาง Email/Line
❌ อย่าเก็บใน Cloud

"Not your keys, not your coins"
```

### ขั้นตอนที่ 3: เพิ่ม Sepolia Testnet

```
วิธีเพิ่ม Network ใน MetaMask:

1. คลิกที่ Network dropdown (บนสุด)
2. คลิก "Add Network"
3. คลิก "Add a network manually"

กรอกข้อมูล:
Network Name: Sepolia Testnet
RPC URL: https://rpc.sepolia.org
Chain ID: 11155111
Currency Symbol: SepoliaETH
Block Explorer: https://sepolia.etherscan.io
```

### ขั้นตอนที่ 4: รับ Test ETH

```bash
# 1. Copy Ethereum Address จาก MetaMask
#    ตัวอย่าง: 0x742d35Cc6634C0532925a3b8D4C9...

# 2. ไปที่ Faucet
#    https://sepoliafaucet.com

# 3. Paste Address และ คลิก "Send ETH"

# 4. ตรวจสอบยอดใน MetaMask
#    ควรได้รับ 0.1 - 0.5 SepoliaETH
```

### ขั้นตอนที่ 5: ส่ง Transaction แรก

```
ส่ง Test ETH ให้ตัวเอง (ทดสอบ):

1. เปิด MetaMask
2. คลิก "Send"
3. ใส่ Address ปลายทาง (สร้าง Account ที่ 2)
4. ใส่จำนวน: 0.01 ETH
5. ตั้ง Gas: ใช้ค่า Default
6. คลิก "Confirm"
7. รอ ~12 วินาที
8. ✅ Transaction สำเร็จ!

ดู Transaction บน Etherscan:
https://sepolia.etherscan.io/tx/0x...
```

### ขั้นตอนที่ 6: อ่าน Transaction บน Etherscan

```
ข้อมูลใน Etherscan:

Transaction Hash: 0x8f3a4b2c...  ← ID ของ Transaction
Status: ✅ Success
Block: 5,234,521
Timestamp: 10 secs ago
From: 0xYourAddress
To: 0xRecipientAddress
Value: 0.01 ETH
Transaction Fee: 0.000441 ETH (21,000 Gas × 21 Gwei)
Gas Price: 21 Gwei
```

---

## สรุป Part 01

สิ่งที่เรียนรู้ในบทนี้:
- ✅ Blockchain คืออะไร และทำงานอย่างไร
- ✅ ความแตกต่างระหว่าง Bitcoin และ Ethereum
- ✅ EVM คืออะไรและทำงานอย่างไร
- ✅ Smart Contracts คืออะไร
- ✅ Gas และ Transaction Fees
- ✅ Account Types ใน Ethereum
- ✅ Transaction Lifecycle
- ✅ Consensus Mechanisms
- ✅ Networks ต่างๆ
- ✅ ใช้งาน MetaMask

## Quiz

1. ทำไม Blockchain จึง Immutable?
2. ความแตกต่างหลักของ EOA และ Contract Account คืออะไร?
3. Gas Price กับ Gas Limit ต่างกันอย่างไร?
4. ทำไม Ethereum จึงเปลี่ยนจาก PoW เป็น PoS?

## แบบฝึกหัด

1. ติดตั้ง MetaMask และสร้าง Wallet ใหม่
2. เพิ่ม Sepolia Testnet ใน MetaMask
3. รับ Test ETH จาก Faucet
4. ส่ง Transaction ทดสอบ
5. ดู Transaction บน Etherscan และอ่านข้อมูลทุกส่วน

---

## Next: Part 02 - การติดตั้งเครื่องมือพัฒนา

ใน Part ถัดไป เราจะติดตั้ง:
- Node.js
- Hardhat Framework
- VS Code + Extensions
- และเขียน Smart Contract แรก!
