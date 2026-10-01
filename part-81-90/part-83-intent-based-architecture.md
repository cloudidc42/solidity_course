# Part 83: Intent-Based Architecture (สถาปัตยกรรมแบบ Intent)

## บทนำ

**Intent-Based Architecture** คือแนวคิดใหม่ในการออกแบบ DeFi ที่เปลี่ยนวิธีที่ผู้ใช้โต้ตอบกับ blockchain จากแบบ **Imperative** (สั่งทำทีละขั้นตอน) เป็นแบบ **Declarative** (บอกสิ่งที่ต้องการ แล้วปล่อยให้ระบบหาวิธีทำ)

ในปี 2024 Intent-based protocols กำลังครองตลาด:
- **UniswapX**: $5B+ volume ต่อเดือน
- **1inch Fusion**: ผู้ใช้ได้ราคาดีกว่า 0.3% เฉลี่ย
- **CoW Protocol**: MEV protection + coincidence of wants

---

## 83.1 Intents vs Transactions

### แบบดั้งเดิม (Imperative Transaction)

```
ผู้ใช้ต้องระบุทุกอย่าง:
- Contract address ที่จะเรียก
- Function ที่จะ call
- Parameters ทั้งหมด
- Gas limit
- Slippage
- Route การ swap (token A → token B → token C)
- DEX ที่จะใช้

ปัญหา:
✗ ผู้ใช้ต้องมีความรู้ทางเทคนิค
✗ ต้องเลือก route เอง อาจได้ราคาไม่ดีที่สุด
✗ เสี่ยง MEV (sandwich attacks)
✗ ต้องจ่าย gas ก่อน แม้จะ fail
```

### แบบ Intent (Declarative)

```
ผู้ใช้บอกแค่ว่าต้องการอะไร:
"ฉันต้องการแลก 1 ETH เป็น USDC ไม่น้อยกว่า $3,000
 ภายใน 10 นาที"

ระบบทำให้:
✓ Solver แข่งกันหา route ที่ดีที่สุด
✓ MEV protection อัตโนมัติ
✓ ไม่ต้องจ่าย gas ถ้าไม่ได้ราคาที่ต้องการ
✓ Cross-DEX, Cross-chain ได้โดยอัตโนมัติ
```

### Intent Stack

```
┌─────────────────────────────────────────────────────┐
│                   User Intent                        │
│  "Swap 1 ETH → ≥3000 USDC within 10 minutes"       │
└─────────────────────┬───────────────────────────────┘
                      │ Sign + Submit
┌─────────────────────▼───────────────────────────────┐
│              Intent Mempool / Orderbook              │
│         (UniswapX, 1inch, CoW Protocol)             │
└──────────┬──────────────────────────────────────────┘
           │ Compete for fill
  ┌────────┴──────────────────────┐
  │         Solvers/Fillers       │
  │  - UniswapX Fillers          │
  │  - Market Makers             │
  │  - Aggregators               │
  │  - MEV Searchers (backrun)   │
  └────────┬──────────────────────┘
           │ Execute on-chain
┌──────────▼──────────────────────────────────────────┐
│            Settlement Contract                       │
│     (validates + executes atomically)               │
└─────────────────────────────────────────────────────┘
```

---

## 83.2 ERC-7683: Cross-Chain Intent Standard

### 83.2.1 Core Interfaces

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title ERC-7683: Cross-Chain Intents Standard
 * @notice Interface สำหรับ cross-chain intent settlement
 * @dev https://eips.ethereum.org/EIPS/eip-7683
 */

// ============ Data Structures ============

/**
 * @notice Cross-chain order structure
 */
struct CrossChainOrder {
    address settlementContract;  // contract ที่ settle order นี้
    address swapper;             // ผู้สร้าง order
    uint256 nonce;               // unique nonce
    uint32 originChainId;        // chain ที่ order ถูกสร้าง
    uint32 initiateDeadline;     // deadline สำหรับ initiate
    uint32 fillDeadline;         // deadline สำหรับ fill
    bytes orderData;             // custom data (encoded per settlement contract)
}

/**
 * @notice Resolved cross-chain order (หลัง decode orderData)
 */
struct ResolvedCrossChainOrder {
    address settlementContract;
    address swapper;
    uint256 nonce;
    uint32 originChainId;
    uint32 initiateDeadline;
    uint32 fillDeadline;
    Input[] swapperInputs;      // tokens ที่ swapper ส่ง
    Output[] swapperOutputs;    // tokens ที่ swapper ต้องการ
    Output[] fillerOutputs;     // tokens ที่ filler ต้องส่ง
}

/**
 * @notice Token input/output specification
 */
struct Input {
    address token;
    uint256 amount;
}

struct Output {
    bytes32 token;      // bytes32 เพื่อรองรับทุก chain
    uint256 amount;
    bytes32 recipient;  // bytes32 เพื่อรองรับทุก chain
    uint32 chainId;
}

// ============ Interfaces ============

/**
 * @title IOriginSettler
 * @notice Interface สำหรับ origin chain (ที่ user อยู่)
 */
interface IOriginSettler {
    
    event Open(
        bytes32 indexed orderHash,
        ResolvedCrossChainOrder resolvedOrder
    );
    
    /**
     * @notice เปิด order ด้วย on-chain call
     * @param order CrossChainOrder ที่ต้องการ open
     * @param signature signature จาก swapper
     * @param fillerData data สำหรับ filler ที่เลือก
     */
    function open(
        CrossChainOrder calldata order,
        bytes calldata signature,
        bytes calldata fillerData
    ) external;
    
    /**
     * @notice Resolve order data เป็น ResolvedCrossChainOrder
     */
    function resolve(
        CrossChainOrder calldata order,
        bytes calldata fillerData
    ) external view returns (ResolvedCrossChainOrder memory);
}

/**
 * @title IDestinationSettler
 * @notice Interface สำหรับ destination chain (ที่ fill order)
 */
interface IDestinationSettler {
    
