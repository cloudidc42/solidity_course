# Part 84: Restaking & EigenLayer (การ Restake และ EigenLayer)

## บทนำ

**Restaking** คือนวัตกรรมที่ช่วยให้ stake ETH ที่ล็อคอยู่แล้วใน Ethereum consensus สามารถ "นำมาใช้ซ้ำ" เพื่อ secure additional services ได้ **EigenLayer** เป็น protocol แรกที่ทำให้แนวคิดนี้เป็นจริง ก่อตั้งโดย Sreeram Kannan ใน 2021 และ launch บน mainnet ปี 2024

**ทำไม Restaking ถึงสำคัญ?**

ก่อน EigenLayer: การสร้าง decentralized service ใหม่ (เช่น oracle, bridge, rollup sequencer) ต้องสร้าง economic security ใหม่ทั้งหมด → ยากมาก, ต้นทุนสูง

หลัง EigenLayer: ใช้ security ของ Ethereum ที่มีอยู่แล้ว ($30B+ staked ETH) → สร้าง service ได้เร็วขึ้น ปลอดภัยกว่า

---

## 84.1 EigenLayer Architecture

### ภาพรวมระบบ

```
EigenLayer Architecture:
                                                            
Ethereum Stakers                EigenLayer                  AVS Services
───────────────                ──────────                   ─────────────
                                                            
ETH Stakers  ──────►  IStrategyManager              ┌──── EigenDA
             restake  (deposit LSTs,                 │     (Data Availability)
                       native ETH)                   │
                              │                      ├──── AltLayer
LST Holders ──────►  IStrategy                      │     (Rollup Sequencer)
(stETH, etc)  restake  (deposit/withdraw)            │
                              │                      ├──── Lagrange
                       IDelegationManager            │     (ZK Coprocessor)
                       (delegate to operators)       │
                              │                      └──── Your Custom AVS
                       IOperators
                       (validate AVS tasks)
                              │
                       ISlasher
                       (slash on misbehavior)
```

### 84.1.1 Key Participants

| บทบาท | หน้าที่ | ความเสี่ยง |
|--------|---------|-----------|
| **Restakers** | ฝาก ETH/LST เพื่อ earn extra yield | Slashing หากไม่ทำหน้าที่ |
| **Operators** | รัน AVS software, validate tasks | Slashing หากทุจริต |
| **AVS (Actively Validated Services)** | สร้าง service ที่ใช้ EigenLayer security | ต้องออกแบบ slashing ให้ดี |
| **Delegators** | มอบ stake ให้ operator | ได้ส่วนแบ่งจาก operator |

### 84.1.2 AVS (Actively Validated Services)

```
AVS คือ service ใดก็ได้ที่ต้องการ:
1. Economic Security (stake เป็นหลักประกัน)
2. Decentralized validation (operators ตรวจสอบ)
3. Cryptographic slashing (บทลงโทษสำหรับการทุจริต)

ตัวอย่าง AVS:
- Oracles: ส่งราคาถูกต้อง หรือถูก slash
- Bridges: relay message ถูกต้อง หรือถูก slash  
- Rollup Sequencers: order transactions ถูกต้อง หรือถูก slash
- Keeper Networks: execute tasks ตามเวลา หรือถูก slash
- AI Inference: ผล inference ถูกต้อง หรือถูก slash
```

---

## 84.2 IStrategy Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title IStrategy
 * @notice Interface สำหรับ EigenLayer Strategy contracts
 * @dev แต่ละ Strategy จัดการ deposit/withdraw ของ token หนึ่งชนิด
 *      เช่น stETHStrategy, rETHStrategy, cbETHStrategy
 */
interface IStrategy {
    
    event Deposit(address depositor, IERC20 token, uint256 shares);
    event Withdrawal(address depositor, IERC20 token, uint256 shares);
    
    /**
     * @notice ฝาก token และรับ shares
     * @param token token ที่จะฝาก
     * @param amount จำนวนที่ฝาก
     * @return newShares shares ที่ได้รับ
     */
    function deposit(
        IERC20 token,
        uint256 amount
    ) external returns (uint256 newShares);
    
    /**
     * @notice ถอน shares คืนเป็น token
     * @param depositor เจ้าของ shares
     * @param token token ที่ต้องการถอน
     * @param amountShares จำนวน shares ที่ถอน
     */
    function withdraw(
        address depositor,
        IERC20 token,
        uint256 amountShares
    ) external;
    
    /**
     * @notice แปลง shares เป็นจำนวน token
     */
    function sharesToUnderlying(uint256 amountShares) 
        external view returns (uint256);
    
    /**
     * @notice แปลง token เป็นจำนวน shares
     */
    function underlyingToShares(uint256 amountUnderlying) 
        external view returns (uint256);
    
    /**
     * @notice ดู shares ทั้งหมด
     */
    function totalShares() external view returns (uint256);
    
    /**
     * @notice token ที่ strategy นี้รองรับ
     */
    function underlyingToken() external view returns (IERC20);
    
    /**
     * @notice คำนวณ underlying balance ของ address
     */
    function userUnderlyingView(address user) external view returns (uint256);
}

/**
 * @title IStrategyManager
 * @notice Interface หลักสำหรับ manage strategies
 */
interface IStrategyManager {
    
    event Deposit(
        address depositor,
        IERC20 token,
        IStrategy strategy,
        uint256 shares
    );
    
    event ShareWithdrawalQueued(
        address depositor,
        uint96 nonce,
        address withdrawer,
        IStrategy[] strategies,
        uint256[] shares,
        bytes32 withdrawalRoot
    );
    
