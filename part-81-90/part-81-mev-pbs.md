# Part 81: MEV & Proposer-Builder Separation (การแยกผู้เสนอและผู้สร้างบล็อก)

## บทนำ

**Maximal Extractable Value (MEV)** คือมูลค่าสูงสุดที่สามารถดึงออกจากการผลิตบล็อกได้ นอกเหนือจาก block reward และค่าธรรมเนียมปกติ โดยการจัดลำดับ เพิ่ม หรือลบธุรกรรมออกจากบล็อก ในปี 2024 MEV สะสมทั้งหมดบน Ethereum เกิน **$1 พันล้านดอลลาร์** และกำลังกลายเป็นประเด็นสำคัญที่นักพัฒนา DeFi ทุกคนต้องเข้าใจอย่างลึกซึ้ง

---

## 81.1 MEV Taxonomy: ประเภทและกลยุทธ์ของ MEV

### 81.1.1 Arbitrage MEV (อาร์บิทราจ)

**แนวคิด:** เมื่อราคาของ token เดียวกันแตกต่างกันระหว่าง DEX สอง Exchange ผู้ค้นหา (searcher) สามารถซื้อถูกและขายแพงในธุรกรรมเดียวกัน

**ตัวอย่างเชิงตัวเลข:**
- Uniswap V3: ETH/USDC = $3,000
- Curve: ETH/USDC = $3,050
- กำไร: $50 ต่อ ETH (ก่อนค่า gas)

**ประเภทของ Arbitrage:**
1. **Triangle Arbitrage:** ETH → USDC → WBTC → ETH
2. **Cross-DEX:** Uniswap ↔ Sushiswap ↔ Balancer
3. **Cross-Chain:** เมื่อรวมกับ bridge (ซับซ้อนกว่า)
4. **Statistical Arbitrage:** ใช้ ML คาดการณ์ราคาล่วงหน้า

### 81.1.2 Liquidation MEV (การชำระหนี้)

เมื่อผู้กู้ใน Aave/Compound มีหลักประกันต่ำกว่า threshold ใครก็ตามสามารถชำระหนี้และรับ **liquidation bonus (5-10%)** ซึ่ง searcher แข่งกันส่งธุรกรรมก่อน

**กระบวนการ:**
1. ตรวจสอบ health factor ของ positions ทั้งหมดอย่างต่อเนื่อง
2. เมื่อราคาตก → position บางอันใกล้ threshold
3. ส่ง bundle ที่รวม flashloan + liquidation + swap
4. ได้กำไรจาก bonus ลบด้วยค่า gas

### 81.1.3 Sandwich Attacks (การโจมตีแบบแซนด์วิช)

เป็น MEV ประเภทที่ **เป็นอันตราย** ต่อผู้ใช้มากที่สุด:

```
Front-run: ซื้อ token ก่อนเหยื่อ (ทำให้ราคาสูงขึ้น)
    Victim: เหยื่อซื้อในราคาแพงขึ้น
Back-run:  ขาย token ทันทีหลังเหยื่อ (ได้กำไร)
```

**เงื่อนไขที่ทำให้ profitable:**
- Slippage tolerance สูง (>1%)
- ขนาด trade ใหญ่
- Pool liquidity ต่ำ (AMM curve ชัน)

**ตัวเลขจริง (2024):**
- Sandwich attacks ทำให้ผู้ใช้ Uniswap สูญเสีย ~$60M+ ต่อปี
- Average sandwich profit: $100-500 per attack

### 81.1.4 JIT Liquidity (Just-In-Time Liquidity)

**กลยุทธ์:** เพิ่ม liquidity เข้า concentrated range ก่อน trade ขนาดใหญ่ จากนั้นถอนออกทันที เพื่อเก็บค่าธรรมเนียมโดยแทบไม่เสี่ยง impermanent loss

```
Block N-1: เห็น large swap ใน mempool
Block N:   [TX1] addLiquidity ที่ราคาปัจจุบัน
           [TX2] swap ของเหยื่อ (เก็บค่าธรรมเนียม 0.3%)
           [TX3] removeLiquidity ทันที
```

### 81.1.5 Long-Tail MEV

MEV ที่ซับซ้อนและหายากกว่า แต่มูลค่าสูง:
- **NFT MEV:** Snipe NFT ราคาต่ำ → ขายทันที
- **Governance MEV:** Vote buying, proposal front-running
- **Oracle manipulation:** ในบางระบบที่มีช่องโหว่
- **Cross-protocol MEV:** รวมหลาย protocol ในธุรกรรมเดียว

---

## 81.2 PBS Architecture: Proposer-Builder Separation

### ปัญหาเดิม (Pre-PBS)

ก่อน PBS ผู้ validate (proposer) ต้องสร้างบล็อกเองด้วย ทำให้:
1. Validator ขนาดใหญ่มีข้อได้เปรียบด้าน MEV
2. เกิด **centralization pressure** - validator เล็กๆ เสียเปรียบ
3. Validator ต้องรัน MEV software เอง (ซับซ้อน)

### PBS Architecture ปัจจุบัน (mev-boost)

```
┌──────────────┐     ┌─────────────┐     ┌──────────────────┐
│   Searchers  │────▶│   Builders  │────▶│     Relays       │
│              │     │ (assemble   │     │ (mev-boost       │
│ Submit MEV   │     │  blocks)    │     │  compatible)     │
│ bundles      │     │             │     │                  │
└──────────────┘     └─────────────┘     └────────┬─────────┘
                                                   │ bid
                                          ┌────────▼─────────┐
                                          │    Proposers     │
                                          │ (Validators)     │
                                          │ choose highest   │
                                          │ bid block        │
                                          └──────────────────┘
```

**บทบาทของแต่ละฝ่าย:**

| บทบาท | หน้าที่ | จำนวน (2024) |
|--------|---------|--------------|
| Searchers | ค้นหาโอกาส MEV, ส่ง bundles | ~1,000+ |
| Builders | รวม bundles → block | ~20 หลัก |
| Relays | ตรวจสอบ bid, ส่ง header | ~10 |
| Proposers | เลือก bid สูงสุด | ~500,000+ validators |

