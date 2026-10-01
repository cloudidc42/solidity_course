# Part 76: Layer 2 Optimization Techniques

## บทนำ

Layer 2 (L2) solutions เช่น Arbitrum, Optimism, zkSync และ Base ได้เปลี่ยนแปลงวิธีที่นักพัฒนา Solidity คิดเรื่อง gas optimization อย่างสิ้นเชิง บน L2 ต้นทุนหลักไม่ใช่ execution gas บน L2 เอง แต่เป็น **calldata cost** ที่ต้องส่งไปบน Ethereum L1 เพื่อความ availability

ในบทนี้เราจะเจาะลึก:
1. การทำงานของ calldata compression บน L2
2. Batch transaction patterns ด้วย Multicall
3. L2-specific gas oracle ของ Arbitrum และ Optimism
4. State channel basics สำหรับ off-chain transactions
5. Plasma-style merkle commitment ไปยัง L1

---

## 76.1 Calldata Compression: ต้นทุนที่แท้จริงบน L2

### ทำไม Calldata จึงสำคัญบน L2?

บน Ethereum mainnet (L1), ต้นทุน gas มาจาก:
- **Execution**: opcode computation
- **Storage**: SSTORE/SLOAD
- **Memory**: expansion costs

บน L2 ต้นทุนหลักแบ่งเป็น 2 ส่วน:
1. **L2 execution fee**: ถูกมาก (มักน้อยกว่า L1 100-1000x)
2. **L1 data fee**: ต้นทุนในการ post calldata ขึ้น L1 เพื่อ data availability

EIP-4844 (Proto-Danksharding) ลดต้นทุน L1 data ด้วย "blob transactions" แต่การ optimize calldata ยังคงสำคัญ

### การนับ Gas ของ Calldata

ก่อน EIP-4844:
- Zero byte: **4 gas** ต่อ byte
- Non-zero byte: **16 gas** ต่อ byte

ดังนั้น address `0x0000000000000000000000001234567890abcdef` (20 bytes)
= 15 zero bytes × 4 + 5 non-zero bytes × 16 = 60 + 80 = **140 gas**

เทียบกับ address เต็ม `0xAbCd...` = 20 × 16 = **320 gas**

