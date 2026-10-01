# Part 68: Writing & Proposing EIPs (Ethereum Improvement Proposals)

## บทนำ

**EIP (Ethereum Improvement Proposal)** คือเอกสารออกแบบที่ให้ข้อมูลแก่ชุมชน Ethereum หรืออธิบายฟีเจอร์ใหม่สำหรับ Ethereum หรือกระบวนการของมัน EIP เป็นกลไกหลักในการเสนอและประสานงานการเปลี่ยนแปลงโปรโตคอล Ethereum

ในบทนี้คุณจะได้เรียนรู้:
- ประเภทของ EIP และความแตกต่าง
- โครงสร้าง Template ที่ต้องใช้
- กรณีศึกษา: เขียน Mini EIP สำหรับ Streaming Token Standard
- วงจรชีวิตของ EIP
- เทคนิคการสื่อสารและการโน้มน้าวชุมชน

---

## 1. ประเภทของ EIP

EIP แบ่งออกเป็น 3 ประเภทหลัก:

### 1.1 Standards Track EIP

เป็น EIP ที่มีผลต่อการใช้งาน Ethereum ส่วนใหญ่หรือทั้งหมด แบ่งย่อยเป็น:

#### Core
- การเปลี่ยนแปลงที่ต้องการ consensus fork เช่น EIP-1559 (gas fee mechanism)
- การเปลี่ยนแปลง EVM opcode
- ตัวอย่าง: EIP-3074 (AUTH/AUTHCALL), EIP-4844 (Proto-Danksharding)

#### Networking
- การเปลี่ยนแปลง devp2p และ subprotocol เช่น wire protocol
- ตัวอย่าง: EIP-8 (devp2p forward compatibility)

#### Interface
- การปรับปรุง client API / RPC specification
- ตัวอย่าง: EIP-6963 (Multi Injected Provider Discovery)

#### ERC (Ethereum Request for Comment)
- มาตรฐานระดับ application เช่น token standards
- ตัวอย่าง: ERC-20, ERC-721, ERC-1155, ERC-4626

### 1.2 Meta EIP
- อธิบายกระบวนการ Ethereum หรือเสนอการเปลี่ยนแปลงกระบวนการ
- ตัวอย่าง: EIP-1 (EIP Purpose and Guidelines)

### 1.3 Informational EIP
- อธิบายปัญหาการออกแบบ Ethereum หรือให้แนวทางทั่วไป
- ไม่เสนอฟีเจอร์ใหม่และไม่ต้องการ consensus
- ตัวอย่าง: EIP-2364 (eth/64: forkid-extended protocol handshake)

---

## 2. EIP Template: โครงสร้างมาตรฐาน

```markdown
---
eip: XXXX
title: [ชื่อ EIP]
description: [คำอธิบายสั้นๆ หนึ่งประโยค]
author: [ชื่อ (@GitHub handle) <email>]
discussions-to: https://ethereum-magicians.org/t/...
status: Draft
type: Standards Track
category: ERC
created: YYYY-MM-DD
requires: [EIP numbers ที่ต้องพึ่งพา, ถ้ามี]
---

## Abstract
[สรุปสั้นๆ เกี่ยวกับ EIP นี้ ไม่เกิน 200 คำ]

## Motivation
[อธิบายว่าทำไมถึงต้องการ EIP นี้ ปัญหาที่แก้ไขคืออะไร]

## Specification
[คำอธิบายทางเทคนิคโดยละเอียด ใช้ keyword SHALL/MUST/SHOULD/MAY ตาม RFC 2119]

## Rationale
[อธิบายว่าทำไมถึงเลือกการออกแบบนี้ ทางเลือกที่พิจารณาแล้วปฏิเสธคืออะไร]

## Backwards Compatibility
[อธิบาย backward compatibility issues ถ้า breaking changes ต้องอธิบายให้ชัดเจน]

## Test Cases
[Test cases สำหรับ implementation ถ้าไม่จำเป็นต้องใส่ N/A]

## Reference Implementation
[Optional: implementation ตัวอย่างเพื่อช่วยให้เข้าใจ specification]

## Security Considerations
[วิเคราะห์ความปลอดภัย attacks ที่เป็นไปได้ และ mitigations]

## Copyright
Copyright and related rights waived via [CC0](../LICENSE.md).
```

---

## 3. กรณีศึกษา: ERC-XXXX Streaming Token Standard

### 3.1 แนวคิดและ Motivation

**Streaming Payments** คือแนวคิดที่เงินไหลต่อเนื่องแบบ real-time แทนที่จะจ่ายเป็นก้อน ใช้ใน:
- เงินเดือนแบบ real-time (pay per second)
- Subscription services
- Yield streaming จาก DeFi protocols

ปัญหาปัจจุบัน: ยังไม่มีมาตรฐาน interface ที่ชุมชนยอมรับ ทำให้ front-end และ integration ทำได้ยาก

### 3.2 Interface Definition

