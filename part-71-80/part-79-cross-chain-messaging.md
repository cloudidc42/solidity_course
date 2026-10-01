# Part 79: Advanced Cross-Chain Messaging (การส่งข้อความข้ามเชน)

## บทนำ

Cross-chain messaging คือหัวใจของ multi-chain future ที่เราจะเห็น dApp เดียวทำงานบนหลาย blockchain พร้อมกัน ใน Part นี้เราจะเรียนรู้วิธีการส่งข้อความและ state ข้ามเชนด้วย protocols หลักสามตัว:

- **LayerZero V2**: OApp (Omnichain App) interface สำหรับ generic message passing
- **Wormhole**: VAA (Verifiable Action Approval) และ Guardian network
- **Axelar**: General Message Passing พร้อม token transfer

เราจะเรียนรู้:
- โครงสร้าง messaging ของแต่ละ protocol
- การจัดการ message ordering และ idempotency
- การ sync state ข้ามเชน (OmniToken)
- Security considerations และ failure handling

---

## 1. LayerZero V2: OApp Interface

### 1.1 ภาพรวม LayerZero V2 Architecture

LayerZero V2 ใช้ **Decentralized Verifier Network (DVN)** แทน relayer เดิม:

```
Source Chain:
  User → OApp → Endpoint → DVN(s) → Executor

Destination Chain:
  Executor → Endpoint → OApp._lzReceive()
```

**Key Components:**
- `OApp`: Contract ที่ implement OmniApp interface
- `EndpointV2`: LayerZero endpoint บนแต่ละ chain
- `DVN`: Decentralized Verifier Network ที่ตรวจสอบ message
- `Executor`: ส่ง message ไปยัง destination chain

### 1.2 OApp Base Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title OAppBase
 * @dev Base contract สำหรับ LayerZero V2 OmniApp
 * Implement interface ที่จำเป็นสำหรับ cross-chain messaging
 *
 * Dependencies (จาก LayerZero V2 SDK):
 * - @layerzerolabs/lz-evm-oapp-v2/contracts/oapp/OApp.sol
 */

// Interface สำหรับ LayerZero V2 Endpoint
interface ILayerZeroEndpointV2 {
    struct MessagingFee {
        uint256 nativeFee;    // fee ใน native token (ETH/BNB/MATIC)
        uint256 lzTokenFee;   // fee ใน ZRO token (optional)
    }

    struct MessagingReceipt {
        bytes32 guid;         // unique identifier ของ message
        uint64 nonce;         // sequence number
        MessagingFee fee;
    }

    struct MessagingParams {
        uint32 dstEid;        // destination endpoint ID
        bytes32 receiver;     // receiver address (bytes32 format)
        bytes message;        // payload
        bytes options;        // executor options
        bool payInLzToken;    // ใช้ ZRO token จ่าย fee
    }

    function send(
        MessagingParams calldata _params,
        address _refundAddress
    ) external payable returns (MessagingReceipt memory receipt);

    function quote(
        MessagingParams calldata _params,
        address _sender
    ) external view returns (MessagingFee memory fee);

    function setConfig(
        address _oapp,
        address _lib,
        SetConfigParam[] calldata _params
    ) external;

    function eid() external view returns (uint32);
}

struct SetConfigParam {
    uint32 eid;      // endpoint ID
    uint32 configType;
    bytes config;
}

interface ILayerZeroReceiver {
    struct Origin {
        uint32 srcEid;    // source endpoint ID
        bytes32 sender;   // sender address (bytes32)
        uint64 nonce;     // message nonce
    }

    function lzReceive(
        Origin calldata _origin,
        bytes32 _guid,
        bytes calldata _message,
        address _executor,
        bytes calldata _extraData
    ) external payable;
}

/**
 * @title OmniCounter
 * @dev ตัวอย่าง OApp ที่ sync counter ข้ามเชน
 * ใช้ LayerZero V2 ส่ง increment message ไปยัง chain อื่น
 */
