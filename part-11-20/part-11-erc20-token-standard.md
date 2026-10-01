# Part 11: ERC-20 Token Standard

## สารบัญ
1. ERC-20 คืออะไร
2. ERC-20 Interface และ Events
3. Implementation สมบูรณ์
4. ERC-20 Extensions
5. SafeERC20 Pattern
6. Token Vesting
7. Token với Permit (EIP-2612)
8. Workshop: Governance Token

---

## 1. ERC-20 คืออะไร

ERC-20 (Ethereum Request for Comments 20) คือมาตรฐาน token บน Ethereum ที่กำหนด interface สำหรับ fungible tokens

```
Fungible = แทนกันได้ (1 MTK = 1 MTK ทุกที่)
Non-fungible = ไม่แทนกัน (NFT #1 ≠ NFT #2)
```

**Use cases:**
- Utility Tokens (ใช้ในระบบ)
- Governance Tokens (โหวต)
- Stablecoins (USDC, DAI)
- Wrapped Tokens (WETH)
- LP Tokens (Uniswap)
- Reward Tokens

---

## 2. ERC-20 Interface และ Events

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface IERC20 {
    // Events - ต้อง emit ทุกครั้งที่เปลี่ยน balance/allowance
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    // Functions
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
}

interface IERC20Metadata is IERC20 {
    function name() external view returns (string memory);
    function symbol() external view returns (string memory);
    function decimals() external view returns (uint8);
}
```

---

## 3. ERC-20 Implementation สมบูรณ์

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract ERC20 is IERC20Metadata {
    
    mapping(address account => uint256) private _balances;
    mapping(address account => mapping(address spender => uint256)) private _allowances;
    
    uint256 private _totalSupply;
    string private _name;
    string private _symbol;
    
    // Custom errors (gas efficient)
    error ERC20InsufficientBalance(address sender, uint256 balance, uint256 needed);
    error ERC20InvalidSender(address sender);
    error ERC20InvalidReceiver(address receiver);
    error ERC20InsufficientAllowance(address spender, uint256 allowance, uint256 needed);
    error ERC20InvalidApprover(address approver);
    error ERC20InvalidSpender(address spender);
    
    constructor(string memory name_, string memory symbol_) {
        _name = name_;
        _symbol = symbol_;
    }
    
    function name() public view virtual returns (string memory) { return _name; }
    function symbol() public view virtual returns (string memory) { return _symbol; }
    function decimals() public view virtual returns (uint8) { return 18; }
    function totalSupply() public view virtual returns (uint256) { return _totalSupply; }
    function balanceOf(address account) public view virtual returns (uint256) { return _balances[account]; }
    
    function transfer(address to, uint256 value) public virtual returns (bool) {
        address owner = msg.sender;
        _transfer(owner, to, value);
        return true;
    }
    
    function allowance(address owner, address spender) public view virtual returns (uint256) {
        return _allowances[owner][spender];
    }
    
    function approve(address spender, uint256 value) public virtual returns (bool) {
        address owner = msg.sender;
        _approve(owner, spender, value);
        return true;
    }
    
    function transferFrom(address from, address to, uint256 value) public virtual returns (bool) {
        address spender = msg.sender;
        _spendAllowance(from, spender, value);
        _transfer(from, to, value);
        return true;
    }
    
    function _transfer(address from, address to, uint256 value) internal virtual {
        if (from == address(0)) revert ERC20InvalidSender(address(0));
        if (to == address(0)) revert ERC20InvalidReceiver(address(0));
        _update(from, to, value);
    }
    
    function _update(address from, address to, uint256 value) internal virtual {
        if (from == address(0)) {
            // Mint
            _totalSupply += value;
        } else {
            uint256 fromBalance = _balances[from];
            if (fromBalance < value) {
                revert ERC20InsufficientBalance(from, fromBalance, value);
            }
            unchecked { _balances[from] = fromBalance - value; }
        }
        
        if (to == address(0)) {
            // Burn
            unchecked { _totalSupply -= value; }
        } else {
            unchecked { _balances[to] += value; }
        }
        
        emit Transfer(from, to, value);
    }
    
    function _mint(address account, uint256 value) internal virtual {
        if (account == address(0)) revert ERC20InvalidReceiver(address(0));
        _update(address(0), account, value);
    }
    
    function _burn(address account, uint256 value) internal virtual {
        if (account == address(0)) revert ERC20InvalidSender(address(0));
        _update(account, address(0), value);
    }
    
    function _approve(address owner, address spender, uint256 value) internal virtual {
        _approve(owner, spender, value, true);
    }
    
    function _approve(address owner, address spender, uint256 value, bool emitEvent) internal virtual {
        if (owner == address(0)) revert ERC20InvalidApprover(address(0));
        if (spender == address(0)) revert ERC20InvalidSpender(address(0));
        _allowances[owner][spender] = value;
        if (emitEvent) emit Approval(owner, spender, value);
    }
    
    function _spendAllowance(address owner, address spender, uint256 value) internal virtual {
        uint256 currentAllowance = allowance(owner, spender);
        if (currentAllowance != type(uint256).max) {
            if (currentAllowance < value) {
                revert ERC20InsufficientAllowance(spender, currentAllowance, value);
            }
            unchecked { _approve(owner, spender, currentAllowance - value, false); }
        }
    }
}
```