```solidity
// SPDX-License-Identifier: CC0-1.0
pragma solidity ^0.8.24;

/**
 * @title IStreamToken
 * @notice ERC-XXXX: Streaming Token Standard Interface
 * @dev Interface สำหรับ token ที่รองรับการส่งเงินแบบ streaming
 *      โดยเงินจะไหลต่อเนื่องต่อ second จาก sender ไปยัง recipient
 */
interface IStreamToken {
    // ============================================================
    //                          STRUCTS
    // ============================================================

    /**
     * @notice ข้อมูลของ stream แต่ละอัน
     * @param sender ผู้ส่ง token
     * @param recipient ผู้รับ token
     * @param ratePerSecond อัตราการไหลต่อวินาที (wei per second)
     * @param startTime เวลาเริ่มต้น stream (Unix timestamp)
     * @param stopTime เวลาสิ้นสุด stream (Unix timestamp)
     * @param deposit จำนวน token ที่ deposit ไว้ใน stream
     * @param remainingBalance จำนวน token ที่เหลือใน stream
     * @param isActive สถานะของ stream
     */
    struct Stream {
        address sender;
        address recipient;
        uint256 ratePerSecond;
        uint256 startTime;
        uint256 stopTime;
        uint256 deposit;
        uint256 remainingBalance;
        bool isActive;
    }

    // ============================================================
    //                          EVENTS
    // ============================================================

    /**
     * @notice Event เมื่อ stream ถูกสร้าง
     * @param streamId ID ของ stream
     * @param sender ผู้ส่ง
     * @param recipient ผู้รับ
     * @param deposit จำนวน token ที่ deposit
     * @param tokenAddress ที่อยู่ของ token contract
     * @param startTime เวลาเริ่ม
     * @param stopTime เวลาสิ้นสุด
     */
    event StreamCreated(
        uint256 indexed streamId,
        address indexed sender,
        address indexed recipient,
        uint256 deposit,
        address tokenAddress,
        uint256 startTime,
        uint256 stopTime
    );

    /**
     * @notice Event เมื่อ stream ถูกยกเลิก
     * @param streamId ID ของ stream
     * @param sender ผู้ส่ง
     * @param recipient ผู้รับ
     * @param senderBalance จำนวน token ที่คืนให้ sender
     * @param recipientBalance จำนวน token ที่ recipient ได้รับ
     */
    event StreamCancelled(
        uint256 indexed streamId,
        address indexed sender,
        address indexed recipient,
        uint256 senderBalance,
        uint256 recipientBalance
    );

    /**
     * @notice Event เมื่อมีการถอน token จาก stream
     * @param streamId ID ของ stream
     * @param recipient ผู้รับ
     * @param amount จำนวน token ที่ถอน
     */
    event WithdrawFromStream(
        uint256 indexed streamId,
        address indexed recipient,
        uint256 amount
    );

    // ============================================================
    //                          ERRORS
    // ============================================================

    /// @notice Stream ไม่พบด้วย ID ที่ให้มา
    error StreamNotFound(uint256 streamId);

    /// @notice Stream ได้ถูกยกเลิกหรือสิ้นสุดแล้ว
    error StreamNotActive(uint256 streamId);

    /// @notice ผู้เรียกไม่มีสิทธิ์ดำเนินการนี้
    error Unauthorized(address caller, uint256 streamId);

    /// @notice จำนวน deposit ไม่เพียงพอสำหรับ duration ที่กำหนด
    error InsufficientDeposit(uint256 required, uint256 provided);

    /// @notice ช่วงเวลาไม่ถูกต้อง
    error InvalidTimeRange(uint256 startTime, uint256 stopTime);

    /// @notice ที่อยู่ไม่ถูกต้อง (zero address)
    error InvalidAddress(address addr);

    /// @notice Rate per second เป็น 0
    error ZeroRate();

    /// @notice จำนวนที่ถอนเกินกว่าที่มี
    error WithdrawExceedsBalance(uint256 requested, uint256 available);

    // ============================================================
    //                     WRITE FUNCTIONS
    // ============================================================

    /**
     * @notice สร้าง stream ใหม่
     * @dev MUST emit StreamCreated event
     *      MUST revert ถ้า recipient เป็น zero address
     *      MUST revert ถ้า deposit ไม่พอสำหรับอย่างน้อย 1 second
     *      MUST revert ถ้า startTime >= stopTime
     *      SHOULD allow startTime ในอนาคต
     * @param recipient ผู้รับ token
     * @param deposit จำนวน token ทั้งหมดที่จะส่ง
     * @param tokenAddress ที่อยู่ของ ERC-20 token
     * @param startTime เวลาเริ่ม stream (Unix timestamp)
     * @param stopTime เวลาสิ้นสุด stream (Unix timestamp)
     * @return streamId ID ของ stream ที่สร้างขึ้น
     */
    function createStream(
        address recipient,
        uint256 deposit,
        address tokenAddress,
        uint256 startTime,
        uint256 stopTime
    ) external returns (uint256 streamId);

    /**
     * @notice ยกเลิก stream
     * @dev MUST emit StreamCancelled event
     *      MUST transfer token ที่เหลือคืนให้ sender
     *      MUST transfer token ที่ earned แล้วให้ recipient
     *      MUST revert ถ้า caller ไม่ใช่ sender หรือ recipient
     *      MUST revert ถ้า stream ไม่ active
     * @param streamId ID ของ stream ที่จะยกเลิก
     */
    function cancelStream(uint256 streamId) external;

    /**
     * @notice ถอน token ที่ earned แล้วจาก stream
     * @dev MUST emit WithdrawFromStream event
     *      MUST revert ถ้า amount > balance ที่ earned
     *      MUST revert ถ้า caller ไม่ใช่ recipient (หรือ approved)
     * @param streamId ID ของ stream
     * @param amount จำนวน token ที่จะถอน
     */
    function withdrawFromStream(uint256 streamId, uint256 amount) external;

    // ============================================================
    //                      READ FUNCTIONS
    // ============================================================

    /**
     * @notice ดึงข้อมูล stream
     * @param streamId ID ของ stream
     * @return stream ข้อมูลของ stream
     */
    function getStream(uint256 streamId) external view returns (Stream memory stream);

    /**
     * @notice คำนวณ balance ที่ recipient สามารถถอนได้ตอนนี้
     * @param streamId ID ของ stream
     * @return balance จำนวน token ที่ถอนได้
     */
    function balanceOf(uint256 streamId) external view returns (uint256 balance);

    /**
     * @notice จำนวน stream ทั้งหมดที่สร้าง
     * @return count จำนวน stream
     */
    function nextStreamId() external view returns (uint256 count);
}
```