contract OmniCounter is ILayerZeroReceiver {
    ILayerZeroEndpointV2 public immutable endpoint;
    address public owner;

    // ==================== State ====================

    mapping(uint32 => bytes32) public peers;    // eid => peer OApp address
    mapping(uint32 => uint256) public counters; // eid => counter value (received from that eid)
    uint256 public localCounter;

    // Message ordering tracking
    mapping(uint32 => uint64) public nextNonce; // expected next nonce per src eid

    // ==================== Events ====================

    event CounterIncremented(uint32 indexed srcEid, uint256 newValue, bytes32 guid);
    event MessageSent(uint32 indexed dstEid, bytes32 guid, uint256 nativeFee);

    // ==================== Modifiers ====================

    modifier onlyEndpoint() {
        require(msg.sender == address(endpoint), "Only endpoint");
        _;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner");
        _;
    }

    // ==================== Constructor ====================

    constructor(address _endpoint) {
        endpoint = ILayerZeroEndpointV2(_endpoint);
        owner = msg.sender;
    }

    // ==================== Admin ====================

    /**
     * @dev ตั้งค่า peer contract บน chain อื่น
     * peers ต้องมี address เดียวกันบนทุก chain (deterministic deploy)
     */
    function setPeer(uint32 eid, bytes32 peer) external onlyOwner {
        peers[eid] = peer;
    }

    /**
     * @dev ตั้งค่า DVN config สำหรับ security
     * เพิ่ม DVN หลายตัวเพื่อ require multiple confirmations
     */
    function setDVNConfig(
        address sendLib,
        uint32 dstEid,
        address[] calldata dvns,
        uint8 requiredDVNs
    ) external onlyOwner {
        // ULN Config structure
        bytes memory config = abi.encode(
            UlnConfig({
                confirmations: 15,          // block confirmations ก่อน verify
                requiredDVNCount: requiredDVNs,
                optionalDVNCount: 0,
                optionalDVNThreshold: 0,
                requiredDVNs: dvns,
                optionalDVNs: new address[](0)
            })
        );

        SetConfigParam[] memory params = new SetConfigParam[](1);
        params[0] = SetConfigParam({
            eid: dstEid,
            configType: 2, // CONFIG_TYPE_ULN
            config: config
        });

        endpoint.setConfig(address(this), sendLib, params);
    }

    // ==================== Send ====================

    /**
     * @dev ส่ง increment message ไปยัง chain อื่น
     * @param dstEid destination endpoint ID
     *   - Ethereum mainnet: 30101
     *   - BNB Chain: 30102
     *   - Avalanche: 30106
     *   - Polygon: 30109
     *   - Arbitrum: 30110
     *   - Optimism: 30111
     */
    function incrementRemote(uint32 dstEid) external payable {
        require(peers[dstEid] != bytes32(0), "Peer not set");

        localCounter++;

        // Encode message: (action, value)
        bytes memory message = abi.encode(uint8(1), localCounter); // action=1: increment

        // Executor options: specify gas limit for _lzReceive
        bytes memory options = _buildOptions(200_000); // 200k gas on dst

        // Quote fee ก่อนส่ง
        ILayerZeroEndpointV2.MessagingFee memory fee = endpoint.quote(
            ILayerZeroEndpointV2.MessagingParams({
                dstEid: dstEid,
                receiver: peers[dstEid],
                message: message,
                options: options,
                payInLzToken: false
            }),
            address(this)
        );

        require(msg.value >= fee.nativeFee, "Insufficient fee");

        // ส่ง message
        ILayerZeroEndpointV2.MessagingReceipt memory receipt = endpoint.send{value: fee.nativeFee}(
            ILayerZeroEndpointV2.MessagingParams({
                dstEid: dstEid,
                receiver: peers[dstEid],
                message: message,
                options: options,
                payInLzToken: false
            }),
            msg.sender // refund address
        );

        emit MessageSent(dstEid, receipt.guid, fee.nativeFee);

        // Refund excess fee
        if (msg.value > fee.nativeFee) {
            payable(msg.sender).transfer(msg.value - fee.nativeFee);
        }
    }

    /**
     * @dev Quote fee สำหรับ cross-chain message
     */
    function quoteIncrementFee(uint32 dstEid) external view returns (uint256 nativeFee) {
        bytes memory message = abi.encode(uint8(1), localCounter + 1);
        bytes memory options = _buildOptions(200_000);

        ILayerZeroEndpointV2.MessagingFee memory fee = endpoint.quote(
            ILayerZeroEndpointV2.MessagingParams({
                dstEid: dstEid,
                receiver: peers[dstEid],
                message: message,
                options: options,
                payInLzToken: false
            }),
            address(this)
        );

        nativeFee = fee.nativeFee;
    }

    // ==================== Receive ====================

    /**
     * @dev Callback ที่ endpoint เรียกเมื่อ message มาถึง
     * ต้อง verify ว่ามาจาก peer ที่รู้จัก
     */
    function lzReceive(
        ILayerZeroReceiver.Origin calldata _origin,
        bytes32 _guid,
        bytes calldata _message,
        address _executor,
        bytes calldata _extraData
    ) external payable override onlyEndpoint {
        // Verify sender
        require(peers[_origin.srcEid] == _origin.sender, "Invalid peer");

        // Verify ordering (in-order delivery)
        require(_origin.nonce == nextNonce[_origin.srcEid] + 1, "Out of order");
        nextNonce[_origin.srcEid] = _origin.nonce;

        // Decode message
        (uint8 action, uint256 value) = abi.decode(_message, (uint8, uint256));

        if (action == 1) {
            // Increment counter
            counters[_origin.srcEid] = value;
            emit CounterIncremented(_origin.srcEid, value, _guid);
        }

        // Suppress unused variable warnings
        _executor;
        _extraData;
    }

    // ==================== Helpers ====================

    /**
     * @dev Build executor options (Type 3 = LZ_RECEIVE options)
     * กำหนด gas limit สำหรับ _lzReceive ที่ destination
     */
    function _buildOptions(uint128 gasLimit) internal pure returns (bytes memory) {
        // Options encoding: TYPE_3 | OPTION_TYPE_LZRECEIVE | gasLimit
        return abi.encodePacked(
            uint16(3),       // options type
            uint8(1),        // option type: LZRECEIVE
            uint16(16),      // option length
            gasLimit,        // gas limit (uint128)
            uint128(0)       // value to send (0)
        );
    }

    function addressToBytes32(address addr) internal pure returns (bytes32) {
        return bytes32(uint256(uint160(addr)));
    }
}

struct UlnConfig {
    uint64 confirmations;
    uint8 requiredDVNCount;
    uint8 optionalDVNCount;
    uint8 optionalDVNThreshold;
    address[] requiredDVNs;
    address[] optionalDVNs;
}
```

---

## 2. Wormhole: VAA และ Guardian Network

### 2.1 ภาพรวม Wormhole Architecture

```
Source Chain:
  Contract → Core Bridge.publishMessage() → Guardians observe

Guardian Network:
  19 Guardians → sign observation → produce VAA

Destination Chain:
  Anyone submits VAA → Core Bridge.parseAndVerifyVM() → Contract