---

## 4. ERC-20 Extensions

### ERC-20 Burnable

```solidity
abstract contract ERC20Burnable is ERC20 {
    
    function burn(uint256 value) public virtual {
        _burn(msg.sender, value);
    }
    
    function burnFrom(address account, uint256 value) public virtual {
        _spendAllowance(account, msg.sender, value);
        _burn(account, value);
    }
}
```

### ERC-20 Capped

```solidity
abstract contract ERC20Capped is ERC20 {
    
    uint256 private immutable _cap;
    
    error ERC20ExceededCap(uint256 increasedSupply, uint256 cap);
    
    constructor(uint256 cap_) {
        require(cap_ > 0, "Zero cap");
        _cap = cap_;
    }
    
    function cap() public view virtual returns (uint256) {
        return _cap;
    }
    
    function _update(address from, address to, uint256 value) internal virtual override {
        super._update(from, to, value);
        
        if (from == address(0)) {
            uint256 maxSupply = cap();
            uint256 supply = totalSupply();
            if (supply > maxSupply) {
                revert ERC20ExceededCap(supply, maxSupply);
            }
        }
    }
}
```

### ERC-20 Snapshot (สำหรับ Governance)

```solidity
abstract contract ERC20Snapshot is ERC20 {
    
    struct Snapshot {
        uint256 id;
        uint256 value;
    }
    
    Snapshot[] private _accountBalanceSnapshots;
    uint256 private _currentSnapshotId;
    
    mapping(address => Snapshot[]) private _accountBalanceSnapshotsMap;
    Snapshot[] private _totalSupplySnapshots;
    
    event SnapshotCreated(uint256 indexed id);
    
    function _snapshot() internal virtual returns (uint256) {
        _currentSnapshotId++;
        uint256 currentId = _currentSnapshotId;
        emit SnapshotCreated(currentId);
        return currentId;
    }
    
    function getCurrentSnapshotId() public view virtual returns (uint256) {
        return _currentSnapshotId;
    }
    
    function balanceOfAt(address account, uint256 snapshotId) 
        public view virtual returns (uint256) 
    {
        (bool snapshotted, uint256 value) = _valueAt(snapshotId, _accountBalanceSnapshotsMap[account]);
        return snapshotted ? value : balanceOf(account);
    }
    
    function totalSupplyAt(uint256 snapshotId) public view virtual returns (uint256) {
        (bool snapshotted, uint256 value) = _valueAt(snapshotId, _totalSupplySnapshots);
        return snapshotted ? value : totalSupply();
    }
    
    function _valueAt(uint256 snapshotId, Snapshot[] storage snapshots) 
        private view returns (bool, uint256) 
    {
        require(snapshotId > 0, "Zero id");
        require(snapshotId <= _currentSnapshotId, "Nonexistent id");
        
        uint256 index = _findUpperBound(snapshots, snapshotId);
        
        if (index == 0) return (false, 0);
        
        Snapshot storage snapshot = snapshots[index - 1];
        return (snapshot.id == snapshotId, snapshot.value);
    }
    
    function _findUpperBound(Snapshot[] storage array, uint256 value) 
        private view returns (uint256) 
    {
        if (array.length == 0) return 0;
        
        uint256 low = 0;
        uint256 high = array.length;
        
        while (low < high) {
            uint256 mid = (low + high) / 2;
            if (array[mid].id > value) {
                high = mid;
            } else {
                low = mid + 1;
            }
        }
        
        return high;
    }
    
    function _update(address from, address to, uint256 value) internal virtual override {
        super._update(from, to, value);
        
        if (_currentSnapshotId > 0) {
            uint256 currentId = _currentSnapshotId;
            
            if (from != address(0)) {
                _updateAccountSnapshot(from, currentId);
            }
            if (to != address(0)) {
                _updateAccountSnapshot(to, currentId);
            }
            _updateTotalSupplySnapshot(currentId);
        }
    }
    
    function _updateAccountSnapshot(address account, uint256 currentId) private {
        Snapshot[] storage snapshots = _accountBalanceSnapshotsMap[account];
        _updateSnapshot(snapshots, balanceOf(account), currentId);
    }
    
    function _updateTotalSupplySnapshot(uint256 currentId) private {
        _updateSnapshot(_totalSupplySnapshots, totalSupply(), currentId);
    }
    
    function _updateSnapshot(
        Snapshot[] storage snapshots, 
        uint256 currentValue, 
        uint256 currentId
    ) private {
        uint256 len = snapshots.length;
        if (len > 0 && snapshots[len - 1].id == currentId) {
            snapshots[len - 1].value = currentValue;
        } else {
            snapshots.push(Snapshot({id: currentId, value: currentValue}));
        }
    }
}
```