    /**
     * @notice Fill order บน destination chain
     * @param orderId unique ID ของ order
     * @param originData encoded origin data
     * @param fillerData data สำหรับ filler
     */
    function fill(
        bytes32 orderId,
        bytes calldata originData,
        bytes calldata fillerData
    ) external;
}
```

### 83.2.2 Complete CrossChainSettler Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import "@openzeppelin/contracts/utils/cryptography/MessageHashUtils.sol";

/**
 * @title CrossChainIntentSettler
 * @notice Implementation ของ ERC-7683 สำหรับ cross-chain intent settlement
 * @dev รองรับ:
 *      - Single-chain intents (swap บน chain เดียว)
 *      - Cross-chain intents (bridge + swap)
 *      - Dutch auction pricing
 *      - Exclusive filler window
 */
contract CrossChainIntentSettler is IOriginSettler, ReentrancyGuard {
    using SafeERC20 for IERC20;
    using ECDSA for bytes32;
    using MessageHashUtils for bytes32;
    
    // ============ Custom OrderData for this settler ============
    
    struct SwapOrderData {
        address inputToken;
        uint256 inputAmount;
        address outputToken;
        uint256 outputAmountMin;     // ขั้นต่ำที่ต้องการ
        uint256 outputAmountStart;   // จุดเริ่มต้น dutch auction
        uint32 auctionStartTime;     // เริ่ม dutch auction
        uint32 auctionEndTime;       // สิ้นสุด dutch auction
        address exclusiveFiller;     // filler พิเศษ (ถ้ามี)
        uint32 exclusivityDeadline;  // deadline ของ exclusivity
        bytes32 destinationChain;    // chain ปลายทาง (0 = same chain)
        bytes32 destinationRecipient;// recipient บน destination
    }
    
    // ============ State ============
    
    struct OrderState {
        bytes32 orderHash;
        address swapper;
        address inputToken;
        uint256 inputAmount;
        OrderStatus status;
        address filler;
        uint256 filledAt;
    }
    
    enum OrderStatus {
        Open,
        Filled,
        Cancelled,
        Expired
    }
    
    // orderHash => OrderState
    mapping(bytes32 => OrderState) public orderStates;
    
    // swapper => nonce => used
    mapping(address => mapping(uint256 => bool)) public usedNonces;
    
    // Domain separator for EIP-712
    bytes32 public immutable DOMAIN_SEPARATOR;
    bytes32 public constant ORDER_TYPEHASH = keccak256(
        "CrossChainOrder(address settlementContract,address swapper,uint256 nonce,uint32 originChainId,uint32 initiateDeadline,uint32 fillDeadline,bytes orderData)"
    );
    
    // ============ Events ============
    
    event OrderFilled(
        bytes32 indexed orderHash,
        address indexed filler,
        uint256 outputAmount
    );
    
    event OrderCancelled(bytes32 indexed orderHash);
    
    // ============ Errors ============
    
    error InvalidSignature();
    error NonceAlreadyUsed();
    error OrderExpired();
    error ExclusivityPeriodActive();
    error OutputAmountTooLow();
    error OrderAlreadyFilled();
    error NotOrderOwner();
    
    constructor() {
        DOMAIN_SEPARATOR = keccak256(
            abi.encode(
                keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
                keccak256("CrossChainIntentSettler"),
                keccak256("1"),
                block.chainid,
                address(this)
            )
        );
    }
    
    // ============ IOriginSettler Implementation ============
    
    /**
     * @notice เปิด order (lock input tokens + emit event)
     * @dev Filler จะดู event และ fill บน destination chain
     */
    function open(
        CrossChainOrder calldata order,
        bytes calldata signature,
        bytes calldata fillerData
    ) external override nonReentrant {
        // 1. ตรวจสอบ basic conditions
        require(order.settlementContract == address(this), "Wrong settlement contract");
        require(order.originChainId == uint32(block.chainid), "Wrong chain");
        require(block.timestamp <= order.initiateDeadline, "Initiation deadline passed");
        require(!usedNonces[order.swapper][order.nonce], "Nonce used");
        
        // 2. Verify signature (EIP-712)
        bytes32 orderHash = _hashOrder(order);
        _verifySignature(orderHash, signature, order.swapper);
        
        // 3. Decode order data
        SwapOrderData memory swapData = abi.decode(order.orderData, (SwapOrderData));
        
        // 4. Mark nonce as used
        usedNonces[order.swapper][order.nonce] = true;
        
        // 5. Lock input tokens
        IERC20(swapData.inputToken).safeTransferFrom(
            order.swapper,
            address(this),
            swapData.inputAmount
        );
        
        // 6. Store order state
        orderStates[orderHash] = OrderState({
            orderHash: orderHash,
            swapper: order.swapper,
            inputToken: swapData.inputToken,
            inputAmount: swapData.inputAmount,
            status: OrderStatus.Open,
            filler: address(0),
            filledAt: 0
        });
        
        // 7. Emit event (fillers จะ monitor นี้)
        ResolvedCrossChainOrder memory resolved = resolve(order, fillerData);
        emit Open(orderHash, resolved);
    }
    
    /**
     * @notice Resolve order เป็นรูปแบบมาตรฐาน
     */
    function resolve(
        CrossChainOrder calldata order,
        bytes calldata /* fillerData */
    ) public view override returns (ResolvedCrossChainOrder memory resolved) {
        SwapOrderData memory swapData = abi.decode(order.orderData, (SwapOrderData));
        
        // คำนวณ output amount ตาม dutch auction
        uint256 currentOutput = _getDutchAuctionOutput(swapData, block.timestamp);
        
        resolved.settlementContract = order.settlementContract;
        resolved.swapper = order.swapper;
        resolved.nonce = order.nonce;
        resolved.originChainId = order.originChainId;
        resolved.initiateDeadline = order.initiateDeadline;
        resolved.fillDeadline = order.fillDeadline;
        
        // Input: swapper ส่ง inputToken
        resolved.swapperInputs = new Input[](1);
        resolved.swapperInputs[0] = Input({
            token: swapData.inputToken,
            amount: swapData.inputAmount
        });
        
        // Output: swapper ต้องการ outputToken
        resolved.swapperOutputs = new Output[](1);
        resolved.swapperOutputs[0] = Output({
            token: bytes32(uint256(uint160(swapData.outputToken))),
            amount: currentOutput,
            recipient: swapData.destinationRecipient,
            chainId: swapData.destinationChain == bytes32(0) 
                     ? uint32(block.chainid) 
                     : uint32(uint256(swapData.destinationChain))
        });
        
        // Filler output: filler ต้องส่ง outputToken ให้ swapper
        resolved.fillerOutputs = resolved.swapperOutputs;
    }
    
    /**
     * @notice Fill order บน same chain
     * @dev สำหรับ cross-chain ใช้ IDestinationSettler บน destination chain
     */
    function fillOrder(
        CrossChainOrder calldata order,
        bytes calldata signature,
        uint256 _outputAmount
    ) external nonReentrant {
        bytes32 orderHash = _hashOrder(order);
        OrderState storage state = orderStates[orderHash];
        
        // ตรวจสอบ order status
        if (state.status != OrderStatus.Open) revert OrderAlreadyFilled();
        
        SwapOrderData memory swapData = abi.decode(order.orderData, (SwapOrderData));
        
        // ตรวจสอบ fill deadline
        if (block.timestamp > order.fillDeadline) revert OrderExpired();
        
        // ตรวจสอบ exclusivity
        if (swapData.exclusiveFiller != address(0) &&
            block.timestamp <= swapData.exclusivityDeadline &&
            msg.sender != swapData.exclusiveFiller) {
            revert ExclusivityPeriodActive();
        }
        
        // ตรวจสอบ minimum output
        uint256 requiredOutput = _getDutchAuctionOutput(swapData, block.timestamp);
        if (_outputAmount < requiredOutput) revert OutputAmountTooLow();
        
        // Mark as filled
        state.status = OrderStatus.Filled;
        state.filler = msg.sender;
        state.filledAt = block.timestamp;
        
        // โอน output token จาก filler → swapper
        address recipient = address(uint160(uint256(swapData.destinationRecipient)));
        IERC20(swapData.outputToken).safeTransferFrom(
            msg.sender,
            recipient == address(0) ? order.swapper : recipient,
            _outputAmount
        );
        
        // โอน input token จาก contract → filler (reward)
        IERC20(swapData.inputToken).safeTransfer(msg.sender, swapData.inputAmount);
        
        emit OrderFilled(orderHash, msg.sender, _outputAmount);
    }
    
    /**
     * @notice ยกเลิก order (ถ้า expired หรือ ผู้สร้างต้องการ)
     */
    function cancelOrder(CrossChainOrder calldata order) external nonReentrant {
        bytes32 orderHash = _hashOrder(order);
        OrderState storage state = orderStates[orderHash];
        
        require(state.status == OrderStatus.Open, "Not open");
        require(
            msg.sender == state.swapper || block.timestamp > order.fillDeadline,
            "Cannot cancel"
        );
        
        state.status = msg.sender == state.swapper 
            ? OrderStatus.Cancelled 
            : OrderStatus.Expired;
        
        // คืน input tokens
        IERC20(state.inputToken).safeTransfer(state.swapper, state.inputAmount);
        
        emit OrderCancelled(orderHash);
    }
    
    // ============ Internal Functions ============
    
    /**
     * @dev คำนวณ output amount ตาม Dutch Auction
     *      ราคาเริ่มสูง ลดลงตามเวลา จนถึง minimum
     */
    function _getDutchAuctionOutput(
        SwapOrderData memory _swapData,
        uint256 _timestamp
    ) internal pure returns (uint256 output) {
        if (_timestamp <= _swapData.auctionStartTime) {
            return _swapData.outputAmountStart;
        }
        
        if (_timestamp >= _swapData.auctionEndTime) {
            return _swapData.outputAmountMin;
        }
        
        uint256 elapsed = _timestamp - _swapData.auctionStartTime;
        uint256 duration = _swapData.auctionEndTime - _swapData.auctionStartTime;
        uint256 decay = (_swapData.outputAmountStart - _swapData.outputAmountMin) 
                       * elapsed / duration;
        
        output = _swapData.outputAmountStart - decay;
    }
    
    function _hashOrder(
        CrossChainOrder calldata order
    ) internal view returns (bytes32) {
        return keccak256(
            abi.encodePacked(
                "\x19\x01",
                DOMAIN_SEPARATOR,
                keccak256(abi.encode(
                    ORDER_TYPEHASH,
                    order.settlementContract,
                    order.swapper,
                    order.nonce,
                    order.originChainId,
                    order.initiateDeadline,
                    order.fillDeadline,
                    keccak256(order.orderData)
                ))
            )
        );
    }
    
    function _verifySignature(
        bytes32 _hash,
        bytes memory _signature,
        address _signer
    ) internal pure {
        address recovered = _hash.recover(_signature);
        if (recovered != _signer) revert InvalidSignature();
    }
}
```

