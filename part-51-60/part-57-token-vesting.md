# Part 57: Token Vesting & Distribution

## บทนำ

การกระจาย token อย่างยุติธรรมและโปร่งใสเป็นหัวใจสำคัญของ DeFi protocol ที่ดี บทนี้จะครอบคลุม:

1. **VestingSchedule** - โครงสร้าง vesting ที่มี cliff, duration, และ revocable option
2. **VestingVault** - ระบบ vesting สมบูรณ์สำหรับ team/investor/advisor
3. **MerkleDistributor** - Airdrop ผ่าน Merkle proof ที่ gas-efficient
4. **StreamingPayment** - การจ่าย token แบบ continuous stream (Sablier-style)
5. **AllocationManager** - Pattern สำหรับจัดสรร token ให้ team/investor/community

---

## 1. VestingSchedule & VestingVault

### ทฤษฎี Token Vesting

Token Vesting คือกระบวนการปลดล็อก token ทีละน้อยตามระยะเวลา เพื่อ:
- ป้องกัน team dump tokens ทันทีหลัง launch
- สร้างแรงจูงใจระยะยาว
- ให้ investors มั่นใจว่า team committed

**Components สำคัญ:**
- **Cliff Period**: ระยะเวลาที่ยังไม่ได้ token เลย (e.g., 6 เดือน)
- **Vesting Duration**: ระยะเวลาทั้งหมดในการ vest (e.g., 24 เดือน)
- **Slice Period**: ความถี่ในการ release (e.g., ทุกวัน, ทุกเดือน)
- **Revocable**: admin สามารถยกเลิก vesting ได้หรือไม่

### Smart Contract: VestingVault

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title VestingVault
 * @notice ระบบ token vesting สำหรับ team, investors, และ advisors
 * @dev สนับสนุน cliff, linear vesting, slices, และ revocation
 */