### 3.3 Reference Implementation: StreamToken.sol

```solidity
// SPDX-License-Identifier: CC0-1.0
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import {IStreamToken} from "./IStreamToken.sol";

/**
 * @title StreamToken
 * @notice ERC-XXXX Reference Implementation: Streaming Token Standard
 * @dev Implementation นี้เป็น reference implementation สำหรับ ERC-XXXX
 *      ไม่แนะนำให้ใช้ใน production โดยไม่ผ่าน audit
 *
 * การออกแบบ:
 * - ใช้ pull payment pattern (recipient ต้อง call withdraw เอง)
 * - รองรับ ERC-20 token ใดๆ
 * - Stream สามารถ cancel ได้โดย sender หรือ recipient
 * - ป้องกัน reentrancy ด้วย ReentrancyGuard
 *
 * Invariants:
 * - stream.deposit == sum(withdrawn) + remainingBalance ตลอดเวลา
 * - balanceOf(streamId) <= remainingBalance เสมอ
 */
contract StreamToken is IStreamToken, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ============================================================
    //                         STORAGE
    // ============================================================

    /// @notice counter สำหรับ stream ID (เริ่มที่ 1)
    uint256 private _nextStreamId;

    /// @notice mapping จาก streamId ไปยัง Stream struct
    mapping(uint256 => Stream) private _streams;

    /// @notice mapping จาก streamId ไปยัง token address
    mapping(uint256 => address) private _tokenAddresses;

    /// @notice จำนวนที่ recipient ถอนไปแล้วสะสม (ป้องกัน double-withdrawal)
    mapping(uint256 => uint256) private _withdrawnAmounts;

    // ============================================================
    //                       CONSTRUCTOR
    // ============================================================

    constructor() {
        // Stream IDs เริ่มจาก 1 เพื่อให้ 0 เป็น sentinel value
        _nextStreamId = 1;
    }

    // ============================================================
    //                     WRITE FUNCTIONS
    // ============================================================

    /**
     * @inheritdoc IStreamToken
     */
    function createStream(
        address recipient,
        uint256 deposit,
        address tokenAddress,
        uint256 startTime,
        uint256 stopTime
    ) external override nonReentrant returns (uint256 streamId) {
        // --- Validation ---
        if (recipient == address(0)) revert InvalidAddress(recipient);
        if (recipient == msg.sender) revert InvalidAddress(recipient);
        if (tokenAddress == address(0)) revert InvalidAddress(tokenAddress);

        if (startTime < block.timestamp) {
            startTime = block.timestamp;
        }
        if (stopTime <= startTime) revert InvalidTimeRange(startTime, stopTime);

        uint256 duration = stopTime - startTime;

        // ต้องมี deposit อย่างน้อยพอสำหรับ 1 second
        if (deposit < duration) revert InsufficientDeposit(duration, deposit);

        // คำนวณ rate โดยให้ remainder ไปเก็บใน contract สำหรับ rounding
        uint256 ratePerSecond = deposit / duration;
        if (ratePerSecond == 0) revert ZeroRate();

        // deposit ที่แท้จริง = ratePerSecond * duration (ตัด remainder ออก)
        uint256 actualDeposit = ratePerSecond * duration;

        // --- Create Stream ---
        streamId = _nextStreamId++;

        _streams[streamId] = Stream({
            sender: msg.sender,
            recipient: recipient,
            ratePerSecond: ratePerSecond,
            startTime: startTime,
            stopTime: stopTime,
            deposit: actualDeposit,
            remainingBalance: actualDeposit,
            isActive: true
        });

        _tokenAddresses[streamId] = tokenAddress;
        _withdrawnAmounts[streamId] = 0;

        // --- Transfer Token ---
        // ต้อง approve ก่อน call createStream
        IERC20(tokenAddress).safeTransferFrom(msg.sender, address(this), actualDeposit);

        emit StreamCreated(
            streamId,
            msg.sender,
            recipient,
            actualDeposit,
            tokenAddress,
            startTime,
            stopTime
        );
    }

    /**
     * @inheritdoc IStreamToken
     */
    function cancelStream(uint256 streamId) external override nonReentrant {
        Stream storage stream = _streams[streamId];

        // Validation
        if (stream.sender == address(0)) revert StreamNotFound(streamId);
        if (!stream.isActive) revert StreamNotActive(streamId);
        if (msg.sender != stream.sender && msg.sender != stream.recipient) {
            revert Unauthorized(msg.sender, streamId);
        }

        // คำนวณจำนวนที่แต่ละฝ่ายได้รับ
        uint256 recipientBalance = _calculateBalance(streamId);
        uint256 senderBalance = stream.remainingBalance - recipientBalance;

        // อัพเดท state ก่อน transfer (CEI pattern)
        stream.isActive = false;
        stream.remainingBalance = 0;

        address tokenAddress = _tokenAddresses[streamId];

        // Transfer ให้ recipient (ถ้ามี)
        if (recipientBalance > 0) {
            IERC20(tokenAddress).safeTransfer(stream.recipient, recipientBalance);
        }

        // Transfer ที่เหลือคืน sender
        if (senderBalance > 0) {
            IERC20(tokenAddress).safeTransfer(stream.sender, senderBalance);
        }

        emit StreamCancelled(
            streamId,
            stream.sender,
            stream.recipient,
            senderBalance,
            recipientBalance
        );
    }

    /**
     * @inheritdoc IStreamToken
     */
    function withdrawFromStream(
        uint256 streamId,
        uint256 amount
    ) external override nonReentrant {
        Stream storage stream = _streams[streamId];

        // Validation
        if (stream.sender == address(0)) revert StreamNotFound(streamId);
        if (!stream.isActive) revert StreamNotActive(streamId);
        if (msg.sender != stream.recipient) {
            revert Unauthorized(msg.sender, streamId);
        }

        uint256 available = _calculateBalance(streamId);
        if (amount > available) revert WithdrawExceedsBalance(amount, available);

        // อัพเดท state ก่อน transfer
        stream.remainingBalance -= amount;
        _withdrawnAmounts[streamId] += amount;

        // Transfer
        IERC20(_tokenAddresses[streamId]).safeTransfer(stream.recipient, amount);

        emit WithdrawFromStream(streamId, stream.recipient, amount);
    }

    // ============================================================
    //                      READ FUNCTIONS
    // ============================================================

    /**
     * @inheritdoc IStreamToken
     */
    function getStream(uint256 streamId) external view override returns (Stream memory) {
        if (_streams[streamId].sender == address(0)) revert StreamNotFound(streamId);
        return _streams[streamId];
    }

    /**
     * @inheritdoc IStreamToken
     */
    function balanceOf(uint256 streamId) external view override returns (uint256) {
        if (_streams[streamId].sender == address(0)) revert StreamNotFound(streamId);
        return _calculateBalance(streamId);
    }

    /**
     * @inheritdoc IStreamToken
     */
    function nextStreamId() external view override returns (uint256) {
        return _nextStreamId;
    }

    /**
     * @notice ดึง token address ของ stream
     * @param streamId ID ของ stream
     * @return address ของ ERC-20 token
     */
    function getTokenAddress(uint256 streamId) external view returns (address) {
        if (_streams[streamId].sender == address(0)) revert StreamNotFound(streamId);
        return _tokenAddresses[streamId];
    }

    // ============================================================
    //                     INTERNAL FUNCTIONS
    // ============================================================

    /**
     * @notice คำนวณ balance ที่ recipient สามารถถอนได้ตอนนี้
     * @dev คำนวณจาก elapsed time และ ratePerSecond ลบด้วยที่ถอนไปแล้ว
     * @param streamId ID ของ stream
     * @return balance ที่ถอนได้
     */
    function _calculateBalance(uint256 streamId) internal view returns (uint256 balance) {
        Stream storage stream = _streams[streamId];

        if (!stream.isActive) return 0;

        uint256 currentTime = block.timestamp;

        // ถ้ายังไม่ถึงเวลาเริ่ม
        if (currentTime <= stream.startTime) return 0;

        // คำนวณเวลาที่ผ่านไป (capped at stopTime)
        uint256 elapsedTime;
        if (currentTime >= stream.stopTime) {
            elapsedTime = stream.stopTime - stream.startTime;
        } else {
            elapsedTime = currentTime - stream.startTime;
        }

        // token ที่ earn ไปแล้วทั้งหมด
        uint256 totalEarned = elapsedTime * stream.ratePerSecond;

        // ลบส่วนที่ถอนไปแล้ว
        uint256 withdrawn = _withdrawnAmounts[streamId];

        // ป้องกัน underflow (ไม่ควรเกิด แต่ป้องกันไว้)
        if (withdrawn >= totalEarned) return 0;

        balance = totalEarned - withdrawn;

        // ไม่เกิน remainingBalance
        if (balance > stream.remainingBalance) {
            balance = stream.remainingBalance;
        }
    }
}
```