### 81.2.1 Flashbots MEV-Boost

```go
// ตัวอย่าง Bundle submission (ภาษา Go)
bundle := &flashbotsgo.SendBundleRequest{
    Txs:         []string{signedTx1, signedTx2},
    BlockNumber:  "0x" + strconv.FormatInt(targetBlock, 16),
    MinTimestamp: &minTimestamp,
    MaxTimestamp: &maxTimestamp,
}
response, err := flashbotsClient.SendBundle(bundle)
```

### 81.2.2 Relay Mechanics

Relay ทำหน้าที่:
1. รับ `ExecutionPayload` จาก Builder
2. ตรวจสอบความถูกต้อง (validity)
3. เก็บ payload ไว้ (escrow)
4. ส่งเฉพาะ `ExecutionPayloadHeader` (blind) ให้ Proposer
5. เมื่อ Proposer sign → เปิดเผย full payload

**ความเสี่ยง:** Relay ต้อง trusted → centralization risk

---

## 81.3 SUAVE: อนาคตของ MEV Infrastructure

**SUAVE (Single Unifying Auction for Value Expression)** คือ blockchain แยกต่างหากที่ Flashbots กำลังพัฒนาเพื่อเป็น **decentralized block building layer**

### สถาปัตยกรรม SUAVE

```
┌────────────────────────────────────────────────────┐
│                  SUAVE Chain                        │
│  ┌─────────────┐  ┌───────────────┐  ┌──────────┐  │
│  │  Preference │  │  MEVM (MEV-   │  │  Orderflow│  │
│  │  Environment│  │  Ethereum VM) │  │  Auction  │  │
│  │  (Intents)  │  │               │  │           │  │
│  └─────────────┘  └───────────────┘  └──────────┘  │
└────────────────────────────────────────────────────┘
          │ cross-chain messages
┌─────────▼──────────────────────────────────────────┐
│          Target Chains (Ethereum, L2s, etc.)        │
└────────────────────────────────────────────────────┘
```

**แนวคิดหลัก:**
- **Confidential Compute:** ประมวลผล bundle โดยไม่เปิดเผยข้อมูล
- **SUAVE Transactions:** แทน MEV bundles ด้วย intents
- **Kettle:** Node พิเศษที่รัน SGX (Intel trusted execution)

---

## 81.4 Flashbots Protect & Private Mempool

### Flashbots Protect

ผู้ใช้สามารถส่ง transaction ผ่าน `https://rpc.flashbots.net` เพื่อ:
1. **ปกป้องจาก Sandwich:** ธุรกรรมไม่ปรากฏใน public mempool
2. **Backrun Only:** อนุญาตเฉพาะ MEV ที่ไม่เป็นอันตราย
3. **Refund:** ได้รับ ETH คืนจากกำไรที่ Flashbots ทำได้

```javascript
// ตั้งค่า RPC ใน MetaMask / ethers.js
const provider = new ethers.JsonRpcProvider("https://rpc.flashbots.net");

// ส่ง transaction ปกติ → ป้องกัน sandwich อัตโนมัติ
const tx = await wallet.sendTransaction({
  to: "0x...",
  value: ethers.parseEther("1.0"),
  gasLimit: 21000,
});
```

### MEV Blocker และทางเลือกอื่น

| บริการ | กลไก | ข้อดี |
|--------|------|-------|
| Flashbots Protect | Private mempool + backrun refund | ง่ายที่สุด |
| MEV Blocker (CoW Protocol) | OFA auction | Refund สูงสุด |
| Titan Builder | Private + builder rebate | L1 + L2 |
| bloXroute | Private relay | ความเร็วสูง |

---

## 81.5 MEV-Resistant Protocol Design

### กลยุทธ์ที่ 1: Commit-Reveal Scheme

แบ่งการ submit เป็น 2 ขั้นตอน เพื่อซ่อนข้อมูลจาก frontrunner:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title CommitRevealAuction
 * @notice ตัวอย่าง commit-reveal เพื่อป้องกัน front-running
 */
