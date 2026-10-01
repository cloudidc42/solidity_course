# Part 82: Privacy in DeFi (ความเป็นส่วนตัวใน DeFi)

## บทนำ

Blockchain โดยธรรมชาติเป็น **สาธารณะ (public)** — ทุกธุรกรรมสามารถติดตามได้ นี่คือปัญหาสำหรับผู้ใช้ที่ต้องการความเป็นส่วนตัว เช่น บริษัทที่ไม่ต้องการให้คู่แข่งเห็นกลยุทธ์ทางการเงิน หรือบุคคลทั่วไปที่ต้องการปกป้องข้อมูลส่วนตัว

ในปาร์ตนี้เราจะศึกษา:
- **Tornado Cash**: ระบบ privacy แรกที่ประสบความสำเร็จบน Ethereum
- **Stealth Addresses** (ERC-5564): มาตรฐาน one-time address
- **Confidential Transactions**: แนวคิดการซ่อนจำนวน
- **Privacy-Preserving Voting**: ลงคะแนนโดยไม่เปิดเผยตัวตน
- **Aztec Connect**: สถาปัตยกรรมผสม ZK + Account Abstraction

---

## 82.1 Tornado Cash Architecture

### แนวคิดพื้นฐาน

Tornado Cash ทำงานโดยใช้ **Merkle Tree** และ **Zero-Knowledge Proofs (ZKP)**:

```
ขั้นตอนการทำงาน:
1. DEPOSIT: ผู้ใช้ A สร้าง secret + nullifier
           → hash เป็น commitment
           → ส่ง commitment + ETH เข้า contract
           
2. MERKLE TREE อัปเดต: เพิ่ม commitment เป็น leaf ใหม่

3. WITHDRAW (ด้วย address ใหม่):
   → สร้าง ZK Proof ว่า "ฉันรู้ secret ของ commitment ที่อยู่ใน tree"
   → ส่ง nullifierHash (ป้องกัน double-spend)
   → contract ตรวจสอบ proof + nullifier ยังไม่ได้ใช้
   → โอน ETH ให้ recipient (address ใหม่)
```

**ทำไมจึงเป็น private?**
- Deposit address และ Withdraw address ไม่เชื่อมกัน
- ZK Proof ไม่เปิดเผย secret หรือ commitment ใด
- ถ้ามีหลายคน deposit ในจำนวนเท่ากัน → ไม่รู้ว่า withdrawal ไหนมาจาก deposit ไหน

### 82.1.1 Merkle Tree Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title MerkleTreeWithHistory
 * @notice Merkle Tree ที่เก็บประวัติ root หลาย roots
 *         (เพื่อให้ withdrawal ทำได้แม้ root เปลี่ยนไปแล้ว)
 * @dev ใช้ Poseidon hash หรือ MiMC hash (ZK-friendly)
 *      ในตัวอย่างนี้ใช้ keccak256 เพื่อความง่าย
 *      ในการใช้งานจริงต้องใช้ ZK-friendly hash
 */
