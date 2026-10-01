# Part 25: MEV และ Front-Running Protection

## สารบัญ
1. MEV คืออะไร
2. Types of MEV
3. Flashbots และ PBS
4. Protection Mechanisms
5. Workshop: MEV-Resistant Auction

---

## 1. MEV คืออะไร

```
MEV (Maximal Extractable Value / Miner Extractable Value):
กำไรที่ block producers สามารถหาได้จากการ
- จัดลำดับ transactions
- Include/exclude transactions
- Insert transactions ของตัวเอง

Sandwich Attack:
1. Bot เห็น tx ของ user ใน mempool (large swap)
2. Bot ส่ง buy tx ก่อน (frontrun) → ราคาขึ้น
3. User tx ทำงาน → ซื้อได้ราคาแพงกว่า
4. Bot ขาย (backrun) → profit

Arbitrage MEV:
- Price difference ระหว่าง DEX
- Bot ซื้อถูก ขายแพง ทันที (atomic)

Liquidation MEV:
- Monitor unhealthy positions
- Race to liquidate first (get bonus)

JIT (Just-In-Time) Liquidity:
- Add liquidity ก่อน tx → earn fee
- Remove liquidity หลัง tx
- ทำให้ real LPs ได้ fee น้อยลง
```

---

## 2. Flashbots และ MEV-Boost

```
Traditional Mining:
Miner เห็น public mempool → เลือก tx ตาม gas price

Flashbots:
- Private transaction relay
- Bundle: กลุ่ม tx ที่ทำงาน atomically
- Searcher ส่ง bundle ผ่าน Flashbots RPC
- Miner/Validator เลือก bundle ที่ให้ profit สูงสุด

PBS (Proposer-Builder Separation):
- Builder: รวบ transactions สร้าง block
- Proposer: เลือก block สูงสุด (blind bid)
- ลด centralization risk

MEV-Boost:
- Ethereum post-Merge
- Validator ใช้ external block builders
- แยก block building ออกจาก validation
```

---

## 3. Protection Mechanisms

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * 1. Slippage Protection (Basic)
 * กำหนด minimum output ที่ยอมรับได้
 */
contract SlippageProtectedSwap {
    
    IMiniDEX public immutable dex;
    
    function swapWithSlippage(
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 minAmountOut, // ต่ำสุดที่ยอมรับ
        uint256 deadline      // หมดอายุ tx
    ) external {
        require(block.timestamp <= deadline, "Expired");
        
        uint256 amountOut = dex.swap(tokenIn, tokenOut, amountIn);
        
        require(amountOut >= minAmountOut, "Slippage too high");
        
        IERC20(tokenOut).transfer(msg.sender, amountOut);
    }
}

/**
 * 2. Commit-Reveal Scheme
 * ซ่อน intent ก่อน reveal → ป้องกัน frontrun
 */
contract CommitRevealAuction {
    
    struct Bid {
        bytes32 commit;
        uint256 amount;
        bool revealed;
    }
    
    mapping(address => Bid) public bids;
    
    uint256 public commitDeadline;
    uint256 public revealDeadline;
    
    address public highestBidder;
    uint256 public highestBid;
    
    event CommitMade(address indexed bidder);
    event BidRevealed(address indexed bidder, uint256 amount);
    event AuctionEnded(address indexed winner, uint256 amount);
    
    constructor(uint256 commitPeriod, uint256 revealPeriod) {
        commitDeadline = block.timestamp + commitPeriod;
        revealDeadline = commitDeadline + revealPeriod;
    }
    
    // Commit: hash(amount, secret) → ไม่มีใครรู้ bid amount
    function commit(bytes32 commitHash) external payable {
        require(block.timestamp <= commitDeadline, "Commit period ended");
        require(msg.value > 0, "Must send ETH");
        
        bids[msg.sender] = Bid({
            commit: commitHash,
            amount: msg.value, // deposit (refund ถ้าแพ้)
            revealed: false
        });
        
        emit CommitMade(msg.sender);
    }
    
    // Reveal: เปิดเผย amount จริงและ secret
    function reveal(uint256 amount, bytes32 secret) external {
        require(block.timestamp > commitDeadline, "Still in commit phase");
        require(block.timestamp <= revealDeadline, "Reveal period ended");
        
        Bid storage bid = bids[msg.sender];
        require(!bid.revealed, "Already revealed");
        
        // Verify commit matches
        bytes32 expectedCommit = keccak256(abi.encodePacked(amount, secret));
        require(bid.commit == expectedCommit, "Commit mismatch");
        
        bid.revealed = true;
        
        if (amount > highestBid) {
            highestBid = amount;
            highestBidder = msg.sender;
        }
        
        emit BidRevealed(msg.sender, amount);
    }
    
    function endAuction() external {
        require(block.timestamp > revealDeadline, "Reveal not ended");
        emit AuctionEnded(highestBidder, highestBid);
    }
    
    // สร้าง commit hash (off-chain helper)
    function hashBid(uint256 amount, bytes32 secret) external pure returns (bytes32) {
        return keccak256(abi.encodePacked(amount, secret));
    }
}

