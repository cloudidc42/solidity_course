# Part 21: DAO และ Governance

## สารบัญ
1. DAO Basics
2. Governor Contract (OpenZeppelin Style)
3. Voting Mechanisms
4. Timelock Integration
5. Workshop: Full DAO System

---

## 1. DAO Basics

```
DAO = Decentralized Autonomous Organization
- ตัดสินใจผ่าน on-chain voting
- ไม่มี CEO/กรรมการบริษัท
- Code is law: rules อยู่ใน smart contracts
- Token holders = shareholders + board

Governance Lifecycle:
1. Proposal: สมาชิกเสนอ action
2. Delay: รอ voting ให้เริ่ม (voting delay)
3. Vote: token holders ลงคะแนน
4. Queue: ผ่านแล้ว รอ timelock
5. Execute: รัน action จริง

Vote Types:
- For: เห็นด้วย
- Against: ไม่เห็นด้วย
- Abstain: งดออกเสียง (นับ quorum แต่ไม่มีผลชนะ/แพ้)
```

---

## 2. Governor Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title DaoGovernor
 * @dev Governor สำหรับ DAO ที่ใช้ ERC20Votes
 * 
 * Proposal States:
 * 0: Pending  - รอ voting delay
 * 1: Active   - กำลัง voting
 * 2: Canceled
 * 3: Defeated - ไม่ผ่าน (quorum ไม่ถึง หรือ Against มากกว่า)
 * 4: Succeeded - ผ่าน
 * 5: Queued   - รอ timelock
 * 6: Expired  - หมดเวลา execute
 * 7: Executed - ดำเนินการแล้ว
 */