    /**
     * @notice ฝาก token เข้า strategy
     */
    function depositIntoStrategy(
        IStrategy strategy,
        IERC20 token,
        uint256 amount
    ) external returns (uint256 shares);
    
    /**
     * @notice ฝากด้วย signature (EIP-712)
     */
    function depositIntoStrategyWithSignature(
        IStrategy strategy,
        IERC20 token,
        uint256 amount,
        address staker,
        uint256 expiry,
        bytes calldata signature
    ) external returns (uint256 shares);
    
    /**
     * @notice ดู shares ที่ depositor มีใน strategy
     */
    function stakerStrategyShares(
        address staker,
        IStrategy strategy
    ) external view returns (uint256 shares);
    
    /**
     * @notice ดู strategies ทั้งหมดของ staker
     */
    function getDeposits(address depositor) 
        external view returns (IStrategy[] memory, uint256[] memory);
    
    /**
     * @notice ดูว่า strategy ใดที่ approved
     */
    function strategyIsWhitelistedForDeposit(IStrategy strategy) 
        external view returns (bool);
}
```

### 84.2.1 LST Strategy Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title LSTRestakingStrategy
 * @notice Strategy สำหรับ restake Liquid Staking Tokens (stETH, rETH, etc.)
 * @dev Implements IStrategy interface
 *      แปลง LST เป็น shares ตาม exchange rate ปัจจุบัน
 */
contract LSTRestakingStrategy is IStrategy, ReentrancyGuard, Ownable {
    using SafeERC20 for IERC20;
    
    // ============ State ============
    
    IERC20 public immutable override underlyingToken;
    
    uint256 private _totalShares;
    mapping(address => uint256) private _shares;
    
    // StrategyManager เป็นผู้จัดการ
    address public immutable strategyManager;
    
    // Exchange rate cache (อัปเดตจาก oracle)
    uint256 public exchangeRate;         // shares ต่อ 1 underlying token (scaled 1e18)
    uint256 public lastExchangeRateUpdate;
    
    uint256 public constant MAX_TOTAL_DEPOSITS = 1_000_000 ether; // 1M token limit
    uint256 public totalDeposited;
    
    // ============ Events ============
    
    event ExchangeRateUpdated(uint256 newRate, uint256 timestamp);
    
    // ============ Modifiers ============
    
    modifier onlyStrategyManager() {
        require(msg.sender == strategyManager, "Not strategy manager");
        _;
    }
    
    // ============ Constructor ============
    
    constructor(
        IERC20 _underlyingToken,
        address _strategyManager
    ) Ownable(msg.sender) {
        underlyingToken = _underlyingToken;
        strategyManager = _strategyManager;
        exchangeRate = 1e18; // 1:1 เริ่มต้น
        lastExchangeRateUpdate = block.timestamp;
    }
    
    // ============ IStrategy Implementation ============
    
    /**
     * @notice ฝาก token และรับ shares
     * @dev เรียกโดย StrategyManager เท่านั้น
     */
    function deposit(
        IERC20 token,
        uint256 amount
    ) external override onlyStrategyManager nonReentrant returns (uint256 newShares) {
        require(token == underlyingToken, "Wrong token");
        require(amount > 0, "Zero amount");
        require(totalDeposited + amount <= MAX_TOTAL_DEPOSITS, "Deposit limit exceeded");
        
        // คำนวณ shares ตาม current exchange rate
        newShares = underlyingToShares(amount);
        require(newShares > 0, "Zero shares");
        
        // อัปเดต state
        _totalShares += newShares;
        totalDeposited += amount;
        
        // StrategyManager จะ track shares ต่อ depositor
        // (strategy ไม่ track เอง - ให้ StrategyManager จัดการ)
        
        emit Deposit(msg.sender, token, newShares);
    }
    
    /**
     * @notice ถอน shares คืนเป็น token
     * @dev เรียกโดย StrategyManager เท่านั้น
     */
    function withdraw(
        address depositor,
        IERC20 token,
        uint256 amountShares
    ) external override onlyStrategyManager nonReentrant {
        require(token == underlyingToken, "Wrong token");
        require(amountShares > 0, "Zero shares");
        
        // คำนวณ underlying amount
        uint256 underlyingAmount = sharesToUnderlying(amountShares);
        require(underlyingAmount > 0, "Zero underlying");
        
        // อัปเดต state
        require(_totalShares >= amountShares, "Insufficient total shares");
        _totalShares -= amountShares;
        totalDeposited -= underlyingAmount;
        
        // โอน token คืน
        underlyingToken.safeTransfer(depositor, underlyingAmount);
        
        emit Withdrawal(depositor, token, amountShares);
    }
    
    // ============ View Functions ============
    
    function sharesToUnderlying(uint256 amountShares) 
        public view override returns (uint256) 
    {
        if (_totalShares == 0 || totalDeposited == 0) {
            return amountShares; // 1:1 when empty
        }
        return (amountShares * totalDeposited) / _totalShares;
    }
    
    function underlyingToShares(uint256 amountUnderlying) 
        public view override returns (uint256) 
    {
        if (_totalShares == 0 || totalDeposited == 0) {
            return amountUnderlying; // 1:1 when empty
        }
        return (amountUnderlying * _totalShares) / totalDeposited;
    }
    
    function totalShares() external view override returns (uint256) {
        return _totalShares;
    }
    
    function userUnderlyingView(address user) external view override returns (uint256) {
        // ต้อง query จาก StrategyManager
        // สำหรับตัวอย่างนี้ simplified
        return 0;
    }
    
    // ============ Admin Functions ============
    
    /**
     * @notice อัปเดต exchange rate จาก oracle
     */
    function updateExchangeRate(uint256 _newRate) external onlyOwner {
        require(_newRate > 0, "Invalid rate");
        exchangeRate = _newRate;
        lastExchangeRateUpdate = block.timestamp;
        emit ExchangeRateUpdated(_newRate, block.timestamp);
    }
}
```