/**
 * 3. Private Mempool (Flashbots)
 * ส่ง tx ผ่าน private relay ไม่ผ่าน public mempool
 */
// ไม่ใช่ smart contract แต่เป็น ethers.js code:
/*
import { FlashbotsBundleProvider } from "@flashbots/ethers-provider-bundle";

const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
const flashbotsProvider = await FlashbotsBundleProvider.create(
    provider,
    wallet,
    "https://relay.flashbots.net" // mainnet
);

const bundle = [
    {
        signer: wallet,
        transaction: {
            to: contractAddress,
            data: calldata,
            gasLimit: 200000,
        }
    }
];

const targetBlock = (await provider.getBlockNumber()) + 1;
const signedBundle = await flashbotsProvider.signBundle(bundle);
const simulation = await flashbotsProvider.simulate(signedBundle, targetBlock);

if ("error" in simulation) {
    console.error("Simulation failed:", simulation.error);
} else {
    const receipt = await flashbotsProvider.sendRawBundle(signedBundle, targetBlock);
    console.log("Bundle included:", receipt);
}
*/

/**
 * 4. Time-Weighted Average Price (TWAP)
 * ใช้ราคาเฉลี่ยแทนราคา spot → ป้องกัน oracle manipulation
 */
contract TWAPOracle {
    
    struct Observation {
        uint256 timestamp;
        uint256 price0Cumulative;
        uint256 price1Cumulative;
    }
    
    Observation[] public observations;
    
    uint256 public constant PERIOD = 30 minutes;
    
    IUniswapV2Pair public immutable pair;
    
    constructor(address _pair) {
        pair = IUniswapV2Pair(_pair);
        _update();
    }
    
    function _update() internal {
        (uint112 reserve0, uint112 reserve1, uint32 blockTimestampLast) = pair.getReserves();
        
        observations.push(Observation({
            timestamp: block.timestamp,
            price0Cumulative: pair.price0CumulativeLast(),
            price1Cumulative: pair.price1CumulativeLast()
        }));
    }
    
    // Update TWAP (anyone can call)
    function update() external {
        require(
            observations.length == 0 || 
            block.timestamp - observations[observations.length - 1].timestamp >= PERIOD,
            "Too soon"
        );
        _update();
    }
    
    // Get TWAP price (token0 in terms of token1)
    function consult(address token, uint256 amountIn) external view returns (uint256) {
        require(observations.length >= 2, "Not enough observations");
        
        Observation storage first = observations[observations.length - 2];
        Observation storage last = observations[observations.length - 1];
        
        uint256 timeElapsed = last.timestamp - first.timestamp;
        
        uint256 priceCumulativeDiff = last.price0Cumulative - first.price0Cumulative;
        
        // TWAP = priceCumulative diff / time elapsed
        uint256 twap = priceCumulativeDiff / timeElapsed;
        
        return (amountIn * twap) >> 112; // UQ112x112 format
    }
}

interface IUniswapV2Pair {
    function getReserves() external view returns (uint112, uint112, uint32);
    function price0CumulativeLast() external view returns (uint256);
    function price1CumulativeLast() external view returns (uint256);
}

