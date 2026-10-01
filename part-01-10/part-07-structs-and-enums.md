# Part 07: Structs และ Enums

## สารบัญ
1. Structs เชิงลึก
2. Struct Patterns
3. Enums เชิงลึก
4. State Machines ด้วย Enums
5. Struct Packing
6. Inheritance กับ Structs
7. Events กับ Structs
8. ABI Encoding ของ Structs
9. Workshop: Decentralized Marketplace

---

## 1. Structs เชิงลึก

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract AdvancedStructs {
    
    // === Basic Struct ===
    
    struct Point {
        uint256 x;
        uint256 y;
    }
    
    // === Struct ใน Struct ===
    
    struct Rectangle {
        Point topLeft;
        Point bottomRight;
        string color;
    }
    
    // === Struct กับ Array ===
    
    struct Team {
        string name;
        address[] members;
        uint256 score;
        bool active;
    }
    
    // === Struct กับ Mapping ===
    
    struct Vault {
        address owner;
        mapping(address => uint256) deposits;
        mapping(address => bool) authorized;
        uint256 totalDeposits;
    }
    
    // === Struct Arrays และ Mappings ===
    
    Team[] public teams;
    mapping(uint256 => Vault) public vaults;
    mapping(address => Point[]) public userPositions;
    
    // === สร้าง Struct ===
    
    function createPoint() public pure returns (Point memory) {
        // วิธี 1: Named
        Point memory p1 = Point({x: 10, y: 20});
        
        // วิธี 2: Positional
        Point memory p2 = Point(10, 20);
        
        // วิธี 3: Default + assign
        Point memory p3;
        p3.x = 10;
        p3.y = 20;
        
        return p1;
    }
    
    function createRectangle() public pure returns (Rectangle memory) {
        return Rectangle({
            topLeft: Point({x: 0, y: 10}),
            bottomRight: Point({x: 10, y: 0}),
            color: "red"
        });
    }
    
    // === Memory vs Storage Struct ===
    
    uint256 public counter;
    Point public currentPosition;
    
    // ❌ Wrong: memory copy ไม่เปลี่ยน storage
    function wrongUpdate() public {
        Point memory p = currentPosition; // copy to memory
        p.x = 100; // แก้ memory เท่านั้น
        // storage ไม่เปลี่ยน!
    }
    
    // ✅ Correct: storage pointer
    function correctUpdate(uint256 newX) public {
        Point storage p = currentPosition; // pointer to storage
        p.x = newX; // แก้โดยตรงใน storage
    }
    
    // ✅ Alternative: direct assignment
    function directUpdate(uint256 newX) public {
        currentPosition.x = newX;
    }
    
    // ✅ Replace entire struct
    function replaceStruct(uint256 newX, uint256 newY) public {
        currentPosition = Point({x: newX, y: newY});
    }
    
    // === Struct ใน Memory Array ===
    
    function createPoints(uint256 count) public pure returns (Point[] memory) {
        Point[] memory points = new Point[](count);
        
        for (uint256 i = 0; i < count; i++) {
            points[i] = Point({x: i, y: i * 2});
        }
        
        return points;
    }
    
    // === Delete Struct ===
    
    function deletePosition() public {
        delete currentPosition; // reset to default values (0, 0)
    }
    
    // === Comparing Structs ===
    // ไม่สามารถ compare struct โดยตรงได้
    
    function arePointsEqual(Point memory a, Point memory b) public pure returns (bool) {
        return a.x == b.x && a.y == b.y;
    }
    
    // Hash struct สำหรับ comparison
    function hashPoint(Point memory p) public pure returns (bytes32) {
        return keccak256(abi.encode(p.x, p.y));
    }
}
```

---

## 2. Struct Patterns

```solidity
contract StructPatterns {
    
    // === Pattern 1: Entity Registry ===
    
    struct Entity {
        uint256 id;
        address owner;
        string name;
        uint256 createdAt;
        uint256 updatedAt;
        bool active;
    }
    
    mapping(uint256 => Entity) public entities;
    mapping(address => uint256[]) public ownerEntities;
    uint256 public entityCount;
    
    event EntityCreated(uint256 indexed id, address indexed owner, string name);
    event EntityUpdated(uint256 indexed id);
    
    function createEntity(string calldata name) public returns (uint256 id) {
        id = ++entityCount;
        
        entities[id] = Entity({
            id: id,
            owner: msg.sender,
            name: name,
            createdAt: block.timestamp,
            updatedAt: block.timestamp,
            active: true
        });
        
        ownerEntities[msg.sender].push(id);
        emit EntityCreated(id, msg.sender, name);
    }
    
    function updateEntity(uint256 id, string calldata newName) public {
        Entity storage entity = entities[id];
        require(entity.active, "Not active");
        require(entity.owner == msg.sender, "Not owner");
        
        entity.name = newName;
        entity.updatedAt = block.timestamp;
        
        emit EntityUpdated(id);
    }
    
    // === Pattern 2: Linked List ===
    
    struct ListNode {
        uint256 value;
        uint256 next; // 0 = null
        uint256 prev; // 0 = null
    }
    
    mapping(uint256 => ListNode) public nodes;
    uint256 public head;
    uint256 public tail;
    uint256 public nodeCount;
    uint256 private _nodeIdCounter;
    
    function insertFront(uint256 value) public returns (uint256 nodeId) {
        nodeId = ++_nodeIdCounter;
        
        nodes[nodeId] = ListNode({
            value: value,
            next: head,
            prev: 0
        });
        
        if (head != 0) {
            nodes[head].prev = nodeId;
        } else {
            tail = nodeId;
        }
        
        head = nodeId;
        nodeCount++;
    }
    
    function insertBack(uint256 value) public returns (uint256 nodeId) {
        nodeId = ++_nodeIdCounter;
        
        nodes[nodeId] = ListNode({
            value: value,
            next: 0,
            prev: tail
        });
        
        if (tail != 0) {
            nodes[tail].next = nodeId;
        } else {
            head = nodeId;
        }
        
        tail = nodeId;
        nodeCount++;
    }
    
    function removeNode(uint256 nodeId) public {
        ListNode storage node = nodes[nodeId];
        require(node.value != 0 || nodeId != 0, "Node not found");
        
        if (node.prev != 0) {
            nodes[node.prev].next = node.next;
        } else {
            head = node.next;
        }
        
        if (node.next != 0) {
            nodes[node.next].prev = node.prev;
        } else {
            tail = node.prev;
        }
        
        delete nodes[nodeId];
        nodeCount--;
    }
    
    function toArray() public view returns (uint256[] memory result) {
        result = new uint256[](nodeCount);
        uint256 current = head;
        uint256 idx = 0;
        
        while (current != 0) {
            result[idx++] = nodes[current].value;
            current = nodes[current].next;
        }
    }
    
    // === Pattern 3: Time-based Struct ===
    
    struct TimeLock {
        address beneficiary;
        uint256 amount;
        uint256 releaseTime;
        bool released;
    }
    
    mapping(uint256 => TimeLock) public timelocks;
    uint256 public timelockCount;
    
    function createTimeLock(
        address beneficiary,
        uint256 releaseTime
    ) public payable returns (uint256 id) {
        require(msg.value > 0, "No ETH");
        require(releaseTime > block.timestamp, "Must be future");
        require(beneficiary != address(0), "Zero address");
        
        id = ++timelockCount;
        timelocks[id] = TimeLock({
            beneficiary: beneficiary,
            amount: msg.value,
            releaseTime: releaseTime,
            released: false
        });
    }
    
    function release(uint256 id) public {
        TimeLock storage lock = timelocks[id];
        require(!lock.released, "Already released");
        require(block.timestamp >= lock.releaseTime, "Too early");
        require(msg.sender == lock.beneficiary, "Not beneficiary");
        
        lock.released = true;
        uint256 amount = lock.amount;
        lock.amount = 0;
        
        (bool success, ) = lock.beneficiary.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

---

## 3. Enums เชิงลึก

```solidity
contract AdvancedEnums {
    
    // === Basic Enum ===
    
    enum Direction { North, South, East, West }
    enum Status { Inactive, Active, Paused, Terminated }
    enum Priority { Low, Medium, High, Critical }
    
    // === Enum คือ uint8 โดยเริ่มจาก 0 ===
    
    Direction public currentDirection = Direction.North;
    
    function getDirectionValue(Direction dir) public pure returns (uint8) {
        return uint8(dir);
        // North=0, South=1, East=2, West=3
    }
    
    function getDirectionFromValue(uint8 value) public pure returns (Direction) {
        require(value <= uint8(Direction.West), "Invalid value");
        return Direction(value);
    }
    
    // === Enum ใน Struct ===
    
    struct Task {
        uint256 id;
        string title;
        Status status;
        Priority priority;
        address assignee;
        uint256 createdAt;
        uint256 completedAt;
    }
    
    mapping(uint256 => Task) public tasks;
    uint256 public taskCount;
    
    function createTask(
        string calldata title,
        Priority priority,
        address assignee
    ) public returns (uint256 id) {
        id = ++taskCount;
        tasks[id] = Task({
            id: id,
            title: title,
            status: Status.Active,
            priority: priority,
            assignee: assignee,
            createdAt: block.timestamp,
            completedAt: 0
        });
    }
    
    function completeTask(uint256 taskId) public {
        Task storage task = tasks[taskId];
        require(task.status == Status.Active, "Not active");
        require(msg.sender == task.assignee, "Not assignee");
        
        task.status = Status.Inactive;
        task.completedAt = block.timestamp;
    }
    
    // === Enum Validation ===
    
    function isValidStatus(uint8 value) public pure returns (bool) {
        return value <= uint8(type(Status).max);
    }
    
    function statusToString(Status status) public pure returns (string memory) {
        if (status == Status.Inactive) return "Inactive";
        if (status == Status.Active) return "Active";
        if (status == Status.Paused) return "Paused";
        if (status == Status.Terminated) return "Terminated";
        revert("Unknown status");
    }
    
    // === Enum Min/Max (Solidity 0.8.8+) ===
    
    function getMinMax() public pure returns (uint8 min, uint8 max) {
        min = uint8(type(Status).min); // 0 (Inactive)
        max = uint8(type(Status).max); // 3 (Terminated)
    }
}
```

---

## 4. State Machines ด้วย Enums

```solidity
/**
 * @title StateMachine
 * @dev ตัวอย่าง State Machine สำหรับ Order Processing
 */
contract OrderStateMachine {
    
    // === Order States ===
    
    enum OrderState {
        Created,      // 0: สร้างแล้ว รอชำระเงิน
        Paid,         // 1: ชำระเงินแล้ว รอยืนยัน
        Processing,   // 2: กำลังดำเนินการ
        Shipped,      // 3: จัดส่งแล้ว
        Delivered,    // 4: ได้รับสินค้าแล้ว
        Cancelled,    // 5: ยกเลิก
        Disputed,     // 6: มีข้อพิพาท
        Refunded      // 7: คืนเงินแล้ว
    }
    
    // === State Transitions ===
    // 
    // Created → Paid → Processing → Shipped → Delivered
    //         ↓              ↓
    //    Cancelled       Cancelled
    //
    // Delivered → Disputed → Refunded
    //
    
    struct Order {
        uint256 id;
        address buyer;
        address seller;
        uint256 amount;
        OrderState state;
        string item;
        uint256 createdAt;
        uint256 lastUpdatedAt;
        string trackingNumber;
    }
    
    // === State ===
    
    mapping(uint256 => Order) public orders;
    uint256 public orderCount;
    
    // Valid transitions
    mapping(OrderState => mapping(OrderState => bool)) private _validTransitions;
    
    // === Events ===
    
    event OrderCreated(uint256 indexed orderId, address buyer, address seller, uint256 amount);
    event OrderStateChanged(
        uint256 indexed orderId,
        OrderState indexed fromState,
        OrderState indexed toState,
        address changedBy
    );
    
    // === Custom Errors ===
    
    error InvalidTransition(OrderState from, OrderState to);
    error NotBuyer(address caller);
    error NotSeller(address caller);
    error OrderNotFound(uint256 id);
    
    // === Constructor ===
    
    constructor() {
        _setupTransitions();
    }
    
    function _setupTransitions() internal {
        // Created → Paid (buyer pays)
        _validTransitions[OrderState.Created][OrderState.Paid] = true;
        // Created → Cancelled (buyer cancels before payment)
        _validTransitions[OrderState.Created][OrderState.Cancelled] = true;
        
        // Paid → Processing (seller confirms)
        _validTransitions[OrderState.Paid][OrderState.Processing] = true;
        // Paid → Cancelled (seller rejects or buyer cancels)
        _validTransitions[OrderState.Paid][OrderState.Cancelled] = true;
        
        // Processing → Shipped (seller ships)
        _validTransitions[OrderState.Processing][OrderState.Shipped] = true;
        // Processing → Cancelled (seller cancels)
        _validTransitions[OrderState.Processing][OrderState.Cancelled] = true;
        
        // Shipped → Delivered (buyer confirms receipt)
        _validTransitions[OrderState.Shipped][OrderState.Delivered] = true;
        // Shipped → Disputed (buyer disputes)
        _validTransitions[OrderState.Shipped][OrderState.Disputed] = true;
        
        // Delivered → Disputed (buyer opens dispute)
        _validTransitions[OrderState.Delivered][OrderState.Disputed] = true;
        
        // Cancelled → Refunded (admin processes refund)
        _validTransitions[OrderState.Cancelled][OrderState.Refunded] = true;
        // Disputed → Refunded (dispute resolved)
        _validTransitions[OrderState.Disputed][OrderState.Refunded] = true;
    }
    
    // === Modifiers ===
    
    modifier onlyBuyer(uint256 orderId) {
        if (orders[orderId].buyer != msg.sender) revert NotBuyer(msg.sender);
        _;
    }
    
    modifier onlySeller(uint256 orderId) {
        if (orders[orderId].seller != msg.sender) revert NotSeller(msg.sender);
        _;
    }
    
    modifier validOrder(uint256 orderId) {
        if (orderId == 0 || orderId > orderCount) revert OrderNotFound(orderId);
        _;
    }
    
    // === Order Lifecycle ===
    
    function createOrder(address seller, string calldata item) external payable returns (uint256 orderId) {
        require(msg.value > 0, "Payment required");
        require(seller != address(0), "Invalid seller");
        require(seller != msg.sender, "Cannot buy from yourself");
        
        orderId = ++orderCount;
        orders[orderId] = Order({
            id: orderId,
            buyer: msg.sender,
            seller: seller,
            amount: msg.value,
            state: OrderState.Created,
            item: item,
            createdAt: block.timestamp,
            lastUpdatedAt: block.timestamp,
            trackingNumber: ""
        });
        
        emit OrderCreated(orderId, msg.sender, seller, msg.value);
    }
    
    function payOrder(uint256 orderId) external payable onlyBuyer(orderId) validOrder(orderId) {
        Order storage order = orders[orderId];
        require(msg.value == order.amount, "Wrong amount");
        
        _transition(orderId, OrderState.Paid);
    }
    
    function confirmOrder(uint256 orderId) external onlySeller(orderId) validOrder(orderId) {
        _transition(orderId, OrderState.Processing);
    }
    
    function shipOrder(uint256 orderId, string calldata trackingNumber) 
        external onlySeller(orderId) validOrder(orderId) 
    {
        _transition(orderId, OrderState.Shipped);
        orders[orderId].trackingNumber = trackingNumber;
    }
    
    function confirmDelivery(uint256 orderId) external onlyBuyer(orderId) validOrder(orderId) {
        _transition(orderId, OrderState.Delivered);
        
        // Release payment to seller
        uint256 amount = orders[orderId].amount;
        address seller = orders[orderId].seller;
        
        (bool success, ) = seller.call{value: amount}("");
        require(success, "Transfer failed");
    }
    
    function cancelOrder(uint256 orderId) external validOrder(orderId) {
        Order storage order = orders[orderId];
        require(
            msg.sender == order.buyer || msg.sender == order.seller,
            "Not authorized"
        );
        
        _transition(orderId, OrderState.Cancelled);
    }
    
    function disputeOrder(uint256 orderId) external onlyBuyer(orderId) validOrder(orderId) {
        _transition(orderId, OrderState.Disputed);
    }
    
    // === Internal Transition ===
    
    function _transition(uint256 orderId, OrderState newState) internal {
        Order storage order = orders[orderId];
        OrderState oldState = order.state;
        
        if (!_validTransitions[oldState][newState]) {
            revert InvalidTransition(oldState, newState);
        }
        
        order.state = newState;
        order.lastUpdatedAt = block.timestamp;
        
        emit OrderStateChanged(orderId, oldState, newState, msg.sender);
    }
    
    // === View Functions ===
    
    function getOrderState(uint256 orderId) external view returns (OrderState) {
        return orders[orderId].state;
    }
    
    function canTransitionTo(uint256 orderId, OrderState newState) external view returns (bool) {
        return _validTransitions[orders[orderId].state][newState];
    }
    
    function isTerminalState(OrderState state) public pure returns (bool) {
        return state == OrderState.Delivered || 
               state == OrderState.Refunded;
    }
}
```

---

## 5. Struct Packing

```solidity
contract StructPacking {
    
    // === ERC-20 Token ===
    
    // ❌ Unoptimized: 4 slots
    struct TokenInfoUnpacked {
        address tokenAddress;   // 20 bytes → slot 0 (wastes 12 bytes)
        uint256 totalSupply;    // 32 bytes → slot 1
        uint8 decimals;         // 1 byte  → slot 2 (wastes 31 bytes)
        bool paused;            // 1 byte  → slot 3 (wastes 31 bytes)
    }
    
    // ✅ Optimized: 2 slots
    struct TokenInfoPacked {
        address tokenAddress;   // 20 bytes ┐
        uint8 decimals;         //  1 byte  │ = 21 bytes (slot 0: 32 bytes)
        bool paused;            //  1 byte  ┘ + 10 bytes padding
        uint256 totalSupply;    // 32 bytes → slot 1
    }
    
    // === NFT ===
    
    // ❌ 4 slots
    struct NFTUnpacked {
        address owner;          // 20 bytes → slot 0
        uint256 tokenId;        // 32 bytes → slot 1
        uint256 price;          // 32 bytes → slot 2
        bool forSale;           //  1 byte  → slot 3
    }
    
    // ✅ 3 slots
    struct NFTPacked {
        uint256 tokenId;        // 32 bytes → slot 0
        uint256 price;          // 32 bytes → slot 1
        address owner;          // 20 bytes ┐
        bool forSale;           //  1 byte  ┘ slot 2 (uses 21 bytes)
    }
    
    // === User Account ===
    
    // ❌ 5 slots
    struct UserUnpacked {
        address wallet;         // 20 bytes → slot 0
        uint256 balance;        // 32 bytes → slot 1
        uint256 lastActivity;   // 32 bytes → slot 2
        uint8 tier;             //  1 byte  → slot 3
        uint8 flags;            //  1 byte  → slot 4
    }
    
    // ✅ 2 slots
    struct UserPacked {
        uint256 balance;        // 32 bytes → slot 0
        address wallet;         // 20 bytes ┐
        uint40 lastActivity;    //  5 bytes │ slot 1
        uint8 tier;             //  1 byte  │
        uint8 flags;            //  1 byte  ┘ (total: 27 bytes, 5 padding)
    }
    
    // เก็บ timestamp ใน uint32 หรือ uint40 แทน uint256
    // uint32 สามารถเก็บได้ถึงปี 2106
    // uint40 สามารถเก็บได้ถึงปี 36,812
    
    // === Order System ===
    
    // ✅ Optimized Order: 3 slots
    struct Order {
        uint256 amount;         // 32 bytes → slot 0
        address buyer;          // 20 bytes ┐
        uint40 createdAt;       //  5 bytes │ slot 1 (25 bytes, 7 padding)
        uint32 expiry;          //  4 bytes  wait no... let me redo
        // Better packing:
        address seller;         // 20 bytes → separate field
    }
    
    // คำนวณ packing ที่ถูกต้อง:
    // slot = 32 bytes
    // Pack variables ที่รวมกันได้ ≤ 32 bytes
    
    struct BetterOrder {
        // Slot 0: 32 bytes
        uint256 amount;
        // Slot 1: 20 + 5 + 4 + 1 + 1 = 31 bytes ✅
        address buyer;          // 20 bytes
        uint40 createdAt;       //  5 bytes (max ~35k years)
        uint32 expiry;          //  4 bytes (relative days or timestamp truncated)
        uint8 status;           //  1 byte
        bool active;            //  1 byte
        // Slot 2: 20 bytes
        address seller;         // 20 bytes
    }
    
    // === Check slot ด้วย Assembly ===
    
    function checkSlot(uint256 slotNum) public view returns (bytes32 value) {
        assembly {
            value := sload(slotNum)
        }
    }
    
    // === Benchmark: Gas Difference ===
    
    TokenInfoUnpacked public unpacked;
    TokenInfoPacked public packed;
    
    function setUnpacked(address addr, uint256 supply) public {
        unpacked.tokenAddress = addr;
        unpacked.totalSupply = supply;
        // Gas cost: ~40000 (2 SSTOREs)
    }
    
    function setPacked(address addr, uint256 supply) public {
        packed.tokenAddress = addr;
        packed.totalSupply = supply;
        // Gas cost: ~40000 (2 SSTOREs)
        // อันนี้เท่ากันเพราะ 2 slot ทั้งคู่
        // แต่ถ้า add decimals และ paused ด้วย:
        // unpacked: 2 SSTOREs เพิ่ม = 40000 gas
        // packed: 0 SSTORE เพิ่ม (อยู่ใน slot เดียวกับ addr) = ฟรี!
    }
    
    function setPackedFull(address addr, uint256 supply, uint8 decimals, bool paused) public {
        packed.tokenAddress = addr;
        packed.decimals = decimals;
        packed.paused = paused;
        packed.totalSupply = supply;
        // Gas: 2 SSTOREs (addr+decimals+paused ใน slot เดียว, supply ใน slot 1)
    }
}
```

---

## 6. Workshop: Decentralized Marketplace

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title Marketplace
 * @dev Decentralized Marketplace สำหรับซื้อขายสินค้า
 */
contract Marketplace {
    
    // === Enums ===
    
    enum ListingStatus { Active, Sold, Cancelled, Expired }
    enum DisputeStatus { None, Open, ResolvedBuyer, ResolvedSeller }
    enum OfferStatus { Pending, Accepted, Rejected, Withdrawn }
    
    // === Structs ===
    
    struct Listing {
        uint256 id;
        address seller;
        string title;
        string description;
        uint256 price;       // in Wei
        uint256 quantity;
        ListingStatus status;
        uint40 createdAt;
        uint40 expiresAt;
        string[] tags;
        uint256 saleCount;
    }
    
    struct Purchase {
        uint256 id;
        uint256 listingId;
        address buyer;
        address seller;
        uint256 amount;
        uint256 quantity;
        uint40 purchasedAt;
        DisputeStatus dispute;
        bool buyerConfirmed;
        bool sellerConfirmed;
        bool fundsReleased;
    }
    
    struct Offer {
        uint256 id;
        uint256 listingId;
        address buyer;
        uint256 offerPrice;
        uint256 quantity;
        uint40 createdAt;
        uint40 expiresAt;
        OfferStatus status;
    }
    
    struct SellerProfile {
        address addr;
        string name;
        uint256 totalSales;
        uint256 totalRevenue;
        uint256 reputationScore; // 0-100
        uint256 reviewCount;
        bool verified;
        uint40 joinedAt;
    }
    
    struct Review {
        address reviewer;
        address seller;
        uint256 purchaseId;
        uint8 rating;    // 1-5
        string comment;
        uint40 createdAt;
    }
    
    // === State ===
    
    address public immutable ADMIN;
    uint256 public feePercent = 250; // 2.5% (basis points)
    uint256 public constant FEE_DENOMINATOR = 10000;
    uint256 public accumulatedFees;
    
    mapping(uint256 => Listing) public listings;
    uint256 public listingCount;
    
    mapping(uint256 => Purchase) public purchases;
    uint256 public purchaseCount;
    
    mapping(uint256 => Offer) public offers;
    uint256 public offerCount;
    
    mapping(address => SellerProfile) public sellerProfiles;
    mapping(address => uint256[]) public sellerListings;
    mapping(address => uint256[]) public buyerPurchases;
    
    mapping(uint256 => Review[]) public listingReviews;
    mapping(address => Review[]) public sellerReviews;
    
    // Funds in escrow
    mapping(uint256 => uint256) public escrowBalance; // purchaseId => amount
    
    // === Events ===
    
    event ListingCreated(uint256 indexed id, address indexed seller, uint256 price);
    event ListingUpdated(uint256 indexed id);
    event ListingCancelled(uint256 indexed id);
    event ItemPurchased(uint256 indexed purchaseId, uint256 indexed listingId, address indexed buyer);
    event OfferMade(uint256 indexed offerId, uint256 indexed listingId, address indexed buyer);
    event OfferAccepted(uint256 indexed offerId);
    event FundsReleased(uint256 indexed purchaseId, address indexed seller, uint256 amount);
    event DisputeOpened(uint256 indexed purchaseId);
    event DisputeResolved(uint256 indexed purchaseId, DisputeStatus resolution);
    event ReviewAdded(uint256 indexed listingId, address indexed reviewer, uint8 rating);
    
    // === Custom Errors ===
    
    error NotFound(uint256 id);
    error NotSeller(address caller, uint256 listingId);
    error NotBuyer(address caller, uint256 purchaseId);
    error NotActive(uint256 listingId);
    error InsufficientPayment(uint256 required, uint256 sent);
    error AlreadyConfirmed(address user, uint256 purchaseId);
    error FundsAlreadyReleased(uint256 purchaseId);
    error DisputeAlreadyOpen(uint256 purchaseId);
    error NotAuthorized();
    
    // === Constructor ===
    
    constructor() {
        ADMIN = msg.sender;
    }
    
    // === Modifiers ===
    
    modifier onlyAdmin() {
        if (msg.sender != ADMIN) revert NotAuthorized();
        _;
    }
    
    modifier listingExists(uint256 listingId) {
        if (listingId == 0 || listingId > listingCount) revert NotFound(listingId);
        _;
    }
    
    // === Seller Registration ===
    
    function registerSeller(string calldata name) external {
        require(bytes(name).length > 0 && bytes(name).length <= 50, "Invalid name");
        require(!_isSeller(msg.sender), "Already registered");
        
        sellerProfiles[msg.sender] = SellerProfile({
            addr: msg.sender,
            name: name,
            totalSales: 0,
            totalRevenue: 0,
            reputationScore: 50,
            reviewCount: 0,
            verified: false,
            joinedAt: uint40(block.timestamp)
        });
    }
    
    // === Listing Management ===
    
    function createListing(
        string calldata title,
        string calldata description,
        uint256 price,
        uint256 quantity,
        uint256 duration,
        string[] calldata tags
    ) external returns (uint256 listingId) {
        require(_isSeller(msg.sender), "Not a seller");
        require(bytes(title).length > 0, "Title required");
        require(price > 0, "Price must be > 0");
        require(quantity > 0, "Quantity must be > 0");
        require(duration > 0 && duration <= 90 days, "Invalid duration");
        require(tags.length <= 10, "Too many tags");
        
        listingId = ++listingCount;
        
        listings[listingId] = Listing({
            id: listingId,
            seller: msg.sender,
            title: title,
            description: description,
            price: price,
            quantity: quantity,
            status: ListingStatus.Active,
            createdAt: uint40(block.timestamp),
            expiresAt: uint40(block.timestamp + duration),
            tags: tags,
            saleCount: 0
        });
        
        sellerListings[msg.sender].push(listingId);
        
        emit ListingCreated(listingId, msg.sender, price);
    }
    
    function updateListing(
        uint256 listingId,
        uint256 newPrice,
        uint256 newQuantity
    ) external listingExists(listingId) {
        Listing storage listing = listings[listingId];
        
        if (listing.seller != msg.sender) revert NotSeller(msg.sender, listingId);
        if (listing.status != ListingStatus.Active) revert NotActive(listingId);
        
        if (newPrice > 0) listing.price = newPrice;
        if (newQuantity > 0) listing.quantity = newQuantity;
        
        emit ListingUpdated(listingId);
    }
    
    function cancelListing(uint256 listingId) external listingExists(listingId) {
        Listing storage listing = listings[listingId];
        
        if (listing.seller != msg.sender && msg.sender != ADMIN) {
            revert NotSeller(msg.sender, listingId);
        }
        if (listing.status != ListingStatus.Active) revert NotActive(listingId);
        
        listing.status = ListingStatus.Cancelled;
        emit ListingCancelled(listingId);
    }
    
    // === Buying ===
    
    function buyItem(
        uint256 listingId,
        uint256 quantity
    ) external payable listingExists(listingId) returns (uint256 purchaseId) {
        Listing storage listing = listings[listingId];
        
        if (listing.status != ListingStatus.Active) revert NotActive(listingId);
        require(block.timestamp < listing.expiresAt, "Listing expired");
        require(quantity > 0 && quantity <= listing.quantity, "Invalid quantity");
        
        uint256 totalPrice = listing.price * quantity;
        if (msg.value < totalPrice) revert InsufficientPayment(totalPrice, msg.value);
        
        // Calculate fee
        uint256 fee = (totalPrice * feePercent) / FEE_DENOMINATOR;
        uint256 sellerAmount = totalPrice - fee;
        
        // Update listing
        listing.quantity -= quantity;
        listing.saleCount += quantity;
        if (listing.quantity == 0) {
            listing.status = ListingStatus.Sold;
        }
        
        // Create purchase
        purchaseId = ++purchaseCount;
        purchases[purchaseId] = Purchase({
            id: purchaseId,
            listingId: listingId,
            buyer: msg.sender,
            seller: listing.seller,
            amount: sellerAmount,
            quantity: quantity,
            purchasedAt: uint40(block.timestamp),
            dispute: DisputeStatus.None,
            buyerConfirmed: false,
            sellerConfirmed: false,
            fundsReleased: false
        });
        
        // Escrow funds
        escrowBalance[purchaseId] = sellerAmount;
        accumulatedFees += fee;
        
        // Refund excess payment
        uint256 excess = msg.value - totalPrice;
        if (excess > 0) {
            (bool success, ) = msg.sender.call{value: excess}("");
            require(success, "Refund failed");
        }
        
        buyerPurchases[msg.sender].push(purchaseId);
        
        emit ItemPurchased(purchaseId, listingId, msg.sender);
    }
    
    // === Offers ===
    
    function makeOffer(
        uint256 listingId,
        uint256 offerPrice,
        uint256 quantity,
        uint256 duration
    ) external payable listingExists(listingId) returns (uint256 offerId) {
        Listing storage listing = listings[listingId];
        
        if (listing.status != ListingStatus.Active) revert NotActive(listingId);
        require(offerPrice > 0, "Invalid price");
        require(quantity > 0 && quantity <= listing.quantity, "Invalid quantity");
        require(msg.value == offerPrice * quantity, "Wrong payment");
        
        offerId = ++offerCount;
        offers[offerId] = Offer({
            id: offerId,
            listingId: listingId,
            buyer: msg.sender,
            offerPrice: offerPrice,
            quantity: quantity,
            createdAt: uint40(block.timestamp),
            expiresAt: uint40(block.timestamp + duration),
            status: OfferStatus.Pending
        });
        
        emit OfferMade(offerId, listingId, msg.sender);
    }
    
    function acceptOffer(uint256 offerId) external returns (uint256 purchaseId) {
        Offer storage offer = offers[offerId];
        require(offer.status == OfferStatus.Pending, "Not pending");
        require(block.timestamp < offer.expiresAt, "Offer expired");
        
        Listing storage listing = listings[offer.listingId];
        require(listing.seller == msg.sender, "Not seller");
        
        offer.status = OfferStatus.Accepted;
        
        // Create purchase from offer
        purchaseId = ++purchaseCount;
        uint256 totalPrice = offer.offerPrice * offer.quantity;
        uint256 fee = (totalPrice * feePercent) / FEE_DENOMINATOR;
        
        purchases[purchaseId] = Purchase({
            id: purchaseId,
            listingId: offer.listingId,
            buyer: offer.buyer,
            seller: msg.sender,
            amount: totalPrice - fee,
            quantity: offer.quantity,
            purchasedAt: uint40(block.timestamp),
            dispute: DisputeStatus.None,
            buyerConfirmed: false,
            sellerConfirmed: false,
            fundsReleased: false
        });
        
        escrowBalance[purchaseId] = totalPrice - fee;
        accumulatedFees += fee;
        
        emit OfferAccepted(offerId);
        emit ItemPurchased(purchaseId, offer.listingId, offer.buyer);
    }
    
    // === Confirmation & Release ===
    
    function confirmDelivery(uint256 purchaseId) external {
        Purchase storage purchase = purchases[purchaseId];
        
        require(purchaseId > 0 && purchaseId <= purchaseCount, "Not found");
        if (purchase.buyer != msg.sender) revert NotBuyer(msg.sender, purchaseId);
        if (purchase.buyerConfirmed) revert AlreadyConfirmed(msg.sender, purchaseId);
        if (purchase.fundsReleased) revert FundsAlreadyReleased(purchaseId);
        
        purchase.buyerConfirmed = true;
        
        // Auto-release if both confirmed
        if (purchase.sellerConfirmed) {
            _releaseFunds(purchaseId);
        }
    }
    
    function confirmShipped(uint256 purchaseId) external {
        Purchase storage purchase = purchases[purchaseId];
        
        if (purchase.seller != msg.sender) revert NotSeller(msg.sender, purchaseId);
        if (purchase.sellerConfirmed) revert AlreadyConfirmed(msg.sender, purchaseId);
        
        purchase.sellerConfirmed = true;
        
        if (purchase.buyerConfirmed) {
            _releaseFunds(purchaseId);
        }
    }
    
    function _releaseFunds(uint256 purchaseId) internal {
        Purchase storage purchase = purchases[purchaseId];
        
        if (purchase.fundsReleased) revert FundsAlreadyReleased(purchaseId);
        
        purchase.fundsReleased = true;
        uint256 amount = escrowBalance[purchaseId];
        escrowBalance[purchaseId] = 0;
        
        // Update seller stats
        SellerProfile storage profile = sellerProfiles[purchase.seller];
        profile.totalSales++;
        profile.totalRevenue += amount;
        
        (bool success, ) = purchase.seller.call{value: amount}("");
        require(success, "Transfer failed");
        
        emit FundsReleased(purchaseId, purchase.seller, amount);
    }
    
    // === Reviews ===
    
    function addReview(
        uint256 purchaseId,
        uint8 rating,
        string calldata comment
    ) external {
        Purchase storage purchase = purchases[purchaseId];
        
        require(purchase.buyer == msg.sender, "Not buyer");
        require(purchase.fundsReleased, "Purchase not completed");
        require(rating >= 1 && rating <= 5, "Rating must be 1-5");
        
        Review memory review = Review({
            reviewer: msg.sender,
            seller: purchase.seller,
            purchaseId: purchaseId,
            rating: rating,
            comment: comment,
            createdAt: uint40(block.timestamp)
        });
        
        listingReviews[purchase.listingId].push(review);
        sellerReviews[purchase.seller].push(review);
        
        // Update reputation
        SellerProfile storage profile = sellerProfiles[purchase.seller];
        uint256 totalRating = profile.reputationScore * profile.reviewCount + rating * 20;
        profile.reviewCount++;
        profile.reputationScore = totalRating / profile.reviewCount;
        
        emit ReviewAdded(purchase.listingId, msg.sender, rating);
    }
    
    // === Admin ===
    
    function withdrawFees() external onlyAdmin {
        uint256 fees = accumulatedFees;
        accumulatedFees = 0;
        
        (bool success, ) = ADMIN.call{value: fees}("");
        require(success, "Transfer failed");
    }
    
    function verifySeller(address seller) external onlyAdmin {
        require(_isSeller(seller), "Not a seller");
        sellerProfiles[seller].verified = true;
    }
    
    function resolveDispute(
        uint256 purchaseId,
        bool favorBuyer
    ) external onlyAdmin {
        Purchase storage purchase = purchases[purchaseId];
        require(purchase.dispute == DisputeStatus.Open, "No open dispute");
        
        uint256 amount = escrowBalance[purchaseId];
        escrowBalance[purchaseId] = 0;
        purchase.fundsReleased = true;
        
        if (favorBuyer) {
            purchase.dispute = DisputeStatus.ResolvedBuyer;
            (bool success, ) = purchase.buyer.call{value: amount}("");
            require(success, "Transfer failed");
        } else {
            purchase.dispute = DisputeStatus.ResolvedSeller;
            (bool success, ) = purchase.seller.call{value: amount}("");
            require(success, "Transfer failed");
        }
        
        emit DisputeResolved(purchaseId, purchase.dispute);
    }
    
    // === View Functions ===
    
    function getListingsByTag(string calldata tag) 
        external view returns (uint256[] memory result) 
    {
        uint256 count = 0;
        for (uint256 i = 1; i <= listingCount; i++) {
            if (_hasTag(i, tag)) count++;
        }
        
        result = new uint256[](count);
        uint256 idx = 0;
        for (uint256 i = 1; i <= listingCount; i++) {
            if (_hasTag(i, tag)) result[idx++] = i;
        }
    }
    
    function _hasTag(uint256 listingId, string calldata tag) internal view returns (bool) {
        string[] storage tags = listings[listingId].tags;
        for (uint256 i = 0; i < tags.length; i++) {
            if (keccak256(bytes(tags[i])) == keccak256(bytes(tag))) return true;
        }
        return false;
    }
    
    function _isSeller(address addr) internal view returns (bool) {
        return sellerProfiles[addr].joinedAt != 0;
    }
    
    function getSellerListings(address seller) external view returns (uint256[] memory) {
        return sellerListings[seller];
    }
    
    function getBuyerPurchases(address buyer) external view returns (uint256[] memory) {
        return buyerPurchases[buyer];
    }
    
    function getListingReviews(uint256 listingId) external view returns (Review[] memory) {
        return listingReviews[listingId];
    }
    
    receive() external payable {}
}
```

---

## สรุป Part 07

Structs และ Enums ที่เรียนรู้:
- ✅ Structs เชิงลึก (nested, with arrays, with mappings)
- ✅ Struct patterns (entity registry, linked list, timelock)
- ✅ Enums เชิงลึก (type casting, validation)
- ✅ State machines ด้วย Enums
- ✅ Struct packing (gas optimization)
- ✅ Decentralized Marketplace

## Quiz

1. ทำไม `address` ควรอยู่ท้ายใน Packed Struct?
2. State Machine pattern ช่วยแก้ปัญหาอะไร?
3. ต่างกันอย่างไรระหว่าง `memory` และ `storage` เมื่อใช้กับ struct?
4. `uint40` เหมาะกับ timestamp หรือไม่ ทำไม?

---

## Next: Part 08 - Events และ Error Handling