contract VestingVault is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ============ Structs ============

    /**
     * @notice โครงสร้าง vesting schedule
     * @param beneficiary ผู้รับ token
     * @param cliff เวลา (timestamp) ที่เริ่ม release token ได้ครั้งแรก
     * @param start เวลาเริ่มต้น vesting
     * @param duration ระยะเวลาทั้งหมด (seconds)
     * @param slicePeriodSeconds ความถี่ release (seconds per slice)
     * @param revocable สามารถ revoke ได้หรือไม่
     * @param amountTotal จำนวน token ทั้งหมดที่ vesting
     * @param released จำนวน token ที่ release ไปแล้ว
     * @param revoked ถูก revoke แล้วหรือไม่
     */
    struct VestingSchedule {
        address beneficiary;
        uint256 cliff;
        uint256 start;
        uint256 duration;
        uint256 slicePeriodSeconds;
        bool revocable;
        uint256 amountTotal;
        uint256 released;
        bool revoked;
    }

    // ============ State Variables ============
    IERC20 public immutable token;

    bytes32[] public vestingScheduleIds;
    mapping(bytes32 => VestingSchedule) public vestingSchedules;
    mapping(address => uint256) public holdersVestingCount;

    uint256 public vestingSchedulesTotalAmount;

    // ============ Events ============
    event VestingScheduleCreated(
        bytes32 indexed vestingScheduleId,
        address indexed beneficiary,
        uint256 start,
        uint256 cliff,
        uint256 duration,
        uint256 slicePeriodSeconds,
        bool revocable,
        uint256 amount
    );
    event TokensReleased(
        bytes32 indexed vestingScheduleId,
        address indexed beneficiary,
        uint256 amount
    );
    event VestingScheduleRevoked(
        bytes32 indexed vestingScheduleId,
        address indexed beneficiary,
        uint256 unreleased
    );
    event TokensWithdrawn(address indexed owner, uint256 amount);

    // ============ Constructor ============
    constructor(address _token) Ownable(msg.sender) {
        require(_token != address(0), "VestingVault: zero address");
        token = IERC20(_token);
    }

    // ============ External Functions ============

    /**
     * @notice สร้าง vesting schedule ใหม่
     * @param _beneficiary ผู้รับ token
     * @param _start เวลาเริ่มต้น (Unix timestamp)
     * @param _cliff ระยะเวลา cliff (seconds หลังจาก start)
     * @param _duration ระยะเวลา vesting ทั้งหมด (seconds)
     * @param _slicePeriodSeconds ความถี่ release token (ขั้นต่ำ 1 วินาที)
     * @param _revocable สามารถ revoke ได้หรือไม่
     * @param _amount จำนวน token ทั้งหมด
     */
    function createVestingSchedule(
        address _beneficiary,
        uint256 _start,
        uint256 _cliff,
        uint256 _duration,
        uint256 _slicePeriodSeconds,
        bool _revocable,
        uint256 _amount
    ) external onlyOwner {
        require(
            getWithdrawableAmount() >= _amount,
            "VestingVault: insufficient tokens"
        );
        require(_beneficiary != address(0), "VestingVault: zero address");
        require(_duration > 0, "VestingVault: duration is 0");
        require(_amount > 0, "VestingVault: amount is 0");
        require(_slicePeriodSeconds >= 1, "VestingVault: slice period too small");
        require(_slicePeriodSeconds <= _duration, "VestingVault: slice > duration");
        require(_cliff <= _duration, "VestingVault: cliff > duration");

        bytes32 vestingScheduleId = computeNextVestingScheduleIdForHolder(_beneficiary);
        uint256 cliff = _start + _cliff;

        vestingSchedules[vestingScheduleId] = VestingSchedule({
            beneficiary: _beneficiary,
            cliff: cliff,
            start: _start,
            duration: _duration,
            slicePeriodSeconds: _slicePeriodSeconds,
            revocable: _revocable,
            amountTotal: _amount,
            released: 0,
            revoked: false
        });

        vestingSchedulesTotalAmount += _amount;
        vestingScheduleIds.push(vestingScheduleId);
        holdersVestingCount[_beneficiary]++;

        emit VestingScheduleCreated(
            vestingScheduleId,
            _beneficiary,
            _start,
            cliff,
            _duration,
            _slicePeriodSeconds,
            _revocable,
            _amount
        );
    }

    /**
     * @notice Release token ที่ vested แล้ว
     * @param vestingScheduleId ID ของ vesting schedule
     * @param amount จำนวน token ที่ต้องการ release
     */
    function release(
        bytes32 vestingScheduleId,
        uint256 amount
    ) external nonReentrant {
        VestingSchedule storage vestingSchedule = vestingSchedules[vestingScheduleId];

        require(
            msg.sender == vestingSchedule.beneficiary || msg.sender == owner(),
            "VestingVault: unauthorized"
        );
        require(!vestingSchedule.revoked, "VestingVault: revoked");

        uint256 vestedAmount = computeReleasableAmount(vestingScheduleId);
        require(amount <= vestedAmount, "VestingVault: insufficient vested");

        vestingSchedule.released += amount;
        vestingSchedulesTotalAmount -= amount;

        token.safeTransfer(vestingSchedule.beneficiary, amount);

        emit TokensReleased(vestingScheduleId, vestingSchedule.beneficiary, amount);
    }

    /**
     * @notice Revoke vesting schedule (เฉพาะ schedule ที่ revocable = true)
     * @param vestingScheduleId ID ของ vesting schedule ที่ต้องการ revoke
     */
    function revoke(bytes32 vestingScheduleId) external onlyOwner {
        VestingSchedule storage vestingSchedule = vestingSchedules[vestingScheduleId];

        require(vestingSchedule.revocable, "VestingVault: not revocable");
        require(!vestingSchedule.revoked, "VestingVault: already revoked");

        uint256 vestedAmount = computeReleasableAmount(vestingScheduleId);

        // Release ส่วนที่ vested แล้วก่อน
        if (vestedAmount > 0) {
            vestingSchedule.released += vestedAmount;
            token.safeTransfer(vestingSchedule.beneficiary, vestedAmount);
            emit TokensReleased(vestingScheduleId, vestingSchedule.beneficiary, vestedAmount);
        }

        // คืน token ที่ยังไม่ vested กลับ
        uint256 unreleased = vestingSchedule.amountTotal - vestingSchedule.released;
        vestingSchedulesTotalAmount -= unreleased;
        vestingSchedule.revoked = true;

        emit VestingScheduleRevoked(vestingScheduleId, vestingSchedule.beneficiary, unreleased);
    }

    /**
     * @notice ถอน token ส่วนเกิน (ที่ไม่ได้ allocate ให้ใคร)
     */
    function withdraw(uint256 amount) external onlyOwner nonReentrant {
        require(getWithdrawableAmount() >= amount, "VestingVault: insufficient balance");
        token.safeTransfer(owner(), amount);
        emit TokensWithdrawn(owner(), amount);
    }

    // ============ View Functions ============

    /**
     * @notice คำนวณจำนวน token ที่ release ได้ตอนนี้
     * @param vestingScheduleId ID ของ vesting schedule
     * @return จำนวน token ที่ release ได้
     */
    function computeReleasableAmount(
        bytes32 vestingScheduleId
    ) public view returns (uint256) {
        VestingSchedule storage vestingSchedule = vestingSchedules[vestingScheduleId];

        if (vestingSchedule.revoked) return 0;

        return _computeVestedAmount(vestingSchedule) - vestingSchedule.released;
    }

    /**
     * @notice คำนวณ token ที่ vested แล้วทั้งหมด (รวมที่ release ไปแล้ว)
     */
    function _computeVestedAmount(
        VestingSchedule storage vestingSchedule
    ) internal view returns (uint256) {
        uint256 currentTime = getCurrentTime();

        // ยังไม่ถึง cliff → ยังไม่ได้ token เลย
        if (currentTime < vestingSchedule.cliff) {
            return 0;
        }
        // ผ่าน duration ไปแล้ว → ได้ token ทั้งหมด
        else if (currentTime >= vestingSchedule.start + vestingSchedule.duration) {
            return vestingSchedule.amountTotal;
        }
        // อยู่ระหว่าง cliff และ end → linear vesting ตาม slice
        else {
            uint256 timeFromStart = currentTime - vestingSchedule.start;
            uint256 secondsPerSlice = vestingSchedule.slicePeriodSeconds;
            uint256 vestedSlices = timeFromStart / secondsPerSlice;
            uint256 vestedSeconds = vestedSlices * secondsPerSlice;

            // คำนวณสัดส่วน token ที่ vested
            uint256 vestedAmount = vestingSchedule.amountTotal * vestedSeconds / vestingSchedule.duration;
            return vestedAmount;
        }
    }

    /**
     * @notice คำนวณจำนวน token ที่ถอนได้ (token ที่ไม่ได้ allocate)
     */
    function getWithdrawableAmount() public view returns (uint256) {
        return token.balanceOf(address(this)) - vestingSchedulesTotalAmount;
    }

    function computeVestingScheduleIdForAddressAndIndex(
        address holder,
        uint256 index
    ) public pure returns (bytes32) {
        return keccak256(abi.encodePacked(holder, index));
    }

    function computeNextVestingScheduleIdForHolder(
        address holder
    ) public view returns (bytes32) {
        return computeVestingScheduleIdForAddressAndIndex(
            holder,
            holdersVestingCount[holder]
        );
    }

    function getVestingSchedulesCount() external view returns (uint256) {
        return vestingScheduleIds.length;
    }

    function getVestingScheduleByAddressAndIndex(
        address holder,
        uint256 index
    ) external view returns (VestingSchedule memory) {
        return vestingSchedules[computeVestingScheduleIdForAddressAndIndex(holder, index)];
    }

    function getCurrentTime() public view virtual returns (uint256) {
        return block.timestamp;
    }

    /**
     * @notice ดู vesting progress ทั้งหมดของ address
     */
    function getVestingProgress(address beneficiary) external view returns (
        uint256 totalAllocated,
        uint256 totalReleased,
        uint256 totalReleasable,
        uint256 totalLocked
    ) {
        uint256 count = holdersVestingCount[beneficiary];

        for (uint256 i = 0; i < count; i++) {
            bytes32 scheduleId = computeVestingScheduleIdForAddressAndIndex(beneficiary, i);
            VestingSchedule storage schedule = vestingSchedules[scheduleId];

            if (!schedule.revoked) {
                totalAllocated += schedule.amountTotal;
                totalReleased += schedule.released;
                totalReleasable += computeReleasableAmount(scheduleId);
            }
        }

        totalLocked = totalAllocated - totalReleased - totalReleasable;
    }
}
```

---

## 2. MerkleDistributor - Airdrop ผ่าน Merkle Proof

### ทฤษฎี Merkle Airdrop

แทนที่จะ loop ส่ง token ทุก address (expensive gas), เราสร้าง Merkle tree ของ recipients ทั้งหมด แล้วให้แต่ละคน claim เอง โดยพิสูจน์ด้วย Merkle proof

**ประหยัด gas อย่างไร:**
- ไม่ต้อง store addresses ทั้งหมด on-chain
- แต่ละ claim ใช้ gas ~O(log n) แทน O(n)
- ไม่ต้อง send transaction ให้ทุก address (deployer ประหยัด gas)

**Bitmap Optimization:**
- ใช้ 1 bit ต่อ account แทน 1 slot (32 bytes)
- ประหยัด storage ถึง 256x

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title MerkleDistributor
 * @notice Gas-efficient airdrop โดยใช้ Merkle proof
 * @dev ใช้ bitmap สำหรับ tracking claims (ประหยัด storage)
 */
contract MerkleDistributor is Ownable {
    using SafeERC20 for IERC20;

    // ============ Immutables ============
    IERC20 public immutable token;
    bytes32 public immutable merkleRoot;

    // ============ State ============
    /// @dev bitmap: claimedBitMap[index/256] bit (index%256) = 1 ถ้าถูก claimed แล้ว
    mapping(uint256 => uint256) private claimedBitMap;

    uint256 public immutable distributionEnd; // เวลาหมดอายุ
    address public immutable treasury;        // ที่รับ token ที่ไม่ถูก claim

    // ============ Events ============
    event Claimed(
        uint256 indexed index,
        address indexed account,
        uint256 amount
    );
    event UnclaimedWithdrawn(address indexed treasury, uint256 amount);

    // ============ Constructor ============
    constructor(
        address _token,
        bytes32 _merkleRoot,
        uint256 _distributionEnd,
        address _treasury
    ) Ownable(msg.sender) {
        require(_token != address(0), "MerkleDistributor: zero token");
        require(_treasury != address(0), "MerkleDistributor: zero treasury");
        require(_distributionEnd > block.timestamp, "MerkleDistributor: invalid end");

        token = IERC20(_token);
        merkleRoot = _merkleRoot;
        distributionEnd = _distributionEnd;
        treasury = _treasury;
    }

    // ============ View Functions ============

    /**
     * @notice ตรวจสอบว่า index นั้นถูก claimed แล้วหรือไม่
     * @param index ลำดับใน Merkle tree
     */
    function isClaimed(uint256 index) public view returns (bool) {
        uint256 claimedWordIndex = index / 256;
        uint256 claimedBitIndex = index % 256;
        uint256 claimedWord = claimedBitMap[claimedWordIndex];
        uint256 mask = (1 << claimedBitIndex);
        return claimedWord & mask == mask;
    }

    // ============ Internal Functions ============

    /**
     * @notice Mark index ว่าถูก claimed แล้ว
     */
    function _setClaimed(uint256 index) internal {
        uint256 claimedWordIndex = index / 256;
        uint256 claimedBitIndex = index % 256;
        claimedBitMap[claimedWordIndex] =
            claimedBitMap[claimedWordIndex] | (1 << claimedBitIndex);
    }

    // ============ External Functions ============

    /**
     * @notice Claim airdrop โดยใช้ Merkle proof
     * @param index ลำดับใน Merkle tree (กำหนดโดย off-chain script)
     * @param account Address ของผู้รับ
     * @param amount จำนวน token ที่ได้รับ
     * @param merkleProof Proof ที่สร้างจาก Merkle tree
     */
    function claim(
        uint256 index,
        address account,
        uint256 amount,
        bytes32[] calldata merkleProof
    ) external {
        require(block.timestamp <= distributionEnd, "MerkleDistributor: distribution ended");
        require(!isClaimed(index), "MerkleDistributor: already claimed");

        // ตรวจสอบ Merkle proof
        // leaf = keccak256(abi.encodePacked(index, account, amount))
        bytes32 node = keccak256(bytes.concat(keccak256(abi.encode(index, account, amount))));
        require(
            MerkleProof.verify(merkleProof, merkleRoot, node),
            "MerkleDistributor: invalid proof"
        );

        _setClaimed(index);
        token.safeTransfer(account, amount);

        emit Claimed(index, account, amount);
    }

    /**
     * @notice Multi-claim: claim หลาย entries ในครั้งเดียว
     */
    function multiClaim(
        uint256[] calldata indexes,
        address[] calldata accounts,
        uint256[] calldata amounts,
        bytes32[][] calldata merkleProofs
    ) external {
        require(
            indexes.length == accounts.length &&
            indexes.length == amounts.length &&
            indexes.length == merkleProofs.length,
            "MerkleDistributor: length mismatch"
        );
        require(block.timestamp <= distributionEnd, "MerkleDistributor: distribution ended");

        for (uint256 i = 0; i < indexes.length; i++) {
            if (!isClaimed(indexes[i])) {
                bytes32 node = keccak256(bytes.concat(keccak256(abi.encode(indexes[i], accounts[i], amounts[i]))));
                if (MerkleProof.verify(merkleProofs[i], merkleRoot, node)) {
                    _setClaimed(indexes[i]);
                    token.safeTransfer(accounts[i], amounts[i]);
                    emit Claimed(indexes[i], accounts[i], amounts[i]);
                }
            }
        }
    }

    /**
     * @notice ถอน token ที่ไม่ถูก claim หลังหมดเวลา
     */
    function withdrawUnclaimed() external {
        require(block.timestamp > distributionEnd, "MerkleDistributor: not ended");
        uint256 balance = token.balanceOf(address(this));
        require(balance > 0, "MerkleDistributor: nothing to withdraw");
        token.safeTransfer(treasury, balance);
        emit UnclaimedWithdrawn(treasury, balance);
    }

    /**
     * @notice ตรวจสอบ proof โดยไม่ claim (สำหรับ UI verification)
     */
    function verifyProof(
        uint256 index,
        address account,
        uint256 amount,
        bytes32[] calldata merkleProof
    ) external view returns (bool valid, bool claimed) {
        claimed = isClaimed(index);
        bytes32 node = keccak256(bytes.concat(keccak256(abi.encode(index, account, amount))));
        valid = MerkleProof.verify(merkleProof, merkleRoot, node);
    }
}
```