contract MerkleTreeWithHistory {
    
    // จำนวน levels ของ tree (depth)
    uint32 public constant LEVELS = 20;
    
    // จำนวน roots ที่เก็บประวัติ
    uint32 public constant ROOT_HISTORY_SIZE = 30;
    
    // Zero values สำหรับแต่ละ level (hash ของ empty subtree)
    bytes32[LEVELS] public zeros;
    
    // Subtree hashes ชั้นซ้ายสุดที่ยังไม่ complete
    bytes32[LEVELS] public filledSubtrees;
    
    // ประวัติ roots
    bytes32[ROOT_HISTORY_SIZE] public roots;
    uint32 public currentRootIndex = 0;
    
    uint32 public nextIndex = 0;
    
    event LeafInserted(bytes32 indexed leaf, uint32 leafIndex, bytes32 root);
    
    constructor() {
        // คำนวณ zero values
        bytes32 currentZero = keccak256(abi.encodePacked(uint256(0)));
        zeros[0] = currentZero;
        filledSubtrees[0] = currentZero;
        
        for (uint32 i = 1; i < LEVELS; i++) {
            currentZero = _hashLeftRight(currentZero, currentZero);
            zeros[i] = currentZero;
            filledSubtrees[i] = currentZero;
        }
        
        roots[0] = _hashLeftRight(currentZero, currentZero);
    }
    
    /**
     * @notice เพิ่ม leaf ใหม่เข้า tree
     * @return index ของ leaf ที่เพิ่ม
     */
    function _insert(bytes32 _leaf) internal returns (uint32 index) {
        uint32 _nextIndex = nextIndex;
        require(_nextIndex != uint32(2**LEVELS), "Merkle tree is full");
        
        uint32 currentIndex = _nextIndex;
        bytes32 currentLevelHash = _leaf;
        bytes32 left;
        bytes32 right;
        
        for (uint32 i = 0; i < LEVELS; i++) {
            if (currentIndex % 2 == 0) {
                // left node - เก็บไว้รอ right
                left = currentLevelHash;
                right = zeros[i];
                filledSubtrees[i] = currentLevelHash;
            } else {
                // right node - hash กับ left ที่เก็บไว้
                left = filledSubtrees[i];
                right = currentLevelHash;
            }
            
            currentLevelHash = _hashLeftRight(left, right);
            currentIndex /= 2;
        }
        
        // อัปเดต root
        uint32 newRootIndex = (currentRootIndex + 1) % ROOT_HISTORY_SIZE;
        currentRootIndex = newRootIndex;
        roots[newRootIndex] = currentLevelHash;
        
        nextIndex = _nextIndex + 1;
        
        emit LeafInserted(_leaf, _nextIndex, currentLevelHash);
        return _nextIndex;
    }
    
    /**
     * @notice ตรวจสอบว่า root นี้เคยใช้งานได้หรือไม่
     */
    function isKnownRoot(bytes32 _root) public view returns (bool) {
        if (_root == 0) return false;
        
        uint32 i = currentRootIndex;
        do {
            if (_root == roots[i]) return true;
            if (i == 0) i = ROOT_HISTORY_SIZE - 1;
            else i--;
        } while (i != currentRootIndex);
        
        return false;
    }
    
    /**
     * @notice ดึง root ปัจจุบัน
     */
    function getLastRoot() public view returns (bytes32) {
        return roots[currentRootIndex];
    }
    
    /**
     * @dev Hash function (ใน production ใช้ Poseidon หรือ MiMC)
     */
    function _hashLeftRight(bytes32 _left, bytes32 _right) 
        internal 
        pure 
        returns (bytes32) 
    {
        return keccak256(abi.encodePacked(_left, _right));
    }
}
```

### 82.1.2 Tornado Cash Core Contract (Simplified)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "./MerkleTreeWithHistory.sol";

/**
 * @title TornadoCashSimplified
 * @notice สำเนา Tornado Cash แบบ simplified สำหรับการศึกษา
 * @dev ในการใช้งานจริง:
 *      - ใช้ Groth16 ZK-SNARK verifier
 *      - Hash function ต้องเป็น ZK-friendly (Poseidon)
 *      - Verifier contract สร้างจาก circuit (circom)
 */

// Interface สำหรับ ZK Proof Verifier
interface IVerifier {
    function verifyProof(
        uint256[2] calldata a,
        uint256[2][2] calldata b,
        uint256[2] calldata c,
        uint256[4] calldata input
    ) external view returns (bool);
}

contract TornadoCashSimplified is MerkleTreeWithHistory {
    
    // จำนวน ETH ต่อ deposit (ต้องเป็นจำนวนคงที่เพื่อ anonymity set)
    uint256 public immutable denomination;
    
    // Verifier contract (สร้างจาก ZK circuit)
    IVerifier public immutable verifier;
    
    // nullifierHash => ใช้ไปแล้วหรือไม่ (ป้องกัน double-spend)
    mapping(bytes32 => bool) public nullifierHashes;
    
    // commitment => ใส่แล้วหรือไม่
    mapping(bytes32 => bool) public commitments;
    
    // ค่าธรรมเนียมสำหรับ relayer
    uint256 public constant RELAYER_FEE_LIMIT = 1e17; // 0.1 ETH max fee
    
    event Deposit(bytes32 indexed commitment, uint32 leafIndex, uint256 timestamp);
    event Withdrawal(
        address to, 
        bytes32 nullifierHash, 
        address indexed relayer, 
        uint256 fee
    );
    
    error AlreadyCommitted();
    error InvalidProof();
    error NullifierAlreadySpent();
    error UnknownRoot();
    error InvalidRecipient();
    error FeeTooHigh();
    
    constructor(
        address _verifier,
        uint256 _denomination
    ) MerkleTreeWithHistory() {
        verifier = IVerifier(_verifier);
        denomination = _denomination;
    }
    
    /**
     * @notice ฝาก ETH
     * @param _commitment = Poseidon(nullifier, secret)
     *        คำนวณ off-chain ด้วย circomlib
     */
    function deposit(bytes32 _commitment) external payable {
        if (commitments[_commitment]) revert AlreadyCommitted();
        if (msg.value != denomination) revert(); // ต้องส่งตรงตามจำนวน
        
        commitments[_commitment] = true;
        uint32 insertedIndex = _insert(_commitment);
        
        emit Deposit(_commitment, insertedIndex, block.timestamp);
    }
    
    /**
     * @notice ถอน ETH โดยใช้ ZK Proof
     * @param _proof ZK proof ที่สร้างจาก circuit
     * @param _root Merkle root ที่ proof อ้างอิง
     * @param _nullifierHash hash ของ nullifier (ป้องกัน double-spend)
     * @param _recipient ที่อยู่ที่รับ ETH
     * @param _relayer ที่อยู่ relayer (ถ้ามี)
     * @param _fee ค่าธรรมเนียม relayer
     * @param _refund ETH refund สำหรับ recipient
     */
    function withdraw(
        bytes calldata _proof,
        bytes32 _root,
        bytes32 _nullifierHash,
        address payable _recipient,
        address payable _relayer,
        uint256 _fee,
        uint256 _refund
    ) external payable {
        if (!isKnownRoot(_root)) revert UnknownRoot();
        if (nullifierHashes[_nullifierHash]) revert NullifierAlreadySpent();
        if (_recipient == address(0)) revert InvalidRecipient();
        if (_fee > denomination) revert FeeTooHigh();
        if (_fee > RELAYER_FEE_LIMIT) revert FeeTooHigh();
        
        // Decode proof
        (
            uint256[2] memory a,
            uint256[2][2] memory b,
            uint256[2] memory c
        ) = abi.decode(_proof, (uint256[2], uint256[2][2], uint256[2]));
        
        // Public inputs: root, nullifierHash, recipient, relayer, fee, refund
        uint256[4] memory inputs = [
            uint256(_root),
            uint256(_nullifierHash),
            uint256(uint160(address(_recipient))),
            _fee
        ];
        
        // ตรวจสอบ ZK Proof
        if (!verifier.verifyProof(a, b, c, inputs)) revert InvalidProof();
        
        // บันทึก nullifier (ป้องกันใช้ซ้ำ)
        nullifierHashes[_nullifierHash] = true;
        
        uint256 sendAmount = denomination - _fee;
        
        // โอน ETH ให้ recipient
        (bool success, ) = _recipient.call{value: sendAmount}("");
        require(success, "ETH transfer failed");
        
        // จ่ายค่าธรรมเนียม relayer
        if (_fee > 0 && _relayer != address(0)) {
            (bool relaySuccess, ) = _relayer.call{value: _fee}("");
            require(relaySuccess, "Relayer fee transfer failed");
        }
        
        emit Withdrawal(_recipient, _nullifierHash, _relayer, _fee);
    }
    
    /**
     * @notice ตรวจสอบว่า note นี้ยังใช้ได้หรือไม่
     */
    function isSpent(bytes32 _nullifierHash) external view returns (bool) {
        return nullifierHashes[_nullifierHash];
    }
    
    /**
     * @notice ตรวจสอบหลาย nullifier พร้อมกัน
     */
    function isSpentArray(
        bytes32[] calldata _nullifierHashes
    ) external view returns (bool[] memory spent) {
        spent = new bool[](_nullifierHashes.length);
        for (uint256 i = 0; i < _nullifierHashes.length; i++) {
            spent[i] = nullifierHashes[_nullifierHashes[i]];
        }
    }
}
```

### 82.1.3 Note Generation (Off-Chain JavaScript)