### 3.4 MockERC20 สำหรับ Testing

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

/**
 * @title MockERC20
 * @notice Token ปลอมสำหรับ testing เท่านั้น
 */
contract MockERC20 is ERC20 {
    constructor(string memory name, string memory symbol) ERC20(name, symbol) {
        // mint ให้ deployer ทันที
        _mint(msg.sender, 1_000_000 ether);
    }

    /// @notice ให้ใครก็ได้ mint สำหรับ testing
    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
}
```

### 3.5 Test Cases ใน Solidity

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test, console} from "forge-std/Test.sol";
import {StreamToken} from "../src/StreamToken.sol";
import {MockERC20} from "../src/mocks/MockERC20.sol";
import {IStreamToken} from "../src/interfaces/IStreamToken.sol";

/**
 * @title StreamTokenTest
 * @notice Test suite สำหรับ ERC-XXXX StreamToken Reference Implementation
 *
 * Test categories:
 * 1. createStream - การสร้าง stream
 * 2. cancelStream - การยกเลิก stream
 * 3. withdrawFromStream - การถอน token
 * 4. balanceOf - การคำนวณ balance
 * 5. Edge cases และ security
 */
contract StreamTokenTest is Test {
    StreamToken public streamToken;
    MockERC20 public token;

    address public alice = makeAddr("alice");   // sender
    address public bob = makeAddr("bob");       // recipient
    address public carol = makeAddr("carol");   // third party

    uint256 public constant INITIAL_BALANCE = 10_000 ether;
    uint256 public constant STREAM_DEPOSIT = 3_600 ether;  // 1 hour at 1 token/sec
    uint256 public constant RATE_PER_SECOND = 1 ether;

    // เวลาพื้นฐาน
    uint256 public startTime;
    uint256 public stopTime;

    function setUp() public {
        streamToken = new StreamToken();
        token = new MockERC20("Test Token", "TEST");

        // mint และ approve
        token.mint(alice, INITIAL_BALANCE);
        token.mint(bob, INITIAL_BALANCE);

        vm.prank(alice);
        token.approve(address(streamToken), type(uint256).max);

        startTime = block.timestamp + 100;   // เริ่มในอีก 100 วินาที
        stopTime = startTime + 3_600;        // ระยะเวลา 1 ชั่วโมง
    }

    // ============================================================
    //               TEST: createStream
    // ============================================================

    function test_createStream_basic() public {
        vm.prank(alice);
        uint256 streamId = streamToken.createStream(
            bob,
            STREAM_DEPOSIT,
            address(token),
            startTime,
            stopTime
        );

        assertEq(streamId, 1, "First stream ID should be 1");

        IStreamToken.Stream memory stream = streamToken.getStream(streamId);
        assertEq(stream.sender, alice);
        assertEq(stream.recipient, bob);
        assertEq(stream.startTime, startTime);
        assertEq(stream.stopTime, stopTime);
        assertEq(stream.isActive, true);
    }

    function test_createStream_transfersTokens() public {
        uint256 aliceBalanceBefore = token.balanceOf(alice);
        uint256 contractBalanceBefore = token.balanceOf(address(streamToken));

        vm.prank(alice);
        streamToken.createStream(bob, STREAM_DEPOSIT, address(token), startTime, stopTime);

        assertEq(token.balanceOf(alice), aliceBalanceBefore - STREAM_DEPOSIT);
        assertEq(
            token.balanceOf(address(streamToken)),
            contractBalanceBefore + STREAM_DEPOSIT
        );
    }

    function test_createStream_emitsEvent() public {
        vm.expectEmit(true, true, true, true);
        emit IStreamToken.StreamCreated(
            1,
            alice,
            bob,
            STREAM_DEPOSIT,
            address(token),
            startTime,
            stopTime
        );

        vm.prank(alice);
        streamToken.createStream(bob, STREAM_DEPOSIT, address(token), startTime, stopTime);
    }

    function test_createStream_revert_zeroRecipient() public {
        vm.prank(alice);
        vm.expectRevert(
            abi.encodeWithSelector(IStreamToken.InvalidAddress.selector, address(0))
        );
        streamToken.createStream(address(0), STREAM_DEPOSIT, address(token), startTime, stopTime);
    }

    function test_createStream_revert_selfStream() public {
        vm.prank(alice);
        vm.expectRevert(
            abi.encodeWithSelector(IStreamToken.InvalidAddress.selector, alice)
        );
        streamToken.createStream(alice, STREAM_DEPOSIT, address(token), startTime, stopTime);
    }

    function test_createStream_revert_invalidTimeRange() public {
        vm.prank(alice);
        vm.expectRevert(
            abi.encodeWithSelector(IStreamToken.InvalidTimeRange.selector, startTime, startTime)
        );
        streamToken.createStream(bob, STREAM_DEPOSIT, address(token), startTime, startTime);
    }

    function test_createStream_revert_insufficientDeposit() public {
        uint256 duration = stopTime - startTime;
        uint256 tooSmall = duration - 1;  // น้อยกว่า duration

        vm.prank(alice);
        vm.expectRevert(
            abi.encodeWithSelector(IStreamToken.InsufficientDeposit.selector, duration, tooSmall)
        );
        streamToken.createStream(bob, tooSmall, address(token), startTime, stopTime);
    }

    // ============================================================
    //               TEST: balanceOf (คำนวณ balance)
    // ============================================================

    function test_balanceOf_zeroBeforeStart() public {
        vm.prank(alice);
        uint256 streamId = streamToken.createStream(
            bob, STREAM_DEPOSIT, address(token), startTime, stopTime
        );

        // ยังไม่ถึงเวลาเริ่ม
        assertEq(streamToken.balanceOf(streamId), 0);
    }

    function test_balanceOf_afterHalfTime() public {
        vm.prank(alice);
        uint256 streamId = streamToken.createStream(
            bob, STREAM_DEPOSIT, address(token), startTime, stopTime
        );

        // เลื่อนเวลาไปครึ่งทาง
        uint256 halfDuration = (stopTime - startTime) / 2;
        vm.warp(startTime + halfDuration);

        uint256 expectedBalance = halfDuration * RATE_PER_SECOND;
        assertApproxEqAbs(
            streamToken.balanceOf(streamId),
            expectedBalance,
            RATE_PER_SECOND  // tolerance 1 token สำหรับ rounding
        );
    }

    function test_balanceOf_fullAfterStopTime() public {
        vm.prank(alice);
        uint256 streamId = streamToken.createStream(
            bob, STREAM_DEPOSIT, address(token), startTime, stopTime
        );

        // เลื่อนเวลาหลัง stopTime
        vm.warp(stopTime + 1);

        IStreamToken.Stream memory stream = streamToken.getStream(streamId);
        assertEq(streamToken.balanceOf(streamId), stream.deposit);
    }

    // ============================================================
    //               TEST: withdrawFromStream
    // ============================================================

    function test_withdraw_success() public {
        vm.prank(alice);
        uint256 streamId = streamToken.createStream(
            bob, STREAM_DEPOSIT, address(token), startTime, stopTime
        );

        // เลื่อนเวลาไป 1800 วินาที (ครึ่งทาง)
        vm.warp(startTime + 1800);

        uint256 available = streamToken.balanceOf(streamId);
        uint256 bobBalanceBefore = token.balanceOf(bob);

        vm.prank(bob);
        streamToken.withdrawFromStream(streamId, available);

        assertEq(token.balanceOf(bob), bobBalanceBefore + available);
    }

    function test_withdraw_revert_exceedsBalance() public {
        vm.prank(alice);
        uint256 streamId = streamToken.createStream(
            bob, STREAM_DEPOSIT, address(token), startTime, stopTime
        );

        vm.warp(startTime + 1800);
        uint256 available = streamToken.balanceOf(streamId);

        vm.prank(bob);
        vm.expectRevert(
            abi.encodeWithSelector(
                IStreamToken.WithdrawExceedsBalance.selector,
                available + 1,
                available
            )
        );
        streamToken.withdrawFromStream(streamId, available + 1);
    }

    function test_withdraw_revert_unauthorized() public {
        vm.prank(alice);
        uint256 streamId = streamToken.createStream(
            bob, STREAM_DEPOSIT, address(token), startTime, stopTime
        );

        vm.warp(startTime + 1800);

        vm.prank(carol);
        vm.expectRevert(
            abi.encodeWithSelector(IStreamToken.Unauthorized.selector, carol, streamId)
        );
        streamToken.withdrawFromStream(streamId, 1);
    }

    // ============================================================
    //               TEST: cancelStream
    // ============================================================

    function test_cancelStream_bySender() public {
        vm.prank(alice);
        uint256 streamId = streamToken.createStream(
            bob, STREAM_DEPOSIT, address(token), startTime, stopTime
        );

        // เลื่อนเวลาไป 1800 วินาที
        vm.warp(startTime + 1800);

        uint256 aliceBalanceBefore = token.balanceOf(alice);
        uint256 bobBalanceBefore = token.balanceOf(bob);

        vm.prank(alice);
        streamToken.cancelStream(streamId);

        IStreamToken.Stream memory stream = streamToken.getStream(streamId);
        assertEq(stream.isActive, false);

        // bob ได้รับ token ที่ earn แล้ว
        assertGt(token.balanceOf(bob), bobBalanceBefore);
        // alice ได้รับ token ที่เหลือคืน
        assertGt(token.balanceOf(alice), aliceBalanceBefore);
    }

    function test_cancelStream_byRecipient() public {
        vm.prank(alice);
        uint256 streamId = streamToken.createStream(
            bob, STREAM_DEPOSIT, address(token), startTime, stopTime
        );

        vm.warp(startTime + 1800);

        vm.prank(bob);
        streamToken.cancelStream(streamId);  // bob ยกเลิก stream ได้เช่นกัน

        IStreamToken.Stream memory stream = streamToken.getStream(streamId);
        assertEq(stream.isActive, false);
    }

    function test_cancelStream_revert_unauthorized() public {
        vm.prank(alice);
        uint256 streamId = streamToken.createStream(
            bob, STREAM_DEPOSIT, address(token), startTime, stopTime
        );

        vm.prank(carol);
        vm.expectRevert(
            abi.encodeWithSelector(IStreamToken.Unauthorized.selector, carol, streamId)
        );
        streamToken.cancelStream(streamId);
    }

    function test_cancelStream_revert_alreadyCancelled() public {
        vm.prank(alice);
        uint256 streamId = streamToken.createStream(
            bob, STREAM_DEPOSIT, address(token), startTime, stopTime
        );

        vm.prank(alice);
        streamToken.cancelStream(streamId);

        vm.prank(alice);
        vm.expectRevert(
            abi.encodeWithSelector(IStreamToken.StreamNotActive.selector, streamId)
        );
        streamToken.cancelStream(streamId);  // cancel ซ้ำ
    }

    // ============================================================
    //               TEST: Fuzz Testing
    // ============================================================

    /**
     * @notice Fuzz test สำหรับ createStream
     * @dev ทดสอบด้วย input หลากหลาย
     */
    function testFuzz_createStream_balanceConsistency(
        uint256 depositAmount,
        uint256 durationSeconds
    ) public {
        // Bound inputs ให้อยู่ในช่วงที่สมเหตุสมผล
        depositAmount = bound(depositAmount, 1000, 1_000_000 ether);
        durationSeconds = bound(durationSeconds, 100, 365 days);

        uint256 start = block.timestamp + 1;
        uint256 stop = start + durationSeconds;

        token.mint(alice, depositAmount);

        vm.prank(alice);
        uint256 streamId = streamToken.createStream(
            bob, depositAmount, address(token), start, stop
        );

        // หลังจาก stream จบ balance ต้องเท่ากับ deposit
        vm.warp(stop + 1);

        IStreamToken.Stream memory stream = streamToken.getStream(streamId);
        assertEq(streamToken.balanceOf(streamId), stream.deposit);
    }

    // ============================================================
    //               TEST: Security - Reentrancy
    // ============================================================

    /**
     * @notice Test ป้องกัน reentrancy attack
     */
    function test_noReentrancy() public {
        // สร้าง attacker contract
        ReentrancyAttacker attacker = new ReentrancyAttacker(streamToken, token);
        token.mint(address(attacker), INITIAL_BALANCE);

        vm.prank(address(attacker));
        token.approve(address(streamToken), type(uint256).max);

        // ลอง attack
        vm.expectRevert();  // ควร revert
        attacker.attack(bob, STREAM_DEPOSIT, address(token), startTime, stopTime);
    }
}

/**
 * @notice Contract สำหรับ test reentrancy
 */
contract ReentrancyAttacker {
    StreamToken public target;
    MockERC20 public token;
    uint256 public streamId;

    constructor(StreamToken _target, MockERC20 _token) {
        target = _target;
        token = _token;
    }

    function attack(
        address recipient,
        uint256 deposit,
        address tokenAddr,
        uint256 startTime,
        uint256 stopTime
    ) external {
        streamId = target.createStream(recipient, deposit, tokenAddr, startTime, stopTime);
        // พยายาม cancel ระหว่าง create (reentrancy)
        target.cancelStream(streamId);
    }

    // Fallback ที่พยายาม reenter
    receive() external payable {
        if (streamId > 0) {
            target.cancelStream(streamId);
        }
    }
}
```

