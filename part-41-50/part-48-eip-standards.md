# Part 48: EIP Standards Deep Dive

## สารบัญ
1. Critical EIPs Timeline
2. EIP-2612 Permit
3. EIP-712 Typed Signatures
4. EIP-4626 Vault Standard
5. Workshop: Multi-EIP Contract

---

## 1. Critical EIPs Timeline

```
EIPs (Ethereum Improvement Proposals) สำคัญ:

Token Standards:
- EIP-20 (ERC-20): Fungible tokens
- EIP-721 (ERC-721): NFTs  
- EIP-1155 (ERC-1155): Multi-token
- EIP-4626: Tokenized vault standard
- EIP-3643: Security tokens

Signature/Auth:
- EIP-191: Signed data standard
- EIP-712: Typed structured data signing
- EIP-2612: Permit (gasless approve)
- EIP-1271: Contract signature validation

Account Abstraction:
- EIP-4337: AA via EntryPoint
- EIP-7702: Set EOA code (Pectra)
- EIP-3074: AUTH/AUTHCALL opcodes

Gas Optimization:
- EIP-1559: Fee market reform
- EIP-2929: Gas cost for access lists
- EIP-2930: Access list transactions
- EIP-3855: PUSH0 opcode
- EIP-1153: Transient storage

Protocol:
- EIP-1967: Standard proxy storage slots
- EIP-2535 (Diamond): Multi-facet proxy
- EIP-7201: Namespaced storage (for upgradeable)
```

---

## 2. EIP-2612 Permit (Full Implementation)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * EIP-2612 Permit:
 * Gasless ERC-20 approval via signature
 * 
 * User signs permit off-chain → spender submits on-chain
 * User pays 0 gas for approval!
 */