---

## 83.3 UniswapX OrderReactor

### แนวคิด UniswapX

```
UniswapX Flow:
                                                    
User signs intent     Off-chain         On-chain
─────────────────    ──────────────    ──────────────
                                       
SignedOrder  ──────►  Intent Pool  ───► Filler executes
                      (off-chain)       - validate sig
                                        - check deadline
                      Fillers compete   - check output
                      - AMM swaps       - transfer tokens
                      - Private MM      - emit Fill event
                      - Cross-protocol
```

### 83.3.1 UniswapX Core Types

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title UniswapX Core Types
 * @notice Types ที่ใช้ใน UniswapX protocol
 */

// Order ที่ถูก sign แล้ว
struct SignedOrder {
    bytes order;       // encoded order
    bytes signature;   // EIP-712 signature จาก swapper
}

// Dutch Auction Order (output ลดลงตามเวลา)
struct DutchOrder {
    // Common fields
    address reactor;           // OrderReactor address
    address swapper;
    uint256 nonce;
    uint256 deadline;
    
    // Validation hooks
    address additionalValidationContract;
    bytes additionalValidationData;
    
    // Decay settings
    uint256 decayStartTime;    // เริ่ม decay
    uint256 decayEndTime;      // จบ decay
    
    // Exclusivity
    address exclusiveFiller;
    uint256 exclusivityOverrideBps; // หาก filler อื่นมา จ่ายเพิ่มกี่ bps
    
    // Input/Output
    DutchInput input;
    DutchOutput[] outputs;
}

struct DutchInput {
    address token;
    uint256 startAmount;  // เพิ่มขึ้นตามเวลา (ถ้า filler ช้า จ่ายน้อยลง)
    uint256 endAmount;
}

struct DutchOutput {
    address token;
    uint256 startAmount;  // ลดลงตามเวลา
    uint256 endAmount;    // ขั้นต่ำ
    address recipient;
}