---

## 4. วงจรชีวิตของ EIP

```
Draft → Review → Last Call → Final
  ↓         ↓        ↓
Stagnant  Withdrawn  Withdrawn
```

### 4.1 แต่ละ Status

| Status | ความหมาย | ระยะเวลา |
|--------|----------|---------|
| **Idea** | ก่อนส่ง PR | ไม่จำกัด |
| **Draft** | PR merged ใน ethereum/EIPs | ไม่จำกัด |
| **Review** | Author เชิญให้ peer review | ไม่จำกัด |
| **Last Call** | แจ้งให้ community review 14 วัน | 14 วัน |
| **Final** | มาตรฐานถาวร | ถาวร |
| **Stagnant** | ไม่มี activity 6 เดือน | - |
| **Withdrawn** | Author ถอน | - |
| **Living** | สำหรับ EIP ที่อัพเดทต่อเนื่อง (เช่น EIP-1) | ถาวร |

### 4.2 Champion Responsibilities

EIP Champion คือผู้รับผิดชอบหลักใน EIP:

```
Champion Duties:
├── เขียน Draft
├── ตอบคำถามใน ethereum-magicians
├── แก้ไขตาม feedback
├── ติดตาม editor requests
├── ประสานงานกับ implementation teams
└── ขอ move to Last Call เมื่อพร้อม
```