### เทคนิค Calldata Packing

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title CalldataOptimizer
/// @notice แสดงเทคนิคการบีบอัด calldata สำหรับ L2
contract CalldataOptimizer {
    
    // ❌ วิธีปกติ: ส่ง params แยกกัน = calldata ใหญ่
    function transferUnoptimized(
        address to,
        uint256 amount,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external pure returns (bytes memory) {
        // ต้องการ calldata: 4 + 32 + 32 + 32 + 32 + 32 + 32 = 196 bytes
        return abi.encode(to, amount, deadline, v, r, s);
    }
    
    // ✅ วิธี optimized: pack หลาย params เข้า bytes เดียว
    // โดยใช้ smaller types และ tight packing
    function transferOptimized(bytes calldata packed) external pure returns (
        address to,
        uint96 amount,    // uint96 พอสำหรับ token amount (ลด 160 bits)
        uint32 deadline,  // unix timestamp พอใน uint32 จนถึงปี 2106
        uint8 v,
        bytes32 r,
        bytes32 s
    ) {
        // packed = [20 bytes addr][12 bytes amount][4 bytes deadline][1 byte v][32 bytes r][32 bytes s]
        // รวม = 101 bytes แทน 196 bytes = ประหยัด ~48%
        assembly {
            let ptr := packed.offset
            to := shr(96, calldataload(ptr))            // 20 bytes
            amount := and(calldataload(add(ptr, 20)), 0xFFFFFFFFFFFFFFFFFFFFFFFF)  // 12 bytes
            deadline := and(calldataload(add(ptr, 32)), 0xFFFFFFFF)  // 4 bytes
            v := byte(0, calldataload(add(ptr, 36)))    // 1 byte
            r := calldataload(add(ptr, 37))             // 32 bytes
            s := calldataload(add(ptr, 69))             // 32 bytes
        }
    }
    
    /// @notice เปรียบเทียบขนาด calldata
    function compareCalldataSizes() external pure returns (
        uint256 unoptimizedSize,
        uint256 optimizedSize,
        uint256 savings
    ) {
        unoptimizedSize = 4 + 32 + 32 + 32 + 32 + 32 + 32; // 196 bytes
        optimizedSize = 4 + 20 + 12 + 4 + 1 + 32 + 32;     // 105 bytes
        savings = unoptimizedSize - optimizedSize;           // 91 bytes saved
    }
}

/// @title AddressCompressor
/// @notice ใช้ index แทน full address เพื่อลด calldata
contract AddressCompressor {
    
    mapping(uint16 => address) public addressBook;
    mapping(address => uint16) public addressIndex;
    uint16 public nextIndex;
    
    event AddressRegistered(address indexed addr, uint16 index);
    
    /// @notice ลงทะเบียน address และรับ 2-byte index
    function registerAddress(address addr) external returns (uint16 index) {
        require(addressIndex[addr] == 0, "Already registered");
        index = ++nextIndex;
        addressBook[index] = addr;
        addressIndex[addr] = index;
        emit AddressRegistered(addr, index);
    }
    
    /// @notice ส่ง token โดยใช้ 2-byte index แทน 20-byte address
    /// ประหยัด 18 bytes ต่อ address = 18 × 16 = 288 gas
    function transferByIndex(
        uint16 toIndex,
        uint96 amount
    ) external view returns (address recipient, uint96 transferAmount) {
        recipient = addressBook[toIndex];
        require(recipient != address(0), "Unknown index");
        transferAmount = amount;
        // ทำ transfer จริงที่นี่
    }
    
    /// @notice Batch transfer ด้วย compressed calldata
    /// Format: [uint16 toIndex, uint96 amount] × N = 14 bytes per transfer
    function batchTransferCompressed(bytes calldata data) external pure returns (uint256 count) {
        // แต่ละ transfer = 14 bytes (2 + 12)
        require(data.length % 14 == 0, "Invalid data length");
        count = data.length / 14;
        
        for (uint256 i = 0; i < count; i++) {
            uint256 offset = i * 14;
            uint16 toIndex;
            uint96 amount;
            
            assembly {
                toIndex := shr(240, calldataload(add(data.offset, offset)))
                amount := and(
                    shr(128, calldataload(add(data.offset, add(offset, 2)))),
                    0xFFFFFFFFFFFFFFFFFFFFFFFF
                )
            }
            
            // ทำ transfer logic ที่นี่
            // addressBook[toIndex] รับ amount
            _ = toIndex; // suppress unused warning
            _ = amount;
        }
    }
}
```

---

## 76.2 Batch Transactions: Multicall Pattern

### ทำไมต้อง Multicall?

บน L2 แต่ละ transaction มี overhead ของ L1 data fee ดังนั้นการรวมหลาย operations เข้า transaction เดียวช่วยลดต้นทุนได้มาก

Multicall3 (deployed ที่ `0xcA11bde05977b3631167028862bE2a173976CA11` บนทุก chain) เป็น standard ที่ใช้กันทั่วไป

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title Multicall3
/// @notice Aggregate multiple calls into a single transaction
/// @dev Compatible with Multicall3 standard
contract Multicall3 {
    
    struct Call {
        address target;
        bytes callData;
    }
    
    struct Call3 {
        address target;
        bool allowFailure;
        bytes callData;
    }
    
    struct Call3Value {
        address target;
        bool allowFailure;
        uint256 value;
        bytes callData;
    }
    
    struct Result {
        bool success;
        bytes returnData;
    }
    
    /// @notice รวม calls หลายอัน ถ้าอันใด fail จะ revert ทั้งหมด
    function aggregate(Call[] calldata calls) 
        external 
        payable 
        returns (uint256 blockNumber, bytes[] memory returnData) 
    {
        blockNumber = block.number;
        uint256 length = calls.length;
        returnData = new bytes[](length);
        
        for (uint256 i = 0; i < length; i++) {
            (bool success, bytes memory ret) = calls[i].target.call(calls[i].callData);
            require(success, "Multicall3: call failed");
            returnData[i] = ret;
        }
    }
    
    /// @notice รวม calls โดยแต่ละอันสามารถ fail ได้
    function aggregate3(Call3[] calldata calls) 
        external 
        payable 
        returns (Result[] memory returnData) 
    {
        uint256 length = calls.length;
        returnData = new Result[](length);
        
        for (uint256 i = 0; i < length; i++) {
            Result memory result = returnData[i];
            (result.success, result.returnData) = calls[i].target.call(calls[i].callData);
            
            if (!result.success && !calls[i].allowFailure) {
                assembly {
                    revert(add(mload(add(result, 0x40)), 0x20), mload(mload(add(result, 0x40))))
                }
            }
        }
    }
    
    /// @notice รวม calls พร้อม ETH value
    function aggregate3Value(Call3Value[] calldata calls) 
        external 
        payable 
        returns (Result[] memory returnData) 
    {
        uint256 valAccumulator;
        uint256 length = calls.length;
        returnData = new Result[](length);
        
        for (uint256 i = 0; i < length; i++) {
            Result memory result = returnData[i];
            uint256 val = calls[i].value;
            
            // ตรวจสอบไม่ให้ overflow
            unchecked { valAccumulator += val; }
            require(valAccumulator <= msg.value, "Multicall3: insufficient value");
            
            (result.success, result.returnData) = calls[i].target.call{value: val}(calls[i].callData);
            
            if (!result.success && !calls[i].allowFailure) {
                assembly {
                    revert(add(mload(add(result, 0x40)), 0x20), mload(mload(add(result, 0x40))))
                }
            }
        }
        
        // คืนเงิน ETH ส่วนเกิน
        unchecked {
            uint256 excess = msg.value - valAccumulator;
            if (excess > 0) {
                (bool success,) = msg.sender.call{value: excess}("");
                require(success, "Multicall3: refund failed");
            }
        }
    }
    
    /// @notice ดึง block info ต่างๆ ในการ call เดียว
    function getBlockHash(uint256 blockNumber) external view returns (bytes32) {
        return blockhash(blockNumber);
    }
    
    function getBlockNumber() external view returns (uint256) {
        return block.number;
    }
    
    function getCurrentBlockTimestamp() external view returns (uint256) {
        return block.timestamp;
    }
    
    function getEthBalance(address addr) external view returns (uint256) {
        return addr.balance;
    }
    
    receive() external payable {}
}

/// @title BatchUserOps
/// @notice รวม user operations หลายอันเข้าด้วยกัน (ERC-4337 style)
contract BatchUserOps {
    
    struct UserOp {
        address sender;
        address target;
        uint256 value;
        bytes data;
        uint256 nonce;
        bytes signature;
    }
    
    mapping(address => uint256) public nonces;
    mapping(address => bool) public authorizedBundlers;
    
    bytes32 private constant USEROP_TYPEHASH = keccak256(
        "UserOp(address sender,address target,uint256 value,bytes data,uint256 nonce)"
    );
    
    bytes32 private immutable DOMAIN_SEPARATOR;
    
    constructor() {
        DOMAIN_SEPARATOR = keccak256(abi.encode(
            keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
            keccak256("BatchUserOps"),
            keccak256("1"),
            block.chainid,
            address(this)
        ));
    }
    
    modifier onlyBundler() {
        require(authorizedBundlers[msg.sender], "Not authorized bundler");
        _;
    }
    
    /// @notice Execute batch ของ user operations
    /// @dev Bundler รวบรวม ops จาก mempool แล้วส่งเป็น batch เดียว
    function executeBatch(UserOp[] calldata ops) external onlyBundler {
        for (uint256 i = 0; i < ops.length; i++) {
            _executeOp(ops[i]);
        }
    }
    
    function _executeOp(UserOp calldata op) internal {
        // ตรวจสอบ nonce
        require(op.nonce == nonces[op.sender], "Invalid nonce");
        
        // สร้าง hash
        bytes32 opHash = keccak256(abi.encodePacked(
            "\x19\x01",
            DOMAIN_SEPARATOR,
            keccak256(abi.encode(
                USEROP_TYPEHASH,
                op.sender,
                op.target,
                op.value,
                keccak256(op.data),
                op.nonce
            ))
        ));
        
        // ตรวจสอบ signature
        address signer = _recoverSigner(opHash, op.signature);
        require(signer == op.sender, "Invalid signature");
        
        // อัพเดต nonce
        nonces[op.sender]++;
        
        // Execute
        (bool success, bytes memory result) = op.target.call{value: op.value}(op.data);
        if (!success) {
            assembly {
                revert(add(result, 32), mload(result))
            }
        }
    }
    
    function _recoverSigner(bytes32 hash, bytes memory sig) internal pure returns (address) {
        require(sig.length == 65, "Invalid signature length");
        bytes32 r;
        bytes32 s;
        uint8 v;
        assembly {
            r := mload(add(sig, 32))
            s := mload(add(sig, 64))
            v := byte(0, mload(add(sig, 96)))
        }
        return ecrecover(hash, v, r, s);
    }
    
    function addBundler(address bundler) external {
        // ใน production ต้องมี governance
        authorizedBundlers[bundler] = true;
    }
}
```

---

## 76.3 L2-Specific Gas Oracle

### Arbitrum Gas Oracle

Arbitrum ใช้ `ArbGasInfo` precompile ที่ address `0x000000000000000000000000000000000000006C`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title ArbitrumGasOracle
/// @notice Interface กับ Arbitrum ArbGasInfo precompile
interface IArbGasInfo {
    /// @notice ราคา gas ต่างๆ
    function getPricesInWei() external view returns (
        uint256 perL2Tx,
        uint256 perL1CalldataUnit,
        uint256 perStorageAllocation,
        uint256 perArbGasBase,
        uint256 perArbGasCongestion,
        uint256 perArbGasTotal
    );
    
    function getGasBacklog() external view returns (uint64);
    function getL1BaseFeeEstimate() external view returns (uint256);
    function isUsingL1PricingData() external view returns (bool);
}

/// @title OptimismGasOracle
/// @notice Interface กับ Optimism L1Block precompile
interface IL1Block {
    function basefee() external view returns (uint256);
    function blobBaseFee() external view returns (uint256);
    function hash() external view returns (bytes32);
    function number() external view returns (uint64);
    function sequenceNumber() external view returns (uint64);
    function timestamp() external view returns (uint64);
    function l1FeeOverhead() external view returns (uint256);    // Legacy
    function l1FeeScalar() external view returns (uint256);     // Legacy
}

interface IGasPriceOracle {
    function baseFee() external view returns (uint256);
    function decimals() external view returns (uint256);
    function gasPrice() external view returns (uint256);
    function getL1Fee(bytes memory _data) external view returns (uint256);
    function getL1GasUsed(bytes memory _data) external view returns (uint256);
    function l1BaseFee() external view returns (uint256);
    function overhead() external view returns (uint256);
    function scalar() external view returns (uint256);
    function version() external view returns (string memory);
}

/// @title L2GasEstimator
/// @notice คำนวณ gas cost ที่แม่นยำบน L2
contract L2GasEstimator {
    
    // Arbitrum precompile addresses
    address constant ARB_GAS_INFO = 0x000000000000000000000000000000000000006C;
    
    // Optimism precompile addresses
    address constant OP_GAS_PRICE_ORACLE = 0x420000000000000000000000000000000000000F;
    address constant OP_L1_BLOCK = 0x4200000000000000000000000000000000000015;
    
    enum L2Type { UNKNOWN, ARBITRUM, OPTIMISM, BASE, ZKSYNC }
    
    /// @notice ตรวจสอบว่าอยู่บน L2 ไหน
    function detectL2Type() external view returns (L2Type) {
        // ตรวจสอบ Arbitrum โดยดู chainId
        if (block.chainid == 42161 || block.chainid == 421614) {
            return L2Type.ARBITRUM;
        }
        if (block.chainid == 10 || block.chainid == 11155420) {
            return L2Type.OPTIMISM;
        }
        if (block.chainid == 8453 || block.chainid == 84532) {
            return L2Type.BASE;
        }
        if (block.chainid == 324 || block.chainid == 300) {
            return L2Type.ZKSYNC;
        }
        return L2Type.UNKNOWN;
    }
    
    /// @notice ดึง L1 fee สำหรับ transaction บน Arbitrum
    function getArbitrumL1Fee(uint256 calldataBytes) external view returns (uint256 l1Fee) {
        if (block.chainid != 42161 && block.chainid != 421614) {
            return 0;
        }
        
        (
            ,
            uint256 perL1CalldataUnit,
            ,
            ,
            ,
        ) = IArbGasInfo(ARB_GAS_INFO).getPricesInWei();
        
        // Arbitrum ใช้ units = (calldataBytes * 16 + fixedOverhead) / compressionRatio
        // simplified estimate
        l1Fee = calldataBytes * 16 * perL1CalldataUnit / 1e9;
    }
    
    /// @notice ดึง L1 fee สำหรับ transaction บน Optimism/Base
    function getOptimismL1Fee(bytes calldata txData) external view returns (uint256 l1Fee) {
        if (block.chainid != 10 && block.chainid != 8453 && 
            block.chainid != 11155420 && block.chainid != 84532) {
            return 0;
        }
        
        l1Fee = IGasPriceOracle(OP_GAS_PRICE_ORACLE).getL1Fee(txData);
    }
    
    /// @notice คำนวณ total cost ของ transaction
    struct TransactionCost {
        uint256 l2ExecutionFee;
        uint256 l1DataFee;
        uint256 totalFee;
        uint256 l1BaseFee;
        uint256 l2GasPrice;
    }
    
    function estimateTransactionCost(
        uint256 estimatedGas,
        uint256 calldataSize
    ) external view returns (TransactionCost memory cost) {
        cost.l2GasPrice = tx.gasprice;
        cost.l2ExecutionFee = estimatedGas * cost.l2GasPrice;
        
        if (block.chainid == 42161) {
            // Arbitrum
            cost.l1BaseFee = IArbGasInfo(ARB_GAS_INFO).getL1BaseFeeEstimate();
            // คร่าวๆ: calldata bytes * non-zero cost * L1 base fee
            cost.l1DataFee = calldataSize * 16 * cost.l1BaseFee / 1e9;
        } else if (block.chainid == 10 || block.chainid == 8453) {
            // Optimism/Base
            cost.l1BaseFee = IL1Block(OP_L1_BLOCK).basefee();
            // Bedrock formula: (calldataGas + overhead) * baseFee * scalar / 1e6
            uint256 overhead = IGasPriceOracle(OP_GAS_PRICE_ORACLE).overhead();
            uint256 scalar = IGasPriceOracle(OP_GAS_PRICE_ORACLE).scalar();
            uint256 calldataGas = calldataSize * 16;
            cost.l1DataFee = (calldataGas + overhead) * cost.l1BaseFee * scalar / 1e6 / 1e9;
        }
        
        cost.totalFee = cost.l2ExecutionFee + cost.l1DataFee;
    }
    
    /// @notice Dynamic gas limit เพื่อ avoid overpaying
    function safeGasLimit(uint256 baseGasEstimate) external pure returns (uint256) {
        // เพิ่ม buffer 20% บน L2 เพราะ gas ถูกกว่า
        return baseGasEstimate * 120 / 100;
    }
}

/// @title GasOptimizedStorage
/// @notice Patterns สำหรับลด storage reads บน L2
contract GasOptimizedStorage {
    
    // บน L2, SLOAD ยังคงแพง (ด้าน L2 execution)
    // ใช้ packing เพื่อลด storage slots
    
    struct UserData {
        uint128 balance;      // เหลือพอสำหรับ token balances
        uint64 lastActivity;  // unix timestamp
        uint32 nonce;         // nonce ไม่น่าเกิน 4B
        uint16 tier;          // membership tier
        uint8 flags;          // packed boolean flags
        bool active;          // 1 bit
    }
    // ทั้งหมด = 128+64+32+16+8+8 = 256 bits = 1 storage slot!
    
    mapping(address => UserData) private userData;
    
    // Transient storage สำหรับ within-transaction caching (EIP-1153)
    // transient storage reset ทุก transaction โดยอัตโนมัติ
    uint256 private constant CACHED_PRICE_SLOT = uint256(keccak256("cached.price"));
    
    function setUserData(
        address user,
        uint128 balance,
        uint64 lastActivity,
        uint32 nonce,
        uint16 tier,
        uint8 flags,
        bool active
    ) external {
        userData[user] = UserData({
            balance: balance,
            lastActivity: lastActivity,
            nonce: nonce,
            tier: tier,
            flags: flags,
            active: active
        });
    }
    
    function getUserData(address user) external view returns (UserData memory) {
        return userData[user]; // 1 SLOAD ดึงทุก fields
    }
    
    /// @notice ใช้ unchecked arithmetic เมื่อรู้ว่าไม่ overflow
    function batchUpdateBalances(
        address[] calldata users,
        uint128[] calldata amounts
    ) external {
        require(users.length == amounts.length, "Length mismatch");
        
        unchecked {
            for (uint256 i = 0; i < users.length; i++) {
                userData[users[i]].balance += amounts[i];
                // ไม่ใส่ overflow check เพราะ uint128 + uint128 ไม่น่าเกิน
                // ประหยัด ~20 gas ต่อ iteration
            }
        }
    }
}
```

---

## 76.4 State Channels: Off-Chain Transactions

### แนวคิดของ State Channels

State channels ให้ผู้ใช้ทำ transactions นับพันครั้งโดยไม่ต้องแตะ blockchain จนกว่าจะปิด channel

```
Alice ↔ Bob: เปิด channel ด้วย deposit
  → ทำ transactions ได้เรื่อยๆ (off-chain)
  → เมื่อจบ: ใครก็ได้ submit state สุดท้าย → ปิด channel
  → ถ้า dispute: on-chain judge ตัดสิน
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title PaymentChannel
/// @notice Simple bidirectional payment channel
contract PaymentChannel {
    
    struct Channel {
        address payable alice;
        address payable bob;
        uint256 aliceBalance;
        uint256 bobBalance;
        uint256 expiry;
        uint256 nonce;
        bool isOpen;
    }
    
    mapping(bytes32 => Channel) public channels;
    
    uint256 public constant DISPUTE_PERIOD = 1 days;
    uint256 public constant CHANNEL_EXPIRY = 30 days;
    
    event ChannelOpened(bytes32 indexed channelId, address alice, address bob, uint256 totalDeposit);
    event ChannelClosed(bytes32 indexed channelId, uint256 aliceFinal, uint256 bobFinal);
    event DisputeRaised(bytes32 indexed channelId, address raiser, uint256 nonce);
    
    /// @notice เปิด payment channel ระหว่าง alice และ bob
    function openChannel(address payable bob) external payable returns (bytes32 channelId) {
        require(msg.value > 0, "Must deposit ETH");
        require(bob != address(0) && bob != msg.sender, "Invalid partner");
        
        channelId = keccak256(abi.encodePacked(
            msg.sender,
            bob,
            block.timestamp,
            block.number
        ));
        
        require(!channels[channelId].isOpen, "Channel already exists");
        
        channels[channelId] = Channel({
            alice: payable(msg.sender),
            bob: bob,
            aliceBalance: msg.value,
            bobBalance: 0,
            expiry: block.timestamp + CHANNEL_EXPIRY,
            nonce: 0,
            isOpen: true
        });
        
        emit ChannelOpened(channelId, msg.sender, bob, msg.value);
    }
    
    /// @notice Bob เข้าร่วม channel พร้อม deposit
    function joinChannel(bytes32 channelId) external payable {
        Channel storage ch = channels[channelId];
        require(ch.isOpen, "Channel not open");
        require(msg.sender == ch.bob, "Not bob");
        require(ch.bobBalance == 0, "Already joined");
        
        ch.bobBalance = msg.value;
    }
    
    /// @notice ปิด channel ด้วย signed state ที่ทั้งสองฝ่ายเห็นด้วย
    /// @param channelId Channel ID
    /// @param aliceFinal Balance ของ Alice ที่ตกลงกัน
    /// @param bobFinal Balance ของ Bob ที่ตกลงกัน
    /// @param nonce Nonce สูงสุดที่ใช้
    /// @param aliceSig Signature ของ Alice
    /// @param bobSig Signature ของ Bob
    function closeChannelCooperative(
        bytes32 channelId,
        uint256 aliceFinal,
        uint256 bobFinal,
        uint256 nonce,
        bytes calldata aliceSig,
        bytes calldata bobSig
    ) external {
        Channel storage ch = channels[channelId];
        require(ch.isOpen, "Channel not open");
        
        // ตรวจสอบว่า total balance เท่ากัน
        uint256 totalDeposit = ch.aliceBalance + ch.bobBalance;
        require(aliceFinal + bobFinal == totalDeposit, "Balance mismatch");
        require(nonce > ch.nonce, "Stale state");
        
        // สร้าง state hash
        bytes32 stateHash = _createStateHash(channelId, aliceFinal, bobFinal, nonce);
        
        // ตรวจสอบ signatures
        require(_verifySignature(stateHash, aliceSig, ch.alice), "Invalid alice sig");
        require(_verifySignature(stateHash, bobSig, ch.bob), "Invalid bob sig");
        
        // ปิด channel และจ่ายเงิน
        ch.isOpen = false;
        
        if (aliceFinal > 0) {
            ch.alice.transfer(aliceFinal);
        }
        if (bobFinal > 0) {
            ch.bob.transfer(bobFinal);
        }
        
        emit ChannelClosed(channelId, aliceFinal, bobFinal);
    }
    
    struct DisputeData {
        uint256 aliceBalance;
        uint256 bobBalance;
        uint256 nonce;
        uint256 disputeDeadline;
        bool active;
    }
    
    mapping(bytes32 => DisputeData) public disputes;
    
    /// @notice เปิด dispute เมื่ออีกฝ่ายไม่ยอมปิด channel
    function raiseDispute(
        bytes32 channelId,
        uint256 aliceBalance,
        uint256 bobBalance,
        uint256 nonce,
        bytes calldata mySig,
        bytes calldata theirSig
    ) external {
        Channel storage ch = channels[channelId];
        require(ch.isOpen, "Channel not open");
        require(msg.sender == ch.alice || msg.sender == ch.bob, "Not participant");
        
        bytes32 stateHash = _createStateHash(channelId, aliceBalance, bobBalance, nonce);
        
        // ตรวจสอบ signatures
        if (msg.sender == ch.alice) {
            require(_verifySignature(stateHash, mySig, ch.alice), "Invalid alice sig");
            require(_verifySignature(stateHash, theirSig, ch.bob), "Invalid bob sig");
        } else {
            require(_verifySignature(stateHash, mySig, ch.bob), "Invalid bob sig");
            require(_verifySignature(stateHash, theirSig, ch.alice), "Invalid alice sig");
        }
        
        DisputeData storage dispute = disputes[channelId];
        
        // อัพเดต dispute ถ้า nonce ใหม่กว่า
        if (!dispute.active || nonce > dispute.nonce) {
            disputes[channelId] = DisputeData({
                aliceBalance: aliceBalance,
                bobBalance: bobBalance,
                nonce: nonce,
                disputeDeadline: block.timestamp + DISPUTE_PERIOD,
                active: true
            });
            
            emit DisputeRaised(channelId, msg.sender, nonce);
        }
    }
    
    /// @notice Finalize dispute หลัง dispute period
    function finalizeDispute(bytes32 channelId) external {
        Channel storage ch = channels[channelId];
        DisputeData storage dispute = disputes[channelId];
        
        require(ch.isOpen, "Channel not open");
        require(dispute.active, "No active dispute");
        require(block.timestamp > dispute.disputeDeadline, "Dispute period not over");
        
        ch.isOpen = false;
        dispute.active = false;
        
        if (dispute.aliceBalance > 0) {
            ch.alice.transfer(dispute.aliceBalance);
        }
        if (dispute.bobBalance > 0) {
            ch.bob.transfer(dispute.bobBalance);
        }
        
        emit ChannelClosed(channelId, dispute.aliceBalance, dispute.bobBalance);
    }
    
    /// @notice Force close หลัง expiry
    function forceClose(bytes32 channelId) external {
        Channel storage ch = channels[channelId];
        require(ch.isOpen, "Channel not open");
        require(block.timestamp > ch.expiry, "Channel not expired");
        
        ch.isOpen = false;
        
        // คืนเงินตาม deposit ล่าสุด
        if (ch.aliceBalance > 0) {
            ch.alice.transfer(ch.aliceBalance);
        }
        if (ch.bobBalance > 0) {
            ch.bob.transfer(ch.bobBalance);
        }
    }
    
    function _createStateHash(
        bytes32 channelId,
        uint256 aliceBalance,
        uint256 bobBalance,
        uint256 nonce
    ) internal pure returns (bytes32) {
        return keccak256(abi.encodePacked(
            "\x19Ethereum Signed Message:\n32",
            keccak256(abi.encodePacked(channelId, aliceBalance, bobBalance, nonce))
        ));
    }
    
    function _verifySignature(
        bytes32 hash,
        bytes memory sig,
        address expectedSigner
    ) internal pure returns (bool) {
        require(sig.length == 65, "Invalid sig length");
        bytes32 r;
        bytes32 s;
        uint8 v;
        
        assembly {
            r := mload(add(sig, 32))
            s := mload(add(sig, 64))
            v := byte(0, mload(add(sig, 96)))
        }
        
        address recovered = ecrecover(hash, v, r, s);
        return recovered != address(0) && recovered == expectedSigner;
    }
}
```

---

## 76.5 Plasma-Style Merkle Commitment

### Plasma Architecture

Plasma เป็นแนวทางที่นำ transactions ไปประมวลผลบน child chain แล้ว commit Merkle root ไปยัง L1

```
Child Chain: [tx1, tx2, tx3, ... txN] → Merkle Root
L1 Contract: เก็บ Merkle roots ของแต่ละ block
ถ้ามี fraud: ใครก็ได้ submit proof เพื่อ challenge
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title MerkleTree
/// @notice Library สำหรับสร้างและตรวจสอบ Merkle proofs
library MerkleTree {
    
    /// @notice ตรวจสอบ Merkle proof
    function verify(
        bytes32[] memory proof,
        bytes32 root,
        bytes32 leaf,
        uint256 index
    ) internal pure returns (bool) {
        bytes32 hash = leaf;
        
        for (uint256 i = 0; i < proof.length; i++) {
            bytes32 proofElement = proof[i];
            
            if (index % 2 == 0) {
                hash = keccak256(abi.encodePacked(hash, proofElement));
            } else {
                hash = keccak256(abi.encodePacked(proofElement, hash));
            }
            
            index = index / 2;
        }
        
        return hash == root;
    }
    
    /// @notice สร้าง leaf hash จาก transaction data
    function hashLeaf(
        address sender,
        address recipient,
        uint256 amount,
        uint256 nonce,
        uint256 blockNumber
    ) internal pure returns (bytes32) {
        return keccak256(abi.encodePacked(sender, recipient, amount, nonce, blockNumber));
    }
}

/// @title PlasmaCommitter
/// @notice L1 contract ที่รับ Merkle commitments จาก Plasma operator
contract PlasmaCommitter {
    using MerkleTree for *;
    
    struct BlockCommitment {
        bytes32 merkleRoot;
        uint256 timestamp;
        uint256 txCount;
        address operator;
        bool finalized;
    }
    
    struct ExitRequest {
        address owner;
        uint256 amount;
        uint256 plasmaBlock;
        uint256 txIndex;
        uint256 exitTime;
        bool processed;
    }
    
    // Plasma block number → commitment
    mapping(uint256 => BlockCommitment) public commitments;
    
    // Exit ID → exit request
    mapping(bytes32 => ExitRequest) public exits;
    
    // Deposits: depositor → balance
    mapping(address => uint256) public deposits;
    
    uint256 public currentPlasmaBlock;
    address public operator;
    
    uint256 public constant CHALLENGE_PERIOD = 7 days;
    uint256 public constant EXIT_BOND = 0.1 ether;
    
    event Deposited(address indexed user, uint256 amount);
    event BlockCommitted(uint256 indexed plasmaBlock, bytes32 merkleRoot, uint256 txCount);
    event ExitStarted(bytes32 indexed exitId, address owner, uint256 amount);
    event ExitFinalized(bytes32 indexed exitId, address owner, uint256 amount);
    event ExitChallenged(bytes32 indexed exitId, address challenger);
    
    modifier onlyOperator() {
        require(msg.sender == operator, "Not operator");
        _;
    }
    
    constructor(address _operator) {
        operator = _operator;
    }
    
    /// @notice Deposit ETH เข้า Plasma
    function deposit() external payable {
        require(msg.value > 0, "Must deposit");
        deposits[msg.sender] += msg.value;
        emit Deposited(msg.sender, msg.value);
    }
    
    /// @notice Operator commit Merkle root ของ Plasma block
    function commitBlock(
        bytes32 merkleRoot,
        uint256 txCount
    ) external onlyOperator {
        currentPlasmaBlock++;
        
        commitments[currentPlasmaBlock] = BlockCommitment({
            merkleRoot: merkleRoot,
            timestamp: block.timestamp,
            txCount: txCount,
            operator: msg.sender,
            finalized: false
        });
        
        emit BlockCommitted(currentPlasmaBlock, merkleRoot, txCount);
    }
    
    /// @notice เริ่ม exit process โดย user ที่ต้องการถอน
    function startExit(
        uint256 plasmaBlock,
        uint256 txIndex,
        uint256 amount,
        bytes32[] calldata merkleProof,
        bytes calldata txData
    ) external payable {
        require(msg.value >= EXIT_BOND, "Insufficient exit bond");
        require(plasmaBlock <= currentPlasmaBlock, "Invalid block");
        
        BlockCommitment storage commitment = commitments[plasmaBlock];
        require(commitment.merkleRoot != bytes32(0), "Block not committed");
        
        // ตรวจสอบว่า tx อยู่จริงใน Plasma block
        bytes32 txHash = keccak256(txData);
        require(
            MerkleTree.verify(merkleProof, commitment.merkleRoot, txHash, txIndex),
            "Invalid Merkle proof"
        );
        
        // Parse tx data เพื่อตรวจสอบว่า sender เป็น msg.sender
        // (simplified: ใน production ต้องตรวจสอบ signature ด้วย)
        (address txRecipient, uint256 txAmount) = _parseTxData(txData);
        require(txRecipient == msg.sender, "Not tx recipient");
        require(txAmount == amount, "Amount mismatch");
        
        bytes32 exitId = keccak256(abi.encodePacked(
            msg.sender, plasmaBlock, txIndex
        ));
        
        require(exits[exitId].owner == address(0), "Exit already started");
        
        exits[exitId] = ExitRequest({
            owner: msg.sender,
            amount: amount,
            plasmaBlock: plasmaBlock,
            txIndex: txIndex,
            exitTime: block.timestamp + CHALLENGE_PERIOD,
            processed: false
        });
        
        emit ExitStarted(exitId, msg.sender, amount);
    }
    
    /// @notice Challenge exit ถ้ามี double spend
    function challengeExit(
        bytes32 exitId,
        uint256 spendingBlock,
        uint256 spendingTxIndex,
        bytes32[] calldata spendingProof,
        bytes calldata spendingTxData
    ) external {
        ExitRequest storage exitReq = exits[exitId];
        require(!exitReq.processed, "Exit already processed");
        require(block.timestamp < exitReq.exitTime, "Challenge period over");
        
        // ตรวจสอบว่า spending block มีอยู่จริง
        require(spendingBlock <= currentPlasmaBlock, "Invalid spending block");
        
        BlockCommitment storage spendCommitment = commitments[spendingBlock];
        bytes32 spendTxHash = keccak256(spendingTxData);
        
        require(
            MerkleTree.verify(spendingProof, spendCommitment.merkleRoot, spendTxHash, spendingTxIndex),
            "Invalid spending proof"
        );
        
        // ตรวจสอบว่า spending tx ใช้ output ที่ถูก exit
        // (simplified check)
        (address spendSender,) = _parseTxData(spendingTxData);
        require(spendSender == exitReq.owner, "Not spending this exit");
        
        // ยกเลิก exit และให้ bond กับ challenger
        exitReq.processed = true;
        payable(msg.sender).transfer(EXIT_BOND);
        
        emit ExitChallenged(exitId, msg.sender);
    }
    
    /// @notice Finalize exit หลัง challenge period
    function finalizeExit(bytes32 exitId) external {
        ExitRequest storage exitReq = exits[exitId];
        require(exitReq.owner == msg.sender, "Not exit owner");
        require(!exitReq.processed, "Already processed");
        require(block.timestamp >= exitReq.exitTime, "Challenge period not over");
        
        exitReq.processed = true;
        
        // คืนเงินพร้อม bond
        uint256 totalAmount = exitReq.amount + EXIT_BOND;
        payable(msg.sender).transfer(totalAmount);
        
        emit ExitFinalized(exitId, msg.sender, exitReq.amount);
    }
    
    /// @notice Parse tx data (simplified)
    function _parseTxData(bytes calldata txData) internal pure returns (
        address recipient,
        uint256 amount
    ) {
        // ใน production จะมี proper encoding/decoding
        (recipient, amount) = abi.decode(txData, (address, uint256));
    }
    
    receive() external payable {}
}
```

---

## Workshop: L2 Optimization Challenge

**โจทย์**: สร้าง `L2EfficientDEX` ที่ optimize สำหรับ L2

### Requirements:
1. Batch swap: รับ N swaps เป็น compressed calldata
2. Gas estimation: แสดง estimated fee ก่อน execute
3. Channel-based liquidity: LP เปิด channel กับ DEX
4. Merkle order book: Orders ถูก commit เป็น batch

### Starter Code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title L2EfficientDEX
/// @notice DEX ที่ optimize สำหรับ Layer 2
contract L2EfficientDEX is ReentrancyGuard {
    
    // Token pair → reserve
    mapping(address => mapping(address => uint256)) public reserves;
    
    // LP positions
    mapping(bytes32 => uint256) public lpShares;
    mapping(bytes32 => uint256) public totalShares;
    
    // Committed order batches
    mapping(uint256 => bytes32) public orderBatchRoots;
    uint256 public currentBatch;
    
    // Compressed swap format: [2 bytes tokenInIndex][2 bytes tokenOutIndex][12 bytes amountIn][12 bytes minOut]
    // = 28 bytes per swap
    uint256 constant SWAP_SIZE = 28;
    
    // Registered token list (for compression)
    address[] public tokenList;
    mapping(address => uint16) public tokenIndex;
    
    event TokenRegistered(address token, uint16 index);
    event BatchSwapExecuted(uint256 swapCount, uint256 totalVolumeEth);
    event OrderBatchCommitted(uint256 batchId, bytes32 merkleRoot, uint256 orderCount);
    
    constructor() {
        // Register ETH as token 0
        tokenList.push(address(0));
        tokenIndex[address(0)] = 0;
    }
    
    /// @notice ลงทะเบียน token เพื่อใช้ 2-byte index
    function registerToken(address token) external returns (uint16 index) {
        require(tokenIndex[token] == 0 && token != address(0), "Already registered");
        index = uint16(tokenList.length);
        tokenList.push(token);
        tokenIndex[token] = index;
        emit TokenRegistered(token, index);
    }
    
    /// @notice Batch swap ด้วย compressed calldata
    /// @param compressedSwaps Packed swap data (28 bytes per swap)
    function batchSwap(bytes calldata compressedSwaps) 
        external 
        nonReentrant 
        returns (uint256 swapsExecuted) 
    {
        require(compressedSwaps.length % SWAP_SIZE == 0, "Invalid swap data");
        uint256 swapCount = compressedSwaps.length / SWAP_SIZE;
        
        for (uint256 i = 0; i < swapCount; i++) {
            uint256 offset = i * SWAP_SIZE;
            
            uint16 tokenInIdx;
            uint16 tokenOutIdx;
            uint96 amountIn;
            uint96 minOut;
            
            assembly {
                let ptr := add(compressedSwaps.offset, offset)
                tokenInIdx := shr(240, calldataload(ptr))
                tokenOutIdx := shr(240, calldataload(add(ptr, 2)))
                amountIn := and(
                    shr(128, calldataload(add(ptr, 4))),
                    0xFFFFFFFFFFFFFFFFFFFFFFFF
                )
                minOut := and(
                    shr(128, calldataload(add(ptr, 16))),
                    0xFFFFFFFFFFFFFFFFFFFFFFFF
                )
            }
            
            address tokenIn = tokenList[tokenInIdx];
            address tokenOut = tokenList[tokenOutIdx];
            
            // Execute swap
            uint256 amountOut = _swap(tokenIn, tokenOut, amountIn, minOut, msg.sender);
            if (amountOut >= minOut) {
                swapsExecuted++;
            }
        }
        
        emit BatchSwapExecuted(swapsExecuted, 0);
    }
    
    /// @notice Commit batch ของ limit orders เป็น Merkle root
    function commitOrderBatch(
        bytes32 merkleRoot,
        uint256 orderCount
    ) external {
        currentBatch++;
        orderBatchRoots[currentBatch] = merkleRoot;
        emit OrderBatchCommitted(currentBatch, merkleRoot, orderCount);
    }
    
    /// @notice คำนวณ gas estimate สำหรับ batch
    function estimateBatchCost(uint256 swapCount) external view returns (
        uint256 calldataBytes,
        uint256 estimatedL2Gas,
        uint256 estimatedTotalGas
    ) {
        // 4 bytes selector + swapCount * 28 bytes
        calldataBytes = 4 + swapCount * SWAP_SIZE;
        estimatedL2Gas = 50000 + swapCount * 30000; // Base + per-swap
        
        // L1 data cost (non-zero bytes @ 16 gas each, estimated 50% non-zero)
        uint256 l1DataGas = calldataBytes * 8; // rough estimate
        estimatedTotalGas = estimatedL2Gas + l1DataGas;
    }
    
    function _swap(
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 minOut,
        address user
    ) internal returns (uint256 amountOut) {
        bytes32 pairKey = _pairKey(tokenIn, tokenOut);
        uint256 reserveIn = reserves[tokenIn][tokenOut];
        uint256 reserveOut = reserves[tokenOut][tokenIn];
        
        require(reserveIn > 0 && reserveOut > 0, "No liquidity");
        
        // AMM formula: x * y = k, with 0.3% fee
        uint256 amountInWithFee = amountIn * 997;
        amountOut = (amountInWithFee * reserveOut) / (reserveIn * 1000 + amountInWithFee);
        
        require(amountOut >= minOut, "Insufficient output");
        
        // Transfer tokens
        IERC20(tokenIn).transferFrom(user, address(this), amountIn);
        IERC20(tokenOut).transfer(user, amountOut);
        
        // Update reserves
        reserves[tokenIn][tokenOut] += amountIn;
        reserves[tokenOut][tokenIn] -= amountOut;
        
        _ = pairKey; // suppress unused
    }
    
    function _pairKey(address a, address b) internal pure returns (bytes32) {
        return a < b 
            ? keccak256(abi.encodePacked(a, b))
            : keccak256(abi.encodePacked(b, a));
    }
    
    function addLiquidity(
        address tokenA,
        address tokenB,
        uint256 amountA,
        uint256 amountB
    ) external returns (uint256 shares) {
        IERC20(tokenA).transferFrom(msg.sender, address(this), amountA);
        IERC20(tokenB).transferFrom(msg.sender, address(this), amountB);
        
        bytes32 pairKey = _pairKey(tokenA, tokenB);
        uint256 total = totalShares[pairKey];
        
        if (total == 0) {
            shares = _sqrt(amountA * amountB);
        } else {
            uint256 sharesA = amountA * total / reserves[tokenA][tokenB];
            uint256 sharesB = amountB * total / reserves[tokenB][tokenA];
            shares = sharesA < sharesB ? sharesA : sharesB;
        }
        
        reserves[tokenA][tokenB] += amountA;
        reserves[tokenB][tokenA] += amountB;
        
        bytes32 lpKey = keccak256(abi.encodePacked(pairKey, msg.sender));
        lpShares[lpKey] += shares;
        totalShares[pairKey] += shares;
    }
    
    function _sqrt(uint256 y) internal pure returns (uint256 z) {
        if (y > 3) {
            z = y;
            uint256 x = y / 2 + 1;
            while (x < z) {
                z = x;
                x = (y / x + x) / 2;
            }
        } else if (y != 0) {
            z = 1;
        }
    }
}
```

---

## สรุป Part 76

- **Calldata compression** คือกุญแจสำคัญในการลด L1 data fee บน L2 — ใช้ smaller types, packed encoding และ address indices
- **Multicall pattern** ช่วยรวมหลาย transactions เข้าเป็นหนึ่ง ลด per-transaction overhead
- **Arbitrum Gas Oracle** (`ArbGasInfo`) และ **Optimism Gas Oracle** (`GasPriceOracle`) ให้ข้อมูล L1 fee แบบ real-time
- **State channels** ให้ผู้ใช้ทำ transactions off-chain นับพันครั้งโดยแทบไม่มี cost จนกว่าจะ close
- **Plasma commitments** ใช้ Merkle trees commit state จาก child chain ไปยัง L1 พร้อม challenge mechanism

## Next: Part 77 - Advanced Governance Systems