```

**VAA Structure:**
```
Version (1 byte)
Guardian Set Index (4 bytes)
Signatures (n * 66 bytes)
Timestamp (4 bytes)
Nonce (4 bytes)
Emitter Chain (2 bytes)
Emitter Address (32 bytes)
Sequence (8 bytes)
Consistency Level (1 byte)
Payload (variable)
```

### 2.2 Wormhole Message Publishing

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title WormholeMessenger
 * @dev ส่ง/รับ message ผ่าน Wormhole protocol
 * ใช้ VAA (Verifiable Action Approval) สำหรับ cross-chain verification
 */

// Wormhole Core Bridge interface
interface IWormhole {
    struct VM {
        uint8 version;
        uint32 timestamp;
        uint32 nonce;
        uint16 emitterChainId;
        bytes32 emitterAddress;
        uint64 sequence;
        uint8 consistencyLevel;
        bytes payload;
        uint32 guardianSetIndex;
        Signature[] signatures;
        bytes32 hash;
    }

    struct Signature {
        bytes32 r;
        bytes32 s;
        uint8 v;
        uint8 guardianIndex;
    }

    /**
     * @dev publish message ไปยัง Wormhole
     * Guardians จะ observe event นี้และสร้าง VAA
     *
     * @param nonce arbitrary nonce (ไม่ unique ต้อง track sequence เอง)
     * @param payload ข้อมูลที่ต้องการส่ง
     * @param consistencyLevel ระดับ finality ที่ต้องการ
     *   1 = finalized (รอ finality)
     *   200 = instant (ไม่รอ)
     * @return sequence unique sequence number สำหรับ message นี้
     */
    function publishMessage(
        uint32 nonce,
        bytes memory payload,
        uint8 consistencyLevel
    ) external payable returns (uint64 sequence);

    /**
     * @dev parse และ verify VAA
     * @param encodedVM raw VAA bytes
     * @return vm parsed VM struct
     * @return valid true ถ้า VAA valid
     * @return reason error message ถ้าไม่ valid
     */
    function parseAndVerifyVM(
        bytes calldata encodedVM
    ) external view returns (VM memory vm, bool valid, string memory reason);

    function messageFee() external view returns (uint256);
    function chainId() external view returns (uint16);
}

/**
 * @title WormholeCrossChainApp
 * @dev Application ที่ใช้ Wormhole สำหรับ cross-chain state sync
 *
 * Pattern: Manual Verification
 * - User ต้อง fetch VAA จาก Wormhole Guardian API
 * - Submit VAA ไปยัง destination contract เอง
 * - Contract verify VAA และ process
 */
contract WormholeCrossChainApp {
    IWormhole public immutable wormhole;

    // ==================== State ====================

    // Chain registration
    mapping(uint16 => bytes32) public registeredEmitters; // chainId => emitter address

    // Replay protection
    mapping(bytes32 => bool) public processedVAAs; // vaaHash => processed

    // Cross-chain state
    mapping(uint16 => uint256) public remoteValues; // chainId => value

    uint256 public localValue;

    // Wormhole chain IDs:
    // Solana: 1
    // Ethereum: 2
    // Terra: 3
    // BSC: 4
    // Polygon: 5
    // Avalanche: 6
    // Oasis: 7
    // Algorand: 8
    // Aurora: 9
    // Fantom: 10
    // Karura: 11
    // Acala: 12
    // Klaytn: 13
    // Celo: 14
    // NEAR: 15
    // Moonbeam: 16
    // Arbitrum: 23
    // Optimism: 24
    // Aptos: 22

    // ==================== Events ====================

    event MessagePublished(uint64 sequence, bytes32 payloadHash);
    event VAAProcessed(uint16 srcChain, bytes32 emitter, uint64 sequence);
    event ValueUpdated(uint16 srcChain, uint256 newValue);

    // ==================== Constructor ====================

    constructor(address _wormhole) {
        wormhole = IWormhole(_wormhole);
    }

    // ==================== Admin ====================

    /**
     * @dev ลงทะเบียน emitter contract จาก chain อื่น
     * ใช้ address แบบ bytes32 เพื่อรองรับ non-EVM chains
     */
    function registerEmitter(uint16 chainId, bytes32 emitterAddress) external {
        // ใน production: onlyOwner
        registeredEmitters[chainId] = emitterAddress;
    }

    // ==================== Send ====================

    /**
     * @dev ส่ง value update ไปยัง chain อื่นผ่าน Wormhole
     * Guardians จะ observe และสร้าง VAA
     *
     * @param newValue ค่าใหม่ที่ต้องการ sync
     * @param nonce arbitrary nonce
     */
    function updateValue(uint256 newValue, uint32 nonce) external payable {
        uint256 messageFee = wormhole.messageFee();
        require(msg.value >= messageFee, "Insufficient message fee");

        localValue = newValue;

        // Encode payload: (action, value, sender, timestamp)
        bytes memory payload = abi.encode(
            uint8(1),         // action type: VALUE_UPDATE
            newValue,         // new value
            msg.sender,       // original sender
            block.timestamp   // timestamp
        );

        // Publish message (Guardians จะ observe event นี้)
        uint64 sequence = wormhole.publishMessage{value: messageFee}(
            nonce,
            payload,
            1    // finalized consistency
        );

        emit MessagePublished(sequence, keccak256(payload));

        // Refund excess fee
        if (msg.value > messageFee) {
            payable(msg.sender).transfer(msg.value - messageFee);
        }
    }

    // ==================== Receive ====================

    /**
     * @dev รับและ process VAA จาก chain อื่น
     * ผู้ใช้ต้อง fetch VAA จาก:
     * https://api.wormholescan.io/v1/signed_vaa/{chain_id}/{emitter_address}/{sequence}
     *
     * @param encodedVAA raw VAA bytes (ดึงจาก Wormhole Guardian API)
     */
    function receiveMessage(bytes calldata encodedVAA) external {
        // 1. Parse และ verify VAA
        (IWormhole.VM memory vm, bool valid, string memory reason) =
            wormhole.parseAndVerifyVM(encodedVAA);

        require(valid, string(abi.encodePacked("Invalid VAA: ", reason)));

        // 2. Replay protection
        require(!processedVAAs[vm.hash], "VAA already processed");
        processedVAAs[vm.hash] = true;

        // 3. Verify emitter
        require(
            registeredEmitters[vm.emitterChainId] == vm.emitterAddress,
            "Unknown emitter"
        );

        // 4. Decode payload
        (uint8 action, uint256 value, address sender, uint256 timestamp) =
            abi.decode(vm.payload, (uint8, uint256, address, uint256));

        // 5. Process message
        if (action == 1) {
            remoteValues[vm.emitterChainId] = value;
            emit ValueUpdated(vm.emitterChainId, value);
        }

        // Suppress unused variable warnings
        sender;
        timestamp;

        emit VAAProcessed(vm.emitterChainId, vm.emitterAddress, vm.sequence);
    }

    /**
     * @dev Helper สำหรับ encode address เป็น bytes32
     * Wormhole ใช้ bytes32 สำหรับ address (รองรับ non-EVM)
     */
    function addressToBytes32(address addr) public pure returns (bytes32) {
        return bytes32(uint256(uint160(addr)));
    }

    function bytes32ToAddress(bytes32 b) public pure returns (address) {
        return address(uint160(uint256(b)));
    }
}
```