```javascript
// ตัวอย่าง off-chain code สำหรับสร้าง Tornado Cash note
// ต้องใช้ circomlib และ snarkjs

const { utils } = require("ffjavascript");
const { buildPoseidon } = require("circomlibjs");

async function generateNote(currency, amount, netId) {
    const poseidon = await buildPoseidon();
    
    // สร้าง random nullifier และ secret
    const nullifier = utils.leBuff2int(crypto.getRandomValues(new Uint8Array(31)));
    const secret = utils.leBuff2int(crypto.getRandomValues(new Uint8Array(31)));
    
    // คำนวณ commitment = Poseidon(nullifier, secret)
    const commitment = poseidon([nullifier, secret]);
    const commitmentHex = "0x" + poseidon.F.toString(commitment, 16).padStart(64, "0");
    
    // คำนวณ nullifierHash = Poseidon(nullifier, isSpending=1, treeIndex=0)
    const nullifierHash = poseidon([nullifier, 1, 0]);
    const nullifierHashHex = "0x" + poseidon.F.toString(nullifierHash, 16).padStart(64, "0");
    
    // เข้ารหัส note (เก็บ secret ไว้ใช้ถอน)
    const note = Buffer.concat([
        toBuffer(nullifier, 31),
        toBuffer(secret, 31)
    ]);
    
    const noteString = `tornado-${currency}-${amount}-${netId}-` + 
                      note.toString("hex");
    
    return {
        noteString,
        commitment: commitmentHex,
        nullifierHash: nullifierHashHex
    };
}

// ใช้งาน:
// const note = await generateNote("eth", "0.1", 1);
// await tornado.deposit(note.commitment, { value: ethers.parseEther("0.1") });
// เก็บ note.noteString ไว้อย่างปลอดภัย!
```

---

## 82.2 Stealth Addresses: ERC-5564

### แนวคิด

**Stealth Address** คือ one-time address ที่สร้างขึ้นเฉพาะสำหรับ transaction หนึ่งๆ ผู้รับสามารถ scan เพื่อค้นหาว่า transaction ไหนถูกส่งมาให้ตน โดยไม่ต้องเปิดเผย address ถาวรของตัวเอง

**Diffie-Hellman Key Exchange สำหรับ Stealth Addresses:**

```
ผู้รับ Bob:
  - private key: b (สุ่ม)
  - public key: B = b * G (บน elliptic curve)
  - publishing key: p (อีกคู่)
  - publishing public key: P = p * G

ผู้ส่ง Alice:
  1. สร้าง ephemeral key: r (สุ่ม)
  2. คำนวณ shared secret: s = hash(r * B) = hash(r * b * G)
  3. สร้าง stealth address: S = s*G + P (หรือ P + s*G)
  4. ประกาศ ephemeral public key: R = r * G

Bob scan:
  1. ดู R จาก announcement
  2. คำนวณ: s = hash(b * R) = hash(b * r * G) = hash(r * b * G) ✓
  3. คำนวณ stealth address: S = s*G + P
  4. ถ้า S มี ETH/token → เป็นของ Bob!
  5. Private key ของ S: b + s (Bob ควบคุมได้)
```

### 82.2.1 ERC-5564 Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title IERC5564Announcer
 * @notice ERC-5564: Stealth Meta-Addresses
 * @dev Standard interface สำหรับ stealth address announcements
 */
interface IERC5564Announcer {
    /**
     * @notice ประกาศการโอนไปยัง stealth address
     * @param schemeId รหัสของ stealth address scheme (0 = secp256k1)
     * @param stealthAddress ที่อยู่ stealth ที่ถูกสร้าง
     * @param ephemeralPubKey ephemeral public key ของผู้ส่ง
     * @param metadata ข้อมูลเพิ่มเติม (view tag + token info)
     */
    function announce(
        uint256 schemeId,
        address indexed stealthAddress,
        bytes calldata ephemeralPubKey,
        bytes calldata metadata
    ) external;
    
    event Announcement(
        uint256 indexed schemeId,
        address indexed stealthAddress,
        address indexed caller,
        bytes ephemeralPubKey,
        bytes metadata
    );
}

/**
 * @title IERC5564Registry
 * @notice Registry สำหรับ stealth meta-addresses
 */
interface IERC5564Registry {
    
    event StealthMetaAddressSet(
        address indexed registrant,
        uint256 indexed schemeId,
        bytes stealthMetaAddress
    );
    
    /**
     * @notice ลงทะเบียน stealth meta-address ของตัวเอง
     */
    function registerKeys(
        uint256 schemeId, 
        bytes calldata stealthMetaAddress
    ) external;
    
    /**
     * @notice ลงทะเบียนให้คนอื่นโดยใช้ signature
     */
    function registerKeysOnBehalf(
        address registrant,
        uint256 schemeId,
        bytes calldata signature,
        bytes calldata stealthMetaAddress
    ) external;
    