### Off-chain Merkle Tree Generation (JavaScript Reference)

```javascript
// ตัวอย่างการสร้าง Merkle Tree ด้วย JavaScript
// (ไม่ใช่ Solidity แต่อธิบายวิธีสร้าง off-chain)

/*
const { MerkleTree } = require('merkletreejs');
const { ethers } = require('ethers');

// รายชื่อผู้รับ airdrop
const airdropList = [
    { index: 0, address: '0x1234...', amount: ethers.parseEther('100') },
    { index: 1, address: '0x5678...', amount: ethers.parseEther('200') },
    // ... รายชื่อทั้งหมด
];

// สร้าง leaf nodes
const leaves = airdropList.map(({ index, address, amount }) => {
    const encoded = ethers.solidityPackedKeccak256(
        ['uint256', 'address', 'uint256'],
        [index, address, amount]
    );
    return ethers.keccak256(encoded); // double-hash for security
});

// สร้าง Merkle Tree
const tree = new MerkleTree(leaves, ethers.keccak256, { sortPairs: true });
const root = tree.getHexRoot();

// สร้าง proof สำหรับแต่ละ address
const proof = tree.getHexProof(leaves[0]);
console.log('Merkle Root:', root);
console.log('Proof for index 0:', proof);
*/
```

---

