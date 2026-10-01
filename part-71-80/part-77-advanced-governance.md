# Part 77: Advanced Governance Systems

## บทนำ

Governance คือ "รัฐธรรมนูญ" ของ DeFi protocol — กำหนดว่าใครมีสิทธิ์เปลี่ยนแปลงอะไรได้บ้าง และผ่านกระบวนการอย่างไร จากยุค Compound Governor Alpha ง่ายๆ สู่ระบบ governance ซับซ้อนที่รองรับ partial delegation, quadratic voting และ rage-quit mechanism ในปัจจุบัน

บทนี้ครอบคลุม:
1. วิวัฒนาการจาก Compound Governor Alpha → Bravo → OpenZeppelin Governor
2. Flexible voting ด้วย delegation และ vote-by-signature
3. Optimistic governance สำหรับ low-contention changes
4. Rage-quit mechanism เพื่อปกป้อง minority
5. Quadratic voting implementation

---

## 77.1 วิวัฒนาการของ On-Chain Governance

### Governor Alpha (Compound, 2020)

```
Simple majority vote
Fixed 3-day voting period
2-day timelock
No delegation
```

### Governor Bravo (Compound, 2021)
```
Proposal thresholds
Abstain votes
Upgradeable
Improved delegation
```

### OpenZeppelin Governor (2021+)
```
Modular architecture
Extensions: ERC20Votes, Timelock, Settings
Vote weight snapshots
EIP-712 signing
```

### Modern Governance (2022+)
```
Partial delegation
Fractional voting
Optimistic proposals
Cross-chain governance
```

---

## 77.2 OpenZeppelin Governor แบบ Complete

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/governance/Governor.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorSettings.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorCountingSimple.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorVotes.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorVotesQuorumFraction.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorTimelockControl.sol";
import "@openzeppelin/contracts/governance/TimelockController.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Votes.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/utils/cryptography/EIP712.sol";

/// @title GovernanceToken
/// @notice ERC20 token ที่รองรับ delegation และ vote snapshots
contract GovernanceToken is ERC20, ERC20Votes {
    
    uint256 public constant MAX_SUPPLY = 100_000_000e18; // 100M tokens
    
    constructor(string memory name, string memory symbol) 
        ERC20(name, symbol) 
        EIP712(name, "1") 
    {}
    
    function mint(address to, uint256 amount) external {
        require(totalSupply() + amount <= MAX_SUPPLY, "Exceeds max supply");
        _mint(to, amount);
    }
    
    // Override ที่จำเป็น
    function _update(address from, address to, uint256 amount)
        internal
        override(ERC20, ERC20Votes)
    {
        super._update(from, to, amount);
    }
    
    function nonces(address owner)
        public
        view
        override(ERC20Permit, Nonces)
        returns (uint256)
    {
        return super.nonces(owner);
    }
}