// Priority Order (gas auction)
struct PriorityOrder {
    address reactor;
    address swapper;
    uint256 nonce;
    uint256 deadline;
    address auctionStartBlock;
    uint256 baselinePriorityFeeWei;
    PriorityInput input;
    PriorityOutput[] outputs;
}

struct PriorityInput {
    address token;
    uint256 amount;
    uint256 mpsPerPriorityFeeWei; // scaling factor
}

struct PriorityOutput {
    address token;
    uint256 amount;
    uint256 mpsPerPriorityFeeWei;
    address recipient;
}
```

### 83.3.2 OrderReactor Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title DutchOrderReactor
 * @notice Simplified UniswapX Dutch Order Reactor
 * @dev ให้ fillers execute orders พร้อม dutch auction pricing
 */
contract DutchOrderReactor is ReentrancyGuard {
    using SafeERC20 for IERC20;
    
    // ============ State ============
    
    // order hash => is filled
    mapping(bytes32 => bool) public filledOrders;
    
    // permit2 contract สำหรับ token approval
    address public immutable permit2;
    
    // ============ Events ============
    
    event Fill(
        bytes32 indexed orderHash,
        address indexed filler,
        address indexed swapper,
        uint256 nonce
    );
    
    // ============ Errors ============
    
    error OrderAlreadyFilled();
    error OrderDeadlinePassed();
    error InsufficientOutput();
    error ExclusiveFiller(address exclusiveFiller, uint256 until);
    error InvalidSignature();
    
    constructor(address _permit2) {
        permit2 = _permit2;
    }
    
    // ============ Core Fill Functions ============
    
    /**
     * @notice Fill single order
     * @param _signedOrder order + signature
     * @param _fillData data สำหรับ filler callback
     */
    function execute(
        SignedOrder calldata _signedOrder,
        bytes calldata _fillData
    ) external nonReentrant {
        DutchOrder memory order = abi.decode(_signedOrder.order, (DutchOrder));
        
        _validateOrder(order);
        
        bytes32 orderHash = _getOrderHash(order);
        if (filledOrders[orderHash]) revert OrderAlreadyFilled();
        
        // ตรวจสอบ signature
        _verifySignature(orderHash, _signedOrder.signature, order.swapper);
        
        // คำนวณ amounts ตาม dutch auction
        (uint256 inputAmount, uint256[] memory outputAmounts) = 
            _resolveAmounts(order, block.timestamp);
        
        // Mark as filled BEFORE external calls (reentrancy protection)
        filledOrders[orderHash] = true;
        
        // โอน input จาก swapper → filler (via Permit2)
        _transferInputFromSwapper(
            order.swapper, 
            msg.sender, 
            order.input.token, 
            inputAmount
        );
        
        // Callback: filler execute strategy
        if (_fillData.length > 0) {
            IFillCallback(msg.sender).reactorCallback(
                _signedOrder,
                inputAmount,
                outputAmounts,
                _fillData
            );
        }
        
        // ตรวจสอบและโอน outputs จาก filler → recipients
        for (uint256 i = 0; i < order.outputs.length; i++) {
            DutchOutput memory output = order.outputs[i];
            IERC20(output.token).safeTransferFrom(
                msg.sender,
                output.recipient,
                outputAmounts[i]
            );
        }
        
        emit Fill(orderHash, msg.sender, order.swapper, order.nonce);
    }
    
    /**
     * @notice Fill หลาย orders พร้อมกัน (gas efficient)
     */
    function executeBatch(
        SignedOrder[] calldata _signedOrders,
        bytes calldata _fillData
    ) external nonReentrant {
        uint256 len = _signedOrders.length;
        
        DutchOrder[] memory orders = new DutchOrder[](len);
        bytes32[] memory orderHashes = new bytes32[](len);
        uint256[] memory inputAmounts = new uint256[](len);
        uint256[][] memory outputAmountsList = new uint256[][](len);
        
        // Validate all orders first
        for (uint256 i = 0; i < len; i++) {
            orders[i] = abi.decode(_signedOrders[i].order, (DutchOrder));
            _validateOrder(orders[i]);
            
            orderHashes[i] = _getOrderHash(orders[i]);
            if (filledOrders[orderHashes[i]]) revert OrderAlreadyFilled();
            
            _verifySignature(
                orderHashes[i], 
                _signedOrders[i].signature, 
                orders[i].swapper
            );
            
            (inputAmounts[i], outputAmountsList[i]) = 
                _resolveAmounts(orders[i], block.timestamp);
            
            filledOrders[orderHashes[i]] = true;
        }
        
        // Transfer all inputs to filler
        for (uint256 i = 0; i < len; i++) {
            _transferInputFromSwapper(
                orders[i].swapper,
                msg.sender,
                orders[i].input.token,
                inputAmounts[i]
            );
        }
        
        // Filler callback
        if (_fillData.length > 0) {
            IBatchFillCallback(msg.sender).reactorBatchCallback(
                _signedOrders,
                inputAmounts,
                outputAmountsList,
                _fillData
            );
        }
        
        // Transfer all outputs from filler
        for (uint256 i = 0; i < len; i++) {
            for (uint256 j = 0; j < orders[i].outputs.length; j++) {
                IERC20(orders[i].outputs[j].token).safeTransferFrom(
                    msg.sender,
                    orders[i].outputs[j].recipient,
                    outputAmountsList[i][j]
                );
            }
            
            emit Fill(orderHashes[i], msg.sender, orders[i].swapper, orders[i].nonce);
        }
    }
    
    // ============ Internal Functions ============
    
    function _validateOrder(DutchOrder memory _order) internal view {
        if (block.timestamp > _order.deadline) revert OrderDeadlinePassed();
        
        // ตรวจสอบ exclusivity
        if (_order.exclusiveFiller != address(0) &&
            msg.sender != _order.exclusiveFiller &&
            block.timestamp < _order.decayStartTime) {
            revert ExclusiveFiller(
                _order.exclusiveFiller, 
                _order.decayStartTime
            );
        }
    }
    
    function _resolveAmounts(
        DutchOrder memory _order,
        uint256 _timestamp
    ) internal pure returns (
        uint256 inputAmount,
        uint256[] memory outputAmounts
    ) {
        // คำนวณ decay factor (0 = start, 1e18 = end)
        uint256 decayFactor;
        if (_timestamp <= _order.decayStartTime) {
            decayFactor = 0;
        } else if (_timestamp >= _order.decayEndTime) {
            decayFactor = 1e18;
        } else {
            uint256 elapsed = _timestamp - _order.decayStartTime;
            uint256 duration = _order.decayEndTime - _order.decayStartTime;
            decayFactor = elapsed * 1e18 / duration;
        }
        
        // Input เพิ่มขึ้น (swapper จ่ายมากขึ้นถ้า filler ช้า)
        inputAmount = _order.input.startAmount + 
            (_order.input.endAmount - _order.input.startAmount) * decayFactor / 1e18;
        
        // Outputs ลดลง (filler ต้องส่งน้อยลงถ้า filler ช้า)
        outputAmounts = new uint256[](_order.outputs.length);
        for (uint256 i = 0; i < _order.outputs.length; i++) {
            DutchOutput memory out = _order.outputs[i];
            uint256 decay = (out.startAmount - out.endAmount) * decayFactor / 1e18;
            outputAmounts[i] = out.startAmount - decay;
        }
    }
    
    function _getOrderHash(DutchOrder memory _order) internal pure returns (bytes32) {
        return keccak256(abi.encode(_order));
    }
    
    function _verifySignature(
        bytes32 _hash,
        bytes memory _signature,
        address _signer
    ) internal pure {
        bytes32 ethSignedHash = keccak256(
            abi.encodePacked("\x19Ethereum Signed Message:\n32", _hash)
        );
        
        bytes32 r;
        bytes32 s;
        uint8 v;
        assembly {
            r := mload(add(_signature, 32))
            s := mload(add(_signature, 64))
            v := byte(0, mload(add(_signature, 96)))
        }
        
        address recovered = ecrecover(ethSignedHash, v, r, s);
        if (recovered != _signer) revert InvalidSignature();
    }
    
    function _transferInputFromSwapper(
        address _swapper,
        address _filler,
        address _token,
        uint256 _amount
    ) internal {
        // ในการใช้งานจริงใช้ Permit2 สำหรับ gas-efficient approval
        // สำหรับตัวอย่างใช้ standard transferFrom
        IERC20(_token).safeTransferFrom(_swapper, _filler, _amount);
    }
}

// Callback interfaces
interface IFillCallback {
    function reactorCallback(
        SignedOrder calldata order,
        uint256 inputAmount,
        uint256[] calldata outputAmounts,
        bytes calldata callbackData
    ) external;
}

interface IBatchFillCallback {
    function reactorBatchCallback(
        SignedOrder[] calldata orders,
        uint256[] calldata inputAmounts,
        uint256[][] calldata outputAmountsList,
        bytes calldata callbackData
    ) external;
}
```