contract CommitRevealAuction {
    struct Bid {
        bytes32 commitment; // keccak256(abi.encodePacked(amount, salt))
        uint256 deposit;
        bool revealed;
        uint256 revealedAmount;
    }
    
    mapping(address => Bid) public bids;
    
    uint256 public commitDeadline;
    uint256 public revealDeadline;
    
    address public highestBidder;
    uint256 public highestBid;
    
    event BidCommitted(address indexed bidder);
    event BidRevealed(address indexed bidder, uint256 amount);
    
    constructor(uint256 _commitPeriod, uint256 _revealPeriod) {
        commitDeadline = block.timestamp + _commitPeriod;
        revealDeadline = commitDeadline + _revealPeriod;
    }
    
    // Phase 1: Commit (ส่ง hash เท่านั้น ไม่รู้ราคาจริง)
    function commit(bytes32 _commitment) external payable {
        require(block.timestamp < commitDeadline, "Commit phase ended");
        require(bids[msg.sender].commitment == bytes32(0), "Already committed");
        require(msg.value > 0, "Deposit required");
        
        bids[msg.sender] = Bid({
            commitment: _commitment,
            deposit: msg.value,
            revealed: false,
            revealedAmount: 0
        });
        
        emit BidCommitted(msg.sender);
    }
    
    // Phase 2: Reveal (เปิดเผยราคาจริง พร้อม salt)
    function reveal(uint256 _amount, bytes32 _salt) external {
        require(block.timestamp >= commitDeadline, "Reveal phase not started");
        require(block.timestamp < revealDeadline, "Reveal phase ended");
        
        Bid storage bid = bids[msg.sender];
        require(!bid.revealed, "Already revealed");
        
        // ตรวจสอบว่า commitment ตรงกัน
        bytes32 expectedCommitment = keccak256(
            abi.encodePacked(_amount, _salt, msg.sender)
        );
        require(bid.commitment == expectedCommitment, "Invalid reveal");
        require(bid.deposit >= _amount, "Insufficient deposit for bid");
        
        bid.revealed = true;
        bid.revealedAmount = _amount;
        
        if (_amount > highestBid) {
            highestBid = _amount;
            highestBidder = msg.sender;
        }
        
        emit BidRevealed(msg.sender, _amount);
    }
    
    // Helper: สร้าง commitment ฝั่ง client
    function createCommitment(
        uint256 _amount,
        bytes32 _salt,
        address _bidder
    ) external pure returns (bytes32) {
        return keccak256(abi.encodePacked(_amount, _salt, _bidder));
    }
}
```

### กลยุทธ์ที่ 2: Time-Weighted Average Price (TWAP)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title TWAPOracle
 * @notice ใช้ราคาเฉลี่ยระยะเวลาแทนราคา spot เพื่อต้านการโจมตี
 */
contract TWAPOracle {
    struct Observation {
        uint32 timestamp;
        uint256 priceCumulative; // sum of price * time
    }
    
    Observation[65536] public observations;
    uint16 public observationIndex;
    uint16 public observationCardinality;
    uint16 public observationCardinalityNext;
    
    uint256 public lastPrice;
    uint256 public lastTimestamp;
    uint256 public priceCumulative;
    
    event PriceUpdated(uint256 price, uint256 timestamp);
    
    // อัปเดตราคา (เรียกจาก price feed หรือ trading activity)
    function updatePrice(uint256 _newPrice) external {
        uint256 timeElapsed = block.timestamp - lastTimestamp;
        
        if (timeElapsed > 0 && lastPrice > 0) {
            priceCumulative += lastPrice * timeElapsed;
        }
        
        lastPrice = _newPrice;
        lastTimestamp = block.timestamp;
        
        // บันทึก observation
        observations[observationIndex] = Observation({
            timestamp: uint32(block.timestamp),
            priceCumulative: priceCumulative
        });
        
        unchecked {
            observationIndex = (observationIndex + 1) % 65536;
        }
        
        emit PriceUpdated(_newPrice, block.timestamp);
    }
    
    // คำนวณ TWAP สำหรับช่วงเวลา
    function getTWAP(uint256 _secondsAgo) external view returns (uint256 twap) {
        require(_secondsAgo > 0, "Must specify period");
        
        uint256 currentTime = block.timestamp;
        uint256 targetTime = currentTime - _secondsAgo;
        
        // คำนวณ cumulative ณ ปัจจุบัน
        uint256 currentCumulative = priceCumulative;
        if (lastTimestamp < currentTime) {
            currentCumulative += lastPrice * (currentTime - lastTimestamp);
        }
        
        // หา observation ที่ใกล้ targetTime ที่สุด
        (uint256 pastCumulative, uint256 pastTimestamp) = _findObservation(targetTime);
        
        uint256 timeDelta = currentTime - pastTimestamp;
        require(timeDelta > 0, "No time elapsed");
        
        twap = (currentCumulative - pastCumulative) / timeDelta;
    }
    
    function _findObservation(uint256 _targetTimestamp) 
        internal 
        view 
        returns (uint256 cumulative, uint256 timestamp) 
    {
        // Simplified - ในการใช้งานจริงต้องทำ binary search
        // สำหรับ Uniswap V3 ใช้ algorithm ที่ซับซ้อนกว่านี้
        for (uint16 i = observationIndex; ; ) {
            if (i == 0) i = 65535;
            else unchecked { i--; }
            
            if (observations[i].timestamp <= _targetTimestamp) {
                return (observations[i].priceCumulative, observations[i].timestamp);
            }
            
            if (i == observationIndex) break;
        }
        
        // Fallback ถ้าไม่มี observation เก่าพอ
        return (0, _targetTimestamp);
    }
}
```

### กลยุทธ์ที่ 3: Batch Auctions (CowSwap-style)

แทนที่จะ settle ทีละ order ใช้การ settle พร้อมกันทีละ batch เพื่อหา **uniform clearing price**

---

## 81.6 MEVProtectedDEX: Complete Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import "@openzeppelin/contracts/utils/cryptography/MessageHashUtils.sol";

/**
 * @title MEVProtectedDEX
 * @notice DEX ที่ป้องกัน MEV ด้วย batch auction settlement
 *         และ uniform clearing price
 * @dev ใช้หลักการ CowSwap/Gnosis Protocol:
 *      1. รวม orders ใน batch
 *      2. หา clearing price ที่ดีที่สุดสำหรับทุกคน
 *      3. Settle พร้อมกัน (ป้องกัน sandwich)
 */