---

## 84.3 IServiceManager: AVS Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title IServiceManager
 * @notice Interface หลักสำหรับ AVS (Actively Validated Service)
 * @dev AVS ต้อง implement interface นี้เพื่อ integrate กับ EigenLayer
 */
interface IServiceManager {
    
    // ============ Operator Management ============
    
    /**
     * @notice ลงทะเบียน operator สำหรับ AVS นี้
     * @param operatorSignature signature ที่ยืนยันว่า operator consent
     */
    function registerOperatorToAVS(
        address operator,
        bytes calldata operatorSignature
    ) external;
    
    /**
     * @notice ยกเลิกการลงทะเบียน operator
     */
    function deregisterOperatorFromAVS(address operator) external;
    
    /**
     * @notice ดูว่า operator ลงทะเบียนกับ AVS นี้หรือไม่
     */
    function isOperatorRegistered(address operator) external view returns (bool);
    
    // ============ Slashing ============
    
    /**
     * @notice ส่งคำขอ slash operator (ต้องผ่าน EigenLayer)
     * @dev เรียกเมื่อ operator ทำผิดพลาดหรือทุจริต
     */
    function freezeOperator(address operator) external;
    
    /**
     * @notice ดู staking ทั้งหมดที่ delegate ให้ operator
     */
    function getOperatorStake(address operator) external view returns (uint256);
    
    // ============ Task Management ============
    
    /**
     * @notice สร้าง task ใหม่ที่ operators ต้องตอบ
     */
    function createNewTask(bytes32 taskData) external returns (uint32 taskIndex);
    
    /**
     * @notice Respond to task (โดย operators)
     */
    function respondToTask(
        uint32 taskIndex,
        bytes32 response,
        bytes calldata signature
    ) external;
    
    /**
     * @notice Challenge response (ถ้า response ผิด)
     */
    function challengeResponse(
        uint32 taskIndex,
        address operator,
        bytes calldata proof
    ) external;
}

/**
 * @title ISlasher
 * @notice Interface สำหรับ EigenLayer Slasher
 */
interface ISlasher {
    
    event OperatorFrozen(address indexed frozenOperator, address indexed slashingContract);
    event FrozenStatusReset(address indexed previouslySlashedAddress);
    
    /**
     * @notice Freeze operator (ทำให้ withdraw ไม่ได้ชั่วคราว)
     */
    function freezeOperator(address toBeSlashed) external;
    
    /**
     * @notice Reset frozen status (หลังจัดการเรื่อง slashing แล้ว)
     */
    function resetFrozenStatus(address[] calldata frozenAddresses) external;
    
    /**
     * @notice ดูว่า operator ถูก freeze อยู่หรือไม่
     */
    function isFrozen(address staker) external view returns (bool);
    
    /**
     * @notice ดู contracts ที่สามารถ slash operator นี้ได้
     */
    function canSlash(address toBeSlashed, address slashingContract) 
        external view returns (bool);
}

/**
 * @title IDelegationManager
 * @notice Interface สำหรับ delegation ของ stake
 */
interface IDelegationManager {
    
    event OperatorRegistered(address indexed operator, OperatorDetails details);
    event OperatorMetadataURIUpdated(address indexed operator, string metadataURI);
    event Delegated(address indexed staker, address indexed operator);
    event Undelegated(address indexed staker, address indexed operator);
    
    struct OperatorDetails {
        address earningsReceiver;       // รับ earnings
        address delegationApprover;     // approve delegations
        uint32 stakerOptOutWindowBlocks; // blocks ก่อน undelegation มีผล
    }
    
    /**
     * @notice ลงทะเบียนเป็น operator
     */
    function registerAsOperator(
        OperatorDetails calldata registeringOperatorDetails,
        string calldata metadataURI
    ) external;
    
    /**
     * @notice Delegate stake ให้ operator
     */
    function delegateTo(
        address operator,
        bytes calldata approverSignatureAndExpiry,
        bytes32 approverSalt
    ) external;
    
    /**
     * @notice ยกเลิก delegation
     */
    function undelegate() external;
    
    /**
     * @notice ดู operator ที่ staker delegate ให้
     */
    function delegatedTo(address staker) external view returns (address);
    
    /**
     * @notice ดูว่า address เป็น operator หรือไม่
     */
    function isOperator(address operator) external view returns (bool);
    