---

## 83.4 1inch Fusion: Dutch Auction Resolver

### สถาปัตยกรรม 1inch Fusion

```
1inch Fusion Flow:
                                              
User Intent                Dutch Auction                  Settlement
──────────                 ──────────────                 ──────────
                                                           
"swap 1000 USDC ──────►  Block 0: price $2,990  ──────►  Best filler
 → ETH, get best          Block 1: price $2,985            executes
 price"                   Block 2: price $2,980            settlement()
                           Block 3: price $2,975
                           ...
                           Block N: price $2,800 (floor)
```

### 83.4.1 Fusion Order Types

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title 1inch Fusion Core Contracts
 * @notice Simplified implementation ของ 1inch Fusion protocol
 */

struct FusionOrder {
    // Order identification
    address makerAsset;      // token ที่ maker ขาย
    address takerAsset;      // token ที่ maker ต้องการ
    address maker;           // ผู้สร้าง order
    uint256 makingAmount;    // จำนวนที่ขาย
    uint256 takingAmount;    // จำนวนที่ต้องการ (base)
    uint256 salt;            // unique salt
    
    // Dutch auction params
    uint40 auctionStartTime;   // เวลาเริ่ม auction
    uint40 duration;           // ระยะเวลา auction (seconds)
    uint24 initialRateBump;    // อัตราส่วนเริ่มต้นสูงกว่า base เท่าไร (bps)
    
    // Resolver requirements
    address allowedResolver;   // resolver ที่ได้รับอนุญาต (0 = ใครก็ได้)
    
    // Extra params
    bytes permit;              // ERC-2612 permit
    bytes predicate;           // conditions ที่ต้องเป็นจริง
    bytes interaction;         // callback data
}