interface IMiniDEX {
    function swap(address, address, uint256) external returns (uint256);
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
}
```

---

## 4. MEV-Resistant AMM

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Batch Auction AMM
 * รวบ orders ใน batch → ทำงาน end of block
 * ทุก order ใน batch ได้ราคาเดียวกัน (clearing price)
 * 
 * ป้องกัน sandwich attack ได้ดี
 * เพราะ frontrunner ไม่รู้ว่า batch จะ clear ที่ราคาไหน
 */
contract BatchAuction {
    
    IERC20 public immutable tokenA;
    IERC20 public immutable tokenB;
    
    struct Order {
        address trader;
        bool isBuy;      // true = ซื้อ A ด้วย B, false = ขาย A รับ B
        uint256 amount;  // amount of tokenA
        uint256 limit;   // limit price (B per A, 18 decimals)
        bool filled;
    }
    
    Order[] public orders;
    
    uint256 public currentBatch;
    uint256 public batchEnd;
    uint256 public constant BATCH_DURATION = 1 minutes;
    
    event OrderPlaced(uint256 indexed orderId, address trader, bool isBuy, uint256 amount);
    event BatchSettled(uint256 batchId, uint256 clearingPrice);
    
    constructor(address _tokenA, address _tokenB) {
        tokenA = IERC20(_tokenA);
        tokenB = IERC20(_tokenB);
        batchEnd = block.timestamp + BATCH_DURATION;
    }
    
    function placeOrder(
        bool isBuy,
        uint256 amount,
        uint256 limitPrice
    ) external returns (uint256 orderId) {
        // Collect tokens upfront
        if (isBuy) {
            uint256 maxPayment = (amount * limitPrice) / 1e18;
            tokenB.transferFrom(msg.sender, address(this), maxPayment);
        } else {
            tokenA.transferFrom(msg.sender, address(this), amount);
        }
        
        orderId = orders.length;
        orders.push(Order({
            trader: msg.sender,
            isBuy: isBuy,
            amount: amount,
            limit: limitPrice,
            filled: false
        }));
        
        emit OrderPlaced(orderId, msg.sender, isBuy, amount);
    }
    
    // Settle batch at clearing price
    function settleBatch(uint256 clearingPrice) external {
        require(block.timestamp >= batchEnd, "Batch not ended");
        
        uint256 batchId = currentBatch;
        
        // Find all orders at clearing price
        for (uint256 i = 0; i < orders.length; i++) {
            Order storage order = orders[i];
            if (order.filled) continue;
            
            if (order.isBuy && order.limit >= clearingPrice) {
                // Buy order: fills at clearingPrice
                order.filled = true;
                uint256 cost = (order.amount * clearingPrice) / 1e18;
                uint256 maxCost = (order.amount * order.limit) / 1e18;
                
                tokenA.transfer(order.trader, order.amount);
                if (maxCost > cost) {
                    tokenB.transfer(order.trader, maxCost - cost); // refund difference
                }
            } else if (!order.isBuy && order.limit <= clearingPrice) {
                // Sell order: fills at clearingPrice
                order.filled = true;
                uint256 proceeds = (order.amount * clearingPrice) / 1e18;
                tokenB.transfer(order.trader, proceeds);
            }
        }
        
        currentBatch++;
        batchEnd = block.timestamp + BATCH_DURATION;
        
        emit BatchSettled(batchId, clearingPrice);
    }
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
}
```

---

## สรุป Part 25

MEV ที่เรียนรู้:
- ✅ MEV types (sandwich, arbitrage, liquidation)
- ✅ Flashbots private mempool
- ✅ Slippage protection
- ✅ Commit-reveal scheme
- ✅ TWAP oracle (ป้องกัน manipulation)
- ✅ Batch auction AMM

## Quiz

1. Sandwich attack ทำงานอย่างไร? ป้องกันได้อย่างไร?
2. TWAP ช่วยป้องกัน oracle manipulation อย่างไร?
3. Commit-reveal มีข้อเสียอะไร?
4. Flashbots bundle คืออะไร?

---

## Next: Part 26 - Advanced Token Economics