/// @title AdvancedGovernor
/// @notice Governor ที่มีฟีเจอร์ครบครัน
contract AdvancedGovernor is
    Governor,
    GovernorSettings,
    GovernorCountingSimple,
    GovernorVotes,
    GovernorVotesQuorumFraction,
    GovernorTimelockControl
{
    constructor(
        IVotes _token,
        TimelockController _timelock,
        uint48 _votingDelay,        // blocks
        uint32 _votingPeriod,       // blocks
        uint256 _proposalThreshold, // min tokens to propose
        uint256 _quorumFraction     // % of total supply needed (e.g. 4 = 4%)
    )
        Governor("AdvancedGovernor")
        GovernorSettings(_votingDelay, _votingPeriod, _proposalThreshold)
        GovernorVotes(_token)
        GovernorVotesQuorumFraction(_quorumFraction)
        GovernorTimelockControl(_timelock)
    {}
    
    // Override ที่จำเป็นเมื่อใช้หลาย extensions
    function votingDelay()
        public view override(Governor, GovernorSettings)
        returns (uint256)
    {
        return super.votingDelay();
    }
    
    function votingPeriod()
        public view override(Governor, GovernorSettings)
        returns (uint256)
    {
        return super.votingPeriod();
    }
    
    function quorum(uint256 blockNumber)
        public view override(Governor, GovernorVotesQuorumFraction)
        returns (uint256)
    {
        return super.quorum(blockNumber);
    }
    
    function proposalThreshold()
        public view override(Governor, GovernorSettings)
        returns (uint256)
    {
        return super.proposalThreshold();
    }
    
    function state(uint256 proposalId)
        public view override(Governor, GovernorTimelockControl)
        returns (ProposalState)
    {
        return super.state(proposalId);
    }
    
    function proposalNeedsQueuing(uint256 proposalId)
        public view override(Governor, GovernorTimelockControl)
        returns (bool)
    {
        return super.proposalNeedsQueuing(proposalId);
    }
    
    function _queueOperations(
        uint256 proposalId,
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        bytes32 descriptionHash
    ) internal override(Governor, GovernorTimelockControl) returns (uint48) {
        return super._queueOperations(proposalId, targets, values, calldatas, descriptionHash);
    }
    
    function _executeOperations(
        uint256 proposalId,
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        bytes32 descriptionHash
    ) internal override(Governor, GovernorTimelockControl) {
        super._executeOperations(proposalId, targets, values, calldatas, descriptionHash);
    }
    
    function _cancel(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        bytes32 descriptionHash
    ) internal override(Governor, GovernorTimelockControl) returns (uint256) {
        return super._cancel(targets, values, calldatas, descriptionHash);
    }
    
    function _executor()
        internal view override(Governor, GovernorTimelockControl)
        returns (address)
    {
        return super._executor();
    }
}
```

---

## 77.3 Flexible Voting: Partial Delegation

### ปัญหาของ Full Delegation

ปกติ ERC20Votes ให้ delegate votes ทั้งหมดให้คนเดียว ไม่ยืดหยุ่น:
- ถ้า delegate ทำ bad proposal → ต้อง un-delegate ก่อน vote
- ไม่สามารถกระจาย voting power ให้หลายคน

### Partial Delegation Solution

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title PartialDelegationToken
/// @notice ERC20 token รองรับ partial delegation ให้หลายคน
contract PartialDelegationToken {
    
    string public name;
    string public symbol;
    uint8 public constant decimals = 18;
    
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    uint256 public totalSupply;
    
    // Delegation: holder → [(delegate, percentage)]
    // percentage ใช้ basis points (10000 = 100%)
    struct DelegationEntry {
        address delegate;
        uint16 bps; // basis points: 100 = 1%, 10000 = 100%
    }
    
    mapping(address => DelegationEntry[]) public delegations;
    mapping(address => uint256) public delegatedVotes; // votes ที่ได้รับ delegation
    
    // Checkpoint สำหรับ vote snapshots
    struct Checkpoint {
        uint32 fromBlock;
        uint224 votes;
    }
    
    mapping(address => Checkpoint[]) public checkpoints;
    
    uint256 public constant MAX_DELEGATES = 10;
    uint256 public constant TOTAL_BPS = 10000;
    
    event DelegationSet(address indexed from, address[] delegates, uint16[] bps);
    event Transfer(address indexed from, address indexed to, uint256 value);
    event DelegateVotesChanged(address indexed delegate, uint256 prevVotes, uint256 newVotes);
    
    constructor(string memory _name, string memory _symbol) {
        name = _name;
        symbol = _symbol;
    }
    
    /// @notice Set partial delegation ให้หลายคน
    /// @param delegates Array ของ delegate addresses
    /// @param bpsArray Array ของ basis points (ต้องรวมกัน ≤ 10000)
    function setDelegations(
        address[] calldata delegates,
        uint16[] calldata bpsArray
    ) external {
        require(delegates.length == bpsArray.length, "Length mismatch");
        require(delegates.length <= MAX_DELEGATES, "Too many delegates");
        
        // ตรวจสอบว่า total bps ≤ 10000
        uint256 totalBps;
        for (uint256 i = 0; i < bpsArray.length; i++) {
            require(delegates[i] != address(0), "Invalid delegate");
            totalBps += bpsArray[i];
        }
        require(totalBps <= TOTAL_BPS, "Total BPS exceeds 100%");
        
        // ลบ delegation เก่า
        _removeDelegations(msg.sender);
        
        // ตั้ง delegation ใหม่
        uint256 senderBalance = balanceOf[msg.sender];
        
        for (uint256 i = 0; i < delegates.length; i++) {
            if (bpsArray[i] == 0) continue;
            
            delegations[msg.sender].push(DelegationEntry({
                delegate: delegates[i],
                bps: bpsArray[i]
            }));
            
            // เพิ่ม votes ให้ delegate
            uint256 votes = senderBalance * bpsArray[i] / TOTAL_BPS;
            _addDelegateVotes(delegates[i], votes);
        }
        
        emit DelegationSet(msg.sender, delegates, bpsArray);
    }
    
    function _removeDelegations(address from) internal {
        DelegationEntry[] storage entries = delegations[from];
        uint256 senderBalance = balanceOf[from];
        
        for (uint256 i = 0; i < entries.length; i++) {
            uint256 votes = senderBalance * entries[i].bps / TOTAL_BPS;
            _removeDelegateVotes(entries[i].delegate, votes);
        }
        
        delete delegations[from];
    }
    
    function _addDelegateVotes(address delegate, uint256 amount) internal {
        uint256 current = delegatedVotes[delegate];
        uint256 newAmount = current + amount;
        delegatedVotes[delegate] = newAmount;
        _writeCheckpoint(delegate, current, newAmount);
        emit DelegateVotesChanged(delegate, current, newAmount);
    }
    
    function _removeDelegateVotes(address delegate, uint256 amount) internal {
        uint256 current = delegatedVotes[delegate];
        uint256 newAmount = current >= amount ? current - amount : 0;
        delegatedVotes[delegate] = newAmount;
        _writeCheckpoint(delegate, current, newAmount);
        emit DelegateVotesChanged(delegate, current, newAmount);
    }
    
    function _writeCheckpoint(address account, uint256 oldVotes, uint256 newVotes) internal {
        Checkpoint[] storage ckpts = checkpoints[account];
        uint256 pos = ckpts.length;
        
        if (pos > 0 && ckpts[pos - 1].fromBlock == block.number) {
            ckpts[pos - 1].votes = uint224(newVotes);
        } else {
            ckpts.push(Checkpoint({
                fromBlock: uint32(block.number),
                votes: uint224(newVotes)
            }));
        }
        
        _ = oldVotes;
    }
    
    /// @notice ดึง votes ณ block ที่กำหนด
    function getPastVotes(address account, uint256 blockNumber) external view returns (uint256) {
        Checkpoint[] storage ckpts = checkpoints[account];
        
        if (ckpts.length == 0) return 0;
        if (ckpts[0].fromBlock > blockNumber) return 0;
        
        uint256 lower = 0;
        uint256 upper = ckpts.length - 1;
        
        while (lower < upper) {
            uint256 center = upper - (upper - lower) / 2;
            if (ckpts[center].fromBlock <= blockNumber) {
                lower = center;
            } else {
                upper = center - 1;
            }
        }
        
        return ckpts[lower].votes;
    }
    
    function getVotes(address account) external view returns (uint256) {
        return delegatedVotes[account];
    }
    
    function getDelegations(address from) external view returns (DelegationEntry[] memory) {
        return delegations[from];
    }
    
    // ERC20 transfers ต้องอัพเดต delegations
    function transfer(address to, uint256 amount) external returns (bool) {
        _transfer(msg.sender, to, amount);
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        allowance[from][msg.sender] -= amount;
        _transfer(from, to, amount);
        return true;
    }
    
    function _transfer(address from, address to, uint256 amount) internal {
        require(balanceOf[from] >= amount, "Insufficient balance");
        
        // อัพเดต delegations สำหรับ sender
        _updateDelegatesOnTransfer(from, to, amount);
        
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        
        emit Transfer(from, to, amount);
    }
    
    function _updateDelegatesOnTransfer(address from, address to, uint256 amount) internal {
        // ลด votes ของ from's delegates
        DelegationEntry[] storage fromDelegates = delegations[from];
        for (uint256 i = 0; i < fromDelegates.length; i++) {
            uint256 votes = amount * fromDelegates[i].bps / TOTAL_BPS;
            _removeDelegateVotes(fromDelegates[i].delegate, votes);
        }
        
        // เพิ่ม votes ให้ to's delegates
        DelegationEntry[] storage toDelegates = delegations[to];
        for (uint256 i = 0; i < toDelegates.length; i++) {
            uint256 votes = amount * toDelegates[i].bps / TOTAL_BPS;
            _addDelegateVotes(toDelegates[i].delegate, votes);
        }
    }
    
    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount;
        totalSupply += amount;
        
        // เพิ่ม votes ให้ to's delegates
        DelegationEntry[] storage toDelegates = delegations[to];
        for (uint256 i = 0; i < toDelegates.length; i++) {
            uint256 votes = amount * toDelegates[i].bps / TOTAL_BPS;
            _addDelegateVotes(toDelegates[i].delegate, votes);
        }
        
        emit Transfer(address(0), to, amount);
    }
    
    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }
}
```