contract MEVProtectedDEX is ReentrancyGuard, Ownable {
    using SafeERC20 for IERC20;
    using ECDSA for bytes32;
    using MessageHashUtils for bytes32;

    // ============ Structs ============

    struct Order {
        address owner;          // เจ้าของ order
        address sellToken;      // token ที่ต้องการขาย
        address buyToken;       // token ที่ต้องการซื้อ
        uint256 sellAmount;     // จำนวนที่ขาย
        uint256 buyAmountMin;   // จำนวนขั้นต่ำที่ต้องการซื้อ
        uint256 validTo;        // หมดอายุ (timestamp)
        uint256 nonce;          // ป้องกัน replay
        bool partialFill;       // อนุญาต partial fill หรือไม่
        OrderStatus status;     // สถานะ
        uint256 filledSell;     // จำนวนที่ขายไปแล้ว
        uint256 filledBuy;      // จำนวนที่ซื้อได้แล้ว
    }

    struct BatchSettlement {
        bytes32[] orderUIDs;        // orders ที่จะ settle
        uint256[] executedSell;     // จำนวนที่ขายจริง
        uint256[] executedBuy;      // จำนวนที่ซื้อจริง
        address[] tokens;           // tokens ที่เกี่ยวข้อง
        uint256[] clearingPrices;   // uniform clearing prices
        uint256 batchId;
    }

    struct TokenBalance {
        uint256 balance;
        uint256 lockedForOrders;
    }

    enum OrderStatus {
        Pending,
        PartiallyFilled,
        Filled,
        Cancelled,
        Expired
    }

    // ============ State Variables ============

    // orderUID => Order
    mapping(bytes32 => Order) public orders;
    
    // user => token => TokenBalance
    mapping(address => mapping(address => TokenBalance)) public balances;
    
    // user => nonce (ป้องกัน replay attack)
    mapping(address => uint256) public nonces;
    
    // solver ที่ได้รับอนุญาต (เพื่อ submit batch settlements)
    mapping(address => bool) public authorizedSolvers;
    
    // batch counter
    uint256 public currentBatchId;
    
    // ค่าธรรมเนียม (basis points, 1 = 0.01%)
    uint256 public constant FEE_BPS = 10; // 0.1%
    uint256 public constant BPS_DENOMINATOR = 10000;
    
    // ระยะเวลา batch (ทุก N วินาที settle หนึ่งครั้ง)
    uint256 public batchInterval = 30; // 30 วินาที
    uint256 public lastBatchTimestamp;
    
    // accumulated fees
    mapping(address => uint256) public protocolFees;

    // ============ Events ============

    event OrderPlaced(
        bytes32 indexed orderUID,
        address indexed owner,
        address sellToken,
        address buyToken,
        uint256 sellAmount,
        uint256 buyAmountMin
    );
    
    event OrderCancelled(bytes32 indexed orderUID, address indexed owner);
    
    event BatchSettled(
        uint256 indexed batchId,
        uint256 ordersSettled,
        address indexed solver
    );
    
    event OrderFilled(
        bytes32 indexed orderUID,
        uint256 executedSell,
        uint256 executedBuy,
        uint256 fee
    );
    
    event Deposit(address indexed user, address indexed token, uint256 amount);
    event Withdrawal(address indexed user, address indexed token, uint256 amount);
    event SolverAuthorized(address indexed solver, bool authorized);

    // ============ Modifiers ============

    modifier onlySolver() {
        require(authorizedSolvers[msg.sender], "Not authorized solver");
        _;
    }

    modifier orderExists(bytes32 _orderUID) {
        require(orders[_orderUID].owner != address(0), "Order does not exist");
        _;
    }

    // ============ Constructor ============

    constructor() Ownable(msg.sender) {
        lastBatchTimestamp = block.timestamp;
        authorizedSolvers[msg.sender] = true; // owner เป็น solver เริ่มต้น
    }

    // ============ User Functions ============

    /**
     * @notice ฝาก token เข้า DEX
     */
    function deposit(address _token, uint256 _amount) external nonReentrant {
        require(_amount > 0, "Amount must be positive");
        require(_token != address(0), "Invalid token");
        
        IERC20(_token).safeTransferFrom(msg.sender, address(this), _amount);
        balances[msg.sender][_token].balance += _amount;
        
        emit Deposit(msg.sender, _token, _amount);
    }

    /**
     * @notice ถอน token ออกจาก DEX
     */
    function withdraw(address _token, uint256 _amount) external nonReentrant {
        TokenBalance storage tokenBal = balances[msg.sender][_token];
        uint256 available = tokenBal.balance - tokenBal.lockedForOrders;
        require(available >= _amount, "Insufficient available balance");
        
        tokenBal.balance -= _amount;
        IERC20(_token).safeTransfer(msg.sender, _amount);
        
        emit Withdrawal(msg.sender, _token, _amount);
    }

    /**
     * @notice วาง order ใหม่
     * @param _sellToken token ที่ต้องการขาย
     * @param _buyToken token ที่ต้องการซื้อ
     * @param _sellAmount จำนวนที่ขาย
     * @param _buyAmountMin จำนวนขั้นต่ำที่ต้องการซื้อ
     * @param _validDuration ระยะเวลาที่ order ยังใช้ได้ (วินาที)
     * @param _partialFill อนุญาต partial fill หรือไม่
     */
    function placeOrder(
        address _sellToken,
        address _buyToken,
        uint256 _sellAmount,
        uint256 _buyAmountMin,
        uint256 _validDuration,
        bool _partialFill
    ) external nonReentrant returns (bytes32 orderUID) {
        require(_sellToken != _buyToken, "Same token");
        require(_sellAmount > 0, "Sell amount must be positive");
        require(_buyAmountMin > 0, "Buy amount min must be positive");
        require(_validDuration > 0 && _validDuration <= 7 days, "Invalid duration");
        
        // ตรวจสอบ balance
        uint256 available = balances[msg.sender][_sellToken].balance 
            - balances[msg.sender][_sellToken].lockedForOrders;
        require(available >= _sellAmount, "Insufficient balance");
        
        // Lock balance สำหรับ order นี้
        balances[msg.sender][_sellToken].lockedForOrders += _sellAmount;
        
        uint256 currentNonce = nonces[msg.sender]++;
        
        // สร้าง unique order ID
        orderUID = keccak256(abi.encodePacked(
            msg.sender,
            _sellToken,
            _buyToken,
            _sellAmount,
            _buyAmountMin,
            block.timestamp + _validDuration,
            currentNonce
        ));
        
        orders[orderUID] = Order({
            owner: msg.sender,
            sellToken: _sellToken,
            buyToken: _buyToken,
            sellAmount: _sellAmount,
            buyAmountMin: _buyAmountMin,
            validTo: block.timestamp + _validDuration,
            nonce: currentNonce,
            partialFill: _partialFill,
            status: OrderStatus.Pending,
            filledSell: 0,
            filledBuy: 0
        });
        
        emit OrderPlaced(
            orderUID, 
            msg.sender, 
            _sellToken, 
            _buyToken, 
            _sellAmount, 
            _buyAmountMin
        );
    }

    /**
     * @notice ยกเลิก order
     */
    function cancelOrder(bytes32 _orderUID) 
        external 
        nonReentrant 
        orderExists(_orderUID) 
    {
        Order storage order = orders[_orderUID];
        require(order.owner == msg.sender, "Not order owner");
        require(
            order.status == OrderStatus.Pending || 
            order.status == OrderStatus.PartiallyFilled,
            "Cannot cancel"
        );
        
        // คืน locked balance
        uint256 remainingSell = order.sellAmount - order.filledSell;
        balances[msg.sender][order.sellToken].lockedForOrders -= remainingSell;
        
        order.status = OrderStatus.Cancelled;
        emit OrderCancelled(_orderUID, msg.sender);
    }

    // ============ Solver Functions ============

    /**
     * @notice Settle batch ของ orders พร้อมกัน
     * @dev Core function ที่ทำให้ DEX นี้ป้องกัน MEV:
     *      - ทุก order ใน batch ได้รับ uniform clearing price
     *      - ไม่มีลำดับที่ได้เปรียบกว่า
     *      - Solver ต้องหา solution ที่ maximize surplus ให้ users
     */
    function settleBatch(BatchSettlement calldata _settlement) 
        external 
        nonReentrant 
        onlySolver 
    {
        require(
            block.timestamp >= lastBatchTimestamp + batchInterval,
            "Batch interval not reached"
        );
        require(_settlement.batchId == currentBatchId, "Invalid batch ID");
        require(
            _settlement.orderUIDs.length == _settlement.executedSell.length &&
            _settlement.orderUIDs.length == _settlement.executedBuy.length,
            "Array length mismatch"
        );
        
        // ตรวจสอบ clearing prices (ต้องสอดคล้องกัน)
        _validateClearingPrices(
            _settlement.tokens, 
            _settlement.clearingPrices
        );
        
        uint256 ordersSettled = 0;
        
        for (uint256 i = 0; i < _settlement.orderUIDs.length; i++) {
            bytes32 uid = _settlement.orderUIDs[i];
            uint256 execSell = _settlement.executedSell[i];
            uint256 execBuy = _settlement.executedBuy[i];
            
            if (_settleOrder(uid, execSell, execBuy)) {
                ordersSettled++;
            }
        }
        
        lastBatchTimestamp = block.timestamp;
        currentBatchId++;
        
        emit BatchSettled(currentBatchId - 1, ordersSettled, msg.sender);
    }

    /**
     * @dev Settle order เดี่ยว ภายใน batch
     */
    function _settleOrder(
        bytes32 _orderUID,
        uint256 _execSell,
        uint256 _execBuy
    ) internal returns (bool success) {
        if (orders[_orderUID].owner == address(0)) return false;
        
        Order storage order = orders[_orderUID];
        
        // ตรวจสอบ status
        if (order.status != OrderStatus.Pending && 
            order.status != OrderStatus.PartiallyFilled) {
            return false;
        }
        
        // ตรวจสอบ expiry
        if (block.timestamp > order.validTo) {
            order.status = OrderStatus.Expired;
            // คืน locked balance
            uint256 remaining = order.sellAmount - order.filledSell;
            balances[order.owner][order.sellToken].lockedForOrders -= remaining;
            return false;
        }
        
        // ตรวจสอบว่า execSell ไม่เกิน remaining
        uint256 remainingSell = order.sellAmount - order.filledSell;
        if (_execSell > remainingSell) return false;
        if (!order.partialFill && _execSell < remainingSell) return false;
        
        // ตรวจสอบ minimum buy amount (ตามสัดส่วน)
        uint256 minBuyForExec = (order.buyAmountMin * _execSell) / order.sellAmount;
        if (_execBuy < minBuyForExec) return false;
        
        // คำนวณ fee
        uint256 fee = (_execBuy * FEE_BPS) / BPS_DENOMINATOR;
        uint256 netBuy = _execBuy - fee;
        
        // อัปเดต balances
        // Seller: เสีย sellToken, ได้ buyToken
        balances[order.owner][order.sellToken].balance -= _execSell;
        balances[order.owner][order.sellToken].lockedForOrders -= _execSell;
        balances[order.owner][order.buyToken].balance += netBuy;
        
        // Collect protocol fee
        protocolFees[order.buyToken] += fee;
        
        // อัปเดต order
        order.filledSell += _execSell;
        order.filledBuy += _execBuy;
        
        if (order.filledSell >= order.sellAmount) {
            order.status = OrderStatus.Filled;
        } else {
            order.status = OrderStatus.PartiallyFilled;
        }
        
        emit OrderFilled(_orderUID, _execSell, _execBuy, fee);
        return true;
    }

    /**
     * @dev ตรวจสอบว่า clearing prices สอดคล้องกัน
     *      (ราคา A/B และ B/A ต้องไม่ขัดแย้งกัน)
     */
    function _validateClearingPrices(
        address[] calldata _tokens,
        uint256[] calldata _prices
    ) internal pure {
        require(_tokens.length == _prices.length, "Mismatched price arrays");
        
        for (uint256 i = 0; i < _prices.length; i++) {
            require(_prices[i] > 0, "Zero clearing price");
        }
        
        // ในการใช้งานจริง ต้องตรวจสอบ arbitrage-free conditions
        // สำหรับตัวอย่างนี้ simplified
    }

    // ============ Admin Functions ============

    function authorizeSolver(address _solver, bool _authorized) external onlyOwner {
        authorizedSolvers[_solver] = _authorized;
        emit SolverAuthorized(_solver, _authorized);
    }

    function setBatchInterval(uint256 _interval) external onlyOwner {
        require(_interval >= 12 && _interval <= 300, "Invalid interval");
        batchInterval = _interval;
    }

    function collectProtocolFees(address _token, address _recipient) 
        external 
        onlyOwner 
    {
        uint256 amount = protocolFees[_token];
        require(amount > 0, "No fees to collect");
        protocolFees[_token] = 0;
        IERC20(_token).safeTransfer(_recipient, amount);
    }

    // ============ View Functions ============

    function getOrder(bytes32 _orderUID) external view returns (Order memory) {
        return orders[_orderUID];
    }

    function getAvailableBalance(
        address _user, 
        address _token
    ) external view returns (uint256) {
        TokenBalance memory tb = balances[_user][_token];
        return tb.balance - tb.lockedForOrders;
    }

    function isOrderActive(bytes32 _orderUID) external view returns (bool) {
        Order memory order = orders[_orderUID];
        return (
            order.owner != address(0) &&
            (order.status == OrderStatus.Pending || 
             order.status == OrderStatus.PartiallyFilled) &&
            block.timestamp <= order.validTo
        );
    }
    
    /**
     * @notice คำนวณ surplus ที่ user ได้รับจากการ settle
     * @dev ใช้เปรียบเทียบกับ limit price ที่ user ตั้งไว้
     */
    function calculateSurplus(
        bytes32 _orderUID,
        uint256 _execSell,
        uint256 _execBuy
    ) external view returns (uint256 surplus) {
        Order memory order = orders[_orderUID];
        if (order.owner == address(0)) return 0;
        
        // ราคา limit ของ user: buyAmountMin / sellAmount
        // ถ้า execBuy > (buyAmountMin * execSell / sellAmount) = surplus
        uint256 minBuy = (order.buyAmountMin * _execSell) / order.sellAmount;
        if (_execBuy > minBuy) {
            surplus = _execBuy - minBuy;
        }
    }
}
```

---

## 81.7 Advanced MEV: Atomic Arbitrage Bot

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title FlashArbitrageBot
 * @notice ตัวอย่าง atomic arbitrage โดยใช้ flash loan
 * @dev สำหรับการศึกษาเท่านั้น - ในการใช้งานจริงต้องเพิ่ม
 *      - Profitability check ก่อน execute
 *      - Gas estimation
 *      - MEV bundle submission
 */
interface IFlashLoanProvider {
    function flashLoan(
        address receiver,
        address token,
        uint256 amount,
        bytes calldata data
    ) external;
}

interface IDEXRouter {
    function swapExactTokensForTokens(
        uint256 amountIn,
        uint256 amountOutMin,
        address[] calldata path,
        address to,
        uint256 deadline
    ) external returns (uint256[] memory amounts);
    
    function getAmountsOut(
        uint256 amountIn,
        address[] calldata path
    ) external view returns (uint256[] memory amounts);
}

interface IERC20Minimal {
    function approve(address spender, uint256 amount) external returns (bool);
    function transfer(address to, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

contract FlashArbitrageBot {
    address public immutable owner;
    address public immutable flashLoanProvider;
    
    // ค่าธรรมเนียม flash loan (0.09% = 9 BPS)
    uint256 constant FLASH_LOAN_FEE_BPS = 9;
    uint256 constant BPS_DENOM = 10000;
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    constructor(address _flashLoanProvider) {
        owner = msg.sender;
        flashLoanProvider = _flashLoanProvider;
    }
    
    /**
     * @notice เริ่ม arbitrage โดยใช้ flash loan
     * @param _token token ที่จะกู้
     * @param _amount จำนวนที่กู้
     * @param _dexA DEX แรก (ซื้อถูก)
     * @param _dexB DEX สอง (ขายแพง)
     * @param _path เส้นทางการแลกเปลี่ยน
     */
    function executeArbitrage(
        address _token,
        uint256 _amount,
        address _dexA,
        address _dexB,
        address[] calldata _path
    ) external onlyOwner {
        bytes memory data = abi.encode(_dexA, _dexB, _path, _amount);
        IFlashLoanProvider(flashLoanProvider).flashLoan(
            address(this),
            _token,
            _amount,
            data
        );
    }
    
    /**
     * @notice Callback จาก flash loan provider
     * @dev ต้อง return token + fee ภายใน function นี้
     */
    function onFlashLoan(
        address /* initiator */,
        address _token,
        uint256 _amount,
        uint256 _fee,
        bytes calldata _data
    ) external returns (bytes32) {
        require(msg.sender == flashLoanProvider, "Unauthorized");
        
        (
            address dexA, 
            address dexB, 
            address[] memory path,
            uint256 borrowAmount
        ) = abi.decode(_data, (address, address, address[], uint256));
        
        uint256 repayAmount = borrowAmount + _fee;
        
        // Step 1: Swap ที่ DEX A (ได้ tokenB)
        IERC20Minimal(_token).approve(dexA, borrowAmount);
        
        address[] memory reversePath = new address[](path.length);
        for (uint i = 0; i < path.length; i++) {
            reversePath[i] = path[path.length - 1 - i];
        }
        
        uint256[] memory amountsA = IDEXRouter(dexA).swapExactTokensForTokens(
            borrowAmount,
            0, // ใน production ต้องระบุ slippage
            path,
            address(this),
            block.timestamp
        );
        
        uint256 receivedAmount = amountsA[amountsA.length - 1];
        address intermediateToken = path[path.length - 1];
        
        // Step 2: Swap ที่ DEX B (ได้ tokenA คืน)
        IERC20Minimal(intermediateToken).approve(dexB, receivedAmount);
        
        uint256[] memory amountsB = IDEXRouter(dexB).swapExactTokensForTokens(
            receivedAmount,
            repayAmount, // ต้องได้คืนมากกว่า repayAmount
            reversePath,
            address(this),
            block.timestamp
        );
        
        uint256 finalAmount = amountsB[amountsB.length - 1];
        require(finalAmount >= repayAmount, "Arbitrage not profitable");
        
        // Step 3: คืน flash loan
        IERC20Minimal(_token).approve(flashLoanProvider, repayAmount);
        
        // กำไรที่เหลือจะอยู่ใน contract
        uint256 profit = finalAmount - repayAmount;
        
        // ส่งกำไรให้ owner
        if (profit > 0) {
            IERC20Minimal(_token).transfer(owner, profit);
        }
        
        return keccak256("ERC3156FlashBorrower.onFlashLoan");
    }
    
    /**
     * @notice ตรวจสอบกำไรก่อน execute (simulation)
     */
    function checkProfitability(
        address _token,
        uint256 _amount,
        address _dexA,
        address _dexB,
        address[] calldata _path
    ) external view returns (int256 estimatedProfit) {
        // Simulate swap ที่ DEX A
        uint256[] memory amountsA = IDEXRouter(_dexA).getAmountsOut(_amount, _path);
        uint256 intermediate = amountsA[amountsA.length - 1];
        
        // Reverse path
        address[] memory reversePath = new address[](_path.length);
        for (uint i = 0; i < _path.length; i++) {
            reversePath[i] = _path[_path.length - 1 - i];
        }
        
        // Simulate swap ที่ DEX B
        uint256[] memory amountsB = IDEXRouter(_dexB).getAmountsOut(intermediate, reversePath);
        uint256 finalAmount = amountsB[amountsB.length - 1];
        
        // คำนวณค่า flash loan fee
        uint256 flashFee = (_amount * FLASH_LOAN_FEE_BPS) / BPS_DENOM;
        uint256 repayAmount = _amount + flashFee;
        
        estimatedProfit = int256(finalAmount) - int256(repayAmount);
    }
    
    // Emergency withdraw
    function withdrawToken(address _token) external onlyOwner {
        uint256 balance = IERC20Minimal(_token).balanceOf(address(this));
        if (balance > 0) {
            IERC20Minimal(_token).transfer(owner, balance);
        }
    }
    
    receive() external payable {}
    
    function withdrawETH() external onlyOwner {
        payable(owner).transfer(address(this).balance);
    }
}
```