    /**
     * @notice ดึง stealth meta-address ของ address นั้น
     */
    function stealthMetaAddressOf(
        address registrant,
        uint256 schemeId
    ) external view returns (bytes memory);
}
```

### 82.2.2 ERC-5564 Full Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "./IERC5564.sol";

/**
 * @title ERC5564Announcer
 * @notice Implementation ของ ERC-5564 Announcer
 */
contract ERC5564Announcer is IERC5564Announcer {
    
    /**
     * @notice ประกาศ stealth address transaction
     * @dev ใครก็ได้สามารถเรียกฟังก์ชันนี้
     *      (ปกติจะเรียกใน same transaction กับการโอน)
     */
    function announce(
        uint256 schemeId,
        address stealthAddress,
        bytes calldata ephemeralPubKey,
        bytes calldata metadata
    ) external override {
        require(stealthAddress != address(0), "Invalid stealth address");
        require(ephemeralPubKey.length > 0, "Empty ephemeral pubkey");
        
        emit Announcement(
            schemeId,
            stealthAddress,
            msg.sender,
            ephemeralPubKey,
            metadata
        );
    }
}

/**
 * @title ERC5564Registry
 * @notice Registry สำหรับ stealth meta-addresses
 */
contract ERC5564Registry is IERC5564Registry {
    
    // registrant => schemeId => stealthMetaAddress
    mapping(address => mapping(uint256 => bytes)) private _stealthMetaAddresses;
    
    // Domain separator สำหรับ EIP-712 signature
    bytes32 public immutable DOMAIN_SEPARATOR;
    
    bytes32 public constant REGISTER_TYPEHASH = keccak256(
        "RegisterKeys(address registrant,uint256 schemeId,bytes stealthMetaAddress,uint256 nonce)"
    );
    
    mapping(address => uint256) public nonces;
    
    constructor() {
        DOMAIN_SEPARATOR = keccak256(
            abi.encode(
                keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
                keccak256("ERC5564Registry"),
                keccak256("1"),
                block.chainid,
                address(this)
            )
        );
    }
    
    /**
     * @notice ลงทะเบียน stealth meta-address ของตัวเอง
     * @param schemeId scheme ID (0 = secp256k1+ECDH)
     * @param stealthMetaAddress encoded public keys
     */
    function registerKeys(
        uint256 schemeId,
        bytes calldata stealthMetaAddress
    ) external override {
        _setStealthMetaAddress(msg.sender, schemeId, stealthMetaAddress);
    }
    
    /**
     * @notice ลงทะเบียนให้คนอื่นโดยใช้ signature (EIP-712)
     */
    function registerKeysOnBehalf(
        address registrant,
        uint256 schemeId,
        bytes calldata signature,
        bytes calldata stealthMetaAddress
    ) external override {
        // สร้าง digest
        bytes32 digest = keccak256(
            abi.encodePacked(
                "\x19\x01",
                DOMAIN_SEPARATOR,
                keccak256(abi.encode(
                    REGISTER_TYPEHASH,
                    registrant,
                    schemeId,
                    keccak256(stealthMetaAddress),
                    nonces[registrant]++
                ))
            )
        );
        
        // ตรวจสอบ signature
        address recovered = _recover(digest, signature);
        require(recovered == registrant, "Invalid signature");
        
        _setStealthMetaAddress(registrant, schemeId, stealthMetaAddress);
    }
    
    function stealthMetaAddressOf(
        address registrant,
        uint256 schemeId
    ) external view override returns (bytes memory) {
        bytes memory meta = _stealthMetaAddresses[registrant][schemeId];
        require(meta.length > 0, "No stealth meta-address registered");
        return meta;
    }
    
    function _setStealthMetaAddress(
        address registrant,
        uint256 schemeId,
        bytes calldata stealthMetaAddress
    ) internal {
        require(stealthMetaAddress.length > 0, "Empty meta-address");
        _stealthMetaAddresses[registrant][schemeId] = stealthMetaAddress;
        emit StealthMetaAddressSet(registrant, schemeId, stealthMetaAddress);
    }
    
    function _recover(
        bytes32 _hash, 
        bytes memory _sig
    ) internal pure returns (address) {
        require(_sig.length == 65, "Invalid signature length");
        bytes32 r;
        bytes32 s;
        uint8 v;
        assembly {
            r := mload(add(_sig, 32))
            s := mload(add(_sig, 64))
            v := byte(0, mload(add(_sig, 96)))
        }
        return ecrecover(_hash, v, r, s);
    }
}
```

### 82.2.3 Stealth Address Helper (Off-Chain Computation)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title StealthAddressHelper
 * @notice On-chain helper สำหรับ stealth address computation
 * @dev การ compute จริงทำ off-chain แต่ contract นี้ช่วยใน validation
 *      และ stealth ETH/token transfers
 */
contract StealthAddressHelper {
    
    // ERC-5564 Announcer
    address public immutable announcer;
    
    // View tag length (1 byte สำหรับ fast scanning)
    uint8 constant VIEW_TAG_LENGTH = 1;
    
    constructor(address _announcer) {
        announcer = _announcer;
    }
    
    /**
     * @notice โอน ETH ไปยัง stealth address พร้อมประกาศ
     * @param _stealthAddress ที่อยู่ stealth ที่คำนวณ off-chain
     * @param _ephemeralPubKey ephemeral public key ของผู้ส่ง
     * @param _viewTag view tag (1 byte สำหรับ fast scanning)
     */
    function sendEthToStealthAddress(
        address payable _stealthAddress,
        bytes calldata _ephemeralPubKey,
        bytes1 _viewTag
    ) external payable {
        require(msg.value > 0, "No ETH sent");
        require(_stealthAddress != address(0), "Invalid stealth address");
        
        // โอน ETH
        (bool success, ) = _stealthAddress.call{value: msg.value}("");
        require(success, "ETH transfer failed");
        
        // Metadata: view tag + ETH marker
        bytes memory metadata = abi.encodePacked(
            _viewTag,
            bytes1(0xEE), // ETH marker
            bytes11(0),   // padding
            uint128(msg.value)
        );
        
        // ประกาศผ่าน ERC-5564
        IERC5564Announcer(announcer).announce(
            0, // schemeId: secp256k1
            _stealthAddress,
            _ephemeralPubKey,
            metadata
        );
    }
    
    /**
     * @notice โอน ERC-20 ไปยัง stealth address พร้อมประกาศ
     */
    function sendTokenToStealthAddress(
        address _token,
        address _stealthAddress,
        uint256 _amount,
        bytes calldata _ephemeralPubKey,
        bytes1 _viewTag
    ) external {
        require(_amount > 0, "Zero amount");
        
        // โอน token
        (bool success, bytes memory data) = _token.call(
            abi.encodeWithSignature(
                "transferFrom(address,address,uint256)",
                msg.sender,
                _stealthAddress,
                _amount
            )
        );
        require(success && (data.length == 0 || abi.decode(data, (bool))), 
                "Token transfer failed");
        
        // Metadata: view tag + token address + amount
        bytes memory metadata = abi.encodePacked(
            _viewTag,
            bytes1(0x00), // ERC-20 marker
            bytes11(0),
            uint128(_amount)
        );
        
        IERC5564Announcer(announcer).announce(
            0,
            _stealthAddress,
            _ephemeralPubKey,
            metadata
        );
    }
}
```

---

## 82.3 Confidential Transactions (แนวคิด)

### Pedersen Commitments

**Confidential Transactions** ซ่อนจำนวน token แต่ยังพิสูจน์ได้ว่า:
1. ยอดคงเหลือไม่ติดลบ
2. Input = Output (conservation)

```
Commitment: C = r*H + v*G
  r = random blinding factor
  v = จำนวนจริง
  G, H = generator points บน elliptic curve

พิสูจน์ว่า sum inputs = sum outputs:
  ΣC_in = ΣC_out
  Σ(r_in*H + v_in*G) = Σ(r_out*H + v_out*G)
  ∴ Σv_in = Σv_out (conservation)
  Σr_in = Σr_out (blinding factors cancel)
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title PedersenCommitment
 * @notice Simplified Pedersen Commitment สำหรับการศึกษา
 * @dev ในการใช้งานจริงต้องใช้ elliptic curve operations
 *      และ range proofs (Bulletproofs)
 */