### 2.3 Wormhole Token Bridge Pattern

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title WormholeTokenTransfer
 * @dev ส่ง token ข้ามเชนด้วย Wormhole Token Bridge
 * Pattern: Lock-and-Mint
 * - Source chain: lock token ใน bridge contract
 * - Destination chain: mint wrapped token
 */
interface ITokenBridge {
    function transferTokens(
        address token,
        uint256 amount,
        uint16 recipientChain,
        bytes32 recipient,
        uint256 arbiterFee,
        uint32 nonce
    ) external payable returns (uint64 sequence);

    function completeTransfer(bytes memory encodedVm) external;

    function wrappedAsset(uint16 tokenChainId, bytes32 tokenAddress) external view returns (address);
}

contract WormholeTokenBridgeIntegration {
    ITokenBridge public immutable tokenBridge;
    IWormhole public immutable wormhole;

    event TokenBridged(
        address indexed token,
        uint256 amount,
        uint16 dstChain,
        bytes32 recipient,
        uint64 sequence
    );

    constructor(address _tokenBridge, address _wormhole) {
        tokenBridge = ITokenBridge(_tokenBridge);
        wormhole = IWormhole(_wormhole);
    }

    /**
     * @dev ส่ง ERC20 token ไปยัง chain อื่น
     * Token จะถูก lock บน source chain
     * Wrapped token จะถูก mint บน destination chain
     */
    function bridgeToken(
        address token,
        uint256 amount,
        uint16 dstChain,
        address recipient,
        uint32 nonce
    ) external payable {
        uint256 messageFee = wormhole.messageFee();
        require(msg.value >= messageFee, "Insufficient fee");

        // Transfer token เข้า bridge
        IERC20(token).transferFrom(msg.sender, address(this), amount);
        IERC20(token).approve(address(tokenBridge), amount);

        // Bridge token
        uint64 sequence = tokenBridge.transferTokens{value: messageFee}(
            token,
            amount,
            dstChain,
            bytes32(uint256(uint160(recipient))),
            0,    // arbiter fee (0 = no relayer fee)
            nonce
        );

        emit TokenBridged(token, amount, dstChain, bytes32(uint256(uint160(recipient))), sequence);
    }

    /**
     * @dev Complete transfer บน destination chain
     * Submit VAA เพื่อ mint wrapped token
     */
    function completeTransfer(bytes calldata vaa) external {
        tokenBridge.completeTransfer(vaa);
    }
}