contract DaoGovernor {
    
    // Token ที่ใช้ vote (ต้องรองรับ IVotes interface)
    IVotes public immutable token;
    TimelockController public immutable timelock;
    
    // Governance parameters
    uint256 public votingDelay = 1 days;      // รอก่อน vote
    uint256 public votingPeriod = 1 weeks;    // ระยะเวลา vote
    uint256 public proposalThreshold = 1e18;  // min tokens ถึง propose ได้
    uint256 public quorumNumerator = 4;       // 4% quorum
    
    struct ProposalVote {
        uint256 againstVotes;
        uint256 forVotes;
        uint256 abstainVotes;
        mapping(address => bool) hasVoted;
    }
    
    struct ProposalCore {
        address proposer;
        uint256 voteStart;
        uint256 voteEnd;
        bool executed;
        bool canceled;
    }
    
    mapping(uint256 => ProposalCore) private _proposals;
    mapping(uint256 => ProposalVote) private _proposalVotes;
    
    uint256 private _proposalCount;
    
    enum VoteType { Against, For, Abstain }
    
    enum ProposalState {
        Pending,
        Active,
        Canceled,
        Defeated,
        Succeeded,
        Queued,
        Expired,
        Executed
    }
    
    event ProposalCreated(
        uint256 proposalId,
        address proposer,
        address[] targets,
        uint256[] values,
        bytes[] calldatas,
        string description,
        uint256 voteStart,
        uint256 voteEnd
    );
    
    event VoteCast(
        address indexed voter,
        uint256 proposalId,
        uint8 support,
        uint256 weight,
        string reason
    );
    
    event ProposalExecuted(uint256 proposalId);
    event ProposalCanceled(uint256 proposalId);
    
    error GovernorInsufficientProposerVotes(address proposer, uint256 votes, uint256 threshold);
    error GovernorUnexpectedProposalState(uint256 proposalId, ProposalState current);
    error GovernorAlreadyCastVote(address voter);
    error GovernorInvalidVoteType();
    
    constructor(address _token, address payable _timelock) {
        token = IVotes(_token);
        timelock = TimelockController(_timelock);
    }
    
    function propose(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        string memory description
    ) external returns (uint256) {
        address proposer = msg.sender;
        
        // Check proposer has enough votes
        uint256 proposerVotes = token.getPastVotes(proposer, block.number - 1);
        if (proposerVotes < proposalThreshold) {
            revert GovernorInsufficientProposerVotes(proposer, proposerVotes, proposalThreshold);
        }
        
        uint256 proposalId = hashProposal(targets, values, calldatas, keccak256(bytes(description)));
        
        ProposalCore storage proposal = _proposals[proposalId];
        require(proposal.voteStart == 0, "Proposal exists");
        
        uint256 snapshot = block.timestamp + votingDelay;
        uint256 deadline = snapshot + votingPeriod;
        
        proposal.proposer = proposer;
        proposal.voteStart = snapshot;
        proposal.voteEnd = deadline;
        
        emit ProposalCreated(
            proposalId,
            proposer,
            targets,
            values,
            calldatas,
            description,
            snapshot,
            deadline
        );
        
        return proposalId;
    }
    
    function castVote(
        uint256 proposalId,
        uint8 support
    ) external returns (uint256) {
        return _castVote(proposalId, msg.sender, support, "");
    }
    
    function castVoteWithReason(
        uint256 proposalId,
        uint8 support,
        string memory reason
    ) external returns (uint256) {
        return _castVote(proposalId, msg.sender, support, reason);
    }
    
    function _castVote(
        uint256 proposalId,
        address voter,
        uint8 support,
        string memory reason
    ) internal returns (uint256) {
        ProposalState currentState = state(proposalId);
        if (currentState != ProposalState.Active) {
            revert GovernorUnexpectedProposalState(proposalId, currentState);
        }
        
        ProposalVote storage proposalVote = _proposalVotes[proposalId];
        if (proposalVote.hasVoted[voter]) {
            revert GovernorAlreadyCastVote(voter);
        }
        
        proposalVote.hasVoted[voter] = true;
        
        // Get votes at snapshot time
        uint256 weight = token.getPastVotes(voter, _proposals[proposalId].voteStart);
        
        if (support == uint8(VoteType.Against)) {
            proposalVote.againstVotes += weight;
        } else if (support == uint8(VoteType.For)) {
            proposalVote.forVotes += weight;
        } else if (support == uint8(VoteType.Abstain)) {
            proposalVote.abstainVotes += weight;
        } else {
            revert GovernorInvalidVoteType();
        }
        
        emit VoteCast(voter, proposalId, support, weight, reason);
        
        return weight;
    }
    
    function execute(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        bytes32 descriptionHash
    ) external payable returns (uint256) {
        uint256 proposalId = hashProposal(targets, values, calldatas, descriptionHash);
        
        ProposalState currentState = state(proposalId);
        if (currentState != ProposalState.Succeeded && currentState != ProposalState.Queued) {
            revert GovernorUnexpectedProposalState(proposalId, currentState);
        }
        
        _proposals[proposalId].executed = true;
        
        // Execute via timelock
        bytes32 timelockId = timelock.hashOperationBatch(targets, values, calldatas, 0, keccak256(bytes(abi.encode(proposalId))));
        timelock.executeBatch{value: msg.value}(targets, values, calldatas, 0, keccak256(bytes(abi.encode(proposalId))));
        
        emit ProposalExecuted(proposalId);
        
        return proposalId;
    }
    
    function state(uint256 proposalId) public view returns (ProposalState) {
        ProposalCore storage proposal = _proposals[proposalId];
        
        if (proposal.executed) return ProposalState.Executed;
        if (proposal.canceled) return ProposalState.Canceled;
        if (proposal.voteStart == 0) revert("Unknown proposal");
        
        if (block.timestamp < proposal.voteStart) return ProposalState.Pending;
        if (block.timestamp <= proposal.voteEnd) return ProposalState.Active;
        
        // After voting period: check result
        ProposalVote storage votes = _proposalVotes[proposalId];
        uint256 totalVotes = votes.forVotes + votes.againstVotes + votes.abstainVotes;
        
        // Check quorum
        uint256 pastTotalSupply = token.getPastTotalSupply(proposal.voteStart);
        uint256 quorumRequired = (pastTotalSupply * quorumNumerator) / 100;
        
        if (totalVotes < quorumRequired) return ProposalState.Defeated;
        if (votes.forVotes <= votes.againstVotes) return ProposalState.Defeated;
        
        return ProposalState.Succeeded;
    }
    
    function hashProposal(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        bytes32 descriptionHash
    ) public pure returns (uint256) {
        return uint256(keccak256(abi.encode(targets, values, calldatas, descriptionHash)));
    }
    
    function proposalVotes(uint256 proposalId) external view returns (
        uint256 againstVotes,
        uint256 forVotes,
        uint256 abstainVotes
    ) {
        ProposalVote storage votes = _proposalVotes[proposalId];
        return (votes.againstVotes, votes.forVotes, votes.abstainVotes);
    }
    
    function hasVoted(uint256 proposalId, address account) external view returns (bool) {
        return _proposalVotes[proposalId].hasVoted[account];
    }
}

interface IVotes {
    function getPastVotes(address account, uint256 timepoint) external view returns (uint256);
    function getPastTotalSupply(uint256 timepoint) external view returns (uint256);
}

interface TimelockController {
    function hashOperationBatch(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory payloads,
        bytes32 predecessor,
        bytes32 salt
    ) external pure returns (bytes32);
    