**ตัวอย่าง การตอบ Objections:**

```markdown
## การตอบ Objection: "รูปแบบ streamId ควรใช้ bytes32 แทน uint256"

**Objection:** bytes32 ยืดหยุ่นกว่าและ align กับ ERC-721 tokenId บางรูปแบบ

**Response:** ขอคงไว้เป็น uint256 เพราะ:
1. ERC-20, ERC-721, ERC-1155 ล้วนใช้ uint256 สำหรับ IDs
2. Gas efficient กว่า bytes32 ใน common operations
3. ง่ายต่อ off-chain indexing (integers sort ได้ตามธรรมชาติ)
4. bytes32 เพิ่ม complexity โดยไม่จำเป็น

หากมีกรณีใช้งานที่ต้องการ bytes32 สามารถ wrap ผ่าน adapter contract ได้
```

### 4.3 การเขียน Specification ที่ดี

ใช้ RFC 2119 keywords อย่างสม่ำเสมอ:

```markdown
| Keyword | ความหมาย |
|---------|---------|
| MUST / SHALL | บังคับ ถ้าไม่ทำถือว่า non-conformant |
| MUST NOT / SHALL NOT | ห้ามทำ |
| SHOULD / RECOMMENDED | แนะนำ แต่มีเหตุผลที่จะไม่ทำได้ |
| SHOULD NOT / NOT RECOMMENDED | แนะนำว่าอย่าทำ แต่ยอมรับได้ |
| MAY / OPTIONAL | ทำหรือไม่ทำก็ได้ |
```