    /**
     * @notice คำนวณ stake ทั้งหมดที่ delegate ให้ operator (ต่อ strategy)
     */
    function operatorShares(address operator, IStrategy strategy) 
        external view returns (uint256);
}
```

---

## 84.4 Simple AVS Implementation: OracleAVS

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title OracleAVS
 * @notice Simplified Oracle AVS ที่ใช้ EigenLayer security
 * @dev Operators ต้องส่งราคาและ BLS signature aggregate
 *      ถ้าส่งราคาผิด → ถูก slash
 * 
 * สถาปัตยกรรม:
 * 1. Task Creator สร้าง price request task
 * 2. Operators แต่ละคนตอบด้วย signature
 * 3. เมื่อครบ threshold → aggregate + finalize
 * 4. ถ้ามีการ challenge ที่ valid → slash operators ที่ตอบผิด
 */
contract OracleAVS is Ownable, ReentrancyGuard {
    
    // ============ Structs ============
    
    struct Task {
        bytes32 taskHash;           // hash ของ task parameters
        uint32 taskCreatedBlock;    // block ที่สร้าง
        uint32 respondByBlock;      // deadline
        address[] requiredFeeds;    // assets ที่ต้องการราคา
        TaskStatus status;
    }
    
    struct TaskResponse {
        uint32 taskIndex;
        int256[] prices;            // ราคาต่อ asset
        uint256 responseTimestamp;
        bytes signature;            // BLS signature
    }
    
    struct PriceData {
        int256 price;
        uint256 timestamp;
        uint32 respondersCount;
        bool finalized;
    }
    
    struct OperatorInfo {
        bool isRegistered;
        uint256 registeredAt;
        uint256 totalResponseCount;
        uint256 correctResponseCount;
        bool isFrozen;
        bytes blsPublicKey;    // BLS public key สำหรับ signature aggregation
    }
    
    enum TaskStatus {
        Created,
        Responded,    // มีบาง operators ตอบแล้ว
        Finalized,    // ครบ threshold
        Challenged,   // กำลังถูก challenge
        Disputed      // dispute resolved
    }
    
    // ============ State ============
    
    // EigenLayer contracts
    address public immutable eigenLayerServiceManager; // Simplified: address only
    address public immutable slasher;
    
    // Tasks
    Task[] public tasks;
    uint32 public latestTaskNum;
    
    // Task responses: taskIndex => operator => response
    mapping(uint32 => mapping(address => TaskResponse)) public taskResponses;
    mapping(uint32 => address[]) public taskRespondents;
    
    // Finalized prices
    mapping(uint32 => mapping(address => PriceData)) public finalizedPrices;
    
    // Operators
    mapping(address => OperatorInfo) public operators;
    address[] public registeredOperators;
    
    // Configuration
    uint32 public constant RESPONSE_WINDOW_BLOCKS = 50;
    uint32 public constant THRESHOLD_DENOMINATOR = 3; // 2/3 majority
    uint256 public constant PRICE_DEVIATION_THRESHOLD = 100; // 1% in bps
    
    // Task creator whitelist
    mapping(address => bool) public taskCreators;
    
    // ============ Events ============
    
    event OperatorRegistered(address indexed operator, bytes blsPubKey);
    event OperatorDeregistered(address indexed operator);
    event NewTaskCreated(uint32 indexed taskIndex, Task task);
    event TaskResponded(uint32 indexed taskIndex, TaskResponse response, address operator);
    event TaskFinalized(uint32 indexed taskIndex, int256[] medianPrices);
    event OperatorSlashed(address indexed operator, uint32 taskIndex, string reason);
    event PriceUpdated(address indexed asset, int256 price, uint256 timestamp);
    
    // ============ Errors ============
    
    error NotRegisteredOperator();
    error OperatorAlreadyRegistered();
    error TaskNotExists();
    error ResponseWindowClosed();
    error AlreadyResponded();
    error InsufficientResponses();
    error NotTaskCreator();
    error OperatorFrozen();
    
    // ============ Constructor ============
    
    constructor(
        address _eigenLayerServiceManager,
        address _slasher
    ) Ownable(msg.sender) {
        eigenLayerServiceManager = _eigenLayerServiceManager;
        slasher = _slasher;
        taskCreators[msg.sender] = true;
    }
    
    // ============ Operator Management ============
    
    /**
     * @notice ลงทะเบียนเป็น operator ของ AVS นี้
     * @param _blsPublicKey BLS public key สำหรับ signature aggregation
     * @dev ในการใช้งานจริงต้องตรวจสอบว่า operator ลงทะเบียนใน EigenLayer แล้ว
     */
    function registerOperator(bytes calldata _blsPublicKey) external {
        if (operators[msg.sender].isRegistered) revert OperatorAlreadyRegistered();
        
        // ในการใช้งานจริง: ตรวจสอบจาก EigenLayer DelegationManager
        // IDelegationManager(eigenLayerDelegationManager).isOperator(msg.sender)
        
        operators[msg.sender] = OperatorInfo({
            isRegistered: true,
            registeredAt: block.timestamp,
            totalResponseCount: 0,
            correctResponseCount: 0,
            isFrozen: false,
            blsPublicKey: _blsPublicKey
        });
        
        registeredOperators.push(msg.sender);
        
        // ในการใช้งานจริง: เรียก IServiceManager.registerOperatorToAVS()
        
        emit OperatorRegistered(msg.sender, _blsPublicKey);
    }
    
    /**
     * @notice ยกเลิกการลงทะเบียน
     */
    function deregisterOperator() external {
        if (!operators[msg.sender].isRegistered) revert NotRegisteredOperator();
        
        operators[msg.sender].isRegistered = false;
        
        // ลบจาก array
        for (uint256 i = 0; i < registeredOperators.length; i++) {
            if (registeredOperators[i] == msg.sender) {
                registeredOperators[i] = registeredOperators[registeredOperators.length - 1];
                registeredOperators.pop();
                break;
            }
        }
        
        emit OperatorDeregistered(msg.sender);
    }
    
    // ============ Task Management ============
    
    /**
     * @notice สร้าง price request task ใหม่
     * @param _assets assets ที่ต้องการราคา (token addresses)
     */
    function createNewTask(
        address[] calldata _assets
    ) external returns (uint32 taskIndex) {
        if (!taskCreators[msg.sender]) revert NotTaskCreator();
        require(_assets.length > 0 && _assets.length <= 50, "Invalid assets");
        
        taskIndex = latestTaskNum;
        
        tasks.push(Task({
            taskHash: keccak256(abi.encodePacked(
                _assets, 
                block.timestamp, 
                taskIndex
            )),
            taskCreatedBlock: uint32(block.number),
            respondByBlock: uint32(block.number + RESPONSE_WINDOW_BLOCKS),
            requiredFeeds: _assets,
            status: TaskStatus.Created
        }));
        
        latestTaskNum++;
        
        emit NewTaskCreated(taskIndex, tasks[taskIndex]);
    }
    
    /**
     * @notice Operator ตอบ task ด้วยราคาและ signature
     * @param _taskIndex index ของ task
     * @param _prices ราคาของแต่ละ asset (scaled 1e8)
     * @param _signature BLS/ECDSA signature ของ response
     */
    function respondToTask(
        uint32 _taskIndex,
        int256[] calldata _prices,
        bytes calldata _signature
    ) external {
        if (_taskIndex >= latestTaskNum) revert TaskNotExists();
        if (!operators[msg.sender].isRegistered) revert NotRegisteredOperator();
        if (operators[msg.sender].isFrozen) revert OperatorFrozen();
        
        Task storage task = tasks[_taskIndex];
        if (block.number > task.respondByBlock) revert ResponseWindowClosed();
        if (taskResponses[_taskIndex][msg.sender].responseTimestamp != 0) 
            revert AlreadyResponded();
        
        require(_prices.length == task.requiredFeeds.length, "Price count mismatch");
        
        // ตรวจสอบ signature (simplified)
        bytes32 responseHash = keccak256(abi.encodePacked(
            _taskIndex, 
            _prices, 
            msg.sender,
            block.chainid
        ));
        require(_verifySignature(responseHash, _signature, msg.sender), 
                "Invalid signature");
        
        // บันทึก response
        taskResponses[_taskIndex][msg.sender] = TaskResponse({
            taskIndex: _taskIndex,
            prices: _prices,
            responseTimestamp: block.timestamp,
            signature: _signature
        });
        taskRespondents[_taskIndex].push(msg.sender);
        
        operators[msg.sender].totalResponseCount++;
        
        // อัปเดต task status
        if (task.status == TaskStatus.Created) {
            task.status = TaskStatus.Responded;
        }
        
        emit TaskResponded(_taskIndex, taskResponses[_taskIndex][msg.sender], msg.sender);
        
        // ตรวจสอบว่าครบ threshold แล้วหรือยัง
        uint256 responseCount = taskRespondents[_taskIndex].length;
        uint256 requiredCount = (registeredOperators.length * 2) / THRESHOLD_DENOMINATOR;
        
        if (responseCount >= requiredCount) {
            _finalizeTask(_taskIndex);
        }
    }
    
    /**
     * @dev Finalize task โดยคำนวณ median price
     */
    function _finalizeTask(uint32 _taskIndex) internal {
        Task storage task = tasks[_taskIndex];
        if (task.status == TaskStatus.Finalized) return;
        
        address[] memory respondents = taskRespondents[_taskIndex];
        uint256 numFeeds = task.requiredFeeds.length;
        
        int256[] memory medianPrices = new int256[](numFeeds);
        
        // คำนวณ median สำหรับแต่ละ asset
        for (uint256 feedIdx = 0; feedIdx < numFeeds; feedIdx++) {
            int256[] memory pricesForFeed = new int256[](respondents.length);
            
            for (uint256 i = 0; i < respondents.length; i++) {
                pricesForFeed[i] = taskResponses[_taskIndex][respondents[i]].prices[feedIdx];
            }
            
            // Sort (bubble sort สำหรับ simplicity)
            _sortPrices(pricesForFeed);
            
            // Median
            uint256 midIdx = respondents.length / 2;
            medianPrices[feedIdx] = respondents.length % 2 == 0
                ? (pricesForFeed[midIdx - 1] + pricesForFeed[midIdx]) / 2
                : pricesForFeed[midIdx];
            
            // บันทึก finalized price
            finalizedPrices[_taskIndex][task.requiredFeeds[feedIdx]] = PriceData({
                price: medianPrices[feedIdx],
                timestamp: block.timestamp,
                respondersCount: uint32(respondents.length),
                finalized: true
            });
            
            emit PriceUpdated(
                task.requiredFeeds[feedIdx], 
                medianPrices[feedIdx], 
                block.timestamp
            );
        }
        
        task.status = TaskStatus.Finalized;
        
        // อัปเดต correctResponseCount สำหรับ operators ที่ตอบใกล้เคียง median
        _updateOperatorScores(_taskIndex, medianPrices);
        
        emit TaskFinalized(_taskIndex, medianPrices);
    }
    
    /**
     * @dev อัปเดต scores ของ operators
     */
    function _updateOperatorScores(
        uint32 _taskIndex, 
        int256[] memory _medianPrices
    ) internal {
        address[] memory respondents = taskRespondents[_taskIndex];
        
        for (uint256 i = 0; i < respondents.length; i++) {
            address op = respondents[i];
            TaskResponse memory response = taskResponses[_taskIndex][op];
            
            bool allAccurate = true;
            for (uint256 j = 0; j < _medianPrices.length; j++) {
                int256 deviation = response.prices[j] - _medianPrices[j];
                if (deviation < 0) deviation = -deviation;
                
                // deviation > 1% → ถือว่าไม่ accurate
                int256 threshold = (_medianPrices[j] * int256(PRICE_DEVIATION_THRESHOLD)) / 10000;
                if (threshold < 0) threshold = -threshold;
                
                if (deviation > threshold) {
                    allAccurate = false;
                    break;
                }
            }
            
            if (allAccurate) {
                operators[op].correctResponseCount++;
            }
        }
    }
    
    // ============ Challenge Mechanism ============
    
    /**
     * @notice Challenge operator response ที่ผิด
     * @param _taskIndex task index
     * @param _operator operator ที่ถูก challenge
     * @param _proof proof ว่า response ผิด
     */
    function challengeOperatorResponse(
        uint32 _taskIndex,
        address _operator,
        bytes calldata _proof
    ) external {
        Task storage task = tasks[_taskIndex];
        require(task.status == TaskStatus.Finalized, "Task not finalized");
        require(operators[_operator].isRegistered, "Not an operator");
        
        TaskResponse memory response = taskResponses[_taskIndex][_operator];
        require(response.responseTimestamp > 0, "Operator did not respond");
        
        // ตรวจสอบ proof (simplified)
        bool isInvalid = _verifyChallenge(
            _taskIndex, 
            _operator, 
            response.prices, 
            _proof
        );
        
        if (isInvalid) {
            // Slash operator ผ่าน EigenLayer
            _slashOperator(_operator, _taskIndex, "Price deviation too large");
            
            task.status = TaskStatus.Disputed;
        }
    }
    
    /**
     * @dev Slash operator ผ่าน EigenLayer Slasher
     */
    function _slashOperator(
        address _operator,
        uint32 _taskIndex,
        string memory _reason
    ) internal {
        operators[_operator].isFrozen = true;
        
        // ในการใช้งานจริง: เรียก EigenLayer Slasher
        // ISlasher(slasher).freezeOperator(_operator);
        
        emit OperatorSlashed(_operator, _taskIndex, _reason);
    }
    
    // ============ Helper Functions ============
    
    function _sortPrices(int256[] memory arr) internal pure {
        uint256 n = arr.length;
        for (uint256 i = 0; i < n - 1; i++) {
            for (uint256 j = 0; j < n - i - 1; j++) {
                if (arr[j] > arr[j + 1]) {
                    int256 temp = arr[j];
                    arr[j] = arr[j + 1];
                    arr[j + 1] = temp;
                }
            }
        }
    }
    
    function _verifySignature(
        bytes32 _hash,
        bytes memory _signature,
        address _signer
    ) internal pure returns (bool) {
        // Simplified ECDSA verification
        if (_signature.length != 65) return false;
        
        bytes32 r;
        bytes32 s;
        uint8 v;
        assembly {
            r := mload(add(_signature, 32))
            s := mload(add(_signature, 64))
            v := byte(0, mload(add(_signature, 96)))
        }
        
        if (v < 27) v += 27;
        
        bytes32 ethHash = keccak256(
            abi.encodePacked("\x19Ethereum Signed Message:\n32", _hash)
        );
        
        return ecrecover(ethHash, v, r, s) == _signer;
    }
    
    function _verifyChallenge(
        uint32 _taskIndex,
        address _operator,
        int256[] memory _prices,
        bytes calldata _proof
    ) internal view returns (bool) {
        // ในการใช้งานจริง: ตรวจสอบ cryptographic proof
        // สำหรับตัวอย่าง: ตรวจสอบ deviation จาก finalized price
        
        Task memory task = tasks[_taskIndex];
        for (uint256 i = 0; i < task.requiredFeeds.length; i++) {
            PriceData memory finalized = finalizedPrices[_taskIndex][task.requiredFeeds[i]];
            if (!finalized.finalized) continue;
            
            int256 deviation = _prices[i] - finalized.price;
            if (deviation < 0) deviation = -deviation;
            
            int256 threshold = (finalized.price * int256(PRICE_DEVIATION_THRESHOLD * 5)) / 10000; // 5%
            if (threshold < 0) threshold = -threshold;
            
            if (deviation > threshold) return true; // ผิดมาก!
        }
        
        return false;
    }
    
    // ============ View Functions ============
    
    function getLatestPrice(
        address _asset
    ) external view returns (int256 price, uint256 timestamp, bool finalized) {
        // ค้นหา finalized price ล่าสุด
        for (uint32 i = latestTaskNum; i > 0; i--) {
            PriceData memory data = finalizedPrices[i - 1][_asset];
            if (data.finalized) {
                return (data.price, data.timestamp, true);
            }
        }
        return (0, 0, false);
    }
    
    function getOperatorStats(address _operator) external view returns (
        bool isRegistered,
        bool isFrozen,
        uint256 totalResponses,
        uint256 correctResponses,
        uint256 accuracyRate
    ) {
        OperatorInfo memory info = operators[_operator];
        return (
            info.isRegistered,
            info.isFrozen,
            info.totalResponseCount,
            info.correctResponseCount,
            info.totalResponseCount > 0 
                ? (info.correctResponseCount * 10000) / info.totalResponseCount 
                : 0
        );
    }
    
    function getRegisteredOperators() external view returns (address[] memory) {
        return registeredOperators;
    }
    
    // ============ Admin ============
    
    function addTaskCreator(address _creator) external onlyOwner {
        taskCreators[_creator] = true;
    }
}
```