    function executeBatch(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory payloads,
        bytes32 predecessor,
        bytes32 salt
    ) external payable;
}
```

---

## 3. Governance Token with Votes

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title GovernanceToken
 * @dev ERC20 + ERC20Votes รองรับ on-chain governance
 * 
 * Key: ต้อง delegate ก่อน ถึงจะนับ votes
 * delegate(address(self)) = self-delegate
 */
contract GovernanceToken {
    
    string public name;
    string public symbol;
    uint8 public constant decimals = 18;
    uint256 public totalSupply;
    
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    // Votes: checkpoint tracking
    struct Checkpoint {
        uint32 fromBlock;
        uint224 votes;
    }
    
    mapping(address => address) private _delegates;
    mapping(address => Checkpoint[]) private _checkpoints;
    Checkpoint[] private _totalSupplyCheckpoints;
    
    // EIP-712 for permit-style delegation
    bytes32 public constant DELEGATION_TYPEHASH = keccak256("Delegation(address delegatee,uint256 nonce,uint256 expiry)");
    mapping(address => uint256) public nonces;
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    event DelegateChanged(address indexed delegator, address indexed fromDelegate, address indexed toDelegate);
    event DelegateVotesChanged(address indexed delegate, uint256 previousBalance, uint256 newBalance);
    
    constructor(string memory _name, string memory _symbol, uint256 initialSupply) {
        name = _name;
        symbol = _symbol;
        _mint(msg.sender, initialSupply);
    }
    
    // --- ERC20 ---
    
    function transfer(address to, uint256 amount) external returns (bool) {
        _transfer(msg.sender, to, amount);
        return true;
    }
    
    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        if (allowance[from][msg.sender] != type(uint256).max) {
            allowance[from][msg.sender] -= amount;
        }
        _transfer(from, to, amount);
        return true;
    }
    
    function _transfer(address from, address to, uint256 amount) internal {
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        emit Transfer(from, to, amount);
        _moveVotingPower(delegates(from), delegates(to), amount);
    }
    
    // --- Votes ---
    
    function delegates(address account) public view returns (address) {
        return _delegates[account];
    }
    
    function delegate(address delegatee) public {
        _delegate(msg.sender, delegatee);
    }
    
    function _delegate(address delegator, address delegatee) internal {
        address currentDelegate = delegates(delegator);
        _delegates[delegator] = delegatee;
        
        emit DelegateChanged(delegator, currentDelegate, delegatee);
        _moveVotingPower(currentDelegate, delegatee, balanceOf[delegator]);
    }
    
    function _moveVotingPower(address src, address dst, uint256 amount) private {
        if (src != dst && amount > 0) {
            if (src != address(0)) {
                (uint256 oldWeight, uint256 newWeight) = _writeCheckpoint(_checkpoints[src], _subtract, amount);
                emit DelegateVotesChanged(src, oldWeight, newWeight);
            }
            if (dst != address(0)) {
                (uint256 oldWeight, uint256 newWeight) = _writeCheckpoint(_checkpoints[dst], _add, amount);
                emit DelegateVotesChanged(dst, oldWeight, newWeight);
            }
        }
    }
    
    function getVotes(address account) external view returns (uint256) {
        Checkpoint[] storage checkpoints = _checkpoints[account];
        uint256 pos = checkpoints.length;
        return pos == 0 ? 0 : checkpoints[pos - 1].votes;
    }
    
    function getPastVotes(address account, uint256 blockNumber) external view returns (uint256) {
        require(blockNumber < block.number, "Block not yet mined");
        return _checkpointsLookup(_checkpoints[account], blockNumber);
    }
    
    function getPastTotalSupply(uint256 blockNumber) external view returns (uint256) {
        require(blockNumber < block.number, "Block not yet mined");
        return _checkpointsLookup(_totalSupplyCheckpoints, blockNumber);
    }
    
    function _checkpointsLookup(Checkpoint[] storage ckpts, uint256 blockNumber) private view returns (uint256) {
        uint256 high = ckpts.length;
        uint256 low = 0;
        while (low < high) {
            uint256 mid = (low + high + 1) >> 1;
            if (ckpts[mid - 1].fromBlock <= blockNumber) {
                low = mid;
            } else {
                high = mid - 1;
            }
        }
        return high == 0 ? 0 : ckpts[high - 1].votes;
    }
    
    function _writeCheckpoint(
        Checkpoint[] storage ckpts,
        function(uint256, uint256) pure returns (uint256) op,
        uint256 delta
    ) private returns (uint256 oldWeight, uint256 newWeight) {
        uint256 pos = ckpts.length;
        oldWeight = pos == 0 ? 0 : ckpts[pos - 1].votes;
        newWeight = op(oldWeight, delta);
        
        if (pos > 0 && ckpts[pos - 1].fromBlock == block.number) {
            ckpts[pos - 1].votes = uint224(newWeight);
        } else {
            ckpts.push(Checkpoint({fromBlock: uint32(block.number), votes: uint224(newWeight)}));
        }
    }
    
    function _add(uint256 a, uint256 b) private pure returns (uint256) { return a + b; }
    function _subtract(uint256 a, uint256 b) private pure returns (uint256) { return a - b; }
    
    function _mint(address account, uint256 amount) internal {
        totalSupply += amount;
        balanceOf[account] += amount;
        emit Transfer(address(0), account, amount);
        _writeCheckpoint(_totalSupplyCheckpoints, _add, amount);
    }
}
```