contract PedersenCommitment {
    
    // ใน production ใช้ precompile หรือ library สำหรับ EC operations
    // ตัวอย่างนี้ใช้ simplified hash-based commitment
    
    struct Commitment {
        bytes32 value;    // commitment value
        bool isRevealed;
        uint256 amount;   // เปิดเผยเมื่อ reveal
    }
    
    mapping(bytes32 => Commitment) public commitments;
    
    /**
     * @notice สร้าง commitment (simplified)
     * @param _blinder random blinding factor
     * @param _amount จำนวนจริง (ซ่อนอยู่ใน commitment)
     */
    function createCommitment(
        bytes32 _blinder,
        uint256 _amount
    ) external pure returns (bytes32 commitment) {
        // Simplified: ในการใช้งานจริงต้องเป็น C = r*H + v*G
        commitment = keccak256(abi.encodePacked(_blinder, _amount));
    }
    
    /**
     * @notice ตรวจสอบว่า commitments balance กัน
     * @dev C_in = C_out → amounts balance
     */
    function verifyBalance(
        bytes32[] calldata _inputCommitments,
        bytes32[] calldata _outputCommitments,
        bytes32[] calldata _inputBlinders,
        bytes32[] calldata _outputBlinders,
        uint256[] calldata _inputAmounts,
        uint256[] calldata _outputAmounts
    ) external pure returns (bool) {
        require(
            _inputCommitments.length == _inputBlinders.length &&
            _inputCommitments.length == _inputAmounts.length,
            "Input array mismatch"
        );
        require(
            _outputCommitments.length == _outputBlinders.length &&
            _outputCommitments.length == _outputAmounts.length,
            "Output array mismatch"
        );
        
        // ตรวจสอบ commitments ถูกต้อง
        for (uint i = 0; i < _inputCommitments.length; i++) {
            bytes32 expected = keccak256(
                abi.encodePacked(_inputBlinders[i], _inputAmounts[i])
            );
            if (_inputCommitments[i] != expected) return false;
        }
        
        for (uint i = 0; i < _outputCommitments.length; i++) {
            bytes32 expected = keccak256(
                abi.encodePacked(_outputBlinders[i], _outputAmounts[i])
            );
            if (_outputCommitments[i] != expected) return false;
        }
        
        // ตรวจสอบ conservation
        uint256 totalIn = 0;
        uint256 totalOut = 0;
        
        for (uint i = 0; i < _inputAmounts.length; i++) {
            totalIn += _inputAmounts[i];
        }
        for (uint i = 0; i < _outputAmounts.length; i++) {
            totalOut += _outputAmounts[i];
        }
        
        return totalIn == totalOut;
    }
}
```

---

## 82.4 Privacy-Preserving Voting

### BallotBox: Commit-Reveal Voting

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import "@openzeppelin/contracts/utils/cryptography/MessageHashUtils.sol";

/**
 * @title BallotBox
 * @notice ระบบลงคะแนนแบบ commit-reveal เพื่อปกป้องความเป็นส่วนตัว
 *         ในระหว่างช่วงลงคะแนน ไม่มีใครรู้ว่าใครโหวตอะไร
 * 
 * กระบวนการ:
 * Phase 1 (Commit): ส่ง hash ของคะแนนพร้อม salt
 * Phase 2 (Reveal): เปิดเผยคะแนนจริงและ salt
 * Phase 3 (Tally):  นับผล
 */
contract BallotBox is Ownable {
    using ECDSA for bytes32;
    using MessageHashUtils for bytes32;
    
    // ============ Structs ============
    
    struct Proposal {
        string description;
        uint256 voteCount;
        uint256 commitCount;
        uint256 revealCount;
    }
    
    struct VoterState {
        bool isEligible;
        bool hasCommitted;
        bool hasRevealed;
        bytes32 commitment;    // hash ของคะแนน
        uint256 weight;        // น้ำหนักของคะแนน (1 person 1 vote หรือ token-weighted)
    }
    
    enum Phase {
        Registration,  // ลงทะเบียนผู้มีสิทธิ์
        Commit,        // ส่ง commitment
        Reveal,        // เปิดเผยคะแนน
        Finalized      // นับผลแล้ว
    }
    
    // ============ State ============
    
    Proposal[] public proposals;
    mapping(address => VoterState) public voters;
    
    Phase public currentPhase;
    
    uint256 public commitDeadline;
    uint256 public revealDeadline;
    
    uint256 public totalEligibleVoters;
    uint256 public totalCommits;
    uint256 public totalReveals;
    
    uint256 public winningProposalId;
    bool public isFinalized;
    
    // Snapshot block สำหรับ token-weighted voting
    uint256 public snapshotBlock;
    
    // ============ Events ============
    
    event VoterRegistered(address indexed voter, uint256 weight);
    event CommitMade(address indexed voter, bytes32 commitment);
    event VoteRevealed(address indexed voter, uint256 proposalId, uint256 weight);
    event ResultFinalized(uint256 indexed winningProposal, uint256 voteCount);
    event PhaseChanged(Phase newPhase);
    
    // ============ Errors ============
    
    error WrongPhase(Phase current, Phase required);
    error AlreadyCommitted();
    error AlreadyRevealed();
    error NotEligible();
    error InvalidReveal();
    error InvalidProposalId();
    error CommitDeadlinePassed();
    error RevealDeadlinePassed();
    
    // ============ Constructor ============
    
    constructor(
        string[] memory _proposalDescriptions,
        uint256 _commitDuration,
        uint256 _revealDuration
    ) Ownable(msg.sender) {
        require(_proposalDescriptions.length >= 2, "Need at least 2 proposals");
        require(_proposalDescriptions.length <= 100, "Too many proposals");
        
        for (uint256 i = 0; i < _proposalDescriptions.length; i++) {
            proposals.push(Proposal({
                description: _proposalDescriptions[i],
                voteCount: 0,
                commitCount: 0,
                revealCount: 0
            }));
        }
        
        currentPhase = Phase.Registration;
        snapshotBlock = block.number;
    }
    
    // ============ Admin Functions ============
    
    /**
     * @notice เพิ่มผู้มีสิทธิ์ลงคะแนน
     * @param _voter ที่อยู่ของผู้ลงคะแนน
     * @param _weight น้ำหนักคะแนน (ปกติ = 1)
     */
    function registerVoter(address _voter, uint256 _weight) external onlyOwner {
        if (currentPhase != Phase.Registration) 
            revert WrongPhase(currentPhase, Phase.Registration);
        require(!voters[_voter].isEligible, "Already registered");
        require(_weight > 0, "Weight must be positive");
        
        voters[_voter] = VoterState({
            isEligible: true,
            hasCommitted: false,
            hasRevealed: false,
            commitment: bytes32(0),
            weight: _weight
        });
        
        totalEligibleVoters++;
        emit VoterRegistered(_voter, _weight);
    }
    
    /**
     * @notice เพิ่มหลาย voters พร้อมกัน
     */
    function registerVotersBatch(
        address[] calldata _voters,
        uint256[] calldata _weights
    ) external onlyOwner {
        require(_voters.length == _weights.length, "Array mismatch");
        for (uint256 i = 0; i < _voters.length; i++) {
            if (!voters[_voters[i]].isEligible) {
                voters[_voters[i]] = VoterState({
                    isEligible: true,
                    hasCommitted: false,
                    hasRevealed: false,
                    commitment: bytes32(0),
                    weight: _weights[i]
                });
                totalEligibleVoters++;
                emit VoterRegistered(_voters[i], _weights[i]);
            }
        }
    }
    
    /**
     * @notice เปิด commit phase
     */
    function startCommitPhase(uint256 _duration) external onlyOwner {
        if (currentPhase != Phase.Registration)
            revert WrongPhase(currentPhase, Phase.Registration);
        
        currentPhase = Phase.Commit;
        commitDeadline = block.timestamp + _duration;
        
        emit PhaseChanged(Phase.Commit);
    }
    
    /**
     * @notice เปิด reveal phase
     */
    function startRevealPhase(uint256 _duration) external onlyOwner {
        if (currentPhase != Phase.Commit)
            revert WrongPhase(currentPhase, Phase.Commit);
        
        currentPhase = Phase.Reveal;
        revealDeadline = block.timestamp + _duration;
        
        emit PhaseChanged(Phase.Reveal);
    }
    
    // ============ Voter Functions ============
    
    /**
     * @notice ส่ง commitment (Phase: Commit)
     * @param _commitment = keccak256(abi.encodePacked(proposalId, salt, voterAddress))
     * @dev commitment ซ่อนคะแนนจริง ไม่มีใครรู้ว่าโหวตอะไร
     */
    function commit(bytes32 _commitment) external {
        if (currentPhase != Phase.Commit)
            revert WrongPhase(currentPhase, Phase.Commit);
        if (block.timestamp > commitDeadline) revert CommitDeadlinePassed();
        
        VoterState storage voter = voters[msg.sender];
        if (!voter.isEligible) revert NotEligible();
        if (voter.hasCommitted) revert AlreadyCommitted();
        
        voter.commitment = _commitment;
        voter.hasCommitted = true;
        
        proposals[0].commitCount++; // นับรวม
        totalCommits++;
        
        emit CommitMade(msg.sender, _commitment);
    }
    
    /**
     * @notice เปิดเผยคะแนน (Phase: Reveal)
     * @param _proposalId ID ของ proposal ที่โหวต
     * @param _salt salt ที่ใช้ตอน commit
     */
    function reveal(uint256 _proposalId, bytes32 _salt) external {
        if (currentPhase != Phase.Reveal)
            revert WrongPhase(currentPhase, Phase.Reveal);
        if (block.timestamp > revealDeadline) revert RevealDeadlinePassed();
        
        VoterState storage voter = voters[msg.sender];
        if (!voter.isEligible) revert NotEligible();
        if (!voter.hasCommitted) revert(); // ต้อง commit ก่อน
        if (voter.hasRevealed) revert AlreadyRevealed();
        if (_proposalId >= proposals.length) revert InvalidProposalId();
        
        // ตรวจสอบว่า commitment ตรงกับที่ commit ไว้
        bytes32 expectedCommitment = keccak256(
            abi.encodePacked(_proposalId, _salt, msg.sender)
        );
        if (voter.commitment != expectedCommitment) revert InvalidReveal();
        
        // บันทึกการโหวต
        voter.hasRevealed = true;
        proposals[_proposalId].voteCount += voter.weight;
        proposals[_proposalId].revealCount++;
        totalReveals++;
        
        emit VoteRevealed(msg.sender, _proposalId, voter.weight);
    }
    
    /**
     * @notice นับผลและประกาศผู้ชนะ
     */
    function finalizeVote() external {
        if (currentPhase != Phase.Reveal)
            revert WrongPhase(currentPhase, Phase.Reveal);
        require(
            block.timestamp > revealDeadline || totalReveals == totalCommits,
            "Reveal phase not ended"
        );
        
        uint256 maxVotes = 0;
        uint256 winnerId = 0;
        bool tie = false;
        
        for (uint256 i = 0; i < proposals.length; i++) {
            if (proposals[i].voteCount > maxVotes) {
                maxVotes = proposals[i].voteCount;
                winnerId = i;
                tie = false;
            } else if (proposals[i].voteCount == maxVotes && maxVotes > 0) {
                tie = true;
            }
        }
        
        winningProposalId = winnerId;
        isFinalized = true;
        currentPhase = Phase.Finalized;
        
        emit ResultFinalized(winnerId, maxVotes);
        emit PhaseChanged(Phase.Finalized);
    }
    
    // ============ View Functions ============
    
    function getProposalCount() external view returns (uint256) {
        return proposals.length;
    }
    
    function getProposal(uint256 _id) external view returns (
        string memory description,
        uint256 voteCount,
        uint256 commitCount,
        uint256 revealCount
    ) {
        require(_id < proposals.length, "Invalid proposal");
        Proposal memory p = proposals[_id];
        return (p.description, p.voteCount, p.commitCount, p.revealCount);
    }
    
    function getVoterState(address _voter) external view returns (
        bool isEligible,
        bool hasCommitted,
        bool hasRevealed,
        uint256 weight
    ) {
        VoterState memory v = voters[_voter];
        return (v.isEligible, v.hasCommitted, v.hasRevealed, v.weight);
    }
    
    /**
     * @notice Helper: สร้าง commitment hash
     */
    function createCommitment(
        uint256 _proposalId,
        bytes32 _salt
    ) external view returns (bytes32) {
        return keccak256(abi.encodePacked(_proposalId, _salt, msg.sender));
    }
    
    /**
     * @notice ดูสถิติ participation
     */
    function getParticipationStats() external view returns (
        uint256 eligible,
        uint256 committed,
        uint256 revealed,
        uint256 commitRate,    // percentage * 100
        uint256 revealRate     // percentage * 100
    ) {
        eligible = totalEligibleVoters;
        committed = totalCommits;
        revealed = totalReveals;
        commitRate = eligible > 0 ? (committed * 10000) / eligible : 0;
        revealRate = committed > 0 ? (revealed * 10000) / committed : 0;
    }
}
```