---

## 5. EIP-2612: Permit (Gasless Approve)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// Permit ช่วยให้ user ไม่ต้อง send tx approve ก่อน
// ใช้ signature แทน - gasless approve

interface IERC20Permit {
    function permit(
        address owner,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external;
    
    function nonces(address owner) external view returns (uint256);
    function DOMAIN_SEPARATOR() external view returns (bytes32);
}

abstract contract ERC20Permit is ERC20, IERC20Permit {
    
    mapping(address account => uint256) private _nonces;
    
    bytes32 private immutable _hashedName;
    bytes32 private constant TYPE_HASH = 
        keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)");
    bytes32 private constant PERMIT_TYPEHASH = 
        keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)");
    
    error ERC2612ExpiredSignature(uint256 deadline);
    error ERC2612InvalidSigner(address signer, address owner);
    
    constructor(string memory name_) {
        _hashedName = keccak256(bytes(name_));
    }
    
    function permit(
        address owner,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) public virtual override {
        if (block.timestamp > deadline) {
            revert ERC2612ExpiredSignature(deadline);
        }
        
        bytes32 structHash = keccak256(abi.encode(
            PERMIT_TYPEHASH, 
            owner, 
            spender, 
            value, 
            _useNonce(owner), 
            deadline
        ));
        
        bytes32 hash = _hashTypedDataV4(structHash);
        address signer = ecrecover(hash, v, r, s);
        
        if (signer != owner) {
            revert ERC2612InvalidSigner(signer, owner);
        }
        
        _approve(owner, spender, value);
    }
    
    function nonces(address owner) public view virtual override returns (uint256) {
        return _nonces[owner];
    }
    
    function DOMAIN_SEPARATOR() external view virtual override returns (bytes32) {
        return _domainSeparatorV4();
    }
    
    function _domainSeparatorV4() internal view returns (bytes32) {
        return keccak256(abi.encode(
            TYPE_HASH,
            _hashedName,
            keccak256("1"),
            block.chainid,
            address(this)
        ));
    }
    
    function _hashTypedDataV4(bytes32 structHash) internal view virtual returns (bytes32) {
        return keccak256(abi.encodePacked("\x19\x01", _domainSeparatorV4(), structHash));
    }
    
    function _useNonce(address owner) internal virtual returns (uint256) {
        unchecked {
            return _nonces[owner]++;
        }
    }
}
```

---

## 6. Token Vesting

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract TokenVesting {
    
    struct VestingSchedule {
        address beneficiary;
        uint256 cliff;       // timestamp ที่เริ่ม vesting
        uint256 start;       // timestamp เริ่มต้น
        uint256 duration;    // ระยะเวลา vesting (seconds)
        uint256 totalAmount; // จำนวนทั้งหมด
        uint256 released;    // จำนวนที่ release แล้ว
        bool revocable;
        bool revoked;
    }
    
    IERC20 public immutable token;
    address public immutable owner;
    
    mapping(bytes32 => VestingSchedule) public vestingSchedules;
    mapping(address => bytes32[]) public beneficiarySchedules;
    bytes32[] public allScheduleIds;
    
    event VestingScheduleCreated(bytes32 indexed id, address indexed beneficiary, uint256 amount);
    event TokensReleased(bytes32 indexed id, address indexed beneficiary, uint256 amount);
    event VestingRevoked(bytes32 indexed id);
    
    error NotOwner();
    error AlreadyRevoked();
    error NothingToRelease();
    error NotRevocable();
    error Unauthorized();
    
    modifier onlyOwner() {
        if (msg.sender != owner) revert NotOwner();
        _;
    }
    
    constructor(address _token) {
        token = IERC20(_token);
        owner = msg.sender;
    }
    
    function createVestingSchedule(
        address beneficiary,
        uint256 start,
        uint256 cliff,
        uint256 duration,
        uint256 totalAmount,
        bool revocable
    ) external onlyOwner returns (bytes32 id) {
        require(beneficiary != address(0), "Zero beneficiary");
        require(duration > 0, "Zero duration");
        require(totalAmount > 0, "Zero amount");
        require(cliff <= duration, "Cliff > duration");
        
        // Transfer tokens vesting contract
        token.transferFrom(msg.sender, address(this), totalAmount);
        
        id = keccak256(abi.encodePacked(beneficiary, start, duration, totalAmount, block.timestamp));
        
        vestingSchedules[id] = VestingSchedule({
            beneficiary: beneficiary,
            cliff: start + cliff,
            start: start,
            duration: duration,
            totalAmount: totalAmount,
            released: 0,
            revocable: revocable,
            revoked: false
        });
        
        beneficiarySchedules[beneficiary].push(id);
        allScheduleIds.push(id);
        
        emit VestingScheduleCreated(id, beneficiary, totalAmount);
    }
    
    function release(bytes32 id) external {
        VestingSchedule storage schedule = vestingSchedules[id];
        
        require(!schedule.revoked, "Revoked");
        require(msg.sender == schedule.beneficiary, "Not beneficiary");
        
        uint256 releasable = _computeReleasableAmount(schedule);
        if (releasable == 0) revert NothingToRelease();
        
        schedule.released += releasable;
        token.transfer(schedule.beneficiary, releasable);
        
        emit TokensReleased(id, schedule.beneficiary, releasable);
    }
    
    function revoke(bytes32 id) external onlyOwner {
        VestingSchedule storage schedule = vestingSchedules[id];
        
        if (!schedule.revocable) revert NotRevocable();
        if (schedule.revoked) revert AlreadyRevoked();
        
        uint256 releasable = _computeReleasableAmount(schedule);
        
        if (releasable > 0) {
            schedule.released += releasable;
            token.transfer(schedule.beneficiary, releasable);
            emit TokensReleased(id, schedule.beneficiary, releasable);
        }
        
        uint256 refund = schedule.totalAmount - schedule.released;
        schedule.revoked = true;
        
        if (refund > 0) {
            token.transfer(owner, refund);
        }
        
        emit VestingRevoked(id);
    }
    
    function _computeReleasableAmount(VestingSchedule storage schedule) 
        internal view returns (uint256) 
    {
        if (block.timestamp < schedule.cliff) return 0;
        
        uint256 elapsed = block.timestamp - schedule.start;
        uint256 vested;
        
        if (elapsed >= schedule.duration) {
            vested = schedule.totalAmount;
        } else {
            vested = (schedule.totalAmount * elapsed) / schedule.duration;
        }
        
        return vested - schedule.released;
    }
    
    function getReleasableAmount(bytes32 id) external view returns (uint256) {
        return _computeReleasableAmount(vestingSchedules[id]);
    }
    
    function getVestedAmount(bytes32 id) external view returns (uint256) {
        VestingSchedule storage schedule = vestingSchedules[id];
        if (block.timestamp < schedule.cliff) return 0;
        
        uint256 elapsed = block.timestamp - schedule.start;
        if (elapsed >= schedule.duration) return schedule.totalAmount;
        return (schedule.totalAmount * elapsed) / schedule.duration;
    }
    
    function getBeneficiarySchedules(address beneficiary) 
        external view returns (bytes32[] memory) 
    {
        return beneficiarySchedules[beneficiary];
    }
}
```