struct AuctionPoint {
    uint40 delay;        // วินาทีจาก auction start
    uint24 coefficient;  // rate bump ที่ time point นี้
}
```

### 83.4.2 Fusion Settlement Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title FusionSettlement
 * @notice 1inch Fusion Settlement - จัดการการ settle Dutch Auction orders
 */
contract FusionSettlement is ReentrancyGuard {
    using SafeERC20 for IERC20;
    
    uint256 constant RATE_BUMP_DENOMINATOR = 1e7; // rate bump ใน 10^7 scale
    
    // registered resolvers
    mapping(address => bool) public whitelistedResolvers;
    
    // filled order hashes
    mapping(bytes32 => bool) public filledOrders;
    
    address public owner;
    
    event OrderFilled(
        bytes32 indexed orderHash,
        address indexed resolver,
        uint256 makingAmount,
        uint256 takingAmount,
        uint256 auctionDiscount
    );
    
    event ResolverWhitelisted(address resolver, bool whitelisted);
    
    error NotWhitelistedResolver();
    error OrderAlreadyFilled();
    error AuctionNotStarted();
    error AuctionEnded();
    error TakingAmountTooLow();
    error ResolverNotAllowed();
    
    constructor() {
        owner = msg.sender;
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    /**
     * @notice เพิ่ม/ลบ resolver จาก whitelist
     */
    function setResolver(address _resolver, bool _whitelisted) external onlyOwner {
        whitelistedResolvers[_resolver] = _whitelisted;
        emit ResolverWhitelisted(_resolver, _whitelisted);
    }
    
    /**
     * @notice Settle order โดย resolver
     * @param _order FusionOrder ที่ต้องการ settle
     * @param _takingAmount จำนวนที่ resolver จะจ่าย
     * @param _thresholdAmount จำนวนขั้นต่ำที่ maker ยอมรับ
     */
    function settle(
        FusionOrder calldata _order,
        uint256 _takingAmount,
        uint256 _thresholdAmount
    ) external nonReentrant {
        // ตรวจสอบ resolver
        if (!whitelistedResolvers[msg.sender]) revert NotWhitelistedResolver();
        if (_order.allowedResolver != address(0) && 
            _order.allowedResolver != msg.sender) {
            revert ResolverNotAllowed();
        }
        
        bytes32 orderHash = _hashFusionOrder(_order);
        if (filledOrders[orderHash]) revert OrderAlreadyFilled();
        
        // ตรวจสอบ auction timing
        uint256 auctionStart = _order.auctionStartTime;
        if (block.timestamp < auctionStart) revert AuctionNotStarted();
        if (block.timestamp > auctionStart + _order.duration) revert AuctionEnded();
        
        // คำนวณ current rate
        uint256 currentRate = _getCurrentRate(_order, block.timestamp);
        
        // ตรวจสอบ taking amount ต้องไม่น้อยกว่า threshold
        if (_takingAmount < _thresholdAmount) revert TakingAmountTooLow();
        
        // คำนวณ discount ที่ resolver ได้ (savings จาก auction)
        uint256 baseTaking = _order.takingAmount;
        uint256 auctionRate = baseTaking * currentRate / RATE_BUMP_DENOMINATOR;
        
        // Mark as filled
        filledOrders[orderHash] = true;
        
        // โอน makerAsset จาก maker → resolver
        IERC20(_order.makerAsset).safeTransferFrom(
            _order.maker,
            msg.sender,
            _order.makingAmount
        );
        
        // โอน takerAsset จาก resolver → maker
        IERC20(_order.takerAsset).safeTransferFrom(
            msg.sender,
            _order.maker,
            _takingAmount
        );
        
        uint256 discount = _takingAmount > baseTaking ? _takingAmount - baseTaking : 0;
        
        emit OrderFilled(
            orderHash,
            msg.sender,
            _order.makingAmount,
            _takingAmount,
            discount
        );
    }
    
    /**
     * @dev คำนวณ current rate ตาม dutch auction curve
     * @return rate rate ใน RATE_BUMP_DENOMINATOR scale
     */
    function _getCurrentRate(
        FusionOrder calldata _order,
        uint256 _timestamp
    ) internal pure returns (uint256 rate) {
        uint256 timeElapsed = _timestamp - _order.auctionStartTime;
        
        // Linear decay จาก (1 + initialRateBump) ลง 1
        if (timeElapsed >= _order.duration) {
            return RATE_BUMP_DENOMINATOR; // base rate
        }
        
        uint256 bump = uint256(_order.initialRateBump) * 
                       (_order.duration - timeElapsed) / 
                       _order.duration;
        
        rate = RATE_BUMP_DENOMINATOR + bump;
    }
    
    function _hashFusionOrder(FusionOrder calldata _order) 
        internal pure returns (bytes32) 
    {
        return keccak256(abi.encode(
            _order.makerAsset,
            _order.takerAsset,
            _order.maker,
            _order.makingAmount,
            _order.takingAmount,
            _order.salt
        ));
    }
    
    /**
     * @notice ดู current takingAmount ที่ resolver ต้องจ่าย ณ ขณะนี้
     */
    function getCurrentTakingAmount(
        FusionOrder calldata _order
    ) external view returns (uint256 currentTaking, bool isActive) {
        if (block.timestamp < _order.auctionStartTime) {
            return (_order.takingAmount * (RATE_BUMP_DENOMINATOR + _order.initialRateBump) 
                   / RATE_BUMP_DENOMINATOR, false);
        }
        
        if (block.timestamp > _order.auctionStartTime + _order.duration) {
            return (_order.takingAmount, false);
        }
        
        uint256 rate = _getCurrentRate(_order, block.timestamp);
        return (_order.takingAmount * rate / RATE_BUMP_DENOMINATOR, true);
    }
}
```

---

## 83.5 IntentRouter: Universal Intent Parser

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title IntentRouter
 * @notice Universal router สำหรับ parse และ route intents
 *         ไปยัง solver ที่เหมาะสม
 * @dev รองรับหลาย intent formats:
 *      - UniswapX Dutch Orders
 *      - 1inch Fusion Orders  
 *      - ERC-7683 Cross-chain Orders
 *      - Custom intents
 */