---

## 84.5 Symbiotic: Alternative Restaking

### สถาปัตยกรรม Symbiotic

```
Symbiotic vs EigenLayer:
                    
EigenLayer:                     Symbiotic:
- ETH-centric                  - Any collateral
- Operators model              - Vault-based
- EigenLayer middleware        - Permissionless vaults
- More complex                 - Simpler design
- $15B+ TVL (2024)             - $2B+ TVL (2024)
```

### 84.5.1 Symbiotic Vault Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title ISymbioticVault
 * @notice Interface สำหรับ Symbiotic Vault
 * @dev Vault เป็นหน่วยพื้นฐานของ Symbiotic
 *      - ใครก็ได้สร้าง vault
 *      - รองรับ token ใดก็ได้เป็น collateral
 *      - Networks เลือก vault ที่ต้องการ accept
 */
interface ISymbioticVault {
    
    // ============ Events ============
    
    event Deposit(address indexed depositor, address indexed onBehalfOf, uint256 amount, uint256 shares);
    event Withdraw(address indexed withdrawer, address indexed claimer, uint256 amount, uint256 burnedShares, uint256 mintedShares);
    event Claim(address indexed claimer, address indexed recipient, uint256 amount);
    event Slash(uint256 amount, uint48 captureTimestamp);
    