---

## 81.8 Anti-Sandwich Protection: Slippage Manager

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title SlippageManager
 * @notice Helper contract สำหรับคำนวณ dynamic slippage
 *         เพื่อป้องกัน sandwich attacks
 */
contract SlippageManager {
    
    // ประวัติ price volatility
    struct VolatilityData {
        uint256[] priceHistory;   // ราคา 10 ครั้งล่าสุด
        uint256 historyIndex;
        uint256 lastUpdate;
        uint256 volatilityScore; // 0-10000 (0.00%-100.00%)
    }
    
    mapping(address => mapping(address => VolatilityData)) public volatility;
    
    uint256 constant HISTORY_SIZE = 10;
    uint256 constant BASE_SLIPPAGE_BPS = 50;   // 0.5% base
    uint256 constant MAX_SLIPPAGE_BPS = 300;    // 3% max
    
    /**
     * @notice คำนวณ slippage ที่เหมาะสมตาม volatility
     * @return recommendedSlippageBps slippage ที่แนะนำ (basis points)
     */
    function getRecommendedSlippage(
        address _tokenA,
        address _tokenB,
        uint256 _tradeSize,
        uint256 _poolLiquidity
    ) external view returns (uint256 recommendedSlippageBps) {
        // 1. Base slippage จาก volatility
        uint256 volScore = volatility[_tokenA][_tokenB].volatilityScore;
        uint256 volSlippage = (volScore * BASE_SLIPPAGE_BPS) / 1000;
        
        // 2. Impact slippage จากขนาด trade
        uint256 impactBps = 0;
        if (_poolLiquidity > 0) {
            // เปอร์เซ็นต์ของ pool ที่ trade ใช้
            uint256 tradePercent = (_tradeSize * 10000) / _poolLiquidity;
            impactBps = tradePercent / 2; // x/2 approximation
        }
        
        // 3. รวมกัน
        recommendedSlippageBps = BASE_SLIPPAGE_BPS + volSlippage + impactBps;
        
        // Cap ที่ MAX_SLIPPAGE_BPS
        if (recommendedSlippageBps > MAX_SLIPPAGE_BPS) {
            recommendedSlippageBps = MAX_SLIPPAGE_BPS;
        }
    }
    
    /**
     * @notice อัปเดต volatility data
     */
    function updatePrice(
        address _tokenA,
        address _tokenB,
        uint256 _currentPrice
    ) external {
        VolatilityData storage data = volatility[_tokenA][_tokenB];
        
        if (data.priceHistory.length < HISTORY_SIZE) {
            data.priceHistory.push(_currentPrice);
        } else {
            data.priceHistory[data.historyIndex] = _currentPrice;
            data.historyIndex = (data.historyIndex + 1) % HISTORY_SIZE;
        }
        
        data.lastUpdate = block.timestamp;
        
        // คำนวณ volatility score ใหม่
        if (data.priceHistory.length >= 2) {
            data.volatilityScore = _calculateVolatility(data.priceHistory);
        }
    }
    
    function _calculateVolatility(
        uint256[] memory _prices
    ) internal pure returns (uint256 score) {
        if (_prices.length < 2) return 0;
        
        uint256 maxPrice = _prices[0];
        uint256 minPrice = _prices[0];
        
        for (uint256 i = 1; i < _prices.length; i++) {
            if (_prices[i] > maxPrice) maxPrice = _prices[i];
            if (_prices[i] < minPrice) minPrice = _prices[i];
        }
        
        if (minPrice == 0) return 10000; // max volatility
        
        // Range / Min price as volatility proxy
        score = ((maxPrice - minPrice) * 10000) / minPrice;
        if (score > 10000) score = 10000;
    }
}
```

---

## 81.9 Workshop: สร้าง MEV Simulation

### ภาระกิจที่ 1: Sandwich Attack Simulation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title SandwichSimulator
 * @notice จำลองการโจมตีแบบ sandwich เพื่อทำความเข้าใจ
 *         และทดสอบระบบป้องกัน
 * 
 * Workshop: ลองรัน simulation แล้วดูว่า:
 * 1. เหยื่อสูญเสียเท่าไร
 * 2. Attacker ได้กำไรเท่าไร
 * 3. ค่า slippage เท่าไรที่ทำให้การโจมตีไม่คุ้ม
 */
contract SandwichSimulator {
    
    // Simple AMM state (x * y = k)
    struct Pool {
        uint256 reserveA;
        uint256 reserveB;
        uint256 fee; // fee BPS
    }
    
    Pool public pool;
    
    constructor(uint256 _reserveA, uint256 _reserveB) {
        pool = Pool({
            reserveA: _reserveA,
            reserveB: _reserveB,
            fee: 30 // 0.3%
        });
    }
    
    struct SimulationResult {
        uint256 attackerBuyAmount;      // จำนวนที่ attacker ซื้อ
        uint256 attackerReceived;       // ที่ attacker ได้จากการซื้อ
        uint256 victimExpected;         // ที่เหยื่อคาดหวัง
        uint256 victimActualReceived;   // ที่เหยื่อได้จริง
        uint256 victimLoss;             // ความเสียหายของเหยื่อ
        uint256 attackerProfit;         // กำไรของ attacker (ETH terms)
        bool attackProfitable;          // คุ้มไหม
    }
    
    /**
     * @notice จำลอง sandwich attack
     */
    function simulateSandwich(
        uint256 _victimAmountIn,    // จำนวน token A ที่เหยื่อแลก
        uint256 _victimSlippage,    // max slippage (BPS)
        uint256 _attackerFrontrun,  // จำนวน token A ที่ attacker frontrun
        uint256 _gasPrice,          // ราคา gas
        uint256 _gasUsed            // gas ที่ใช้
    ) external view returns (SimulationResult memory result) {
        Pool memory p = pool;
        
        // ราคาก่อนโจมตี
        uint256 priceBeforeAttack = (p.reserveB * 1e18) / p.reserveA;
        
        // Step 1: Attacker front-runs
        uint256 attackerReceived = _getAmountOut(_attackerFrontrun, p.reserveA, p.reserveB, p.fee);
        p.reserveA += _attackerFrontrun;
        p.reserveB -= attackerReceived;
        
        // Step 2: เหยื่อ trade (ในราคาที่สูงขึ้น)
        uint256 victimReceived = _getAmountOut(_victimAmountIn, p.reserveA, p.reserveB, p.fee);
        
        // คำนวณว่าเหยื่อคาดหวังเท่าไร (ณ ราคาก่อนโจมตี)
        uint256 victimExpected = _getAmountOut(
            _victimAmountIn, 
            pool.reserveA, 
            pool.reserveB, 
            pool.fee
        );
        
        // ตรวจสอบ slippage
        uint256 minReceived = victimExpected * (10000 - _victimSlippage) / 10000;
        bool victimTxSucceeds = victimReceived >= minReceived;
        
        if (!victimTxSucceeds) {
            // Victim TX reverts - sandwich ไม่สำเร็จ
            result.attackProfitable = false;
            return result;
        }
        
        p.reserveA += _victimAmountIn;
        p.reserveB -= victimReceived;
        
        // Step 3: Attacker back-runs (ขายสิ่งที่ได้จาก frontrun)
        uint256 attackerBackrunReturn = _getAmountOut(
            attackerReceived, 
            p.reserveB, 
            p.reserveA, 
            p.fee
        );
        
        // คำนวณกำไร/ขาดทุน
        uint256 gasCost = _gasPrice * _gasUsed;
        
        result.attackerBuyAmount = _attackerFrontrun;
        result.attackerReceived = attackerReceived;
        result.victimExpected = victimExpected;
        result.victimActualReceived = victimReceived;
        
        if (victimReceived < victimExpected) {
            result.victimLoss = victimExpected - victimReceived;
        }
        
        if (attackerBackrunReturn > _attackerFrontrun + gasCost) {
            result.attackerProfit = attackerBackrunReturn - _attackerFrontrun - gasCost;
            result.attackProfitable = true;
        } else {
            result.attackProfitable = false;
        }
    }
    
    function _getAmountOut(
        uint256 _amountIn,
        uint256 _reserveIn,
        uint256 _reserveOut,
        uint256 _feeBps
    ) internal pure returns (uint256 amountOut) {
        uint256 amountInWithFee = _amountIn * (10000 - _feeBps);
        uint256 numerator = amountInWithFee * _reserveOut;
        uint256 denominator = (_reserveIn * 10000) + amountInWithFee;
        amountOut = numerator / denominator;
    }
}
```