**ตัวอย่างการเขียน specification:**

```markdown
## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY",
and "OPTIONAL" in this document are to be interpreted as described
in RFC 2119 and RFC 8174.

### createStream

Implementations MUST revert if:
- `recipient` is the zero address (`0x0000...0000`)
- `recipient` equals `msg.sender`
- `deposit` is zero
- `stopTime <= startTime`
- `deposit < (stopTime - startTime)` (less than 1 token per second)

Implementations SHOULD allow `startTime` to be in the past,
treating it as `block.timestamp` internally.

On success, implementations MUST:
1. Emit a `StreamCreated` event with all parameters
2. Transfer `deposit` tokens from `msg.sender` to the contract
3. Return a unique `streamId` starting from 1
```

---

## 5. Workshop: เขียน EIP ของคุณเอง

### Workshop 5.1: สร้าง EIP สำหรับ Token Vesting Standard

ลองเขียน interface สำหรับ "ERC-YYYY: Token Vesting Standard":

```solidity
// SPDX-License-Identifier: CC0-1.0
pragma solidity ^0.8.24;

/**
 * @title IVesting
 * @notice ERC-YYYY: Token Vesting Standard (Workshop Exercise)
 * @dev นักเรียนต้องทำ:
 *   1. เพิ่ม events: VestingCreated, VestingRevoked, TokensClaimed
 *   2. เพิ่ม errors: NotBeneficiary, AlreadyRevoked, NothingToClaim
 *   3. Implement functions ด้านล่าง
 *   4. เขียน test cases อย่างน้อย 5 tests
 */
interface IVesting {
    struct VestingSchedule {
        address beneficiary;        // ผู้รับ token
        address token;              // token ที่ vest
        uint256 totalAmount;        // จำนวนรวม
        uint256 cliffDuration;      // ระยะเวลา cliff (ถ้าไม่ผ่าน cliff จะไม่ได้อะไร)
        uint256 vestingDuration;    // ระยะเวลา vesting รวม
        uint256 startTime;          // เวลาเริ่มต้น
        uint256 claimed;            // จำนวนที่ claim ไปแล้ว
        bool revocable;             // revoke ได้ไหม
        bool revoked;               // ถูก revoke แล้วไหม
    }

    // TODO: เพิ่ม events และ errors

    /**
     * @notice สร้าง vesting schedule ใหม่
     */
    function createVesting(
        address beneficiary,
        address token,
        uint256 totalAmount,
        uint256 cliffDuration,
        uint256 vestingDuration,
        bool revocable
    ) external returns (uint256 vestingId);

    /**
     * @notice claim token ที่ vested แล้ว
     */
    function claim(uint256 vestingId) external;

    /**
     * @notice revoke vesting schedule (เฉพาะ owner)
     */
    function revoke(uint256 vestingId) external;

    /**
     * @notice คำนวณ vested amount ณ เวลาปัจจุบัน
     */
    function vestedAmount(uint256 vestingId) external view returns (uint256);

    /**
     * @notice คำนวณ releasable amount (vested - claimed)
     */
    function releasableAmount(uint256 vestingId) external view returns (uint256);
}
```