    // ============ Deposit/Withdraw ============
    
    /**
     * @notice ฝาก collateral
     * @return shares จำนวน vault shares ที่ได้
     */
    function deposit(
        address onBehalfOf,
        uint256 amount
    ) external returns (uint256 depositedAmount, uint256 mintedShares);
    
    /**
     * @notice ขอถอน (ต้องรอ epoch)
     */
    function withdraw(
        address claimer,
        uint256 amount
    ) external returns (uint256 burnedShares, uint256 mintedShares);
    
    /**
     * @notice รับ collateral หลัง epoch ผ่าน
     */
    function claim(address recipient, uint256 epoch) external returns (uint256 amount);
    
    // ============ Slashing ============
    
    /**
     * @notice Slash collateral (เรียกโดย Network ที่ได้รับอนุญาต)
     */
    function slash(
        bytes memory subnetwork,
        address operator,
        uint256 amount,
        uint48 captureTimestamp,
        bytes memory hints
    ) external returns (uint256 slashedAmount);
    
    // ============ View ============
    
    function totalStake() external view returns (uint256);
    function activeStake() external view returns (uint256);
    function withdrawals(uint256 epoch) external view returns (uint256);
    function collateral() external view returns (address);
    function epochDuration() external view returns (uint48);
    function currentEpoch() external view returns (uint256);
}
```

### 84.5.2 Symbiotic Network Integration

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title SymbioticNetworkMiddleware
 * @notice Middleware สำหรับ integrate กับ Symbiotic
 * @dev Network ใช้ middleware นี้เพื่อ:
 *      - Register operators
 *      - Request slash
 *      - Track stake
 */
contract SymbioticNetworkMiddleware {
    
    // ============ State ============
    
    address public immutable network;        // address ของ network นี้
    address public immutable operatorRegistry;
    address public immutable vaultRegistry;
    address public immutable slasher;
    
    struct OperatorVaultPair {
        address operator;
        address vault;
        bool isActive;
        uint256 registeredAt;
    }
    
    mapping(address => mapping(address => OperatorVaultPair)) public operatorVaults;
    mapping(address => bool) public registeredOperators;
    
    uint48 public immutable epochDuration;
    uint48 public immutable slashingWindow;
    
    // สำหรับ consensus
    mapping(uint32 => bytes32) public epochConsensusRoots;
    mapping(uint32 => mapping(address => bool)) public hasSubmitted;
    
    uint32 public currentEpochNumber;
    
    // ============ Events ============
    
    event OperatorRegistered(address indexed operator, address indexed vault);
    event SlashRequested(address indexed operator, address indexed vault, uint256 amount);
    event ConsensusReached(uint32 indexed epoch, bytes32 root);
    
    constructor(
        address _network,
        address _operatorRegistry,
        address _vaultRegistry,
        address _slasher,
        uint48 _epochDuration,
        uint48 _slashingWindow
    ) {
        network = _network;
        operatorRegistry = _operatorRegistry;
        vaultRegistry = _vaultRegistry;
        slasher = _slasher;
        epochDuration = _epochDuration;
        slashingWindow = _slashingWindow;
    }
    
    /**
     * @notice Operator ลงทะเบียนพร้อมกำหนด vault ที่ใช้ค้ำประกัน
     */
    function registerOperator(address _vault) external {
        require(!registeredOperators[msg.sender], "Already registered");
        
        // ตรวจสอบว่า vault valid ใน Symbiotic
        // IVaultRegistry(vaultRegistry).isEntity(_vault)
        
        // ตรวจสอบว่า operator ลงทะเบียนใน OperatorRegistry
        // IOperatorRegistry(operatorRegistry).isEntity(msg.sender)
        
        registeredOperators[msg.sender] = true;
        operatorVaults[msg.sender][_vault] = OperatorVaultPair({
            operator: msg.sender,
            vault: _vault,
            isActive: true,
            registeredAt: block.timestamp
        });
        
        emit OperatorRegistered(msg.sender, _vault);
    }
    
    /**
     * @notice Submit consensus root (สำหรับ oracle/validator service)
     */
    function submitConsensus(
        uint32 _epoch,
        bytes32 _root,
        bytes calldata _proof
    ) external {
        require(registeredOperators[msg.sender], "Not registered operator");
        require(!hasSubmitted[_epoch][msg.sender], "Already submitted");
        require(_epoch == currentEpochNumber, "Wrong epoch");
        
        hasSubmitted[_epoch][msg.sender] = true;
        
        // ในการใช้งานจริง: ตรวจสอบว่ามี quorum แล้วหรือยัง
        // ถ้า quorum → finalize epoch
        epochConsensusRoots[_epoch] = _root;
        
        emit ConsensusReached(_epoch, _root);
    }
    
    /**
     * @notice ขอ slash operator ที่ทำผิด
     */
    function requestSlash(
        address _operator,
        address _vault,
        uint256 _amount,
        uint48 _captureTimestamp
    ) external {
        require(msg.sender == network, "Not network");
        require(registeredOperators[_operator], "Not operator");
        
        // ส่ง slash request ไปยัง Symbiotic Slasher
        // ISlasher(slasher).requestSlash(_vault, _operator, _amount, _captureTimestamp);
        
        emit SlashRequested(_operator, _vault, _amount);
    }
}
```