interface IERC20 {
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
    function approve(address spender, uint256 amount) external returns (bool);
    function transfer(address to, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}
```

---

## 3. Axelar: General Message Passing

### 3.1 ภาพรวม Axelar Network

Axelar เป็น blockchain ที่สร้างมาเพื่อ cross-chain communication โดยเฉพาะ:

```
Source Chain:
  App → IAxelarGateway.callContract() → Relayer Network

Axelar Network:
  Validators observe → vote → produce proof

Destination Chain:
  Axelar gateway → IAxelarExecutable.execute()
```

**ความแตกต่างจาก LayerZero:**
- Axelar มี native blockchain ของตัวเอง (Cosmos SDK)
- ใช้ PoS consensus สำหรับ verification
- Native token: AXL
- รองรับ token transfer แบบ native ด้วย InterchainToken

### 3.2 AxelarExecutable Pattern

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title AxelarGMPIntegration
 * @dev General Message Passing ด้วย Axelar
 * ทั้ง send message และ receive message
 *
 * Supported chains (Axelar chain names):
 * - "ethereum"
 * - "binance"
 * - "polygon"
 * - "avalanche"
 * - "fantom"
 * - "moonbeam"
 * - "arbitrum"
 * - "optimism"
 * - "base"
 */

interface IAxelarGateway {
    /**
     * @dev ส่ง message ไปยัง chain อื่น
     * @param destinationChain ชื่อ chain ปลายทาง
     * @param contractAddress address ของ contract ปลายทาง
     * @param payload ข้อมูลที่ต้องการส่ง
     */
    function callContract(
        string calldata destinationChain,
        string calldata contractAddress,
        bytes calldata payload
    ) external;

    /**
     * @dev ส่ง message พร้อม token
     * @param destinationChain ชื่อ chain ปลายทาง
     * @param contractAddress address ของ contract ปลายทาง
     * @param payload ข้อมูลที่ต้องการส่ง
     * @param symbol symbol ของ token ที่จะส่ง
     * @param amount จำนวน token
     */
    function callContractWithToken(
        string calldata destinationChain,
        string calldata contractAddress,
        bytes calldata payload,
        string calldata symbol,
        uint256 amount
    ) external;

    /**
     * @dev validate ว่า contract call มาจาก Axelar จริง
     */
    function validateContractCall(
        bytes32 commandId,
        string calldata sourceChain,
        string calldata sourceAddress,
        bytes32 payloadHash
    ) external returns (bool);

    /**
     * @dev validate contract call with token
     */
    function validateContractCallAndMint(
        bytes32 commandId,
        string calldata sourceChain,
        string calldata sourceAddress,
        bytes32 payloadHash,
        string calldata symbol,
        uint256 amount
    ) external returns (bool);
}

interface IAxelarGasService {
    /**
     * @dev จ่าย gas ล่วงหน้าสำหรับ cross-chain execution
     * @param sender ผู้ส่ง
     * @param destinationChain chain ปลายทาง
     * @param destinationAddress contract ปลายทาง
     * @param payload ข้อมูล
     * @param refundAddress address สำหรับคืนเงิน excess gas
     */
    function payNativeGasForContractCall(
        address sender,
        string calldata destinationChain,
        string calldata destinationAddress,
        bytes calldata payload,
        address refundAddress
    ) external payable;

    function payNativeGasForContractCallWithToken(
        address sender,
        string calldata destinationChain,
        string calldata destinationAddress,
        bytes calldata payload,
        string calldata symbol,
        uint256 amount,
        address refundAddress
    ) external payable;
}

/**
 * @title AxelarCrossChainApp
 * @dev ตัวอย่าง app ที่ใช้ Axelar GMP สำหรับ cross-chain state management
 */
contract AxelarCrossChainApp {
    IAxelarGateway public immutable gateway;
    IAxelarGasService public immutable gasService;

    // ==================== State ====================

    // Trusted remote contracts (chain name => contract address string)
    mapping(string => string) public trustedRemotes;

    // Received messages (idempotency)
    mapping(bytes32 => bool) public processedCommands;

    // Cross-chain data
    mapping(string => uint256) public remoteData; // chain => value

    // ==================== Events ====================

    event MessageSent(string dstChain, string dstContract, bytes payload);
    event MessageReceived(string srcChain, string srcAddress, bytes payload);
    event TokenReceived(string srcChain, string symbol, uint256 amount, address recipient);

    // ==================== Modifiers ====================

    modifier onlyGateway() {
        require(msg.sender == address(gateway), "Only gateway");
        _;
    }

    // ==================== Constructor ====================

    constructor(address _gateway, address _gasService) {
        gateway = IAxelarGateway(_gateway);
        gasService = IAxelarGasService(_gasService);
    }

    // ==================== Admin ====================

    function setTrustedRemote(string calldata chain, string calldata remoteContract) external {
        // ใน production: onlyOwner
        trustedRemotes[chain] = remoteContract;
    }

    // ==================== Send ====================

    /**
     * @dev ส่ง data update ไปยัง chain อื่น
     * @param dstChain ชื่อ chain ปลายทาง
     * @param value ค่าที่ต้องการ sync
     */
    function sendUpdate(string calldata dstChain, uint256 value) external payable {
        string memory dstContract = trustedRemotes[dstChain];
        require(bytes(dstContract).length > 0, "Remote not set");

        bytes memory payload = abi.encode(
            uint8(1),        // action: UPDATE
            value,
            msg.sender
        );

        // จ่าย gas ก่อน (ผ่าน Gas Service)
        if (msg.value > 0) {
            gasService.payNativeGasForContractCall{value: msg.value}(
                address(this),
                dstChain,
                dstContract,
                payload,
                msg.sender // refund address
            );
        }

        // ส่ง message
        gateway.callContract(dstChain, dstContract, payload);

        emit MessageSent(dstChain, dstContract, payload);
    }

    /**
     * @dev ส่ง message พร้อม token (เช่น USDC ข้ามเชน)
     * Token ต้องเป็น Axelar canonical token (axlUSDC, axlETH etc.)
     */
    function sendTokenWithMessage(
        string calldata dstChain,
        string calldata dstContract,
        uint256 value,
        string calldata tokenSymbol,
        uint256 tokenAmount
    ) external payable {
        bytes memory payload = abi.encode(value, msg.sender);

        // Approve token transfer ให้ gateway
        // (ใน production ต้อง approve ก่อน)

        if (msg.value > 0) {
            gasService.payNativeGasForContractCallWithToken{value: msg.value}(
                address(this),
                dstChain,
                dstContract,
                payload,
                tokenSymbol,
                tokenAmount,
                msg.sender
            );
        }

        gateway.callContractWithToken(
            dstChain,
            dstContract,
            payload,
            tokenSymbol,
            tokenAmount
        );
    }

    // ==================== Receive ====================

    /**
     * @dev Callback สำหรับ pure message (ไม่มี token)
     * Axelar gateway จะเรียก function นี้เมื่อ message มาถึง
     *
     * @param commandId unique ID ของ command (สำหรับ idempotency)
     * @param sourceChain chain ที่ส่งมา
     * @param sourceAddress contract ที่ส่งมา
     * @param payload ข้อมูลที่ได้รับ
     */
    function execute(
        bytes32 commandId,
        string calldata sourceChain,
        string calldata sourceAddress,
        bytes calldata payload
    ) external onlyGateway {
        // Idempotency check
        require(!processedCommands[commandId], "Already processed");
        processedCommands[commandId] = true;

        // Verify source
        require(
            keccak256(bytes(trustedRemotes[sourceChain])) == keccak256(bytes(sourceAddress)),
            "Untrusted source"
        );

        // Validate ด้วย gateway (สำคัญมาก!)
        require(
            gateway.validateContractCall(
                commandId,
                sourceChain,
                sourceAddress,
                keccak256(payload)
            ),
            "Not approved by gateway"
        );

        // Decode และ process payload
        (uint8 action, uint256 value, address sender) = abi.decode(payload, (uint8, uint256, address));

        if (action == 1) {
            remoteData[sourceChain] = value;
        }

        // Suppress unused warning
        sender;

        emit MessageReceived(sourceChain, sourceAddress, payload);
    }

    /**
     * @dev Callback สำหรับ message พร้อม token
     * @param symbol token symbol ที่ได้รับ
     * @param amount จำนวน token
     */
    function executeWithToken(
        bytes32 commandId,
        string calldata sourceChain,
        string calldata sourceAddress,
        bytes calldata payload,
        string calldata symbol,
        uint256 amount
    ) external onlyGateway {
        require(!processedCommands[commandId], "Already processed");
        processedCommands[commandId] = true;

        // Validate with gateway
        require(
            gateway.validateContractCallAndMint(
                commandId,
                sourceChain,
                sourceAddress,
                keccak256(payload),
                symbol,
                amount
            ),
            "Not approved by gateway"
        );

        // Decode payload
        (uint256 value, address recipient) = abi.decode(payload, (uint256, address));

        // Token ถูก mint เข้า contract นี้แล้ว โดย gateway
        // ส่งต่อไปยัง recipient
        // IERC20(getTokenAddress(symbol)).transfer(recipient, amount);

        remoteData[sourceChain] = value;

        emit TokenReceived(sourceChain, symbol, amount, recipient);
    }
}
```

---

## 4. Message Ordering และ Idempotency

### 4.1 ปัญหา Out-of-Order Delivery

ใน cross-chain messaging ข้อความอาจมาถึงไม่ตามลำดับเพราะ:
- Network congestion ต่างกันใน แต่ละ chain
- Gas price สูงขึ้นทำให้ relayer ชะลอการส่ง
- Retry mechanism ส่งซ้ำ

### 4.2 Sequence Number Solution

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title OrderedMessageProcessor
 * @dev จัดการ message ordering และ idempotency
 * รองรับทั้ง in-order และ out-of-order delivery
 */
contract OrderedMessageProcessor {
    // ==================== Types ====================

    enum DeliveryMode {
        IN_ORDER,     // ต้องรับ in order (reject out-of-order)
        ORDERED_QUEUE // buffer out-of-order messages
    }

    struct PendingMessage {
        bytes payload;
        bool exists;
    }

    // ==================== State ====================

    // ต่อ source chain
    mapping(uint32 => uint64) public nextExpectedNonce; // srcEid => nonce
    mapping(uint32 => mapping(uint64 => PendingMessage)) public messageQueue; // srcEid => nonce => msg

    // Idempotency (duplicate detection)
    mapping(bytes32 => bool) public processedMessages; // hash(srcEid+nonce) => processed

    DeliveryMode public deliveryMode;

    // ==================== Events ====================

    event MessageProcessed(uint32 indexed srcEid, uint64 nonce, bytes payload);
    event MessageQueued(uint32 indexed srcEid, uint64 nonce);
    event OutOfOrderMessage(uint32 indexed srcEid, uint64 received, uint64 expected);

    // ==================== Constructor ====================

    constructor(DeliveryMode mode) {
        deliveryMode = mode;
    }

    // ==================== Core Logic ====================

    /**
     * @dev รับ message พร้อมจัดการ ordering
     * @param srcEid source endpoint ID
     * @param nonce sequence number ของ message
     * @param payload ข้อมูล
     */
    function receiveMessage(
        uint32 srcEid,
        uint64 nonce,
        bytes calldata payload
    ) external {
        // Idempotency check
        bytes32 msgId = keccak256(abi.encode(srcEid, nonce));
        require(!processedMessages[msgId], "Message already processed");

        uint64 expected = nextExpectedNonce[srcEid];

        if (nonce == expected || expected == 0) {
            // Process immediately
            _processMessage(srcEid, nonce, payload);
            processedMessages[msgId] = true;
            nextExpectedNonce[srcEid] = nonce + 1;

            // Check ว่ามี queued messages ที่รอ process อยู่
            _processQueuedMessages(srcEid);

        } else if (nonce > expected) {
            // Out-of-order message
            emit OutOfOrderMessage(srcEid, nonce, expected);

            if (deliveryMode == DeliveryMode.IN_ORDER) {
                revert("Out of order message rejected");
            } else {
                // Queue for later
                messageQueue[srcEid][nonce] = PendingMessage({
                    payload: payload,
                    exists: true
                });
                emit MessageQueued(srcEid, nonce);
            }
        } else {
            // nonce < expected: duplicate/old message
            revert("Outdated message");
        }
    }

    /**
     * @dev ประมวล messages ที่ queued ไว้หลังจากได้รับ message ที่ missing
     * เรียกซ้ำจนกว่าจะไม่มี queued message ที่ sequential
     */
    function _processQueuedMessages(uint32 srcEid) internal {
        uint64 next = nextExpectedNonce[srcEid];

        while (messageQueue[srcEid][next].exists) {
            PendingMessage memory msg = messageQueue[srcEid][next];
            delete messageQueue[srcEid][next];

            bytes32 msgId = keccak256(abi.encode(srcEid, next));
            processedMessages[msgId] = true;

            _processMessage(srcEid, next, msg.payload);
            next++;
            nextExpectedNonce[srcEid] = next;
        }
    }

    function _processMessage(uint32 srcEid, uint64 nonce, bytes memory payload) internal {
        // Override ใน derived contract
        emit MessageProcessed(srcEid, nonce, payload);
    }

    /**
     * @dev Force skip nonce (ใช้เมื่อ message สูญหาย)
     * ต้องใช้ด้วยความระมัดระวัง เพราะอาจทำให้ state inconsistent
     */
    function skipNonce(uint32 srcEid, uint64 nonce) external {
        // onlyOwner ใน production
        require(nonce == nextExpectedNonce[srcEid], "Not next expected nonce");
        nextExpectedNonce[srcEid] = nonce + 1;
        _processQueuedMessages(srcEid); // Process any queued messages
    }
}
```

---

## 5. Cross-Chain State Sync: OmniToken

### 5.1 ความท้าทายของ OmniToken

OmniToken ต้องการ:
- **Same address บนทุก chain**: ใช้ CREATE2 + deterministic deploy
- **Same total supply ทุก chain รวมกัน**: ต้องระมัดระวัง double-spending
- **Atomic transfer**: ถ้า bridge fail ต้องมี recovery mechanism

**Pattern: Lock-and-Mint**
```
Source: burn/lock tokens → emit event
Bridge: verify event → send message
Destination: mint tokens (same amount)
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title OmniToken
 * @dev ERC20 token ที่ sync supply ข้ามเชนด้วย LayerZero V2
 *
 * Design:
 * - Home chain (Ethereum): mints total supply ครั้งแรก
 * - Remote chains: mint เมื่อได้รับ message จาก home chain
 * - Cross-chain transfer: burn on source → mint on destination
 *
 * Security:
 * - Only trusted peers (same contract, different chains) can mint/burn
 * - Nonce tracking ป้องกัน replay attacks
 * - Total supply invariant ต้องคงที่ across all chains
 */
contract OmniToken is ERC20, Ownable, ReentrancyGuard {

    // ==================== Interfaces ====================

    ILayerZeroEndpointV2 public immutable endpoint;

    // ==================== Constants ====================

    uint8 private constant MSG_TYPE_TRANSFER = 1;
    uint8 private constant MSG_TYPE_SUPPLY_SYNC = 2;

    // ==================== State ====================

    // Peer contracts on other chains (eid => address)
    mapping(uint32 => bytes32) public peers;

    // Cross-chain transfer tracking
    mapping(bytes32 => bool) public processedMessages; // guid => processed

    // Sequence number per destination chain (for ordering)
    mapping(uint32 => uint64) public outboundNonce;
    mapping(uint32 => uint64) public inboundNonce;

    // Bridge statistics
    uint256 public totalBridgedOut; // จำนวน token ที่ bridge ออกไป
    uint256 public totalBridgedIn;  // จำนวน token ที่ bridge เข้ามา

    // ==================== Events ====================

    event CrossChainTransferSent(
        address indexed from,
        uint32 dstEid,
        address indexed to,
        uint256 amount,
        bytes32 guid
    );

    event CrossChainTransferReceived(
        uint32 srcEid,
        address indexed to,
        uint256 amount,
        bytes32 guid
    );

    // ==================== Constructor ====================

    constructor(
        string memory name,
        string memory symbol,
        address _endpoint,
        uint256 initialSupply
    ) ERC20(name, symbol) Ownable(msg.sender) {
        endpoint = ILayerZeroEndpointV2(_endpoint);
        if (initialSupply > 0) {
            _mint(msg.sender, initialSupply);
        }
    }

    // ==================== Admin ====================

    function setPeer(uint32 eid, bytes32 peer) external onlyOwner {
        peers[eid] = peer;
    }

    // ==================== Cross-Chain Transfer ====================

    /**
     * @dev Bridge token ไปยัง chain อื่น (burn-and-mint pattern)
     * 1. Burn token บน source chain
     * 2. ส่ง message ด้วย LayerZero
     * 3. Mint token บน destination chain
     *
     * @param dstEid destination endpoint ID
     * @param to recipient address บน destination chain
     * @param amount จำนวน token
     */
    function bridge(
        uint32 dstEid,
        address to,
        uint256 amount
    ) external payable nonReentrant {
        require(peers[dstEid] != bytes32(0), "Peer not set");
        require(amount > 0, "Amount must be > 0");
        require(balanceOf(msg.sender) >= amount, "Insufficient balance");

        // 1. Burn tokens บน source chain
        _burn(msg.sender, amount);
        totalBridgedOut += amount;

        // 2. Encode message
        outboundNonce[dstEid]++;
        bytes memory message = abi.encode(
            MSG_TYPE_TRANSFER,
            to,
            amount,
            outboundNonce[dstEid]
        );

        // 3. Build options (200k gas ที่ destination)
        bytes memory options = _buildLzReceiveOption(200_000);

        // 4. Quote fee
        ILayerZeroEndpointV2.MessagingFee memory fee = endpoint.quote(
            ILayerZeroEndpointV2.MessagingParams({
                dstEid: dstEid,
                receiver: peers[dstEid],
                message: message,
                options: options,
                payInLzToken: false
            }),
            address(this)
        );

        require(msg.value >= fee.nativeFee, "Insufficient fee");

        // 5. Send message
        ILayerZeroEndpointV2.MessagingReceipt memory receipt = endpoint.send{value: fee.nativeFee}(
            ILayerZeroEndpointV2.MessagingParams({
                dstEid: dstEid,
                receiver: peers[dstEid],
                message: message,
                options: options,
                payInLzToken: false
            }),
            msg.sender
        );

        emit CrossChainTransferSent(msg.sender, dstEid, to, amount, receipt.guid);

        // Refund excess
        if (msg.value > fee.nativeFee) {
            payable(msg.sender).transfer(msg.value - fee.nativeFee);
        }
    }

    /**
     * @dev Quote fee สำหรับ bridge
     */
    function quoteBridge(
        uint32 dstEid,
        address to,
        uint256 amount
    ) external view returns (uint256 nativeFee) {
        bytes memory message = abi.encode(MSG_TYPE_TRANSFER, to, amount, outboundNonce[dstEid] + 1);
        bytes memory options = _buildLzReceiveOption(200_000);

        ILayerZeroEndpointV2.MessagingFee memory fee = endpoint.quote(
            ILayerZeroEndpointV2.MessagingParams({
                dstEid: dstEid,
                receiver: peers[dstEid],
                message: message,
                options: options,
                payInLzToken: false
            }),
            address(this)
        );

        nativeFee = fee.nativeFee;
    }

    // ==================== LayerZero Receive ====================

    /**
     * @dev Callback จาก LayerZero endpoint
     * Mint tokens ให้ recipient บน destination chain
     */
    function lzReceive(
        ILayerZeroReceiver.Origin calldata _origin,
        bytes32 _guid,
        bytes calldata _message,
        address _executor,
        bytes calldata _extraData
    ) external {
        require(msg.sender == address(endpoint), "Only endpoint");
        require(peers[_origin.srcEid] == _origin.sender, "Invalid peer");

        // Replay protection
        require(!processedMessages[_guid], "Already processed");
        processedMessages[_guid] = true;

        // Decode message
        (uint8 msgType, address to, uint256 amount, uint64 nonce) =
            abi.decode(_message, (uint8, address, uint256, uint64));

        // Ordering check
        require(nonce == inboundNonce[_origin.srcEid] + 1, "Wrong nonce");
        inboundNonce[_origin.srcEid] = nonce;

        if (msgType == MSG_TYPE_TRANSFER) {
            // Mint tokens ให้ recipient
            _mint(to, amount);
            totalBridgedIn += amount;

            emit CrossChainTransferReceived(_origin.srcEid, to, amount, _guid);
        }

        // Suppress warnings
        _executor;
        _extraData;
    }

    // ==================== Supply Invariant Check ====================

    /**
     * @dev ตรวจสอบ supply ของ chain นี้
     * Total supply across all chains ควรคงที่เสมอ
     *
     * Invariant: sum(totalSupply[chain_i]) = FIXED_TOTAL_SUPPLY
     */
    function getLocalSupplyStats() external view returns (
        uint256 localSupply,
        uint256 bridgedOut,
        uint256 bridgedIn,
        int256 netBalance // bridgedIn - bridgedOut
    ) {
        localSupply = totalSupply();
        bridgedOut = totalBridgedOut;
        bridgedIn = totalBridgedIn;
        netBalance = int256(bridgedIn) - int256(bridgedOut);
    }

    // ==================== Helpers ====================

    function _buildLzReceiveOption(uint128 gasLimit) internal pure returns (bytes memory) {
        return abi.encodePacked(
            uint16(3),
            uint8(1),
            uint16(16),
            gasLimit,
            uint128(0)
        );
    }
}
```

---

## Workshop: Deploy OmniToken บน 2 Testnets

### เป้าหมาย
1. Deploy OmniToken บน Sepolia (Ethereum testnet)
2. Deploy OmniToken บน Mumbai (Polygon testnet)
3. Set peer addresses
4. Bridge tokens ระหว่าง chains

### Script การ Deploy

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title OmniTokenDeployScript
 * @dev Deployment configuration สำหรับ OmniToken บน testnets
 *
 * LayerZero V2 Testnet Endpoints:
 * - Sepolia:   0x6EDCE65403992e310A62460808c4b910D972f10f (eid: 40161)
 * - Mumbai:    0x6EDCE65403992e310A62460808c4b910D972f10f (eid: 40109)
 * - Fuji:      0x6EDCE65403992e310A62460808c4b910D972f10f (eid: 40106)
 * - BNB Testnet: 0x6EDCE65403992e310A62460808c4b910D972f10f (eid: 40102)
 *
 * ขั้นตอน:
 * 1. Deploy บน Sepolia ด้วย initialSupply = 1_000_000e18
 * 2. Deploy บน Mumbai ด้วย initialSupply = 0 (รับ token จาก bridge)
 * 3. setPeer(40109, sepoliaAddress) บน Mumbai contract
 * 4. setPeer(40161, mumbaiAddress) บน Sepolia contract
 * 5. bridge(40109, recipient, 1000e18) พร้อม ETH สำหรับ fee
 */
contract OmniTokenConfig {
    // Endpoint IDs (eid) ในระบบ LayerZero V2
    uint32 public constant SEPOLIA_EID = 40161;
    uint32 public constant MUMBAI_EID = 40109;
    uint32 public constant FUJI_EID = 40106;
    uint32 public constant BNB_TESTNET_EID = 40102;

    // Testnet endpoints (ทุก chain ใช้ address เดียวกันใน testnet)
    address public constant LZ_ENDPOINT_TESTNET = 0x6EDCE65403992e310A62460808c4b910D972f10f;

    /**
     * @dev คำนวณ deterministic address ด้วย CREATE2
     * ใช้ SAME salt บนทุก chain เพื่อได้ address เดียวกัน
     */
    function computeCreate2Address(
        bytes32 salt,
        bytes memory bytecode,
        address deployer
    ) public pure returns (address) {
        return address(uint160(uint256(keccak256(abi.encodePacked(
            bytes1(0xff),
            deployer,
            salt,
            keccak256(bytecode)
        )))));
    }

    /**
     * @dev Verify cross-chain invariant
     * ตรวจสอบว่า supply รวมทุก chain ถูกต้อง
     */
    function verifySupplyInvariant(
        uint256[] memory chainSupplies,
        uint256 totalExpectedSupply
    ) public pure returns (bool) {
        uint256 total = 0;
        for (uint256 i = 0; i < chainSupplies.length; i++) {
            total += chainSupplies[i];
        }
        return total == totalExpectedSupply;
    }
}
```

---

## สรุป Part 79

- **LayerZero V2** ใช้ DVN (Decentralized Verifier Network) สำหรับ verification; OApp interface ต้อง implement `_lzSend` และ `_lzReceive`; `MessagingFee` quote ก่อนส่งเสมอ
- **Wormhole** ใช้ 19 Guardians สำหรับ VAA generation; ต้อง `parseAndVerifyVM` และ replay protection ด้วย vaaHash; Token bridge ใช้ lock-and-mint pattern
- **Axelar** มี native blockchain สำหรับ consensus; ต้อง `validateContractCall` บน destination; Gas Service จ่าย cross-chain gas ล่วงหน้า
- **Message Ordering** สำคัญมาก; ใช้ sequence numbers และ queue out-of-order messages; Idempotency ด้วย processedMessages mapping
- **OmniToken** ใช้ burn-and-mint pattern; same address บนทุก chain ด้วย CREATE2; ต้องรักษา total supply invariant: sum(supply_i) = fixed

## Next: Part 80 - World-Class Protocol Patterns