## 3. StreamingPayment (Sablier-style)

### ทฤษฎี Continuous Stream

แทนที่จะจ่าย token เป็นก้อน (lump sum) หรือตาม schedule, streaming payment จ่ายแบบ continuous ต่อวินาที ซึ่ง:
- ยืดหยุ่นกว่า traditional vesting
- ผู้รับสามารถ withdraw ได้ทุกเมื่อ (เฉพาะส่วนที่ vested แล้ว)
- ผู้จ่ายสามารถ cancel stream ได้ตลอดเวลา

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title StreamingPayment
 * @notice Sablier-style continuous token streaming
 * @dev Token ไหลออกทุกวินาีในอัตราคงที่
 */
contract StreamingPayment is ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ============ Structs ============
    struct Stream {
        address sender;         // ผู้สร้าง stream
        address recipient;      // ผู้รับ token
        uint256 deposit;        // จำนวน token ทั้งหมด
        address tokenAddress;   // token ที่ stream
        uint256 startTime;      // เวลาเริ่ม
        uint256 stopTime;       // เวลาหยุด
        uint256 ratePerSecond;  // อัตราการไหล (token/second)
        uint256 remainingBalance; // token คงเหลือ
    }

    // ============ State ============
    mapping(uint256 => Stream) public streams;
    uint256 public nextStreamId;

    // ============ Events ============
    event StreamCreated(
        uint256 indexed streamId,
        address indexed sender,
        address indexed recipient,
        uint256 deposit,
        address tokenAddress,
        uint256 startTime,
        uint256 stopTime
    );
    event WithdrawFromStream(
        uint256 indexed streamId,
        address indexed recipient,
        uint256 amount
    );
    event CancelStream(
        uint256 indexed streamId,
        address indexed sender,
        address indexed recipient,
        uint256 senderBalance,
        uint256 recipientBalance
    );

    // ============ Modifiers ============
    modifier onlySenderOrRecipient(uint256 streamId) {
        require(
            msg.sender == streams[streamId].sender ||
            msg.sender == streams[streamId].recipient,
            "StreamingPayment: unauthorized"
        );
        _;
    }

    // ============ Constructor ============
    constructor() {
        nextStreamId = 1;
    }

    // ============ External Functions ============

    /**
     * @notice สร้าง stream ใหม่
     * @param recipient ผู้รับ token
     * @param deposit จำนวน token ทั้งหมด
     * @param tokenAddress token ที่ stream
     * @param startTime เวลาเริ่ม (Unix timestamp)
     * @param stopTime เวลาหยุด (Unix timestamp)
     * @return streamId ID ของ stream ที่สร้าง
     */
    function createStream(
        address recipient,
        uint256 deposit,
        address tokenAddress,
        uint256 startTime,
        uint256 stopTime
    ) external nonReentrant returns (uint256 streamId) {
        require(recipient != address(0), "StreamingPayment: zero recipient");
        require(recipient != address(this), "StreamingPayment: invalid recipient");
        require(recipient != msg.sender, "StreamingPayment: same addresses");
        require(deposit > 0, "StreamingPayment: zero deposit");
        require(startTime >= block.timestamp, "StreamingPayment: past start time");
        require(stopTime > startTime, "StreamingPayment: invalid stop time");

        uint256 duration = stopTime - startTime;

        // ต้องหารลงตัว เพื่อหลีกเลี่ยง rounding errors
        require(deposit % duration == 0, "StreamingPayment: deposit not divisible");

        uint256 ratePerSecond = deposit / duration;
        require(ratePerSecond > 0, "StreamingPayment: rate too small");

        streamId = nextStreamId++;

        streams[streamId] = Stream({
            sender: msg.sender,
            recipient: recipient,
            deposit: deposit,
            tokenAddress: tokenAddress,
            startTime: startTime,
            stopTime: stopTime,
            ratePerSecond: ratePerSecond,
            remainingBalance: deposit
        });

        IERC20(tokenAddress).safeTransferFrom(msg.sender, address(this), deposit);

        emit StreamCreated(
            streamId,
            msg.sender,
            recipient,
            deposit,
            tokenAddress,
            startTime,
            stopTime
        );
    }

    /**
     * @notice Withdraw token จาก stream (เฉพาะผู้รับ)
     * @param streamId ID ของ stream
     * @param amount จำนวนที่ต้องการ withdraw
     */
    function withdrawFromStream(
        uint256 streamId,
        uint256 amount
    ) external nonReentrant {
        Stream storage stream = streams[streamId];
        require(stream.recipient == msg.sender, "StreamingPayment: not recipient");
        require(amount > 0, "StreamingPayment: zero amount");

        uint256 balance = balanceOf(streamId, stream.recipient);
        require(balance >= amount, "StreamingPayment: insufficient balance");

        stream.remainingBalance -= amount;
        IERC20(stream.tokenAddress).safeTransfer(stream.recipient, amount);

        emit WithdrawFromStream(streamId, stream.recipient, amount);
    }

    /**
     * @notice Cancel stream (ทั้ง sender และ recipient ทำได้)
     * @param streamId ID ของ stream
     */
    function cancelStream(
        uint256 streamId
    ) external nonReentrant onlySenderOrRecipient(streamId) {
        Stream storage stream = streams[streamId];

        uint256 senderBalance = balanceOf(streamId, stream.sender);
        uint256 recipientBalance = balanceOf(streamId, stream.recipient);

        // ส่ง token คืนตาม balance ที่แต่ละฝ่ายมี
        stream.remainingBalance = 0;

        address tokenAddress = stream.tokenAddress;

        if (recipientBalance > 0) {
            IERC20(tokenAddress).safeTransfer(stream.recipient, recipientBalance);
        }
        if (senderBalance > 0) {
            IERC20(tokenAddress).safeTransfer(stream.sender, senderBalance);
        }

        emit CancelStream(
            streamId,
            stream.sender,
            stream.recipient,
            senderBalance,
            recipientBalance
        );

        delete streams[streamId];
    }

    // ============ View Functions ============

    /**
     * @notice คำนวณ balance ของ address ใน stream
     * @param streamId ID ของ stream
     * @param who Address ที่ต้องการตรวจสอบ (sender หรือ recipient)
     * @return balance จำนวน token ที่เข้าถึงได้
     */
    function balanceOf(
        uint256 streamId,
        address who
    ) public view returns (uint256 balance) {
        Stream storage stream = streams[streamId];

        // คำนวณ token ที่ recipient ได้รับ ณ เวลาปัจจุบัน
        uint256 delta = deltaOf(streamId);
        uint256 recipientBalance = delta * stream.ratePerSecond;

        // recipient balance = vested - already withdrawn
        // sender balance = remaining - recipient's unvested
        if (who == stream.recipient) {
            return recipientBalance;
        } else if (who == stream.sender) {
            return stream.remainingBalance - recipientBalance;
        }
        return 0;
    }

    /**
     * @notice คำนวณเวลาที่ผ่านไปสำหรับ stream
     */
    function deltaOf(uint256 streamId) public view returns (uint256 delta) {
        Stream storage stream = streams[streamId];

        if (block.timestamp <= stream.startTime) return 0;
        if (block.timestamp < stream.stopTime) {
            return block.timestamp - stream.startTime;
        }
        return stream.stopTime - stream.startTime;
    }

    /**
     * @notice ดูข้อมูล stream
     */
    function getStream(uint256 streamId) external view returns (
        address sender,
        address recipient,
        uint256 deposit,
        address tokenAddress,
        uint256 startTime,
        uint256 stopTime,
        uint256 ratePerSecond,
        uint256 remainingBalance
    ) {
        Stream storage stream = streams[streamId];
        return (
            stream.sender,
            stream.recipient,
            stream.deposit,
            stream.tokenAddress,
            stream.startTime,
            stream.stopTime,
            stream.ratePerSecond,
            stream.remainingBalance
        );
    }
}
```

---

## 4. Team/Investor/Community Allocation Pattern

### AllocationManager

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";

/**
 * @title AllocationManager
 * @notice ระบบจัดการการกระจาย token สำหรับ protocol launch
 * @dev แบ่ง token allocation เป็นหมวดหมู่ต่างๆ
 */
contract AllocationManager is AccessControl {
    using SafeERC20 for IERC20;

    bytes32 public constant ALLOCATOR_ROLE = keccak256("ALLOCATOR_ROLE");

    // ============ Structs ============
    struct AllocationCategory {
        string name;
        uint256 totalAllocation;    // จำนวน token ทั้งหมดในหมวดนี้
        uint256 allocatedAmount;    // จำนวนที่ allocate ไปแล้ว
        uint256 cliffMonths;        // Cliff period (months)
        uint256 vestingMonths;      // Vesting period (months)
        bool canRevoke;             // สามารถ revoke ได้หรือไม่
    }

    struct RecipientInfo {
        address recipient;
        uint256 amount;
        string category;
        bool isActive;
    }

    // ============ State ============
    IERC20 public immutable token;
    VestingVault public immutable vestingVault;

    mapping(string => AllocationCategory) public categories;
    string[] public categoryNames;

    mapping(address => RecipientInfo[]) public recipientAllocations;
    mapping(address => bool) public isRecipient;

    uint256 public constant TOTAL_SUPPLY = 1_000_000_000e18; // 1 billion tokens

    // ============ Events ============
    event CategoryCreated(string name, uint256 allocation);
    event AllocationCreated(
        address indexed recipient,
        string category,
        uint256 amount,
        bytes32 vestingScheduleId
    );

    constructor(address _token, address _vestingVault) {
        token = IERC20(_token);
        vestingVault = VestingVault(_vestingVault);

        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(ALLOCATOR_ROLE, msg.sender);

        // ตั้งค่า allocation categories ตาม tokenomics ทั่วไป
        _setupCategories();
    }

    /**
     * @notice ตั้งค่า default allocation categories
     * @dev ตัวเลขเป็นตัวอย่างเท่านั้น - แต่ละ project มี tokenomics ของตัวเอง
     */
    function _setupCategories() internal {
        // Team & Advisors: 15% | 12 month cliff | 36 month vest | revocable
        _createCategory("team", TOTAL_SUPPLY * 15 / 100, 12, 36, true);

        // Investors (Seed): 10% | 6 month cliff | 24 month vest | not revocable
        _createCategory("seed", TOTAL_SUPPLY * 10 / 100, 6, 24, false);

        // Investors (Series A): 8% | 3 month cliff | 18 month vest
        _createCategory("series_a", TOTAL_SUPPLY * 8 / 100, 3, 18, false);

        // Community/Ecosystem: 30% | no cliff | 48 month vest | revocable
        _createCategory("community", TOTAL_SUPPLY * 30 / 100, 0, 48, true);

        // Treasury: 20% | no cliff | 60 month vest | revocable
        _createCategory("treasury", TOTAL_SUPPLY * 20 / 100, 0, 60, true);

        // Liquidity Mining: 12% | immediate | 12 month vest
        _createCategory("liquidity_mining", TOTAL_SUPPLY * 12 / 100, 0, 12, false);

        // Public Sale: 5% | immediate | immediate release
        _createCategory("public_sale", TOTAL_SUPPLY * 5 / 100, 0, 0, false);
    }

    function _createCategory(
        string memory name,
        uint256 totalAlloc,
        uint256 cliffMonths,
        uint256 vestingMonths,
        bool canRevoke
    ) internal {
        categories[name] = AllocationCategory({
            name: name,
            totalAllocation: totalAlloc,
            allocatedAmount: 0,
            cliffMonths: cliffMonths,
            vestingMonths: vestingMonths,
            canRevoke: canRevoke
        });
        categoryNames.push(name);
        emit CategoryCreated(name, totalAlloc);
    }

    /**
     * @notice Allocate token ให้ recipient พร้อม vesting
     * @param recipient Address ของผู้รับ
     * @param category หมวดหมู่ allocation
     * @param amount จำนวน token
     */
    function allocate(
        address recipient,
        string calldata category,
        uint256 amount
    ) external onlyRole(ALLOCATOR_ROLE) returns (bytes32 vestingScheduleId) {
        AllocationCategory storage cat = categories[category];
        require(cat.totalAllocation > 0, "AllocationManager: invalid category");
        require(
            cat.allocatedAmount + amount <= cat.totalAllocation,
            "AllocationManager: exceeds category allocation"
        );

        cat.allocatedAmount += amount;
        isRecipient[recipient] = true;

        recipientAllocations[recipient].push(RecipientInfo({
            recipient: recipient,
            amount: amount,
            category: category,
            isActive: true
        }));

        // สร้าง vesting schedule
        uint256 start = block.timestamp;
        uint256 cliff = cat.cliffMonths * 30 days;
        uint256 duration = cat.vestingMonths == 0 ? 1 : cat.vestingMonths * 30 days;
        uint256 slicePeriod = 1 days; // release ทุกวัน

        // ต้องมี token ใน vault ก่อน
        token.safeTransfer(address(vestingVault), amount);

        // ดึง vestingScheduleId ที่จะสร้าง
        vestingScheduleId = vestingVault.computeNextVestingScheduleIdForHolder(recipient);

        vestingVault.createVestingSchedule(
            recipient,
            start,
            cliff,
            duration,
            slicePeriod,
            cat.canRevoke,
            amount
        );

        emit AllocationCreated(recipient, category, amount, vestingScheduleId);
    }

    /**
     * @notice ดู allocation summary ทั้งหมด
     */
    function getAllocationSummary() external view returns (
        string[] memory names,
        uint256[] memory totals,
        uint256[] memory allocated,
        uint256[] memory remaining
    ) {
        uint256 count = categoryNames.length;
        names = new string[](count);
        totals = new uint256[](count);
        allocated = new uint256[](count);
        remaining = new uint256[](count);

        for (uint256 i = 0; i < count; i++) {
            AllocationCategory storage cat = categories[categoryNames[i]];
            names[i] = cat.name;
            totals[i] = cat.totalAllocation;
            allocated[i] = cat.allocatedAmount;
            remaining[i] = cat.totalAllocation - cat.allocatedAmount;
        }
    }

    /**
     * @notice ดู allocations ทั้งหมดของ recipient
     */
    function getRecipientAllocations(
        address recipient
    ) external view returns (RecipientInfo[] memory) {
        return recipientAllocations[recipient];
    }
}
```