---

## 7. Workshop: Governance Token

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title GovernanceToken
 * @dev ERC-20 + Votes + Permit สำหรับ DAO Governance
 */
contract GovernanceToken is ERC20, ERC20Permit, ERC20Snapshot, ERC20Burnable, Ownable {
    
    uint256 public constant MAX_SUPPLY = 100_000_000 * 1e18; // 100M tokens
    
    // Delegation
    mapping(address => address) private _delegates;
    mapping(address => uint256) private _votingPower;
    
    event DelegateChanged(
        address indexed delegator,
        address indexed fromDelegate,
        address indexed toDelegate
    );
    event DelegateVotesChanged(
        address indexed delegate,
        uint256 previousVotes,
        uint256 newVotes
    );
    
    constructor(address initialOwner) 
        ERC20("Governance Token", "GOV")
        ERC20Permit("Governance Token")
        Ownable(initialOwner)
    {
        // Mint initial supply to owner
        _mint(initialOwner, 10_000_000 * 1e18); // 10M initial
    }
    
    // === Minting ===
    
    function mint(address to, uint256 amount) public onlyOwner {
        require(totalSupply() + amount <= MAX_SUPPLY, "Exceeds max supply");
        _mint(to, amount);
    }
    
    // === Snapshot ===
    
    function snapshot() external onlyOwner returns (uint256) {
        return _snapshot();
    }
    
    // === Delegation ===
    
    function delegate(address delegatee) public {
        _delegate(msg.sender, delegatee);
    }
    
    function delegates(address account) public view returns (address) {
        return _delegates[account];
    }
    
    function getVotes(address account) public view returns (uint256) {
        return _votingPower[account];
    }
    
    function _delegate(address delegator, address delegatee) internal {
        address currentDelegate = _delegates[delegator];
        uint256 delegatorBalance = balanceOf(delegator);
        
        _delegates[delegator] = delegatee;
        
        emit DelegateChanged(delegator, currentDelegate, delegatee);
        
        _moveVotingPower(currentDelegate, delegatee, delegatorBalance);
    }
    
    function _moveVotingPower(address src, address dst, uint256 amount) private {
        if (src != dst && amount > 0) {
            if (src != address(0)) {
                uint256 oldWeight = _votingPower[src];
                uint256 newWeight = oldWeight - amount;
                _votingPower[src] = newWeight;
                emit DelegateVotesChanged(src, oldWeight, newWeight);
            }
            
            if (dst != address(0)) {
                uint256 oldWeight = _votingPower[dst];
                uint256 newWeight = oldWeight + amount;
                _votingPower[dst] = newWeight;
                emit DelegateVotesChanged(dst, oldWeight, newWeight);
            }
        }
    }
    
    // Override _update เพื่อ handle delegation
    function _update(address from, address to, uint256 value) internal virtual override(ERC20, ERC20Snapshot) {
        super._update(from, to, value);
        
        _moveVotingPower(_delegates[from], _delegates[to], value);
    }
    
    // === Pause ===
    
    bool private _paused;
    
    function pause() external onlyOwner {
        _paused = true;
    }
    
    function unpause() external onlyOwner {
        _paused = false;
    }
    
    function _beforeTokenTransfer(address from, address to, uint256 amount) internal virtual {
        require(!_paused || from == address(0) || to == address(0), "Paused");
    }
    
    // Placeholder interfaces ที่ใช้ด้านบน
    function owner() public view returns (address) { return address(0); }
}
```

---

## 8. TypeScript Tests

```typescript
// test/GovernanceToken.test.ts
import { expect } from "chai";
import { ethers } from "hardhat";
import { GovernanceToken } from "../typechain-types";
import { Signer } from "ethers";