contract IntentRouter is Ownable, ReentrancyGuard {
    
    // ============ Intent Types ============
    
    enum IntentType {
        Unknown,
        UniswapXDutch,
        OneinchFusion,
        ERC7683CrossChain,
        LimitOrder,
        MarketOrder
    }
    
    // ============ Structs ============
    
    struct IntentMetadata {
        IntentType intentType;
        address fromToken;
        address toToken;
        uint256 fromAmount;
        uint256 minToAmount;
        uint256 deadline;
        address swapper;
        bool isCrossChain;
        uint32 destinationChain;
    }
    
    struct SolverInfo {
        address solverContract;
        bool isActive;
        uint256 totalFilled;
        uint256 totalVolume;
        string name;
    }
    
    struct FulfillmentResult {
        bool success;
        uint256 outputAmount;
        address solverUsed;
        uint256 gasUsed;
        string reason;
    }
    
    // ============ State ============
    
    // IntentType => Solver contracts (priority order)
    mapping(IntentType => address[]) public solvers;
    
    // solver address => info
    mapping(address => SolverInfo) public solverInfo;
    
    // intent hash => is processed
    mapping(bytes32 => bool) public processedIntents;
    
    // statistics
    uint256 public totalIntentsProcessed;
    uint256 public totalVolumeRouted;
    
    // ============ Events ============
    
    event IntentReceived(
        bytes32 indexed intentHash,
        IntentType intentType,
        address indexed swapper,
        address fromToken,
        address toToken,
        uint256 fromAmount
    );
    
    event IntentFulfilled(
        bytes32 indexed intentHash,
        address indexed solver,
        uint256 outputAmount,
        uint256 gasUsed
    );
    
    event IntentFailed(
        bytes32 indexed intentHash,
        string reason
    );
    
    event SolverRegistered(
        address indexed solver,
        IntentType intentType,
        string name
    );
    
    // ============ Constructor ============
    
    constructor() Ownable(msg.sender) {}
    
    // ============ Admin Functions ============
    
    /**
     * @notice ลงทะเบียน solver สำหรับ intent type นั้น
     */
    function registerSolver(
        address _solver,
        IntentType _intentType,
        string calldata _name
    ) external onlyOwner {
        require(_solver != address(0), "Invalid solver");
        require(!solverInfo[_solver].isActive, "Already registered");
        
        solvers[_intentType].push(_solver);
        solverInfo[_solver] = SolverInfo({
            solverContract: _solver,
            isActive: true,
            totalFilled: 0,
            totalVolume: 0,
            name: _name
        });
        
        emit SolverRegistered(_solver, _intentType, _name);
    }
    
    /**
     * @notice เปิด/ปิด solver
     */
    function setSolverStatus(address _solver, bool _active) external onlyOwner {
        solverInfo[_solver].isActive = _active;
    }
    
    // ============ Core Routing Functions ============
    
    /**
     * @notice Route intent ไปยัง solver ที่เหมาะสม
     * @param _encodedIntent encoded intent data
     * @param _signature signature จาก user
     */
    function routeIntent(
        bytes calldata _encodedIntent,
        bytes calldata _signature
    ) external nonReentrant returns (FulfillmentResult memory result) {
        
        // 1. Parse intent
        (IntentType intentType, IntentMetadata memory meta) = _parseIntent(_encodedIntent);
        
        bytes32 intentHash = keccak256(_encodedIntent);
        
        require(!processedIntents[intentHash], "Intent already processed");
        require(block.timestamp <= meta.deadline, "Intent expired");
        
        emit IntentReceived(
            intentHash,
            intentType,
            meta.swapper,
            meta.fromToken,
            meta.toToken,
            meta.fromAmount
        );
        
        // 2. ลอง solvers ตามลำดับ priority
        address[] memory availableSolvers = solvers[intentType];
        
        for (uint256 i = 0; i < availableSolvers.length; i++) {
            address solver = availableSolvers[i];
            if (!solverInfo[solver].isActive) continue;
            
            uint256 gasBefore = gasleft();
            
            // ลอง fill ด้วย solver นี้
            (bool success, bytes memory returnData) = solver.call(
                abi.encodeWithSignature(
                    "fillIntent(bytes,bytes,address)",
                    _encodedIntent,
                    _signature,
                    msg.sender
                )
            );
            
            uint256 gasUsed = gasBefore - gasleft();
            
            if (success) {
                uint256 outputAmount = abi.decode(returnData, (uint256));
                
                // ตรวจสอบ minimum output
                if (outputAmount >= meta.minToAmount) {
                    processedIntents[intentHash] = true;
                    
                    // อัปเดต statistics
                    solverInfo[solver].totalFilled++;
                    solverInfo[solver].totalVolume += meta.fromAmount;
                    totalIntentsProcessed++;
                    totalVolumeRouted += meta.fromAmount;
                    
                    result = FulfillmentResult({
                        success: true,
                        outputAmount: outputAmount,
                        solverUsed: solver,
                        gasUsed: gasUsed,
                        reason: ""
                    });
                    
                    emit IntentFulfilled(intentHash, solver, outputAmount, gasUsed);
                    return result;
                }
            }
        }
        
        // ไม่มี solver ไหนทำได้
        result = FulfillmentResult({
            success: false,
            outputAmount: 0,
            solverUsed: address(0),
            gasUsed: 0,
            reason: "No solver found"
        });
        
        emit IntentFailed(intentHash, "No solver found");
    }
    
    /**
     * @dev Parse encoded intent และ return type + metadata
     */
    function _parseIntent(
        bytes calldata _encodedIntent
    ) internal view returns (IntentType intentType, IntentMetadata memory meta) {
        // Parse selector (first 4 bytes indicate type)
        if (_encodedIntent.length < 4) {
            return (IntentType.Unknown, meta);
        }
        
        bytes4 selector = bytes4(_encodedIntent[:4]);
        
        // UniswapX Dutch Order selector
        if (selector == bytes4(keccak256("DutchOrder(address,address,address,uint256,uint256,uint256)"))) {
            intentType = IntentType.UniswapXDutch;
            (meta) = _parseDutchOrderMeta(_encodedIntent[4:]);
        }
        // 1inch Fusion selector
        else if (selector == bytes4(keccak256("FusionOrder(address,address,address,uint256,uint256,uint256)"))) {
            intentType = IntentType.OneinchFusion;
            (meta) = _parseFusionOrderMeta(_encodedIntent[4:]);
        }
        // ERC-7683 Cross-chain selector
        else if (selector == bytes4(keccak256("CrossChainOrder(address,address,uint256,uint32,uint32,uint32,bytes)"))) {
            intentType = IntentType.ERC7683CrossChain;
            (meta) = _parseCrossChainOrderMeta(_encodedIntent[4:]);
        }
        else {
            intentType = IntentType.Unknown;
        }
    }
    
    function _parseDutchOrderMeta(bytes calldata _data) 
        internal pure returns (IntentMetadata memory meta) 
    {
        (
            address reactor,
            address swapper,
            uint256 nonce,
            uint256 deadline,
            address inputToken,
            uint256 inputAmount
        ) = abi.decode(_data, (address, address, uint256, uint256, address, uint256));
        
        meta.swapper = swapper;
        meta.fromToken = inputToken;
        meta.fromAmount = inputAmount;
        meta.deadline = deadline;
        meta.isCrossChain = false;
    }
    
    function _parseFusionOrderMeta(bytes calldata _data) 
        internal pure returns (IntentMetadata memory meta) 
    {
        (
            address makerAsset,
            address takerAsset,
            address maker,
            uint256 makingAmount,
            uint256 takingAmount,
            uint256 salt
        ) = abi.decode(_data, (address, address, address, uint256, uint256, uint256));
        
        meta.swapper = maker;
        meta.fromToken = makerAsset;
        meta.toToken = takerAsset;
        meta.fromAmount = makingAmount;
        meta.minToAmount = takingAmount;
        meta.isCrossChain = false;
    }
    
    function _parseCrossChainOrderMeta(bytes calldata _data)
        internal pure returns (IntentMetadata memory meta)
    {
        (
            address swapper,
            uint32 originChainId,
            uint32 fillDeadline,
            address inputToken,
            uint256 inputAmount
        ) = abi.decode(_data, (address, uint32, uint32, address, uint256));
        
        meta.swapper = swapper;
        meta.fromToken = inputToken;
        meta.fromAmount = inputAmount;
        meta.deadline = fillDeadline;
        meta.isCrossChain = true;
    }
    
    // ============ View Functions ============
    
    function getSolversForType(IntentType _type) 
        external view returns (address[] memory) 
    {
        return solvers[_type];
    }
    
    function estimateBestOutput(
        bytes calldata _encodedIntent
    ) external view returns (
        address bestSolver,
        uint256 estimatedOutput,
        IntentType intentType
    ) {
        (intentType, ) = _parseIntent(_encodedIntent);
        
        address[] memory availableSolvers = solvers[intentType];
        uint256 bestOutput = 0;
        
        for (uint256 i = 0; i < availableSolvers.length; i++) {
            address solver = availableSolvers[i];
            if (!solverInfo[solver].isActive) continue;
            
            try ISolverEstimator(solver).estimateOutput(_encodedIntent) 
                returns (uint256 output) 
            {
                if (output > bestOutput) {
                    bestOutput = output;
                    bestSolver = solver;
                }
            } catch {}
        }
        
        estimatedOutput = bestOutput;
    }
}