### Workshop 5.2: เขียน Security Considerations

```markdown
## Security Considerations

### Token Transfer Risks
Implementations using `transfer` instead of `safeTransfer` MUST handle
tokens that return `false` on failure (e.g., USDT on some networks).
Reference implementations SHOULD use `SafeERC20.safeTransfer`.

### Integer Overflow
All arithmetic MUST use Solidity 0.8.x built-in overflow protection
or equivalent. `ratePerSecond * elapsedTime` may overflow for tokens
with 18 decimals and long durations. Implementers SHOULD verify:
`ratePerSecond <= type(uint256).max / maxDuration`

### Reentrancy
Implementations MUST follow the Checks-Effects-Interactions (CEI) pattern:
1. Validate inputs (Checks)
2. Update state (Effects)
3. External calls (Interactions)

Alternatively, use a reentrancy guard such as OpenZeppelin's `ReentrancyGuard`.

### Front-running
Stream creation can be front-run. Applications SHOULD implement
slippage protection or use commit-reveal schemes for sensitive streams.

### Denial of Service
An attacker controlling the recipient address could deploy a contract
with a failing `receive()` to block `cancelStream`. Implementations
MUST handle this by allowing `transfer` failures to be handled gracefully,
e.g., by storing unclaimed amounts for later withdrawal.
```

---

## สรุป Part 68

- **EIP แบ่งเป็น 3 ประเภท**: Standards Track (Core/Networking/Interface/ERC), Meta, และ Informational
- **Template มาตรฐาน** ต้องมี: Abstract, Motivation, Specification, Rationale, Backwards Compatibility, Test Cases, Reference Implementation, Security Considerations
- **ERC-XXXX Streaming Token** เป็นตัวอย่าง EIP ที่สมบูรณ์ มี interface, reference implementation, และ test cases
- **วงจรชีวิต EIP**: Draft → Review → Last Call (14 วัน) → Final
- **Champion** ต้องตอบ objections, อัพเดท spec, และประสานงานกับ implementers
- **ใช้ RFC 2119** (MUST/SHOULD/MAY) เพื่อให้ specification ชัดเจนและไม่คลุมเครือ

## Next: Part 69 - Protocol Monitoring & Alerting