---

## 4. Complete DAO Setup (Hardhat Script)

```typescript
// scripts/deploy-dao.ts
import { ethers } from "hardhat";

async function main() {
    const [deployer] = await ethers.getSigners();
    console.log("Deploying DAO with:", deployer.address);
    
    // 1. Deploy Governance Token
    const GovernanceToken = await ethers.getContractFactory("GovernanceToken");
    const token = await GovernanceToken.deploy(
        "DAO Token",
        "DAOT",
        ethers.parseEther("10000000") // 10M tokens
    );
    await token.waitForDeployment();
    console.log("Token:", await token.getAddress());
    
    // 2. Self-delegate (ต้อง delegate ก่อน votes จะนับ)
    await token.delegate(deployer.address);
    
    // 3. Deploy TimelockController
    const TimelockController = await ethers.getContractFactory("TimelockController");
    const timelock = await TimelockController.deploy(
        2 * 24 * 3600, // 2 day delay
        [ethers.ZeroAddress], // proposers (will be set to governor)
        [ethers.ZeroAddress], // executors (anyone)
        deployer.address // admin
    );
    await timelock.waitForDeployment();
    console.log("Timelock:", await timelock.getAddress());
    
    // 4. Deploy Governor
    const DaoGovernor = await ethers.getContractFactory("DaoGovernor");
    const governor = await DaoGovernor.deploy(
        await token.getAddress(),
        await timelock.getAddress()
    );
    await governor.waitForDeployment();
    console.log("Governor:", await governor.getAddress());
    
    // 5. Setup roles
    const PROPOSER_ROLE = await timelock.PROPOSER_ROLE();
    const EXECUTOR_ROLE = await timelock.EXECUTOR_ROLE();
    const TIMELOCK_ADMIN_ROLE = await timelock.TIMELOCK_ADMIN_ROLE();
    
    // Governor can propose to timelock
    await timelock.grantRole(PROPOSER_ROLE, await governor.getAddress());
    
    // Anyone can execute
    await timelock.grantRole(EXECUTOR_ROLE, ethers.ZeroAddress);
    
    // Renounce admin role (fully decentralized)
    await timelock.renounceRole(TIMELOCK_ADMIN_ROLE, deployer.address);
    
    console.log("\nDAO deployed successfully!");
    console.log("Token:", await token.getAddress());
    console.log("Timelock:", await timelock.getAddress());
    console.log("Governor:", await governor.getAddress());
}

main().catch(console.error);
```

---

## 5. Workshop: สร้าง Proposal และ Vote

