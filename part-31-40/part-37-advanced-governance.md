# Part 37: Advanced Governance Patterns

## สารบัญ
1. Timelock Controller
2. Multi-sig Governance
3. Optimistic Governance
4. Conviction Voting
5. Workshop: Full DAO System

---

## 1. Timelock Controller

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * TimelockController:
 * บังคับให้รอ delay ก่อน execute proposal
 * ให้ community มีเวลา react หาก proposal ไม่ดี
 * 
 * Roles:
 * - PROPOSER: สามารถ schedule operations
 * - EXECUTOR: สามารถ execute ที่ ready แล้ว
 * - CANCELLER: สามารถ cancel ที่ pending
 * - ADMIN: manage roles
 */
contract TimelockController {
    
    bytes32 public constant PROPOSER_ROLE = keccak256("PROPOSER_ROLE");
    bytes32 public constant EXECUTOR_ROLE = keccak256("EXECUTOR_ROLE");
    bytes32 public constant CANCELLER_ROLE = keccak256("CANCELLER_ROLE");
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");
    
    uint256 public minDelay; // ขั้นต่ำ delay (เช่น 2 days)
    
    mapping(bytes32 => uint256) public timestamps; // operation → ready timestamp
    mapping(bytes32 => mapping(address => bool)) public roles;
    
    event OperationScheduled(
        bytes32 indexed id,
        uint256 indexed index,
        address target,
        uint256 value,
        bytes data,
        bytes32 predecessor,
        uint256 delay
    );
    event OperationExecuted(bytes32 indexed id, uint256 indexed index, address target, uint256 value, bytes data);
    event OperationCancelled(bytes32 indexed id);
    event MinDelayChanged(uint256 oldDelay, uint256 newDelay);
    
    error OperationNotReady(bytes32 id);
    error OperationAlreadyDone(bytes32 id);
    error PredecessorNotDone(bytes32 predecessor);
    
    uint256 constant _DONE_TIMESTAMP = 1;
    
    modifier onlyRole(bytes32 role) {
        require(roles[role][msg.sender], "Access denied");
        _;
    }
    
    constructor(
        uint256 minDelay_,
        address[] memory proposers,
        address[] memory executors,
        address admin
    ) {
        minDelay = minDelay_;
        
        roles[ADMIN_ROLE][admin] = true;
        
        for (uint256 i; i < proposers.length; i++) {
            roles[PROPOSER_ROLE][proposers[i]] = true;
            roles[CANCELLER_ROLE][proposers[i]] = true; // proposers can cancel
        }
        
        for (uint256 i; i < executors.length; i++) {
            // address(0) = anyone can execute
            if (executors[i] == address(0)) {
                roles[EXECUTOR_ROLE][address(0)] = true;
            } else {
                roles[EXECUTOR_ROLE][executors[i]] = true;
            }
        }
    }
    
    // Hash a batch of operations
    function hashOperationBatch(
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata payloads,
        bytes32 predecessor,
        bytes32 salt
    ) public pure returns (bytes32) {
        return keccak256(abi.encode(targets, values, payloads, predecessor, salt));
    }
    
    // Schedule a batch of operations
    function scheduleBatch(
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata payloads,
        bytes32 predecessor,
        bytes32 salt,
        uint256 delay
    ) external onlyRole(PROPOSER_ROLE) {
        require(delay >= minDelay, "Delay too short");
        
        bytes32 id = hashOperationBatch(targets, values, payloads, predecessor, salt);
        require(timestamps[id] == 0, "Operation already scheduled");
        
        timestamps[id] = block.timestamp + delay;
        
        for (uint256 i; i < targets.length; i++) {
            emit OperationScheduled(id, i, targets[i], values[i], payloads[i], predecessor, delay);
        }
    }
    
    // Execute a ready batch
    function executeBatch(
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata payloads,
        bytes32 predecessor,
        bytes32 salt
    ) external payable onlyRole(EXECUTOR_ROLE) {
        bytes32 id = hashOperationBatch(targets, values, payloads, predecessor, salt);
        
        // Check predecessor done
        if (predecessor != bytes32(0)) {
            require(timestamps[predecessor] == _DONE_TIMESTAMP, "Predecessor not done");
        }
        
        // Check delay passed
        uint256 readyTime = timestamps[id];
        require(readyTime != 0 && readyTime != _DONE_TIMESTAMP, "Operation not pending");
        if (block.timestamp < readyTime) revert OperationNotReady(id);
        
        timestamps[id] = _DONE_TIMESTAMP;
        
        for (uint256 i; i < targets.length; i++) {
            (bool success,) = targets[i].call{value: values[i]}(payloads[i]);
            require(success, "Execution failed");
            emit OperationExecuted(id, i, targets[i], values[i], payloads[i]);
        }
    }
    
    // Cancel a pending operation
    function cancel(bytes32 id) external onlyRole(CANCELLER_ROLE) {
        require(timestamps[id] > _DONE_TIMESTAMP, "Not cancellable");
        delete timestamps[id];
        emit OperationCancelled(id);
    }
    
    // Update min delay (via governance)
    function updateDelay(uint256 newDelay) external {
        require(msg.sender == address(this), "Self call only");
        emit MinDelayChanged(minDelay, newDelay);
        minDelay = newDelay;
    }
    
    function isOperationPending(bytes32 id) public view returns (bool) {
        return timestamps[id] > _DONE_TIMESTAMP;
    }
    
    function isOperationReady(bytes32 id) public view returns (bool) {
        return timestamps[id] > _DONE_TIMESTAMP && timestamps[id] <= block.timestamp;
    }
    
    function isOperationDone(bytes32 id) public view returns (bool) {
        return timestamps[id] == _DONE_TIMESTAMP;
    }
    
    receive() external payable {}
}
```

---

## 2. Optimistic Governance

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Optimistic Governance:
 * proposals pass โดย default ถ้าไม่มีใค oppose
 * ลด voting burden สำหรับ routine changes
 * 
 * Flow:
 * 1. Submit proposal
 * 2. Wait veto period (e.g., 3 days)
 * 3. If no veto threshold reached → auto-execute
 * 4. If veto reached → proposal defeated
 */
contract OptimisticGovernor {
    
    IERC20Votes public immutable token;
    
    uint256 public constant VETO_THRESHOLD = 10; // 10% of total supply to veto
    uint256 public constant VETO_PERIOD = 3 days;
    uint256 public constant EXECUTION_DELAY = 1 days; // after veto period
    
    struct Proposal {
        address proposer;
        bytes32 callHash;     // hash of calls to execute
        uint256 createdAt;
        uint256 vetoVotes;
        bool executed;
        bool vetoed;
        mapping(address => bool) hasVetoed;
    }
    
    mapping(uint256 => Proposal) public proposals;
    uint256 public proposalCount;
    
    event ProposalCreated(uint256 indexed id, address proposer, bytes32 callHash);
    event VetoVoted(uint256 indexed id, address voter, uint256 weight);
    event ProposalExecuted(uint256 indexed id);
    event ProposalVetoed(uint256 indexed id, uint256 totalVetoVotes);
    
    constructor(address _token) {
        token = IERC20Votes(_token);
    }
    
    function propose(
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata calldatas
    ) external returns (uint256 id) {
        bytes32 callHash = keccak256(abi.encode(targets, values, calldatas));
        
        id = proposalCount++;
        Proposal storage p = proposals[id];
        p.proposer = msg.sender;
        p.callHash = callHash;
        p.createdAt = block.timestamp;
        
        emit ProposalCreated(id, msg.sender, callHash);
    }
    
    function veto(uint256 id) external {
        Proposal storage p = proposals[id];
        
        require(block.timestamp <= p.createdAt + VETO_PERIOD, "Veto period ended");
        require(!p.hasVetoed[msg.sender], "Already vetoed");
        
        p.hasVetoed[msg.sender] = true;
        
        uint256 weight = token.getPastVotes(msg.sender, p.createdAt - 1);
        p.vetoVotes += weight;
        
        emit VetoVoted(id, msg.sender, weight);
        
        // Check if veto threshold reached
        uint256 totalSupply = token.getPastTotalSupply(p.createdAt - 1);
        if (p.vetoVotes * 100 >= totalSupply * VETO_THRESHOLD) {
            p.vetoed = true;
            emit ProposalVetoed(id, p.vetoVotes);
        }
    }
    
    function execute(
        uint256 id,
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata calldatas
    ) external {
        Proposal storage p = proposals[id];
        
        require(!p.vetoed, "Proposal vetoed");
        require(!p.executed, "Already executed");
        require(
            block.timestamp >= p.createdAt + VETO_PERIOD + EXECUTION_DELAY,
            "Not ready"
        );
        
        // Verify call hash
        require(keccak256(abi.encode(targets, values, calldatas)) == p.callHash, "Invalid calls");
        
        p.executed = true;
        
        for (uint256 i; i < targets.length; i++) {
            (bool success,) = targets[i].call{value: values[i]}(calldatas[i]);
            require(success, "Call failed");
        }
        
        emit ProposalExecuted(id);
    }
}

interface IERC20Votes {
    function getPastVotes(address account, uint256 timepoint) external view returns (uint256);
    function getPastTotalSupply(uint256 timepoint) external view returns (uint256);
}
```