---

## 82.5 Aztec Connect Architecture Overview

### แนวคิดหลัก

**Aztec** คือ ZK-rollup ที่ออกแบบมาเพื่อ **privacy** โดยเฉพาะ ใช้ทั้ง:
- **PLONK proof system**: ZK-SNARK ที่ไม่ต้องมี trusted setup
- **Account Abstraction**: ทุก account เป็น smart contract
- **Note-based model**: state เก็บเป็น "notes" ไม่ใช่ balances

```
Aztec Architecture:
                                          
User (Private)          Aztec Network          Ethereum L1
─────────────          ─────────────          ───────────
                                               
Private Notes  ◄────   Note Pool (encrypted)
               ─────►                         
                         ┌─────────────┐       
Spend Notes    ─────►   │  Rollup     │ ─────► Rollup Contract
               ◄─────   │  Processor  │       (verifies PLONK)
                         └─────────────┘       
Create Notes              
               ─────►   ZK Proof generation    
```

### Aztec Note Encryption

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title AztecStyleNoteProcessor
 * @notice Simplified version ของ Aztec note processing
 * @dev แสดงแนวคิดหลัก ไม่ใช่ implementation จริง
 */
contract AztecStyleNoteProcessor {
    
    // Note commitment (hash ของ note content)
    struct NoteHash {
        bytes32 value;
        bool exists;
    }
    
    // Nullifier (ป้องกันใช้ note ซ้ำ)
    struct Nullifier {
        bytes32 value;
        bool spent;
    }
    
    // Pending shield (L1 → L2 deposit)
    struct PendingShield {
        bytes32 secretHash;     // hash(secret)
        uint256 amount;
        address token;
        uint32 blockNumber;
    }
    
    mapping(bytes32 => NoteHash) public noteHashTree;
    mapping(bytes32 => bool) public nullifiers;
    mapping(bytes32 => PendingShield) public pendingShields;
    
    uint256 public noteCount;
    
    event NoteCreated(bytes32 indexed noteHash, address indexed token, uint256 amount);
    event NoteSpent(bytes32 indexed nullifier);
    event ShieldPending(bytes32 indexed secretHash, address token, uint256 amount);
    
    /**
     * @notice Shield tokens (ย้ายจาก public L1 → private Aztec)
     * @param _token token address
     * @param _amount จำนวน
     * @param _secretHash hash(secret) ที่จะใช้ claim ใน L2
     */
    function shield(
        address _token,
        uint256 _amount,
        bytes32 _secretHash
    ) external payable {
        // โอน token มาที่ contract
        if (_token == address(0)) {
            require(msg.value == _amount, "ETH amount mismatch");
        } else {
            // ERC-20 transfer
            (bool success,) = _token.call(
                abi.encodeWithSignature(
                    "transferFrom(address,address,uint256)",
                    msg.sender, address(this), _amount
                )
            );
            require(success, "Token transfer failed");
        }
        
        bytes32 shieldKey = keccak256(abi.encodePacked(_secretHash, _token, _amount));
        pendingShields[shieldKey] = PendingShield({
            secretHash: _secretHash,
            amount: _amount,
            token: _token,
            blockNumber: uint32(block.number)
        });
        
        emit ShieldPending(_secretHash, _token, _amount);
    }
    
    /**
     * @notice Unshield tokens (ย้ายจาก private Aztec → public L1)
     * @dev เรียกโดย Rollup processor พร้อม ZK proof
     */
    function unshield(
        address _token,
        uint256 _amount,
        address _recipient,
        bytes32 _nullifier,
        bytes calldata /* _proof */ // ในการใช้งานจริงต้อง verify
    ) external {
        // ในการใช้งานจริง: verify ZK proof ที่ยืนยันว่า
        // ผู้เรียกมี spending key ของ note ที่มีมูลค่า >= _amount
        
        require(!nullifiers[_nullifier], "Note already spent");
        nullifiers[_nullifier] = true;
        
        // โอน token ให้ recipient
        if (_token == address(0)) {
            (bool success,) = _recipient.call{value: _amount}("");
            require(success, "ETH transfer failed");
        } else {
            (bool success,) = _token.call(
                abi.encodeWithSignature(
                    "transfer(address,uint256)",
                    _recipient, _amount
                )
            );
            require(success, "Token transfer failed");
        }
        
        emit NoteSpent(_nullifier);
    }
    
    /**
     * @notice เพิ่ม note hash เข้า tree (เรียกโดย rollup)
     */
    function insertNoteHashes(bytes32[] calldata _noteHashes) external {
        for (uint256 i = 0; i < _noteHashes.length; i++) {
            noteHashTree[_noteHashes[i]] = NoteHash({
                value: _noteHashes[i],
                exists: true
            });
            noteCount++;
            emit NoteCreated(_noteHashes[i], address(0), 0);
        }
    }
}
```

---

## 82.6 Workshop: Privacy Analysis

### ภาระกิจที่ 1: ทดสอบ BallotBox

```javascript
// Workshop test: Privacy-preserving voting