---

## 77.4 Vote-by-Signature (EIP-712)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title VoteBySignature
/// @notice ให้ผู้ถือ token vote ด้วย off-chain signature (gasless voting)
contract VoteBySignature {
    
    bytes32 public constant BALLOT_TYPEHASH = keccak256(
        "Ballot(uint256 proposalId,uint8 support,address voter,uint256 nonce,uint256 expiry)"
    );
    
    bytes32 private immutable DOMAIN_SEPARATOR;
    
    mapping(address => uint256) public nonces;
    
    // proposalId → voter → hasVoted
    mapping(uint256 => mapping(address => bool)) public hasVoted;
    
    // proposalId → [forVotes, againstVotes, abstainVotes]
    mapping(uint256 => uint256[3]) public proposalVotes;
    
    event VoteCast(address indexed voter, uint256 indexed proposalId, uint8 support, uint256 weight);
    
    constructor() {
        DOMAIN_SEPARATOR = keccak256(abi.encode(
            keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
            keccak256("VoteBySignature"),
            keccak256("1"),
            block.chainid,
            address(this)
        ));
    }
    
    /// @notice Vote ด้วย signature (เหมาะสำหรับ meta-transaction / gasless)
    /// @param proposalId Proposal ที่ต้องการ vote
    /// @param support 0=Against, 1=For, 2=Abstain
    /// @param voter Address ของ voter จริง
    /// @param nonce Nonce ป้องกัน replay
    /// @param expiry Deadline ของ signature
    /// @param v,r,s Signature components
    function castVoteBySig(
        uint256 proposalId,
        uint8 support,
        address voter,
        uint256 nonce,
        uint256 expiry,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external {
        require(block.timestamp <= expiry, "Signature expired");
        require(support <= 2, "Invalid support value");
        require(!hasVoted[proposalId][voter], "Already voted");
        require(nonce == nonces[voter], "Invalid nonce");
        
        // สร้าง EIP-712 hash
        bytes32 structHash = keccak256(abi.encode(
            BALLOT_TYPEHASH,
            proposalId,
            support,
            voter,
            nonce,
            expiry
        ));
        
        bytes32 hash = keccak256(abi.encodePacked(
            "\x19\x01",
            DOMAIN_SEPARATOR,
            structHash
        ));
        
        // ตรวจสอบ signature
        address signer = ecrecover(hash, v, r, s);
        require(signer != address(0) && signer == voter, "Invalid signature");
        
        // บันทึก vote
        nonces[voter]++;
        hasVoted[proposalId][voter] = true;
        
        uint256 weight = _getVotingWeight(voter);
        proposalVotes[proposalId][support] += weight;
        
        emit VoteCast(voter, proposalId, support, weight);
    }
    
    /// @notice Batch vote signatures — relayer จ่าย gas แทน
    function castVotesBySigBatch(
        uint256[] calldata proposalIds,
        uint8[] calldata supports,
        address[] calldata voters,
        uint256[] calldata noncesArr,
        uint256[] calldata expiries,
        uint8[] calldata vs,
        bytes32[] calldata rs,
        bytes32[] calldata ss
    ) external {
        uint256 len = proposalIds.length;
        require(
            len == supports.length && len == voters.length && 
            len == noncesArr.length && len == expiries.length,
            "Array length mismatch"
        );
        
        for (uint256 i = 0; i < len; i++) {
            // ใช้ try-catch เพื่อไม่ให้ invalid vote หยุด batch
            try this.castVoteBySig(
                proposalIds[i],
                supports[i],
                voters[i],
                noncesArr[i],
                expiries[i],
                vs[i],
                rs[i],
                ss[i]
            ) {} catch {
                // Skip invalid votes
            }
        }
    }
    
    function _getVotingWeight(address voter) internal view returns (uint256) {
        // ใน production จะ query ERC20Votes.getPastVotes()
        // สำหรับ demo คืน balance
        return voter.balance / 1e18; // simplified
    }
    
    function getProposalVotes(uint256 proposalId) external view returns (
        uint256 againstVotes,
        uint256 forVotes,
        uint256 abstainVotes
    ) {
        againstVotes = proposalVotes[proposalId][0];
        forVotes = proposalVotes[proposalId][1];
        abstainVotes = proposalVotes[proposalId][2];
    }
    
    function domainSeparator() external view returns (bytes32) {
        return DOMAIN_SEPARATOR;
    }
}
```

---

## 77.5 Optimistic Governance

### แนวคิด: Default Pass Unless Vetoed

ใน Optimistic governance proposal ผ่านโดยอัตโนมัติหากไม่มีใคร veto ภายใน window  
เหมาะสำหรับ routine/low-risk changes ที่ไม่ต้องการ quorum เต็มทุกครั้ง

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/AccessControl.sol";

/// @title OptimisticGovernance
/// @notice Proposals ผ่านโดย default ถ้าไม่ถูก veto ในเวลาที่กำหนด
contract OptimisticGovernance is AccessControl {
    
    bytes32 public constant GUARDIAN_ROLE = keccak256("GUARDIAN_ROLE");
    bytes32 public constant PROPOSER_ROLE = keccak256("PROPOSER_ROLE");
    
    enum ProposalState {
        PENDING,    // รอ veto window
        QUEUED,     // ผ่าน veto window แล้ว รอ execution
        EXECUTED,   // executed แล้ว
        VETOED,     // ถูก veto
        CANCELLED   // ถูก cancel โดย proposer
    }
    
    struct OptimisticProposal {
        address proposer;
        address[] targets;
        uint256[] values;
        bytes[] calldatas;
        string description;
        uint256 proposedAt;
        uint256 vetoDeadline;    // เส้นตาย veto
        uint256 executionDeadline; // เส้นตาย execute
        ProposalState state;
        uint256 vetoVotes;
        uint256 totalSupplySnapshot;
        uint256 vetoThresholdBps; // basis points ของ total supply ที่ต้องการ veto (e.g. 1000 = 10%)
    }
    
    mapping(uint256 => OptimisticProposal) public proposals;
    mapping(uint256 => mapping(address => bool)) public hasVetoed;
    
    uint256 public proposalCount;
    uint256 public constant VETO_WINDOW = 3 days;
    uint256 public constant EXECUTION_WINDOW = 7 days;
    uint256 public constant DEFAULT_VETO_THRESHOLD_BPS = 1000; // 10%
    
    address public governanceToken;
    
    event ProposalCreated(uint256 indexed id, address proposer, string description);
    event ProposalVetoed(uint256 indexed id, address vetoer, uint256 totalVetoVotes);
    event ProposalExecuted(uint256 indexed id);
    event ProposalCancelled(uint256 indexed id);
    
    constructor(address _token, address[] memory _guardians) {
        governanceToken = _token;
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        
        for (uint256 i = 0; i < _guardians.length; i++) {
            _grantRole(GUARDIAN_ROLE, _guardians[i]);
        }
    }
    
    /// @notice สร้าง optimistic proposal — ผ่านได้เลยถ้าไม่ถูก veto
    function propose(
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata calldatas,
        string calldata description
    ) external onlyRole(PROPOSER_ROLE) returns (uint256 proposalId) {
        require(targets.length > 0, "Empty proposal");
        require(targets.length == values.length && targets.length == calldatas.length, "Length mismatch");
        
        proposalId = ++proposalCount;
        
        uint256 totalSupply = _getTotalSupply();
        
        proposals[proposalId] = OptimisticProposal({
            proposer: msg.sender,
            targets: targets,
            values: values,
            calldatas: calldatas,
            description: description,
            proposedAt: block.timestamp,
            vetoDeadline: block.timestamp + VETO_WINDOW,
            executionDeadline: block.timestamp + VETO_WINDOW + EXECUTION_WINDOW,
            state: ProposalState.PENDING,
            vetoVotes: 0,
            totalSupplySnapshot: totalSupply,
            vetoThresholdBps: DEFAULT_VETO_THRESHOLD_BPS
        });
        
        emit ProposalCreated(proposalId, msg.sender, description);
    }
    
    /// @notice Vote to veto proposal
    function veto(uint256 proposalId) external {
        OptimisticProposal storage prop = proposals[proposalId];
        require(prop.state == ProposalState.PENDING, "Not in veto window");
        require(block.timestamp <= prop.vetoDeadline, "Veto window closed");
        require(!hasVetoed[proposalId][msg.sender], "Already vetoed");
        
        hasVetoed[proposalId][msg.sender] = true;
        
        uint256 voterPower = _getVotingPower(msg.sender);
        prop.vetoVotes += voterPower;
        
        // ตรวจสอบว่าถึง threshold แล้วหรือยัง
        uint256 vetoThreshold = prop.totalSupplySnapshot * prop.vetoThresholdBps / 10000;
        
        if (prop.vetoVotes >= vetoThreshold) {
            prop.state = ProposalState.VETOED;
        }
        
        emit ProposalVetoed(proposalId, msg.sender, prop.vetoVotes);
    }
    
    /// @notice Guardian สามารถ veto ทันทีได้
    function guardianVeto(uint256 proposalId) external onlyRole(GUARDIAN_ROLE) {
        OptimisticProposal storage prop = proposals[proposalId];
        require(
            prop.state == ProposalState.PENDING || prop.state == ProposalState.QUEUED,
            "Cannot veto"
        );
        
        prop.state = ProposalState.VETOED;
        emit ProposalVetoed(proposalId, msg.sender, type(uint256).max);
    }
    
    /// @notice Execute proposal หลัง veto window ปิดแล้ว
    function execute(uint256 proposalId) external payable {
        OptimisticProposal storage prop = proposals[proposalId];
        require(
            prop.state == ProposalState.PENDING || prop.state == ProposalState.QUEUED,
            "Cannot execute"
        );
        require(block.timestamp > prop.vetoDeadline, "Veto window still open");
        require(block.timestamp <= prop.executionDeadline, "Execution expired");
        
        prop.state = ProposalState.EXECUTED;
        
        for (uint256 i = 0; i < prop.targets.length; i++) {
            (bool success, bytes memory returnData) = prop.targets[i].call{value: prop.values[i]}(
                prop.calldatas[i]
            );
            
            if (!success) {
                assembly {
                    revert(add(returnData, 32), mload(returnData))
                }
            }
        }
        
        emit ProposalExecuted(proposalId);
    }
    
    function getProposalState(uint256 proposalId) external view returns (ProposalState) {
        OptimisticProposal storage prop = proposals[proposalId];
        
        if (prop.state == ProposalState.PENDING && block.timestamp > prop.vetoDeadline) {
            return ProposalState.QUEUED;
        }
        
        return prop.state;
    }
    
    function _getTotalSupply() internal view returns (uint256) {
        (bool success, bytes memory data) = governanceToken.staticcall(
            abi.encodeWithSignature("totalSupply()")
        );
        require(success, "Failed to get total supply");
        return abi.decode(data, (uint256));
    }
    
    function _getVotingPower(address voter) internal view returns (uint256) {
        (bool success, bytes memory data) = governanceToken.staticcall(
            abi.encodeWithSignature("getVotes(address)", voter)
        );
        if (!success) return 0;
        return abi.decode(data, (uint256));
    }
    
    receive() external payable {}
}
```

---

## 77.6 Rage-Quit Mechanism

### แนวคิด: Exit ก่อน Bad Proposal Execute

Rage-quit ให้สมาชิกที่ไม่เห็นด้วยกับ proposal ถอนส่วนแบ่งออกไปก่อนที่ proposal จะ execute  
ใช้ใน MolochDAO, DAOhaus และ Metacartel Ventures

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title RageQuitDAO
/// @notice DAO ที่ให้สมาชิกถอนออกก่อน proposal execute
contract RageQuitDAO is ReentrancyGuard {
    
    IERC20 public governanceToken;
    address[] public approvedTokens;  // tokens ใน treasury
    
    struct Proposal {
        uint256 id;
        address proposer;
        string description;
        address[] targets;
        uint256[] values;
        bytes[] calldatas;
        uint256 yesVotes;
        uint256 noVotes;
        uint256 votingDeadline;
        uint256 gracePeriodEnd;  // สิ้นสุด rage-quit window
        bool executed;
        bool cancelled;
    }
    
    mapping(uint256 => Proposal) public proposals;
    mapping(uint256 => mapping(address => bool)) public hasVoted;
    mapping(uint256 => mapping(address => bool)) public hasRageQuit;
    
    uint256 public proposalCount;
    uint256 public constant VOTING_PERIOD = 5 days;
    uint256 public constant GRACE_PERIOD = 3 days; // rage-quit window หลัง vote ผ่าน
    uint256 public constant QUORUM_BPS = 2000; // 20%
    
    event ProposalCreated(uint256 indexed id, address proposer);
    event VoteCast(uint256 indexed id, address voter, bool support, uint256 weight);
    event RageQuit(uint256 indexed id, address member, uint256 shares, uint256[] amounts);
    event ProposalExecuted(uint256 indexed id);
    
    constructor(address _token, address[] memory _approvedTokens) {
        governanceToken = IERC20(_token);
        approvedTokens = _approvedTokens;
    }
    
    function propose(
        string calldata description,
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata calldatas
    ) external returns (uint256) {
        require(governanceToken.balanceOf(msg.sender) >= 1e18, "Insufficient tokens");
        
        uint256 proposalId = ++proposalCount;
        
        proposals[proposalId] = Proposal({
            id: proposalId,
            proposer: msg.sender,
            description: description,
            targets: targets,
            values: values,
            calldatas: calldatas,
            yesVotes: 0,
            noVotes: 0,
            votingDeadline: block.timestamp + VOTING_PERIOD,
            gracePeriodEnd: 0, // set เมื่อ proposal ผ่าน
            executed: false,
            cancelled: false
        });
        
        emit ProposalCreated(proposalId, msg.sender);
        return proposalId;
    }
    
    function vote(uint256 proposalId, bool support) external {
        Proposal storage prop = proposals[proposalId];
        require(block.timestamp <= prop.votingDeadline, "Voting ended");
        require(!prop.cancelled, "Proposal cancelled");
        require(!hasVoted[proposalId][msg.sender], "Already voted");
        
        hasVoted[proposalId][msg.sender] = true;
        
        uint256 weight = governanceToken.balanceOf(msg.sender);
        
        if (support) {
            prop.yesVotes += weight;
        } else {
            prop.noVotes += weight;
        }
        
        // ถ้า proposal ผ่าน quorum และ majority → set grace period
        if (_isPassed(prop) && prop.gracePeriodEnd == 0) {
            prop.gracePeriodEnd = block.timestamp + GRACE_PERIOD;
        }
        
        emit VoteCast(proposalId, msg.sender, support, weight);
    }
    
    /// @notice Rage-quit: ถอนออกก่อน proposal execute
    /// @dev เรียกได้ระหว่าง grace period ของ proposal ที่ผ่านแต่ยังไม่ execute
    function rageQuit(uint256 proposalId, uint256 sharesToBurn) external nonReentrant {
        Proposal storage prop = proposals[proposalId];
        require(_isPassed(prop), "Proposal not passed");
        require(prop.gracePeriodEnd > 0, "Grace period not started");
        require(block.timestamp <= prop.gracePeriodEnd, "Grace period ended");
        require(!prop.executed, "Already executed");
        require(!hasRageQuit[proposalId][msg.sender], "Already rage quit this proposal");
        
        require(sharesToBurn > 0, "Must burn some shares");
        require(
            governanceToken.balanceOf(msg.sender) >= sharesToBurn,
            "Insufficient shares"
        );
        
        hasRageQuit[proposalId][msg.sender] = true;
        
        uint256 totalShares = governanceToken.totalSupply();
        
        // คำนวณสัดส่วนและถอน tokens ออก
        uint256[] memory amounts = new uint256[](approvedTokens.length);
        
        for (uint256 i = 0; i < approvedTokens.length; i++) {
            uint256 treasuryBalance = IERC20(approvedTokens[i]).balanceOf(address(this));
            amounts[i] = treasuryBalance * sharesToBurn / totalShares;
            
            if (amounts[i] > 0) {
                IERC20(approvedTokens[i]).transfer(msg.sender, amounts[i]);
            }
        }
        
        // Burn governance tokens
        // (ในจริง ต้องมี burnFrom function หรือ approve ก่อน)
        governanceToken.transferFrom(msg.sender, address(0xdEaD), sharesToBurn);
        
        emit RageQuit(proposalId, msg.sender, sharesToBurn, amounts);
    }
    
    function execute(uint256 proposalId) external payable nonReentrant {
        Proposal storage prop = proposals[proposalId];
        require(_isPassed(prop), "Proposal not passed");
        require(block.timestamp > prop.gracePeriodEnd, "Grace period not over");
        require(!prop.executed, "Already executed");
        require(!prop.cancelled, "Cancelled");
        
        prop.executed = true;
        
        for (uint256 i = 0; i < prop.targets.length; i++) {
            (bool success, bytes memory returnData) = prop.targets[i].call{value: prop.values[i]}(
                prop.calldatas[i]
            );
            
            if (!success) {
                assembly {
                    revert(add(returnData, 32), mload(returnData))
                }
            }
        }
        
        emit ProposalExecuted(proposalId);
    }
    
    function _isPassed(Proposal storage prop) internal view returns (bool) {
        if (block.timestamp <= prop.votingDeadline) return false;
        if (prop.yesVotes <= prop.noVotes) return false;
        
        uint256 totalVotes = prop.yesVotes + prop.noVotes;
        uint256 quorum = governanceToken.totalSupply() * QUORUM_BPS / 10000;
        
        return totalVotes >= quorum;
    }
    
    function getProposalStatus(uint256 proposalId) external view returns (
        bool isPassed,
        bool isInGracePeriod,
        bool isExecutable,
        uint256 timeLeft
    ) {
        Proposal storage prop = proposals[proposalId];
        isPassed = _isPassed(prop);
        
        if (isPassed && prop.gracePeriodEnd > 0) {
            isInGracePeriod = block.timestamp <= prop.gracePeriodEnd;
            timeLeft = isInGracePeriod ? prop.gracePeriodEnd - block.timestamp : 0;
            isExecutable = !isInGracePeriod && !prop.executed;
        }
    }
    
    receive() external payable {}
}
```

---

## 77.7 Quadratic Voting

### สูตร Quadratic Voting

ใน quadratic voting, cost ของ voice credits เพิ่มแบบ quadratic:
- 1 vote = 1 credit
- 2 votes = 4 credits  
- 3 votes = 9 credits
- N votes = N² credits

ดังนั้นคนรวย (มี credits เยอะ) ไม่สามารถ dominate ได้ง่ายเหมือน linear voting

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title QVToken
/// @notice Token พิเศษสำหรับ quadratic voting
/// ผู้ถือได้รับ voice credits แบบ sqrt(tokens) ไม่ใช่ linear
contract QVToken {
    
    mapping(address => uint256) public tokenBalance;
    mapping(address => uint256) public voiceCredits;  // sqrt(tokens) * SCALE
    uint256 public totalTokens;
    
    uint256 public constant SCALE = 1e9; // precision multiplier
    
    event TokensMinted(address indexed to, uint256 tokens, uint256 credits);
    
    function mint(address to, uint256 amount) external {
        uint256 oldBalance = tokenBalance[to];
        uint256 newBalance = oldBalance + amount;
        
        tokenBalance[to] = newBalance;
        totalTokens += amount;
        
        // คำนวณ voice credits ใหม่: sqrt(newBalance) - sqrt(oldBalance)
        uint256 oldCredits = _sqrt(oldBalance * SCALE * SCALE);
        uint256 newCredits = _sqrt(newBalance * SCALE * SCALE);
        
        voiceCredits[to] += (newCredits - oldCredits);
        
        emit TokensMinted(to, amount, newCredits - oldCredits);
    }
    
    function getVoiceCredits(address holder) external view returns (uint256) {
        return _sqrt(tokenBalance[holder] * SCALE * SCALE);
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

/// @title QuadraticVoting
/// @notice Governance ด้วย quadratic voting
contract QuadraticVoting {
    
    QVToken public immutable token;
    
    struct QVProposal {
        string description;
        mapping(uint8 => int256) netVotes;    // option → net quadratic votes
        mapping(address => mapping(uint8 => int256)) userVotes; // voter → option → votes cast
        mapping(address => uint256) creditsUsed; // total credits used per voter
        uint256 deadline;
        uint8 optionCount;
        bool resolved;
        uint8 winner;
    }
    
    mapping(uint256 => QVProposal) public proposals;
    uint256 public proposalCount;
    
    uint256 public constant MAX_CREDITS_PER_PROPOSAL = 100; // credits ต่อ proposal
    
    event ProposalCreated(uint256 indexed id, string description, uint8 optionCount);
    event VoteCast(uint256 indexed id, address voter, uint8 option, int256 votes, uint256 creditsUsed);
    event ProposalResolved(uint256 indexed id, uint8 winner);
    
    constructor(address _token) {
        token = QVToken(_token);
    }
    
    function createProposal(
        string calldata description,
        uint8 optionCount,
        uint256 durationSeconds
    ) external returns (uint256) {
        require(optionCount >= 2, "Need at least 2 options");
        require(durationSeconds >= 1 days, "Too short");
        
        uint256 id = ++proposalCount;
        QVProposal storage prop = proposals[id];
        prop.description = description;
        prop.optionCount = optionCount;
        prop.deadline = block.timestamp + durationSeconds;
        
        emit ProposalCreated(id, description, optionCount);
        return id;
    }
    
    /// @notice คำนวณ voice credits ที่ต้องใช้สำหรับ N votes
    /// cost = votes² (quadratic cost)
    function calculateCost(int256 votes) public pure returns (uint256 cost) {
        if (votes < 0) votes = -votes;
        uint256 absVotes = uint256(votes);
        return absVotes * absVotes;
    }
    
    /// @notice Cast quadratic votes
    /// @param proposalId Proposal ID
    /// @param option Option index (0 to optionCount-1)
    /// @param votes Positive for support, negative for oppose
    function castVote(uint256 proposalId, uint8 option, int256 votes) external {
        QVProposal storage prop = proposals[proposalId];
        require(block.timestamp <= prop.deadline, "Voting ended");
        require(!prop.resolved, "Already resolved");
        require(option < prop.optionCount, "Invalid option");
        require(votes != 0, "Zero votes");
        
        // คำนวณ cost
        int256 existingVotes = prop.userVotes[msg.sender][option];
        int256 newTotalVotes = existingVotes + votes;
        
        uint256 oldCost = calculateCost(existingVotes);
        uint256 newCost = calculateCost(newTotalVotes);
        
        uint256 totalCreditsUsed = prop.creditsUsed[msg.sender] - oldCost + newCost;
        require(totalCreditsUsed <= MAX_CREDITS_PER_PROPOSAL, "Exceeds credit limit");
        
        // ตรวจสอบว่ามี voice credits พอ
        uint256 availableCredits = token.getVoiceCredits(msg.sender);
        require(availableCredits >= totalCreditsUsed, "Insufficient voice credits");
        
        // บันทึก vote
        prop.netVotes[option] += votes;
        prop.userVotes[msg.sender][option] = newTotalVotes;
        prop.creditsUsed[msg.sender] = totalCreditsUsed;
        
        emit VoteCast(proposalId, msg.sender, option, votes, totalCreditsUsed);
    }
    
    /// @notice Tally votes และหา winner
    function resolveProposal(uint256 proposalId) external returns (uint8 winner) {
        QVProposal storage prop = proposals[proposalId];
        require(block.timestamp > prop.deadline, "Voting not ended");
        require(!prop.resolved, "Already resolved");
        
        int256 maxVotes = type(int256).min;
        winner = 0;
        
        for (uint8 i = 0; i < prop.optionCount; i++) {
            if (prop.netVotes[i] > maxVotes) {
                maxVotes = prop.netVotes[i];
                winner = i;
            }
        }
        
        prop.resolved = true;
        prop.winner = winner;
        
        emit ProposalResolved(proposalId, winner);
    }
    
    /// @notice ดู votes ของแต่ละ option
    function getOptionVotes(uint256 proposalId, uint8 option) external view returns (int256) {
        return proposals[proposalId].netVotes[option];
    }
    
    /// @notice ดู votes ที่ voter cast ให้ option
    function getUserVotes(
        uint256 proposalId,
        address voter,
        uint8 option
    ) external view returns (int256) {
        return proposals[proposalId].userVotes[voter][option];
    }
    
    /// @notice คำนวณ voting power จริงๆ ด้วย quadratic formula
    function getEffectiveVotingPower(address voter) external view returns (uint256) {
        return token.getVoiceCredits(voter);
    }
}
```

---

## Workshop: Advanced DAO Implementation

**โจทย์**: สร้าง `HybridDAO` ที่รวม:
1. Optimistic governance สำหรับ routine changes
2. Full quorum voting สำหรับ critical changes
3. Rage-quit สำหรับ ทั้งสอง paths
4. Partial delegation

### Architecture:

```
HybridDAO
├── GovernanceToken (ERC20Votes + PartialDelegation)
├── OptimisticPath (small changes < $10k)
│   └── 3-day veto window
│   └── Guardian veto
│   └── Rage-quit (1-day grace)
├── StandardPath (large changes > $10k)
│   └── 5-day voting
│   └── 20% quorum
│   └── Rage-quit (3-day grace)
└── EmergencyPath (security fixes)
    └── Guardian only
    └── No timelock
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title HybridDAO
/// @notice DAO ที่ผสม optimistic และ standard governance
contract HybridDAO {
    
    enum ProposalType { OPTIMISTIC, STANDARD, EMERGENCY }
    
    struct Proposal {
        ProposalType pType;
        address proposer;
        string description;
        address[] targets;
        uint256[] values;
        bytes[] calldatas;
        
        // Timing
        uint256 createdAt;
        uint256 votingEnd;      // สำหรับ STANDARD
        uint256 vetoEnd;        // สำหรับ OPTIMISTIC
        uint256 gracePeriodEnd; // rage-quit window
        
        // Votes
        uint256 yesVotes;
        uint256 noVotes;
        uint256 vetoVotes;
        
        // State
        bool executed;
        bool cancelled;
        bool vetoed;
    }
    
    mapping(uint256 => Proposal) public proposals;
    mapping(uint256 => mapping(address => bool)) public voted;
    mapping(uint256 => mapping(address => bool)) public vetoed;
    mapping(uint256 => mapping(address => bool)) public rageQuit;
    
    uint256 public proposalCount;
    address public governanceToken;
    mapping(address => bool) public guardians;
    
    // Thresholds
    uint256 public constant OPTIMISTIC_LIMIT = 10_000e18;  // $10k ETH
    uint256 public constant STANDARD_QUORUM_BPS = 2000;    // 20%
    uint256 public constant OPTIMISTIC_VETO_BPS = 1000;    // 10%
    
    event ProposalCreated(uint256 indexed id, ProposalType pType, address proposer);
    event Voted(uint256 indexed id, address voter, bool support, uint256 weight);
    event Vetoed(uint256 indexed id, address vetoer);
    event RageQuitter(uint256 indexed id, address member, uint256 shares);
    event Executed(uint256 indexed id);
    
    constructor(address _token) {
        governanceToken = _token;
    }
    
    function createProposal(
        ProposalType pType,
        string calldata description,
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata calldatas
    ) external returns (uint256) {
        uint256 totalValue;
        for (uint256 i = 0; i < values.length; i++) {
            totalValue += values[i];
        }
        
        // Emergency ต้องเป็น guardian
        if (pType == ProposalType.EMERGENCY) {
            require(guardians[msg.sender], "Not guardian");
        }
        
        // Auto-detect type จาก value
        if (pType == ProposalType.OPTIMISTIC && totalValue > OPTIMISTIC_LIMIT) {
            pType = ProposalType.STANDARD;
        }
        
        uint256 id = ++proposalCount;
        Proposal storage prop = proposals[id];
        
        prop.pType = pType;
        prop.proposer = msg.sender;
        prop.description = description;
        prop.targets = targets;
        prop.values = values;
        prop.calldatas = calldatas;
        prop.createdAt = block.timestamp;
        
        if (pType == ProposalType.OPTIMISTIC) {
            prop.vetoEnd = block.timestamp + 3 days;
            prop.gracePeriodEnd = prop.vetoEnd + 1 days;
        } else if (pType == ProposalType.STANDARD) {
            prop.votingEnd = block.timestamp + 5 days;
            prop.gracePeriodEnd = prop.votingEnd + 3 days;
        }
        // EMERGENCY: no waiting
        
        emit ProposalCreated(id, pType, msg.sender);
        return id;
    }
    
    function vote(uint256 id, bool support) external {
        Proposal storage prop = proposals[id];
        require(prop.pType == ProposalType.STANDARD, "Not standard proposal");
        require(block.timestamp <= prop.votingEnd, "Voting ended");
        require(!voted[id][msg.sender], "Already voted");
        
        voted[id][msg.sender] = true;
        uint256 weight = _getBalance(msg.sender);
        
        if (support) prop.yesVotes += weight;
        else prop.noVotes += weight;
        
        emit Voted(id, msg.sender, support, weight);
    }
    
    function vetoOptimistic(uint256 id) external {
        Proposal storage prop = proposals[id];
        require(prop.pType == ProposalType.OPTIMISTIC, "Not optimistic");
        require(block.timestamp <= prop.vetoEnd, "Veto window closed");
        require(!vetoed[id][msg.sender], "Already vetoed");
        
        vetoed[id][msg.sender] = true;
        prop.vetoVotes += _getBalance(msg.sender);
        
        uint256 totalSupply = _getTotalSupply();
        if (prop.vetoVotes >= totalSupply * OPTIMISTIC_VETO_BPS / 10000) {
            prop.vetoed = true;
        }
        
        emit Vetoed(id, msg.sender);
    }
    
    function performRageQuit(uint256 id, uint256 shares) external {
        Proposal storage prop = proposals[id];
        require(!prop.executed && !prop.cancelled && !prop.vetoed, "Invalid state");
        require(block.timestamp <= prop.gracePeriodEnd, "Grace period ended");
        require(!rageQuit[id][msg.sender], "Already rage quit");
        
        rageQuit[id][msg.sender] = true;
        
        // Transfer proportional share of treasury to member
        // (simplified: ใน production จะมี token list)
        uint256 totalSupply = _getTotalSupply();
        uint256 ethShare = address(this).balance * shares / totalSupply;
        
        // Burn shares
        // token.burnFrom(msg.sender, shares);
        
        if (ethShare > 0) {
            payable(msg.sender).transfer(ethShare);
        }
        
        emit RageQuitter(id, msg.sender, shares);
    }
    
    function execute(uint256 id) external payable {
        Proposal storage prop = proposals[id];
        require(!prop.executed && !prop.cancelled && !prop.vetoed, "Invalid state");
        
        // Check execution conditions
        if (prop.pType == ProposalType.OPTIMISTIC) {
            require(block.timestamp > prop.gracePeriodEnd, "Grace period not over");
        } else if (prop.pType == ProposalType.STANDARD) {
            require(block.timestamp > prop.gracePeriodEnd, "Grace period not over");
            require(_standardPassed(prop), "Proposal did not pass");
        }
        // EMERGENCY: no conditions
        
        prop.executed = true;
        
        for (uint256 i = 0; i < prop.targets.length; i++) {
            (bool success,) = prop.targets[i].call{value: prop.values[i]}(prop.calldatas[i]);
            require(success, "Execution failed");
        }
        
        emit Executed(id);
    }
    
    function _standardPassed(Proposal storage prop) internal view returns (bool) {
        if (prop.yesVotes <= prop.noVotes) return false;
        uint256 quorum = _getTotalSupply() * STANDARD_QUORUM_BPS / 10000;
        return (prop.yesVotes + prop.noVotes) >= quorum;
    }
    
    function _getBalance(address addr) internal view returns (uint256) {
        (bool s, bytes memory d) = governanceToken.staticcall(
            abi.encodeWithSignature("balanceOf(address)", addr)
        );
        if (!s) return 0;
        return abi.decode(d, (uint256));
    }
    
    function _getTotalSupply() internal view returns (uint256) {
        (bool s, bytes memory d) = governanceToken.staticcall(
            abi.encodeWithSignature("totalSupply()")
        );
        if (!s) return 1;
        return abi.decode(d, (uint256));
    }
    
    receive() external payable {}
}
```

---

## สรุป Part 77

- **Governor Alpha → Bravo → OZ Governor**: วิวัฒนาการจาก monolithic สู่ modular, composable governance
- **Partial delegation**: กระจาย voting power ให้หลาย delegate ด้วย basis points ช่วยให้ flexible กว่า full delegation
- **Vote-by-signature (EIP-712)**: ให้ token holders vote โดยไม่จ่าย gas เอง — relayer จ่ายแทน
- **Optimistic governance**: ลด overhead สำหรับ routine changes — ผ่านโดย default ถ้าไม่ถูก veto
- **Rage-quit**: ปกป้อง minority holders — ถอนส่วนแบ่งออกก่อน bad proposal execute
- **Quadratic voting**: ลด plutocracy — cost เพิ่มแบบ quadratic ทำให้ votes แพงขึ้นสำหรับผู้มีอำนาจมาก

## Next: Part 78 - Advanced Protocol Economics