---

## 5. Multi-Round Vesting Schedule Factory

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title MultiRoundVestingFactory
 * @notice Factory สำหรับสร้าง vesting schedules หลาย rounds พร้อมกัน
 * @dev รองรับ batch allocation และ multi-sig approval
 */
contract MultiRoundVestingFactory is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    struct VestingBatch {
        address[] beneficiaries;
        uint256[] amounts;
        uint256 start;
        uint256 cliffDuration;   // seconds
        uint256 vestingDuration; // seconds
        uint256 slicePeriod;     // seconds
        bool revocable;
    }

    struct RoundInfo {
        string name;
        uint256 totalAmount;
        uint256 price;           // token price in USD (18 decimals)
        uint256 maxAllocation;   // max per wallet
        bool isActive;
        uint256 participantCount;
    }

    IERC20 public immutable token;
    VestingVault public immutable vault;

    mapping(uint256 => RoundInfo) public rounds;
    mapping(uint256 => mapping(address => uint256)) public roundAllocations;
    uint256 public roundCount;

    event RoundCreated(uint256 indexed roundId, string name, uint256 totalAmount);
    event BatchVestingCreated(
        uint256 indexed roundId,
        uint256 beneficiaryCount,
        uint256 totalAmount
    );

    constructor(address _token, address _vault) Ownable(msg.sender) {
        token = IERC20(_token);
        vault = VestingVault(_vault);
    }

    /**
     * @notice สร้าง investment round ใหม่
     */
    function createRound(
        string calldata name,
        uint256 totalAmount,
        uint256 price,
        uint256 maxAllocation
    ) external onlyOwner returns (uint256 roundId) {
        roundId = roundCount++;
        rounds[roundId] = RoundInfo({
            name: name,
            totalAmount: totalAmount,
            price: price,
            maxAllocation: maxAllocation,
            isActive: true,
            participantCount: 0
        });
        emit RoundCreated(roundId, name, totalAmount);
    }

    /**
     * @notice Batch สร้าง vesting schedules สำหรับ round
     * @dev Allocator สร้างทีเดียวหลาย address
     */
    function batchCreateVesting(
        uint256 roundId,
        VestingBatch calldata batch
    ) external onlyOwner nonReentrant {
        RoundInfo storage round = rounds[roundId];
        require(round.isActive, "MultiRoundVestingFactory: round not active");
        require(
            batch.beneficiaries.length == batch.amounts.length,
            "MultiRoundVestingFactory: length mismatch"
        );
        require(batch.beneficiaries.length <= 200, "MultiRoundVestingFactory: too many");

        uint256 totalBatchAmount = 0;
        for (uint256 i = 0; i < batch.amounts.length; i++) {
            totalBatchAmount += batch.amounts[i];
        }

        // Transfer tokens to vault
        token.safeTransfer(address(vault), totalBatchAmount);

        for (uint256 i = 0; i < batch.beneficiaries.length; i++) {
            address beneficiary = batch.beneficiaries[i];
            uint256 amount = batch.amounts[i];

            require(beneficiary != address(0), "MultiRoundVestingFactory: zero address");
            require(
                roundAllocations[roundId][beneficiary] + amount <= round.maxAllocation,
                "MultiRoundVestingFactory: exceeds max allocation"
            );

            roundAllocations[roundId][beneficiary] += amount;
            round.participantCount++;

            vault.createVestingSchedule(
                beneficiary,
                batch.start,
                batch.cliffDuration,
                batch.vestingDuration,
                batch.slicePeriod,
                batch.revocable,
                amount
            );
        }

        emit BatchVestingCreated(roundId, batch.beneficiaries.length, totalBatchAmount);
    }

    /**
     * @notice ปิด round
     */
    function closeRound(uint256 roundId) external onlyOwner {
        rounds[roundId].isActive = false;
    }
}
```

---

## Workshop และ Exercises

### Exercise 1: Vesting Dashboard Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title VestingDashboard
 * @notice Contract สำหรับดูภาพรวมของ vesting ทั้งหมด
 * @dev ใช้ในการ track และ display vesting information
 */
contract VestingDashboard {
    struct VestingOverview {
        uint256 totalAllocated;
        uint256 totalVested;
        uint256 totalClaimed;
        uint256 totalLocked;
        uint256 nextUnlockTime;
        uint256 nextUnlockAmount;
    }

    /**
     * @notice คำนวณ cliff ที่เหลืออยู่
     * @param cliffTime Cliff timestamp
     */
    function getCliffCountdown(uint256 cliffTime) external view returns (
        uint256 timeRemaining,
        bool passed
    ) {
        if (block.timestamp >= cliffTime) {
            return (0, true);
        }
        return (cliffTime - block.timestamp, false);
    }

    /**
     * @notice คำนวณ APR ของ vesting (based on token price appreciation)
     * @param vestedAmount จำนวน token ที่ vested
     * @param totalAmount จำนวน token ทั้งหมด
     * @param timeElapsed เวลาที่ผ่านไป (seconds)
     * @param totalDuration เวลา vesting ทั้งหมด (seconds)
     */
    function getVestingProgress(
        uint256 vestedAmount,
        uint256 totalAmount,
        uint256 timeElapsed,
        uint256 totalDuration
    ) external pure returns (
        uint256 percentVested,        // basis points (100 = 1%)
        uint256 timePercentElapsed,   // basis points
        bool isAhead                  // vesting ahead of schedule
    ) {
        if (totalAmount == 0) return (0, 0, false);

        percentVested = vestedAmount * 10000 / totalAmount;
        timePercentElapsed = totalDuration > 0
            ? timeElapsed * 10000 / totalDuration
            : 10000;

        isAhead = percentVested > timePercentElapsed;
    }
}
```