const { ethers } = require("hardhat");

async function testPrivacyVoting() {
    const [deployer, alice, bob, charlie] = await ethers.getSigners();
    
    // Deploy BallotBox
    const BallotBox = await ethers.getContractFactory("BallotBox");
    const ballot = await BallotBox.deploy(
        ["Proposal A: Increase treasury", "Proposal B: Buy back tokens", "Proposal C: Expand team"],
        3600, // 1 hour commit
        3600  // 1 hour reveal
    );
    
    // Register voters
    await ballot.registerVoter(alice.address, 1);
    await ballot.registerVoter(bob.address, 2);   // Bob มี 2 votes
    await ballot.registerVoter(charlie.address, 1);
    
    // Start commit phase
    await ballot.startCommitPhase(3600);
    
    // สร้าง commitments (off-chain)
    const aliceSalt = ethers.randomBytes(32);
    const bobSalt = ethers.randomBytes(32);
    const charlieSalt = ethers.randomBytes(32);
    
    // Alice โหวต Proposal 0 (ซ่อนอยู่ใน hash)
    const aliceCommit = ethers.keccak256(
        ethers.AbiCoder.defaultAbiCoder().encode(
            ["uint256", "bytes32", "address"],
            [0, aliceSalt, alice.address]
        )
    );
    
    // Bob โหวต Proposal 1
    const bobCommit = ethers.keccak256(
        ethers.AbiCoder.defaultAbiCoder().encode(
            ["uint256", "bytes32", "address"],
            [1, bobSalt, bob.address]
        )
    );
    
    // Charlie โหวต Proposal 1
    const charlieCommit = ethers.keccak256(
        ethers.AbiCoder.defaultAbiCoder().encode(
            ["uint256", "bytes32", "address"],
            [1, charlieSalt, charlie.address]
        )
    );
    
    // Commit (ยังไม่มีใครรู้ว่าใครโหวตอะไร)
    await ballot.connect(alice).commit(aliceCommit);
    await ballot.connect(bob).commit(bobCommit);
    await ballot.connect(charlie).commit(charlieCommit);
    
    console.log("Commits done. No one knows votes yet!");
    
    // เลื่อนเวลา → start reveal phase
    await ethers.provider.send("evm_increaseTime", [3601]);
    await ballot.startRevealPhase(3600);
    
    // Reveal (ตอนนี้เปิดเผยคะแนน)
    await ballot.connect(alice).reveal(0, aliceSalt);
    await ballot.connect(bob).reveal(1, bobSalt);
    await ballot.connect(charlie).reveal(1, charlieSalt);
    
    // Finalize
    await ethers.provider.send("evm_increaseTime", [3601]);
    await ballot.finalizeVote();
    
    const winnerID = await ballot.winningProposalId();
    const winner = await ballot.getProposal(winnerID);
    console.log(`Winner: ${winner.description} with ${winner.voteCount} votes`);
    // Proposal B ชนะด้วย 3 votes (Bob=2, Charlie=1)
}
```

### ภาระกิจที่ 2: Stealth Address Scanner

```javascript
// Off-chain stealth address scanner
// ตรวจสอบว่ามี transaction ไหนถูกส่งมาให้เราบ้าง