---

## 3. Conviction Voting

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Conviction Voting:
 * Votes สะสมตามเวลา - ยิ่ง stake นาน ยิ่ง conviction สูง
 * ป้องกัน plutocracy (ซื้อ vote ในนาทีสุดท้าย)
 * ใช้ใน Gitcoin, Gardens
 * 
 * Formula: conviction(t) = α × conviction(t-1) + balance
 * α = decay rate (e.g., 0.9 = 10% decay per period)
 */
contract ConvictionVoting {
    
    IERC20 public immutable token;
    
    uint256 public constant DECAY = 9000;  // 90% retention per period
    uint256 public constant BASIS = 10000;
    uint256 public constant THRESHOLD = 10e18; // 10 tokens conviction to pass
    uint256 public constant MAX_FUNDING = 1000e18; // max per proposal
    
    struct Proposal {
        address requestor;
        uint256 requested;        // ETH requested
        uint256 totalConviction;
        bool executed;
        uint256 lastUpdate;
    }
    
    struct Stake {
        uint256 amount;
        uint256 conviction;
        uint256 lastUpdate;
    }
    
    mapping(uint256 => Proposal) public proposals;
    mapping(address => mapping(uint256 => Stake)) public stakes; // voter → proposalId → stake
    mapping(address => uint256) public totalStaked; // user's total staked
    
    uint256 public proposalCount;
    
    event ProposalCreated(uint256 indexed id, address requestor, uint256 requested);
    event StakeChanged(address indexed voter, uint256 indexed proposalId, uint256 amount);
    event ProposalExecuted(uint256 indexed id, uint256 conviction);
    
    constructor(address _token) {
        token = IERC20(_token);
    }
    
    function createProposal(uint256 requested) external returns (uint256 id) {
        require(requested <= MAX_FUNDING, "Exceeds max");
        
        id = proposalCount++;
        proposals[id] = Proposal({
            requestor: msg.sender,
            requested: requested,
            totalConviction: 0,
            executed: false,
            lastUpdate: block.timestamp
        });
        
        emit ProposalCreated(id, msg.sender, requested);
    }
    
    function stakeTokens(uint256 proposalId, uint256 amount) external {
        Proposal storage p = proposals[proposalId];
        require(!p.executed, "Executed");
        
        // Transfer tokens to this contract
        token.transferFrom(msg.sender, address(this), amount);
        
        // Update conviction first
        _updateConviction(proposalId, msg.sender);
        
        Stake storage s = stakes[msg.sender][proposalId];
        s.amount += amount;
        totalStaked[msg.sender] += amount;
        
        emit StakeChanged(msg.sender, proposalId, s.amount);
    }
    
    function unstakeTokens(uint256 proposalId, uint256 amount) external {
        _updateConviction(proposalId, msg.sender);
        
        Stake storage s = stakes[msg.sender][proposalId];
        require(s.amount >= amount, "Insufficient");
        
        s.amount -= amount;
        totalStaked[msg.sender] -= amount;
        
        token.transfer(msg.sender, amount);
        
        emit StakeChanged(msg.sender, proposalId, s.amount);
    }
    
    function _updateConviction(uint256 proposalId, address voter) internal {
        Stake storage s = stakes[voter][proposalId];
        Proposal storage p = proposals[proposalId];
        
        uint256 periods = (block.timestamp - s.lastUpdate) / 1 hours;
        
        if (periods > 0) {
            // Decay existing conviction and add new
            // conviction = decay^periods × old_conviction + amount × (1 - decay^periods) / (1-decay)
            uint256 newConviction = s.conviction;
            uint256 decayPow = _pow(DECAY, periods);
            
            newConviction = (newConviction * decayPow / BASIS) + 
                           (s.amount * (BASIS - decayPow) / (BASIS - DECAY));
            
            p.totalConviction = p.totalConviction - s.conviction + newConviction;
            s.conviction = newConviction;
            s.lastUpdate = block.timestamp;
        }
        
        p.lastUpdate = block.timestamp;
    }
    
    function executeProposal(uint256 proposalId) external {
        Proposal storage p = proposals[proposalId];
        require(!p.executed, "Already executed");
        
        // Update all stakers' conviction (simplified: use totalConviction)
        require(p.totalConviction >= THRESHOLD, "Insufficient conviction");
        
        p.executed = true;
        
        // Transfer funds to requestor
        payable(p.requestor).transfer(p.requested);
        
        emit ProposalExecuted(proposalId, p.totalConviction);
    }
    
    function _pow(uint256 base, uint256 exp) internal pure returns (uint256 result) {
        result = BASIS;
        for (uint256 i; i < exp; i++) {
            result = result * base / BASIS;
        }
    }
    
    receive() external payable {}
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
    function balanceOf(address) external view returns (uint256);
}
```

---

## 4. Workshop: Full DAO System

```
DAO Architecture ที่ครบครัน:

1. GovernanceToken (ERC20Votes)
   - checkpoint-based vote weights
   - delegation

2. Governor (OpenZeppelin Governor)
   - propose, vote, queue, execute
   - quorum: 4% of total supply
   - voting delay: 1 day
   - voting period: 5 days

3. TimelockController
   - min delay: 2 days
   - proposers: Governor
   - executors: anyone
   
4. Treasury
   - ETH + ERC-20 assets
   - controlled by Timelock

5. Veto Module (Optimistic)
   - Fast-track small proposals
   - 3-day veto window

Governance Parameters:
- Proposal threshold: 1000 tokens
- Quorum: 4%  
- Voting period: 5 days
- Timelock delay: 2 days
- Total cycle: ~8 days for regular, ~4 days for optimistic
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * DAO Treasury:
 * เก็บ funds และ execute governance decisions
 */
contract DAOTreasury {
    
    address public immutable timelock;
    
    event FundsReceived(address indexed from, uint256 amount);
    event ETHSent(address indexed to, uint256 amount);
    event TokenSent(address indexed token, address indexed to, uint256 amount);
    event ParameterChanged(string indexed param, uint256 oldValue, uint256 newValue);
    
    modifier onlyTimelock() {
        require(msg.sender == timelock, "Only timelock");
        _;
    }
    
    constructor(address _timelock) {
        timelock = _timelock;
    }
    
    receive() external payable {
        emit FundsReceived(msg.sender, msg.value);
    }
    
    // Transfer ETH (called via governance)
    function sendETH(address payable to, uint256 amount) external onlyTimelock {
        require(address(this).balance >= amount, "Insufficient ETH");
        (bool success,) = to.call{value: amount}("");
        require(success, "Transfer failed");
        emit ETHSent(to, amount);
    }
    
    // Transfer ERC-20 (called via governance)
    function sendToken(address token, address to, uint256 amount) external onlyTimelock {
        (bool success, bytes memory data) = token.call(
            abi.encodeWithSignature("transfer(address,uint256)", to, amount)
        );
        require(success && (data.length == 0 || abi.decode(data, (bool))), "Token transfer failed");
        emit TokenSent(token, to, amount);
    }
    
    // Change protocol parameter via governance
    function changeParameter(
        address target,
        string calldata paramName,
        uint256 oldValue,
        uint256 newValue,
        bytes calldata callData
    ) external onlyTimelock {
        (bool success,) = target.call(callData);
        require(success, "Parameter change failed");
        emit ParameterChanged(paramName, oldValue, newValue);
    }
    
    function getBalance() external view returns (uint256) {
        return address(this).balance;
    }
}
```

---

## สรุป Part 37

Advanced Governance ที่เรียนรู้:
- ✅ TimelockController (delay + roles)
- ✅ Optimistic Governance (veto-based)
- ✅ Conviction Voting (time-weighted)
- ✅ Full DAO system architecture
- ✅ Treasury contract

## Quiz

1. TimelockController ป้องกันอะไร?
2. Optimistic governance เหมาะกับ proposals แบบไหน?
3. Conviction voting ต่างจาก token voting อย่างไร?
4. ทำไม DAO treasury ต้องผ่าน timelock?

---

## Next: Part 38 - Proxy Patterns Advanced