### Exercise 2: Merkle Tree Builder Simulation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title MerkleProofVerifier
 * @notice ทดสอบ Merkle proof verification
 */
contract MerkleProofVerifier {
    /**
     * @notice สร้าง leaf hash สำหรับ airdrop claim
     */
    function getLeafHash(
        uint256 index,
        address account,
        uint256 amount
    ) external pure returns (bytes32) {
        return keccak256(bytes.concat(keccak256(abi.encode(index, account, amount))));
    }

    /**
     * @notice Verify proof manually (step by step)
     */
    function verifyProofStepByStep(
        bytes32[] calldata proof,
        bytes32 root,
        bytes32 leaf
    ) external pure returns (bool, bytes32[] memory intermediateHashes) {
        intermediateHashes = new bytes32[](proof.length + 1);
        intermediateHashes[0] = leaf;

        bytes32 computedHash = leaf;
        for (uint256 i = 0; i < proof.length; i++) {
            bytes32 proofElement = proof[i];
            if (computedHash <= proofElement) {
                computedHash = keccak256(abi.encodePacked(computedHash, proofElement));
            } else {
                computedHash = keccak256(abi.encodePacked(proofElement, computedHash));
            }
            intermediateHashes[i + 1] = computedHash;
        }

        return (computedHash == root, intermediateHashes);
    }
}
```

### Exercise 3: Stream Rate Calculator

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title StreamCalculator
 * @notice คำนวณ streaming payment parameters
 */
contract StreamCalculator {
    uint256 constant SECONDS_PER_DAY = 86400;
    uint256 constant SECONDS_PER_MONTH = 30 days;
    uint256 constant SECONDS_PER_YEAR = 365 days;

    /**
     * @notice คำนวณ deposit ที่ต้องใช้สำหรับ streaming payment
     * @param ratePerDay จำนวน token ต่อวัน
     * @param durationDays จำนวนวัน
     * @return deposit จำนวน token ทั้งหมด (ต้องหารลงตัวด้วย duration)
     * @return ratePerSecond อัตราต่อวินาที
     * @return actualDeposit deposit จริงที่หารลงตัว
     */
    function calcStreamParams(
        uint256 ratePerDay,
        uint256 durationDays
    ) external pure returns (
        uint256 deposit,
        uint256 ratePerSecond,
        uint256 actualDeposit,
        uint256 durationSeconds
    ) {
        durationSeconds = durationDays * SECONDS_PER_DAY;
        deposit = ratePerDay * durationDays;
        ratePerSecond = ratePerDay / SECONDS_PER_DAY;

        // ปรับ deposit ให้หารลงตัว
        actualDeposit = ratePerSecond * durationSeconds;
    }

    /**
     * @notice คำนวณ total streamed ณ เวลาหนึ่ง
     */
    function calcStreamedAmount(
        uint256 ratePerSecond,
        uint256 startTime,
        uint256 stopTime,
        uint256 queryTime
    ) external pure returns (uint256) {
        if (queryTime <= startTime) return 0;
        uint256 elapsed = queryTime < stopTime
            ? queryTime - startTime
            : stopTime - startTime;
        return ratePerSecond * elapsed;
    }
}
```

---

## สรุป Part 57

- **VestingSchedule**: โครงสร้างสำคัญที่มี cliff (ระยะเวลาก่อน unlock ครั้งแรก), duration (เวลา vesting ทั้งหมด), slicePeriodSeconds (ความถี่ release), และ revocable flag
- **VestingVault**: ระบบ vesting ครบวงจร รองรับ `createVestingSchedule`, `release` (withdraw by beneficiary), `revoke` (cancel by owner), และ `computeReleasableAmount` ที่คำนวณ linear vesting พร้อม slice rounding
- **MerkleDistributor**: Gas-efficient airdrop ที่ใช้ Merkle proof แทนการ loop ส่ง token - ใช้ bitmap ประหยัด storage 256x เทียบกับ mapping แบบปกติ
- **StreamingPayment**: Sablier-style continuous stream ที่จ่าย token ต่อวินาที รองรับ withdraw บางส่วน และ cancel stream ได้ทุกเวลา
- **AllocationManager**: Pattern สำหรับจัดสรร token ตาม tokenomics ด้วย categories (team 15%, seed 10%, community 30%, treasury 20%, etc.) พร้อม vesting ที่เหมาะสมแต่ละหมวด

## Next: Part 58 - Advanced NFT Patterns