const { ethers } = require("ethers");
const { buildBabyjub } = require("circomlibjs");

async function scanForMyTransactions(
    announcerAddress,
    mySpendingPrivKey,
    myViewingPrivKey,
    fromBlock,
    toBlock
) {
    const babyjub = await buildBabyjub();
    const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
    
    // คำนวณ public keys จาก private keys
    const spendingPubKey = babyjub.mulPointEscalar(
        babyjub.Base8, 
        BigInt(mySpendingPrivKey)
    );
    const viewingPubKey = babyjub.mulPointEscalar(
        babyjub.Base8, 
        BigInt(myViewingPrivKey)
    );
    
    // ดึง Announcement events
    const announcer = new ethers.Contract(
        announcerAddress,
        ["event Announcement(uint256 indexed schemeId, address indexed stealthAddress, address indexed caller, bytes ephemeralPubKey, bytes metadata)"],
        provider
    );
    
    const events = await announcer.queryFilter(
        announcer.filters.Announcement(),
        fromBlock,
        toBlock
    );
    
    const myTransactions = [];
    
    for (const event of events) {
        const { stealthAddress, ephemeralPubKey, metadata } = event.args;
        
        // ถอดรหัส ephemeral public key
        const epk = decodePublicKey(ephemeralPubKey);
        
        // คำนวณ shared secret ด้วย viewing private key
        const sharedSecret = babyjub.mulPointEscalar(epk, BigInt(myViewingPrivKey));
        const sharedSecretHash = ethers.keccak256(
            ethers.AbiCoder.defaultAbiCoder().encode(
                ["uint256", "uint256"],
                [sharedSecret[0], sharedSecret[1]]
            )
        );
        
        // ตรวจสอบ view tag (byte แรกของ metadata)
        const viewTag = metadata[0];
        const expectedViewTag = parseInt(sharedSecretHash.slice(2, 4), 16);
        
        if (viewTag !== expectedViewTag) continue; // ข้ามถ้า view tag ไม่ตรง
        
        // คำนวณ stealth address จาก shared secret + spending key
        const stealthPrivKey = (
            BigInt(mySpendingPrivKey) + BigInt(sharedSecretHash)
        ) % babyjub.order;
        
        const stealthPubKey = babyjub.mulPointEscalar(babyjub.Base8, stealthPrivKey);
        const expectedAddress = pubKeyToAddress(stealthPubKey);
        
        if (expectedAddress.toLowerCase() === stealthAddress.toLowerCase()) {
            console.log(`Found my transaction at ${stealthAddress}!`);
            myTransactions.push({
                stealthAddress,
                stealthPrivKey: stealthPrivKey.toString(16),
                txHash: event.transactionHash,
                blockNumber: event.blockNumber
            });
        }
    }
    
    return myTransactions;
}
```

---

## สรุป Part 82

- **Tornado Cash** ใช้ Merkle Tree + ZK Proofs เพื่อซ่อนการเชื่อมระหว่าง deposit และ withdrawal address
- **Nullifier** เป็นกลไกสำคัญที่ป้องกัน double-spending โดยไม่เปิดเผยว่า note ไหนถูกใช้
- **ERC-5564 Stealth Addresses** ให้ผู้ส่งสร้าง one-time address สำหรับผู้รับแต่ละคน โดยไม่ต้องเปิดเผย address ถาวร
- **Commit-Reveal Voting** ให้ privacy ระหว่างช่วงลงคะแนน แต่ต้องการ reveal เพื่อนับผล (ถ้าไม่ reveal คะแนนไม่นับ)
- **Aztec** เป็น privacy-first ZK-rollup ที่ใช้ note-based model แยก private state จาก public state อย่างสิ้นเชิง
- Privacy ใน DeFi ยังคงเป็น **unsolved problem** อย่างสมบูรณ์ ต้องแลกกับ compliance, gas costs, และ complexity

## Next: Part 83 - Intent-Based Architecture (UniswapX, 1inch Fusion, ERC-7683)