interface ISolverEstimator {
    function estimateOutput(bytes calldata _encodedIntent) 
        external view returns (uint256 estimatedOutput);
}
```

---

## 83.6 Workshop: Building an Intent-Based Swap

```javascript
// Workshop: สร้าง intent และ submit ผ่าน UniswapX

const { ethers } = require("ethers");
require("dotenv").config();

// ============ สร้าง Dutch Order Intent ============

async function createAndSubmitDutchOrder() {
    const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
    const wallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider);
    
    const USDC = "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48";
    const WETH = "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2";
    const REACTOR = "0x6000da47483062A0D734Ba3dc7576Ce6A0B645C4"; // UniswapX
    
    const currentBlock = await provider.getBlockNumber();
    const currentTime = Math.floor(Date.now() / 1000);
    
    // Dutch order: ขาย 1000 USDC, ต้องการ WETH
    // เริ่มต้นต้องการ 0.34 WETH (ราคาสูง), ลดเหลือ 0.33 WETH ใน 30 นาที
    const dutchOrder = {
        reactor: REACTOR,
        swapper: wallet.address,
        nonce: Date.now(), // unique nonce
        deadline: currentTime + 3600, // 1 ชั่วโมง
        additionalValidationContract: ethers.ZeroAddress,
        additionalValidationData: "0x",
        decayStartTime: currentTime + 30,    // เริ่ม decay หลัง 30 วินาที
        decayEndTime: currentTime + 1800,    // decay เสร็จใน 30 นาที
        exclusiveFiller: ethers.ZeroAddress, // ใครก็ได้
        exclusivityOverrideBps: 0,
        input: {
            token: USDC,
            startAmount: ethers.parseUnits("1000", 6),
            endAmount: ethers.parseUnits("1000", 6) // input คงที่
        },
        outputs: [{
            token: WETH,
            startAmount: ethers.parseEther("0.340"),  // สูงเริ่มต้น
            endAmount: ethers.parseEther("0.330"),    // ต่ำสุด
            recipient: wallet.address
        }]
    };
    
    // Sign order (EIP-712)
    const domain = {
        name: "UniswapX",
        version: "1",
        chainId: 1,
        verifyingContract: REACTOR
    };
    
    const types = {
        DutchOrder: [
            { name: "reactor", type: "address" },
            { name: "swapper", type: "address" },
            { name: "nonce", type: "uint256" },
            { name: "deadline", type: "uint256" },
            { name: "decayStartTime", type: "uint256" },
            { name: "decayEndTime", type: "uint256" },
            // ... other fields
        ]
    };
    
    const signature = await wallet.signTypedData(domain, types, dutchOrder);
    
    console.log("Order created!");
    console.log("Signature:", signature);
    
    // Submit ไปยัง UniswapX API
    const response = await fetch("https://api.uniswap.org/v2/orders", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
            encodedOrder: encodeOrder(dutchOrder),
            signature,
            chainId: 1,
            orderType: "Dutch"
        })
    });
    
    const result = await response.json();
    console.log("Order submitted:", result.hash);
    
    return result;
}

// ============ Simulate Solver ============

async function simulateSolverLogic(orderHash, outputAtCurrentTime) {
    // Solver checks profitability:
    // 1. หา route ที่ดีที่สุด
    // 2. คำนวณว่ากำไรไหม (ราคาตลาด vs output required)
    // 3. ถ้ากำไร → execute fill
    
    const marketPrice = await getMarketPrice("USDC", "WETH");
    const requiredOutput = outputAtCurrentTime; // จาก dutch auction
    
    // Market: 1000 USDC → 0.335 WETH
    // Required: 0.332 WETH
    // Profit potential: 0.003 WETH ≈ $9
    
    if (marketPrice > requiredOutput * 1.001) { // 0.1% threshold
        console.log("Order profitable! Filling...");
        await fillOrder(orderHash);
    } else {
        console.log("Waiting for better price...");
    }
}
```

---

## สรุป Part 83

- **Intent-Based Architecture** เปลี่ยนจาก imperative (ระบุขั้นตอน) เป็น declarative (บอกผลลัพธ์ที่ต้องการ)
- **ERC-7683** เป็นมาตรฐาน cross-chain intent ที่ให้ protocol ต่างๆ interoperate กันได้
- **UniswapX** ใช้ dutch auction ที่ output ลดลงตามเวลา เพิ่มแรงจูงใจให้ fillers ทำงานเร็ว
- **1inch Fusion** ใช้ rate bump ที่ลดลง + whitelist resolver system เพื่อจัดการ solver competition
- **IntentRouter** เป็น abstraction layer ที่ route intents ไปยัง solver ที่เหมาะสมที่สุด
- Intents ป้องกัน MEV โดยอัตโนมัติเพราะ solver แข่งกันให้ราคาดีที่สุด ไม่ใช่แข่งกันใส่ order ก่อน

## Next: Part 84 - Restaking & EigenLayer (AVS, IStrategy, OracleAVS)