describe("GovernanceToken", function () {
  let token: GovernanceToken;
  let owner: Signer;
  let alice: Signer;
  let bob: Signer;
  
  beforeEach(async function () {
    [owner, alice, bob] = await ethers.getSigners();
    
    const Token = await ethers.getContractFactory("GovernanceToken");
    token = await Token.deploy(await owner.getAddress());
  });
  
  describe("Permit", function () {
    it("should allow gasless approve with permit", async function () {
      const aliceAddr = await alice.getAddress();
      const bobAddr = await bob.getAddress();
      const amount = ethers.parseEther("100");
      const deadline = Math.floor(Date.now() / 1000) + 3600; // 1 hour
      
      // Alice signs permit
      const nonce = await token.nonces(aliceAddr);
      const domain = {
        name: "Governance Token",
        version: "1",
        chainId: (await ethers.provider.getNetwork()).chainId,
        verifyingContract: await token.getAddress(),
      };
      
      const types = {
        Permit: [
          { name: "owner", type: "address" },
          { name: "spender", type: "address" },
          { name: "value", type: "uint256" },
          { name: "nonce", type: "uint256" },
          { name: "deadline", type: "uint256" },
        ],
      };
      
      const value = {
        owner: aliceAddr,
        spender: bobAddr,
        value: amount,
        nonce: nonce,
        deadline: deadline,
      };
      
      const signature = await alice.signTypedData(domain, types, value);
      const { v, r, s } = ethers.Signature.from(signature);
      
      // Bob calls permit (pays gas, not Alice)
      await token.connect(bob).permit(aliceAddr, bobAddr, amount, deadline, v, r, s);
      
      expect(await token.allowance(aliceAddr, bobAddr)).to.equal(amount);
    });
  });
  
  describe("Delegation", function () {
    it("should track delegated votes", async function () {
      const ownerAddr = await owner.getAddress();
      const aliceAddr = await alice.getAddress();
      
      // Transfer tokens to alice
      await token.transfer(aliceAddr, ethers.parseEther("1000"));
      
      // Alice delegates to herself
      await token.connect(alice).delegate(aliceAddr);
      
      expect(await token.getVotes(aliceAddr)).to.equal(ethers.parseEther("1000"));
      
      // Alice transfers 500 to bob - votes should decrease
      await token.connect(alice).transfer(await bob.getAddress(), ethers.parseEther("500"));
      
      expect(await token.getVotes(aliceAddr)).to.equal(ethers.parseEther("500"));
    });
  });
  
  describe("Vesting", function () {
    it("should vest tokens linearly", async function () {
      const Vesting = await ethers.getContractFactory("TokenVesting");
      const vesting = await Vesting.deploy(await token.getAddress());
      
      const aliceAddr = await alice.getAddress();
      const vestingAddr = await vesting.getAddress();
      const amount = ethers.parseEther("1000");
      
      // Approve vesting contract
      await token.approve(vestingAddr, amount);
      
      const now = Math.floor(Date.now() / 1000);
      const cliff = 30 * 24 * 3600; // 30 days
      const duration = 365 * 24 * 3600; // 1 year
      
      const id = await vesting.createVestingSchedule.staticCall(
        aliceAddr, now, cliff, duration, amount, true
      );
      await vesting.createVestingSchedule(aliceAddr, now, cliff, duration, amount, true);
      
      // Before cliff: nothing releasable
      expect(await vesting.getReleasableAmount(id)).to.equal(0);
    });
  });
});
```

---

## สรุป Part 11

ERC-20 Token Standard ที่เรียนรู้:
- ✅ ERC-20 interface และ events
- ✅ Full implementation
- ✅ Extensions: Burnable, Capped, Snapshot
- ✅ EIP-2612 Permit (gasless approve)
- ✅ Token Vesting contract
- ✅ Governance Token (Votes + Snapshot)

## Quiz

1. ทำไม Transfer event ต้อง `indexed` ทั้ง `from` และ `to`?
2. `type(uint256).max` ใน allowance หมายความว่าอะไร?
3. Permit (EIP-2612) แก้ปัญหาอะไรของ ERC-20?
4. ทำไม Delegation ถึงสำคัญใน Governance?

---

## Next: Part 12 - ERC-721 NFT Standard