---

## 84.6 Workshop: Deploy และทดสอบ OracleAVS

```javascript
// Workshop: ทดสอบ OracleAVS lifecycle

const { ethers } = require("hardhat");

async function testOracleAVS() {
    const [deployer, operator1, operator2, operator3, challenger] = 
        await ethers.getSigners();
    
    // Deploy
    const OracleAVS = await ethers.getContractFactory("OracleAVS");
    const avs = await OracleAVS.deploy(
        ethers.ZeroAddress, // mock eigenlayer
        ethers.ZeroAddress  // mock slasher
    );
    
    // ============ Step 1: Register Operators ============
    
    const blsKey1 = ethers.randomBytes(64);
    const blsKey2 = ethers.randomBytes(64);
    const blsKey3 = ethers.randomBytes(64);
    
    await avs.connect(operator1).registerOperator(blsKey1);
    await avs.connect(operator2).registerOperator(blsKey2);
    await avs.connect(operator3).registerOperator(blsKey3);
    
    console.log("Operators registered:", (await avs.getRegisteredOperators()).length);
    
    // ============ Step 2: Create Task ============
    
    const WETH = "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2";
    const WBTC = "0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599";
    
    await avs.createNewTask([WETH, WBTC]);
    const taskIndex = 0;
    
    console.log("Task created: index =", taskIndex);
    
    // ============ Step 3: Operators Respond ============
    
    // ETH = $3000, BTC = $60000 (scaled 1e8)
    const prices = [
        ethers.parseUnits("3000", 8),   // ETH
        ethers.parseUnits("60000", 8)   // BTC
    ];
    
    // สร้าง response hash และ sign
    async function signResponse(signer, taskIdx, priceValues) {
        const hash = ethers.keccak256(
            ethers.AbiCoder.defaultAbiCoder().encode(
                ["uint32", "int256[]", "address", "uint256"],
                [taskIdx, priceValues, signer.address, (await ethers.provider.getNetwork()).chainId]
            )
        );
        const ethHash = ethers.hashMessage(ethers.getBytes(hash));
        return signer.signMessage(ethers.getBytes(hash));
    }
    
    const sig1 = await signResponse(operator1, taskIndex, prices);
    const sig2 = await signResponse(operator2, taskIndex, prices);
    const sig3 = await signResponse(operator3, taskIndex, [
        ethers.parseUnits("3005", 8), // ต่างนิดหน่อย
        ethers.parseUnits("60100", 8)
    ]);
    
    await avs.connect(operator1).respondToTask(taskIndex, prices, sig1);
    await avs.connect(operator2).respondToTask(taskIndex, prices, sig2);
    
    // 2 operators ตอบแล้ว → ตรวจสอบ threshold (ต้องการ 2/3 * 3 = 2)
    await avs.connect(operator3).respondToTask(taskIndex, [
        ethers.parseUnits("3005", 8),
        ethers.parseUnits("60100", 8)
    ], sig3);
    
    // ============ Step 4: Check Finalized Prices ============
    
    const [ethPrice, ethTimestamp, ethFinalized] = await avs.getLatestPrice(WETH);
    const [btcPrice, btcTimestamp, btcFinalized] = await avs.getLatestPrice(WBTC);
    
    console.log("ETH Price:", ethers.formatUnits(ethPrice, 8));
    console.log("BTC Price:", ethers.formatUnits(btcPrice, 8));
    
    // ============ Step 5: Check Operator Stats ============
    
    const [isReg, isFrozen, total, correct, accuracy] = 
        await avs.getOperatorStats(operator1.address);
    
    console.log(`Operator1: accuracy = ${accuracy / 100}%`);
}
```

---

## สรุป Part 84

- **EigenLayer** ช่วยให้ staked ETH สามารถ secure additional services ได้ (restaking) โดยไม่ต้องล็อค ETH เพิ่ม
- **AVS (Actively Validated Services)** คือ service ที่ใช้ EigenLayer security เช่น oracle, bridge, sequencer
- **IStrategy** จัดการ deposit/withdraw ของ LST tokens พร้อมคำนวณ shares ตาม exchange rate
- **IServiceManager** เป็น interface หลักที่ AVS ต้อง implement สำหรับ operator registration และ slashing
- **OracleAVS** แสดงวิธีสร้าง oracle service ที่ใช้ EigenLayer security ด้วย price aggregation และ BLS signature
- **Symbiotic** เป็น alternative ที่ vault-centric, รองรับ collateral ใดก็ได้ และ permissionless
- การ slash ต้องออกแบบอย่างระมัดระวัง เพราะกระทบ restakers ที่ไม่ได้ทำผิด

## Next: Part 85 - Liquid Staking Protocols (Lido, Rocket Pool, LST Composability)