abstract contract ERC20Permit {
    
    // EIP-712 domain
    bytes32 public DOMAIN_SEPARATOR;
    
    // keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)")
    bytes32 public constant PERMIT_TYPEHASH = 
        0x6e71edae12b1b97f4d1f60370fef10105fa2faae0126114a169c64845d6126c9;
    
    mapping(address => uint256) public nonces;
    
    error PermitExpired();
    error InvalidSignature();
    
    constructor(string memory name, string memory version) {
        DOMAIN_SEPARATOR = keccak256(abi.encode(
            keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
            keccak256(bytes(name)),
            keccak256(bytes(version)),
            block.chainid,
            address(this)
        ));
    }
    
    function permit(
        address owner,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external {
        if (deadline < block.timestamp) revert PermitExpired();
        
        bytes32 structHash = keccak256(abi.encode(
            PERMIT_TYPEHASH,
            owner,
            spender,
            value,
            nonces[owner]++, // Increment nonce
            deadline
        ));
        
        bytes32 hash = keccak256(abi.encodePacked(
            "\x19\x01",
            DOMAIN_SEPARATOR,
            structHash
        ));
        
        address signer = ecrecover(hash, v, r, s);
        
        if (signer == address(0) || signer != owner) revert InvalidSignature();
        
        _approve(owner, spender, value);
    }
    
    // Must be implemented by token
    function _approve(address owner, address spender, uint256 value) internal virtual;
}

/**
 * Full ERC-20 with Permit
 */
contract PermitToken is ERC20Permit {
    
    string public name;
    string public symbol;
    uint8 public constant decimals = 18;
    uint256 public totalSupply;
    
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    constructor(string memory _name, string memory _symbol) 
        ERC20Permit(_name, "1") 
    {
        name = _name;
        symbol = _symbol;
    }
    
    function _approve(address owner, address spender, uint256 value) internal override {
        allowance[owner][spender] = value;
        emit Approval(owner, spender, value);
    }
    
    function transfer(address to, uint256 amount) external returns (bool) {
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        uint256 currentAllowance = allowance[from][msg.sender];
        if (currentAllowance != type(uint256).max) {
            allowance[from][msg.sender] = currentAllowance - amount;
        }
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        emit Transfer(from, to, amount);
        return true;
    }
    
    function approve(address spender, uint256 amount) external returns (bool) {
        _approve(msg.sender, spender, amount);
        return true;
    }
}

/**
 * Usage: Gasless DEX deposit
 */
contract PermitDEX {
    
    function depositWithPermit(
        address token,
        uint256 amount,
        uint256 deadline,
        uint8 v, bytes32 r, bytes32 s
    ) external {
        // No need for user to call approve() first!
        // User signs permit off-chain, we submit here
        PermitToken(token).permit(
            msg.sender, // owner
            address(this), // spender
            amount,
            deadline,
            v, r, s
        );
        
        // Now transfer (allowance was set by permit)
        PermitToken(token).transferFrom(msg.sender, address(this), amount);
        
        // Execute deposit logic...
    }
}
```

---

## 3. EIP-1271 Contract Signature Validation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * EIP-1271: Standard Signature Validation for Contracts
 * 
 * Smart wallets (Gnosis Safe, etc.) can't sign with ECDSA
 * EIP-1271 lets contracts define their own signature validation
 * 
 * Use cases:
 * - Smart wallet approvals
 * - Multi-sig for Permit
 * - AA wallet signatures
 */
interface IERC1271 {
    // Returns this magic value when signature is valid:
    // bytes4(keccak256("isValidSignature(bytes32,bytes)"))
    bytes4 constant MAGIC_VALUE = 0x1626ba7e;
    
    function isValidSignature(
        bytes32 hash,
        bytes calldata signature
    ) external view returns (bytes4 magicValue);
}

/**
 * Gnosis Safe style Multi-sig wallet
 * Implements EIP-1271
 */
contract MultiSigWallet is IERC1271 {
    
    address[] public owners;
    uint256 public threshold;
    
    constructor(address[] memory _owners, uint256 _threshold) {
        require(_threshold > 0 && _threshold <= _owners.length, "Invalid");
        owners = _owners;
        threshold = _threshold;
    }
    
    /**
     * Validate multi-sig: signature is packed bytes of individual sigs
     * Each sig is 65 bytes (r, s, v)
     * Must have >= threshold valid sigs from unique owners
     */
    function isValidSignature(
        bytes32 hash,
        bytes calldata signature
    ) external view override returns (bytes4 magicValue) {
        require(signature.length % 65 == 0, "Invalid sig length");
        
        uint256 sigCount = signature.length / 65;
        require(sigCount >= threshold, "Not enough sigs");
        
        address[] memory signers = new address[](sigCount);
        
        for (uint256 i; i < sigCount; i++) {
            bytes memory sig = signature[i*65 : (i+1)*65];
            
            bytes32 r;
            bytes32 s;
            uint8 v;
            
            assembly {
                r := mload(add(sig, 32))
                s := mload(add(sig, 64))
                v := byte(0, mload(add(sig, 96)))
            }
            
            address signer = ecrecover(hash, v, r, s);
            
            // Must be an owner
            require(_isOwner(signer), "Not owner");
            
            // No duplicate signers
            for (uint256 j; j < i; j++) {
                require(signers[j] != signer, "Duplicate signer");
            }
            
            signers[i] = signer;
        }
        
        return MAGIC_VALUE;
    }
    
    function _isOwner(address addr) internal view returns (bool) {
        for (uint256 i; i < owners.length; i++) {
            if (owners[i] == addr) return true;
        }
        return false;
    }
}

/**
 * Universal Signature Validator
 * Works with both EOA (ECDSA) and contracts (EIP-1271)
 */
library SignatureChecker {
    
    bytes4 constant MAGIC_VALUE = 0x1626ba7e;
    
    function isValidSignature(
        address signer,
        bytes32 hash,
        bytes memory signature
    ) internal view returns (bool) {
        if (signer.code.length == 0) {
            // EOA: ECDSA recovery
            if (signature.length != 65) return false;
            
            bytes32 r;
            bytes32 s;
            uint8 v;
            
            assembly {
                r := mload(add(signature, 32))
                s := mload(add(signature, 64))
                v := byte(0, mload(add(signature, 96)))
            }
            
            address recovered = ecrecover(hash, v, r, s);
            return recovered != address(0) && recovered == signer;
        } else {
            // Contract: EIP-1271
            try IERC1271(signer).isValidSignature(hash, signature) 
                returns (bytes4 magic) {
                return magic == MAGIC_VALUE;
            } catch {
                return false;
            }
        }
    }
}
```

---

## 4. EIP-7201: Namespaced Storage

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * EIP-7201: Namespaced Storage Layouts
 * 
 * สำหรับ upgradeable contracts:
 * ป้องกัน storage collision ระหว่าง parent และ child contracts
 * 
 * แทนที่จะใช้ slot 0, 1, 2... (ซึ่งอาจชนกัน)
 * ใช้ keccak256(namespace) เป็น base slot
 */

/**
 * Namespace pattern สำหรับ upgradeable contracts
 */
abstract contract OwnableUpgradeable {
    
    /// @custom:storage-location erc7201:openzeppelin.storage.Ownable
    struct OwnableStorage {
        address _owner;
        address _pendingOwner;
    }
    
    // keccak256(abi.encode(uint256(keccak256("openzeppelin.storage.Ownable")) - 1)) & ~bytes32(uint256(0xff))
    bytes32 private constant OwnableStorageLocation = 
        0x9016d09d72d40fdae2fd8ceac6b6234c7706214fd39c1cd1e609a0528c199300;
    
    function _getOwnableStorage() private pure returns (OwnableStorage storage $) {
        assembly {
            $.slot := OwnableStorageLocation
        }
    }
    
    function owner() public view returns (address) {
        return _getOwnableStorage()._owner;
    }
    
    function transferOwnership(address newOwner) external {
        require(msg.sender == owner(), "Not owner");
        _getOwnableStorage()._owner = newOwner;
    }
    
    function __Ownable_init(address initialOwner) internal {
        _getOwnableStorage()._owner = initialOwner;
    }
}

abstract contract PausableUpgradeable {
    
    /// @custom:storage-location erc7201:openzeppelin.storage.Pausable
    struct PausableStorage {
        bool _paused;
    }
    
    bytes32 private constant PausableStorageLocation = 
        0xcd5ed15c6e187e77e9aee88184c21f4f2182ab5827cb3b7e07fbedcd63f03300;
    
    function _getPausableStorage() private pure returns (PausableStorage storage $) {
        assembly {
            $.slot := PausableStorageLocation
        }
    }
    
    function paused() public view returns (bool) {
        return _getPausableStorage()._paused;
    }
    
    function _pause() internal {
        _getPausableStorage()._paused = true;
    }
    
    function _unpause() internal {
        _getPausableStorage()._paused = false;
    }
}

/**
 * Upgradeable vault using namespaced storage
 * No storage collision possible!
 */
contract UpgradeableVault is OwnableUpgradeable, PausableUpgradeable {
    
    /// @custom:storage-location erc7201:myprotocol.storage.Vault
    struct VaultStorage {
        mapping(address => uint256) balances;
        uint256 totalAssets;
        address asset;
        uint256 feeRate;
    }
    
    bytes32 private constant VaultStorageLocation = 
        keccak256(abi.encode(uint256(keccak256("myprotocol.storage.Vault")) - 1)) & ~bytes32(uint256(0xff));
    
    function _getVaultStorage() private pure returns (VaultStorage storage $) {
        assembly {
            $.slot := VaultStorageLocation
        }
    }
    
    function initialize(address asset_, address owner_) external {
        __Ownable_init(owner_);
        _getVaultStorage().asset = asset_;
    }
    
    function deposit(uint256 amount) external {
        require(!paused(), "Paused");
        VaultStorage storage $ = _getVaultStorage();
        $.balances[msg.sender] += amount;
        $.totalAssets += amount;
    }
    
    function withdraw(uint256 amount) external {
        require(!paused(), "Paused");
        VaultStorage storage $ = _getVaultStorage();
        $.balances[msg.sender] -= amount;
        $.totalAssets -= amount;
    }
    
    function pause() external {
        require(msg.sender == owner(), "Not owner");
        _pause();
    }
}
```

---

## สรุป Part 48

EIP Standards ที่เรียนรู้:
- ✅ EIP-2612 Permit (gasless approve)
- ✅ EIP-1271 contract signature validation
- ✅ SignatureChecker (EOA + contract support)
- ✅ EIP-7201 namespaced storage
- ✅ Upgradeable contracts with no collision

## Next: Part 49 - Smart Contract Incident Response