### ภาระกิจที่ 2: ทดสอบ MEVProtectedDEX

```javascript
// Workshop test script (Hardhat/Foundry)

// Test 1: วาง order และ settle batch
const sellToken = await deployMockToken("TokenA", 18);
const buyToken = await deployMockToken("TokenB", 18);

// Mint tokens ให้ users
await sellToken.mint(alice.address, ethers.parseEther("1000"));
await sellToken.mint(bob.address, ethers.parseEther("1000"));

// Deploy DEX
const dex = await deploy("MEVProtectedDEX");

// Deposit
await sellToken.connect(alice).approve(dex.address, ethers.MaxUint256);
await dex.connect(alice).deposit(sellToken.address, ethers.parseEther("100"));

// Place orders (Alice ขาย TokenA, Bob ขาย TokenB)
const order1UID = await dex.connect(alice).placeOrder(
  sellToken.address,
  buyToken.address,
  ethers.parseEther("10"),   // ขาย 10 TokenA
  ethers.parseEther("9"),    // ต้องการ TokenB อย่างน้อย 9
  3600,                      // หมดอายุ 1 ชั่วโมง
  true                       // partial fill OK
);

// Test 2: ตรวจสอบว่าไม่มีใน public mempool
// (ในการทดสอบ production ต้องใช้ Flashbots RPC)

// Test 3: Settle batch
const settlement = {
  orderUIDs: [order1UID],
  executedSell: [ethers.parseEther("10")],
  executedBuy: [ethers.parseEther("9.5")], // ได้ surplus 0.5
  tokens: [sellToken.address, buyToken.address],
  clearingPrices: [ethers.parseEther("0.95"), ethers.parseEther("1")],
  batchId: 0
};

await dex.settleBatch(settlement);

// ตรวจสอบ surplus ที่ Alice ได้รับ
const aliceBuyBalance = await dex.getAvailableBalance(alice.address, buyToken.address);
console.log("Alice received:", ethers.formatEther(aliceBuyBalance)); // ควรได้ ~9.5 (ลบ fee)
```

---

## สรุป Part 81

- **MEV** คือมูลค่าที่ดึงได้จากการจัดลำดับธุรกรรม มี 5 ประเภทหลัก: arbitrage, liquidations, sandwich attacks, JIT liquidity, และ long-tail MEV
- **PBS (Proposer-Builder Separation)** แยกหน้าที่การสร้างบล็อก (builder) และการเสนอบล็อก (proposer) เพื่อลด centralization pressure
- **mev-boost** คือ middleware ที่ทำให้ validators สามารถรับ block ที่ดีที่สุดจาก builder marketplace
- **SUAVE** คือ vision ระยะยาวของ Flashbots สำหรับ decentralized MEV infrastructure
- **MEV-resistant design** ต้องใช้: commit-reveal, TWAP pricing, batch auctions, และ private mempool
- **MEVProtectedDEX** ใช้ batch auction + uniform clearing price เพื่อ eliminate sandwich attacks
- Sandwich attacks ทำให้ผู้ใช้สูญเสียกว่า $60M/ปี → เป็นปัญหาที่ต้องแก้ไขในระดับ protocol

## Next: Part 82 - Privacy in DeFi (Tornado Cash, Stealth Addresses, ZK Voting)