```typescript
// test/DAO.test.ts
import { expect } from "chai";
import { ethers } from "hardhat";
import { time, mine } from "@nomicfoundation/hardhat-network-helpers";

describe("DAO Governance", function () {
    let token: any, timelock: any, governor: any, treasury: any;
    let proposer: any, voter1: any, voter2: any, voter3: any;
    
    const VOTING_DELAY = 24 * 3600; // 1 day
    const VOTING_PERIOD = 7 * 24 * 3600; // 7 days
    
    beforeEach(async () => {
        [, proposer, voter1, voter2, voter3] = await ethers.getSigners();
        
        // Deploy token
        const GovernanceToken = await ethers.getContractFactory("GovernanceToken");
        token = await GovernanceToken.deploy("DAO Token", "DAOT", ethers.parseEther("10000000"));
        
        // Distribute tokens
        await token.transfer(proposer.address, ethers.parseEther("100000")); // 1%
        await token.transfer(voter1.address, ethers.parseEther("200000"));   // 2%
        await token.transfer(voter2.address, ethers.parseEther("300000"));   // 3%
        await token.transfer(voter3.address, ethers.parseEther("100000"));   // 1%
        
        // Self-delegate
        await token.connect(proposer).delegate(proposer.address);
        await token.connect(voter1).delegate(voter1.address);
        await token.connect(voter2).delegate(voter2.address);
        await token.connect(voter3).delegate(voter3.address);
        
        // Deploy timelock & governor
        // (simplified - direct deployment)
        const DaoGovernor = await ethers.getContractFactory("DaoGovernor");
        governor = await DaoGovernor.deploy(token.target, ethers.ZeroAddress); // simplified
        
        // Deploy treasury
        const Treasury = await ethers.getContractFactory("DAOTreasury");
        treasury = await Treasury.deploy(governor.target);
        
        // Fund treasury
        await ethers.provider.send("eth_sendTransaction", [{
            from: (await ethers.getSigners())[0].address,
            to: treasury.target,
            value: ethers.parseEther("10").toString(16)
        }]);
    });
    
    it("Full governance flow: propose → vote → execute", async function () {
        // 1. Create proposal
        const transferAmount = ethers.parseEther("1");
        const calldata = treasury.interface.encodeFunctionData("transfer", [
            voter1.address,
            transferAmount
        ]);
        
        const tx = await governor.connect(proposer).propose(
            [treasury.target],
            [0],
            [calldata],
            "Transfer 1 ETH to voter1 as grant"
        );
        
        const receipt = await tx.wait();
        const event = receipt.logs.find((log: any) => log.fragment?.name === "ProposalCreated");
        const proposalId = event.args.proposalId;
        
        // 2. Wait for voting delay
        expect(await governor.state(proposalId)).to.equal(0); // Pending
        await time.increase(VOTING_DELAY + 1);
        await mine(1);
        expect(await governor.state(proposalId)).to.equal(1); // Active
        
        // 3. Cast votes
        await governor.connect(voter1).castVoteWithReason(proposalId, 1, "Good grant!"); // For
        await governor.connect(voter2).castVoteWithReason(proposalId, 1, "Agree");        // For
        await governor.connect(voter3).castVote(proposalId, 0);                          // Against
        
        const votes = await governor.proposalVotes(proposalId);
        console.log("For:", ethers.formatEther(votes.forVotes));
        console.log("Against:", ethers.formatEther(votes.againstVotes));
        
        // 4. Wait for voting period
        await time.increase(VOTING_PERIOD + 1);
        expect(await governor.state(proposalId)).to.equal(4); // Succeeded
        
        // 5. Execute (simplified without timelock)
        const descriptionHash = ethers.id("Transfer 1 ETH to voter1 as grant");
        await governor.execute([treasury.target], [0], [calldata], descriptionHash);
        
        expect(await governor.state(proposalId)).to.equal(7); // Executed
    });
    
    it("Proposal defeated if quorum not met", async function () {
        // Only 1% votes - below 4% quorum
        const calldata = ethers.hexlify(ethers.toUtf8Bytes(""));
        
        const tx = await governor.connect(proposer).propose(
            [ethers.ZeroAddress],
            [0],
            [calldata],
            "Low quorum proposal"
        );
        
        const receipt = await tx.wait();
        const event = receipt.logs.find((log: any) => log.fragment?.name === "ProposalCreated");
        const proposalId = event.args.proposalId;
        
        await time.increase(VOTING_DELAY + 1);
        await mine(1);
        
        // Only voter3 votes (1%) - below 4% quorum
        await governor.connect(voter3).castVote(proposalId, 1);
        
        await time.increase(VOTING_PERIOD + 1);
        
        expect(await governor.state(proposalId)).to.equal(3); // Defeated
    });
});

contract DAOTreasury {
    address public governor;
    
    constructor(address _governor) {
        governor = _governor;
    }
    
    receive() external payable {}
    
    function transfer(address payable to, uint256 amount) external {
        require(msg.sender == governor, "Not governor");
        to.transfer(amount);
    }
}
```

---

## สรุป Part 21

DAO Governance ที่เรียนรู้:
- ✅ Proposal lifecycle (Pending → Active → Succeeded → Executed)
- ✅ Checkpoint-based voting power
- ✅ Quorum mechanism
- ✅ Timelock integration
- ✅ Full DAO deployment flow

## Quiz

1. ทำไมต้อง delegate ก่อน votes ถึงจะนับ?
2. Quorum ป้องกันอะไร?
3. Timelock ทำไมถึงสำคัญสำหรับ DAO?
4. Vote snapshot ณ เวลาไหน?

---

## Next: Part 22 - Oracles และ Chainlink
