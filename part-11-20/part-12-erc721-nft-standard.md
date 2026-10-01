# Part 12: ERC-721 NFT Standard

## สารบัญ
1. ERC-721 คืออะไร
2. ERC-721 Interface
3. Implementation สมบูรณ์
4. Metadata และ IPFS
5. NFT Minting Patterns
6. Royalties (EIP-2981)
7. On-chain Metadata (SVG)
8. Workshop: NFT Collection

---

## 1. ERC-721 คืออะไร

ERC-721 คือมาตรฐาน **Non-Fungible Token (NFT)** - token ที่ไม่สามารถแทนกันได้

```
Fungible (ERC-20):    1 ETH = 1 ETH (เหมือนกัน)
Non-Fungible (ERC-721): TokenId #1 ≠ TokenId #2 (ต่างกัน)
```

**Use cases:**
- Digital Art (CryptoPunks, Bored Apes)
- Gaming Items (Axie Infinity)
- Domain Names (ENS)
- Real Estate NFTs
- Event Tickets
- Certificates/Credentials
- Music Rights

---

## 2. ERC-721 Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface IERC165 {
    function supportsInterface(bytes4 interfaceId) external view returns (bool);
}

interface IERC721 is IERC165 {
    
    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);
    
    function balanceOf(address owner) external view returns (uint256 balance);
    function ownerOf(uint256 tokenId) external view returns (address owner);
    
    // Safe transfer - calls onERC721Received on receiver
    function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId,
        bytes calldata data
    ) external;
    
    function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId
    ) external;
    
    function transferFrom(address from, address to, uint256 tokenId) external;
    
    function approve(address to, uint256 tokenId) external;
    function setApprovalForAll(address operator, bool approved) external;
    function getApproved(uint256 tokenId) external view returns (address operator);
    function isApprovedForAll(address owner, address operator) external view returns (bool);
}

interface IERC721Metadata is IERC721 {
    function name() external view returns (string memory);
    function symbol() external view returns (string memory);
    function tokenURI(uint256 tokenId) external view returns (string memory);
}

// Receiver interface สำหรับ contract ที่รับ NFT
interface IERC721Receiver {
    function onERC721Received(
        address operator,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external returns (bytes4);
}
```

---

## 3. ERC-721 Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract ERC721 is IERC721Metadata {
    
    using Strings for uint256;
    
    string private _name;
    string private _symbol;
    
    mapping(uint256 tokenId => address) private _owners;
    mapping(address owner => uint256) private _balances;
    mapping(uint256 tokenId => address) private _tokenApprovals;
    mapping(address owner => mapping(address operator => bool)) private _operatorApprovals;
    
    // ERC-165 interface IDs
    bytes4 private constant ERC721_INTERFACE_ID = 0x80ac58cd;
    bytes4 private constant ERC721_METADATA_INTERFACE_ID = 0x5b5e139f;
    bytes4 private constant ERC165_INTERFACE_ID = 0x01ffc9a7;
    
    // Custom Errors
    error ERC721InvalidOwner(address owner);
    error ERC721NonexistentToken(uint256 tokenId);
    error ERC721IncorrectOwner(address sender, uint256 tokenId, address owner);
    error ERC721InvalidSender(address sender);
    error ERC721InvalidReceiver(address receiver);
    error ERC721InsufficientApproval(address operator, uint256 tokenId);
    error ERC721InvalidApprover(address approver);
    error ERC721InvalidOperator(address operator);
    
    constructor(string memory name_, string memory symbol_) {
        _name = name_;
        _symbol = symbol_;
    }
    
    function supportsInterface(bytes4 interfaceId) public view virtual override returns (bool) {
        return 
            interfaceId == ERC721_INTERFACE_ID ||
            interfaceId == ERC721_METADATA_INTERFACE_ID ||
            interfaceId == ERC165_INTERFACE_ID;
    }
    
    function name() public view virtual returns (string memory) { return _name; }
    function symbol() public view virtual returns (string memory) { return _symbol; }
    
    function tokenURI(uint256 tokenId) public view virtual returns (string memory) {
        _requireOwned(tokenId);
        string memory baseURI = _baseURI();
        return bytes(baseURI).length > 0 
            ? string.concat(baseURI, tokenId.toString()) 
            : "";
    }
    
    function _baseURI() internal view virtual returns (string memory) { return ""; }
    
    function balanceOf(address owner) public view virtual returns (uint256) {
        if (owner == address(0)) revert ERC721InvalidOwner(address(0));
        return _balances[owner];
    }
    
    function ownerOf(uint256 tokenId) public view virtual returns (address) {
        return _requireOwned(tokenId);
    }
    
    function getApproved(uint256 tokenId) public view virtual returns (address) {
        _requireOwned(tokenId);
        return _getApproved(tokenId);
    }
    
    function _getApproved(uint256 tokenId) internal view virtual returns (address) {
        return _tokenApprovals[tokenId];
    }
    
    function isApprovedForAll(address owner, address operator) public view virtual returns (bool) {
        return _operatorApprovals[owner][operator];
    }
    
    function approve(address to, uint256 tokenId) public virtual {
        _approve(to, tokenId, msg.sender);
    }
    
    function setApprovalForAll(address operator, bool approved) public virtual {
        _setApprovalForAll(msg.sender, operator, approved);
    }
    
    function transferFrom(address from, address to, uint256 tokenId) public virtual {
        if (to == address(0)) revert ERC721InvalidReceiver(address(0));
        address previousOwner = _update(to, tokenId, msg.sender);
        if (previousOwner != from) revert ERC721IncorrectOwner(from, tokenId, previousOwner);
    }
    
    function safeTransferFrom(address from, address to, uint256 tokenId) public virtual {
        safeTransferFrom(from, to, tokenId, "");
    }
    
    function safeTransferFrom(address from, address to, uint256 tokenId, bytes memory data) public virtual {
        transferFrom(from, to, tokenId);
        _checkOnERC721Received(from, to, tokenId, data);
    }
    
    function _ownerOf(uint256 tokenId) internal view virtual returns (address) {
        return _owners[tokenId];
    }
    
    function _requireOwned(uint256 tokenId) internal view returns (address) {
        address owner = _ownerOf(tokenId);
        if (owner == address(0)) revert ERC721NonexistentToken(tokenId);
        return owner;
    }
    
    function _update(address to, uint256 tokenId, address auth) internal virtual returns (address) {
        address from = _ownerOf(tokenId);
        
        if (auth != address(0)) {
            _checkAuthorized(from, auth, tokenId);
        }
        
        if (from != address(0)) {
            _approve(address(0), tokenId, address(0), false);
            unchecked { _balances[from] -= 1; }
        }
        
        if (to != address(0)) {
            unchecked { _balances[to] += 1; }
        }
        
        _owners[tokenId] = to;
        
        emit Transfer(from, to, tokenId);
        
        return from;
    }
    
    function _mint(address to, uint256 tokenId) internal virtual {
        if (to == address(0)) revert ERC721InvalidReceiver(address(0));
        address previousOwner = _update(to, tokenId, address(0));
        if (previousOwner != address(0)) revert ERC721InvalidSender(address(0));
    }
    
    function _safeMint(address to, uint256 tokenId) internal virtual {
        _safeMint(to, tokenId, "");
    }
    
    function _safeMint(address to, uint256 tokenId, bytes memory data) internal virtual {
        _mint(to, tokenId);
        _checkOnERC721Received(address(0), to, tokenId, data);
    }
    
    function _burn(uint256 tokenId) internal virtual {
        address previousOwner = _update(address(0), tokenId, address(0));
        if (previousOwner == address(0)) revert ERC721NonexistentToken(tokenId);
    }
    
    function _transfer(address from, address to, uint256 tokenId) internal virtual {
        if (to == address(0)) revert ERC721InvalidReceiver(address(0));
        address previousOwner = _update(to, tokenId, address(0));
        if (previousOwner == address(0)) revert ERC721NonexistentToken(tokenId);
        if (previousOwner != from) revert ERC721IncorrectOwner(from, tokenId, previousOwner);
    }
    
    function _approve(address to, uint256 tokenId, address auth) internal virtual {
        _approve(to, tokenId, auth, true);
    }
    
    function _approve(address to, uint256 tokenId, address auth, bool emitEvent) internal virtual {
        if (emitEvent || auth != address(0)) {
            address owner = _requireOwned(tokenId);
            
            if (auth != address(0) && owner != auth && !isApprovedForAll(owner, auth)) {
                revert ERC721InvalidApprover(auth);
            }
            
            if (emitEvent) {
                emit Approval(owner, to, tokenId);
            }
        }
        
        _tokenApprovals[tokenId] = to;
    }
    
    function _setApprovalForAll(address owner, address operator, bool approved) internal virtual {
        if (operator == address(0)) revert ERC721InvalidOperator(address(0));
        _operatorApprovals[owner][operator] = approved;
        emit ApprovalForAll(owner, operator, approved);
    }
    
    function _checkAuthorized(address owner, address spender, uint256 tokenId) internal view virtual {
        if (!_isAuthorized(owner, spender, tokenId)) {
            if (owner == address(0)) {
                revert ERC721NonexistentToken(tokenId);
            } else {
                revert ERC721InsufficientApproval(spender, tokenId);
            }
        }
    }
    
    function _isAuthorized(address owner, address spender, uint256 tokenId) internal view virtual returns (bool) {
        return spender != address(0) && (
            owner == spender ||
            isApprovedForAll(owner, spender) ||
            _getApproved(tokenId) == spender
        );
    }
    
    function _checkOnERC721Received(
        address from,
        address to,
        uint256 tokenId,
        bytes memory data
    ) private {
        if (to.code.length > 0) {
            try IERC721Receiver(to).onERC721Received(msg.sender, from, tokenId, data) returns (bytes4 retval) {
                if (retval != IERC721Receiver.onERC721Received.selector) {
                    revert ERC721InvalidReceiver(to);
                }
            } catch (bytes memory reason) {
                if (reason.length == 0) {
                    revert ERC721InvalidReceiver(to);
                } else {
                    assembly { revert(add(32, reason), mload(reason)) }
                }
            }
        }
    }
}

library Strings {
    function toString(uint256 value) internal pure returns (string memory) {
        if (value == 0) return "0";
        
        uint256 temp = value;
        uint256 digits;
        
        while (temp != 0) { digits++; temp /= 10; }
        
        bytes memory buffer = new bytes(digits);
        
        while (value != 0) {
            digits -= 1;
            buffer[digits] = bytes1(uint8(48 + uint256(value % 10)));
            value /= 10;
        }
        
        return string(buffer);
    }
    
    function toHexString(uint256 value) internal pure returns (string memory) {
        if (value == 0) return "0x00";
        
        uint256 temp = value;
        uint256 length = 0;
        
        while (temp != 0) { length++; temp >>= 8; }
        
        return toHexString(value, length);
    }
    
    function toHexString(uint256 value, uint256 length) internal pure returns (string memory) {
        bytes memory buffer = new bytes(2 * length + 2);
        buffer[0] = "0";
        buffer[1] = "x";
        
        for (uint256 i = 2 * length + 1; i > 1; --i) {
            buffer[i] = "0123456789abcdef"[value & 0xf];
            value >>= 4;
        }
        
        require(value == 0, "Hex length insufficient");
        return string(buffer);
    }
}
```

---

## 4. Royalties (EIP-2981)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface IERC2981 is IERC165 {
    function royaltyInfo(
        uint256 tokenId,
        uint256 salePrice
    ) external view returns (address receiver, uint256 royaltyAmount);
}

abstract contract ERC2981 is IERC2981 {
    
    struct RoyaltyInfo {
        address receiver;
        uint96 royaltyFraction; // basis points (100 = 1%)
    }
    
    RoyaltyInfo private _defaultRoyaltyInfo;
    mapping(uint256 tokenId => RoyaltyInfo) private _tokenRoyaltyInfo;
    
    error ERC2981InvalidDefaultRoyalty(uint256 numerator, uint256 denominator);
    error ERC2981InvalidDefaultRoyaltyReceiver(address receiver);
    error ERC2981InvalidTokenRoyalty(uint256 tokenId, uint256 numerator, uint256 denominator);
    error ERC2981InvalidTokenRoyaltyReceiver(uint256 tokenId, address receiver);
    
    function supportsInterface(bytes4 interfaceId) public view virtual override returns (bool) {
        return interfaceId == type(IERC2981).interfaceId || super.supportsInterface(interfaceId);
    }
    
    function royaltyInfo(uint256 tokenId, uint256 salePrice) public view virtual override returns (
        address,
        uint256
    ) {
        RoyaltyInfo memory royalty = _tokenRoyaltyInfo[tokenId];
        
        if (royalty.receiver == address(0)) {
            royalty = _defaultRoyaltyInfo;
        }
        
        uint256 royaltyAmount = (salePrice * royalty.royaltyFraction) / _feeDenominator();
        
        return (royalty.receiver, royaltyAmount);
    }
    
    function _feeDenominator() internal pure virtual returns (uint96) {
        return 10000; // basis points
    }
    
    function _setDefaultRoyalty(address receiver, uint96 feeNumerator) internal virtual {
        uint256 denominator = _feeDenominator();
        if (feeNumerator > denominator) {
            revert ERC2981InvalidDefaultRoyalty(feeNumerator, denominator);
        }
        if (receiver == address(0)) {
            revert ERC2981InvalidDefaultRoyaltyReceiver(address(0));
        }
        
        _defaultRoyaltyInfo = RoyaltyInfo(receiver, feeNumerator);
    }
    
    function _deleteDefaultRoyalty() internal virtual {
        delete _defaultRoyaltyInfo;
    }
    
    function _setTokenRoyalty(uint256 tokenId, address receiver, uint96 feeNumerator) internal virtual {
        uint256 denominator = _feeDenominator();
        if (feeNumerator > denominator) {
            revert ERC2981InvalidTokenRoyalty(tokenId, feeNumerator, denominator);
        }
        if (receiver == address(0)) {
            revert ERC2981InvalidTokenRoyaltyReceiver(tokenId, address(0));
        }
        
        _tokenRoyaltyInfo[tokenId] = RoyaltyInfo(receiver, feeNumerator);
    }
    
    function _resetTokenRoyalty(uint256 tokenId) internal virtual {
        delete _tokenRoyaltyInfo[tokenId];
    }
}
```

---

## 5. Workshop: NFT Collection

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Base64} from "./Base64.sol";

/**
 * @title PixelArtNFT
 * @dev NFT Collection ด้วย on-chain metadata และ SVG
 */
contract PixelArtNFT is ERC721, ERC2981, Ownable {
    
    using Strings for uint256;
    
    uint256 public constant MAX_SUPPLY = 10000;
    uint256 public constant PRICE = 0.05 ether;
    uint256 public constant MAX_PER_WALLET = 10;
    
    uint256 private _nextTokenId;
    bool public mintOpen;
    
    mapping(uint256 => uint256) private _tokenSeeds;
    mapping(address => uint256) public mintedPerWallet;
    
    string[] private _colorPalette = [
        "#FF6B6B", "#FFA07A", "#FFD700", "#90EE90",
        "#87CEEB", "#9370DB", "#FF69B4", "#20B2AA"
    ];
    
    string[] private _rarities = ["Common", "Uncommon", "Rare", "Epic", "Legendary"];
    
    event Minted(address indexed minter, uint256 indexed tokenId, uint256 seed);
    
    error MintClosed();
    error MaxSupplyReached();
    error MaxPerWalletReached();
    error InsufficientPayment();
    error WithdrawFailed();
    
    constructor() 
        ERC721("PixelArt NFT", "PXART")
        Ownable(msg.sender)
    {
        _setDefaultRoyalty(msg.sender, 500); // 5% royalty
    }
    
    // === Minting ===
    
    function mint(uint256 quantity) external payable {
        if (!mintOpen) revert MintClosed();
        if (_nextTokenId + quantity > MAX_SUPPLY) revert MaxSupplyReached();
        if (mintedPerWallet[msg.sender] + quantity > MAX_PER_WALLET) revert MaxPerWalletReached();
        if (msg.value < PRICE * quantity) revert InsufficientPayment();
        
        mintedPerWallet[msg.sender] += quantity;
        
        for (uint256 i = 0; i < quantity; i++) {
            uint256 tokenId = _nextTokenId++;
            uint256 seed = uint256(keccak256(abi.encodePacked(
                block.prevrandao,
                block.timestamp,
                msg.sender,
                tokenId,
                i
            )));
            _tokenSeeds[tokenId] = seed;
            _safeMint(msg.sender, tokenId);
            emit Minted(msg.sender, tokenId, seed);
        }
    }
    
    // Owner mint สำหรับ reserved tokens
    function ownerMint(address to, uint256 quantity) external onlyOwner {
        require(_nextTokenId + quantity <= MAX_SUPPLY, "Max supply");
        
        for (uint256 i = 0; i < quantity; i++) {
            uint256 tokenId = _nextTokenId++;
            _tokenSeeds[tokenId] = uint256(keccak256(abi.encodePacked("owner", tokenId)));
            _safeMint(to, tokenId);
        }
    }
    
    // === Metadata ===
    
    function tokenURI(uint256 tokenId) public view override returns (string memory) {
        _requireOwned(tokenId);
        
        string memory svg = _generateSVG(tokenId);
        string memory attributes = _generateAttributes(tokenId);
        
        string memory json = string.concat(
            '{"name":"PixelArt #', tokenId.toString(), '",',
            '"description":"On-chain generative pixel art NFT",',
            '"image":"data:image/svg+xml;base64,', Base64.encode(bytes(svg)), '",',
            '"attributes":', attributes, '}'
        );
        
        return string.concat(
            "data:application/json;base64,",
            Base64.encode(bytes(json))
        );
    }
    
    function _generateSVG(uint256 tokenId) internal view returns (string memory) {
        uint256 seed = _tokenSeeds[tokenId];
        
        string memory pixels = "";
        uint8 size = 8; // 8x8 grid
        
        for (uint8 y = 0; y < size; y++) {
            for (uint8 x = 0; x < size; x++) {
                uint256 colorIndex = (seed >> ((y * size + x) * 4)) % _colorPalette.length;
                string memory color = _colorPalette[colorIndex];
                
                pixels = string.concat(
                    pixels,
                    '<rect x="', (x * 32).toString(), '" y="', (y * 32).toString(),
                    '" width="32" height="32" fill="', color, '"/>'
                );
            }
        }
        
        return string.concat(
            '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 256 256">',
            '<rect width="256" height="256" fill="#000"/>',
            pixels,
            '</svg>'
        );
    }
    
    function _generateAttributes(uint256 tokenId) internal view returns (string memory) {
        uint256 seed = _tokenSeeds[tokenId];
        
        uint256 rarityIndex = _getRarityIndex(seed);
        uint256 power = 10 + (seed % 91);
        uint256 speed = 10 + ((seed >> 8) % 91);
        uint256 luck = 10 + ((seed >> 16) % 91);
        
        return string.concat(
            '[',
            '{"trait_type":"Rarity","value":"', _rarities[rarityIndex], '"},',
            '{"trait_type":"Power","value":', power.toString(), '},',
            '{"trait_type":"Speed","value":', speed.toString(), '},',
            '{"trait_type":"Luck","value":', luck.toString(), '}',
            ']'
        );
    }
    
    function _getRarityIndex(uint256 seed) internal pure returns (uint256) {
        uint256 roll = seed % 1000;
        if (roll < 500) return 0; // Common 50%
        if (roll < 750) return 1; // Uncommon 25%
        if (roll < 900) return 2; // Rare 15%
        if (roll < 975) return 3; // Epic 7.5%
        return 4;                  // Legendary 2.5%
    }
    
    // === Admin ===
    
    function openMint() external onlyOwner { mintOpen = true; }
    function closeMint() external onlyOwner { mintOpen = false; }
    
    function setDefaultRoyalty(address receiver, uint96 feeNumerator) external onlyOwner {
        _setDefaultRoyalty(receiver, feeNumerator);
    }
    
    function withdraw() external onlyOwner {
        uint256 balance = address(this).balance;
        (bool success,) = owner().call{value: balance}("");
        if (!success) revert WithdrawFailed();
    }
    
    function totalSupply() external view returns (uint256) {
        return _nextTokenId;
    }
    
    function supportsInterface(bytes4 interfaceId) 
        public view override(ERC721, ERC2981) returns (bool) 
    {
        return super.supportsInterface(interfaceId);
    }
    
    function owner() public view returns (address) { return _owner; }
    address private _owner;
}

// Base64 Library
library Base64 {
    bytes internal constant TABLE = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";
    
    function encode(bytes memory data) internal pure returns (string memory) {
        if (data.length == 0) return "";
        
        uint256 encodedLen = 4 * ((data.length + 2) / 3);
        bytes memory result = new bytes(encodedLen + 32);
        
        bytes memory table = TABLE;
        
        assembly {
            let tablePtr := add(table, 1)
            let resultPtr := add(result, 32)
            
            for {
                let dataPtr := data
                let endPtr := add(data, mload(data))
            } lt(dataPtr, endPtr) {} {
                dataPtr := add(dataPtr, 3)
                let input := mload(dataPtr)
                
                mstore8(resultPtr, mload(add(tablePtr, and(shr(18, input), 0x3F))))
                resultPtr := add(resultPtr, 1)
                mstore8(resultPtr, mload(add(tablePtr, and(shr(12, input), 0x3F))))
                resultPtr := add(resultPtr, 1)
                mstore8(resultPtr, mload(add(tablePtr, and(shr(6, input), 0x3F))))
                resultPtr := add(resultPtr, 1)
                mstore8(resultPtr, mload(add(tablePtr, and(input, 0x3F))))
                resultPtr := add(resultPtr, 1)
            }
            
            switch mod(mload(data), 3)
            case 1 {
                mstore(sub(resultPtr, 2), shl(240, 0x3d3d))
            }
            case 2 {
                mstore(sub(resultPtr, 1), shl(248, 0x3d))
            }
            
            mstore(result, encodedLen)
        }
        
        return string(result);
    }
}
```

---

## 6. TypeScript Tests

```typescript
// test/PixelArtNFT.test.ts
import { expect } from "chai";
import { ethers } from "hardhat";
import { PixelArtNFT } from "../typechain-types";
import { Signer } from "ethers";

describe("PixelArtNFT", function () {
  let nft: PixelArtNFT;
  let owner: Signer;
  let alice: Signer;
  let bob: Signer;
  
  const PRICE = ethers.parseEther("0.05");
  
  beforeEach(async function () {
    [owner, alice, bob] = await ethers.getSigners();
    
    const NFT = await ethers.getContractFactory("PixelArtNFT");
    nft = await NFT.deploy();
  });
  
  describe("Minting", function () {
    beforeEach(async function () {
      await nft.openMint();
    });
    
    it("should mint NFT with correct payment", async function () {
      await nft.connect(alice).mint(1, { value: PRICE });
      expect(await nft.balanceOf(await alice.getAddress())).to.equal(1);
      expect(await nft.ownerOf(0)).to.equal(await alice.getAddress());
    });
    
    it("should revert when mint is closed", async function () {
      await nft.closeMint();
      await expect(nft.connect(alice).mint(1, { value: PRICE }))
        .to.be.revertedWithCustomError(nft, "MintClosed");
    });
    
    it("should revert with insufficient payment", async function () {
      await expect(nft.connect(alice).mint(1, { value: ethers.parseEther("0.01") }))
        .to.be.revertedWithCustomError(nft, "InsufficientPayment");
    });
    
    it("should enforce max per wallet", async function () {
      await nft.connect(alice).mint(10, { value: PRICE * 10n });
      
      await expect(nft.connect(alice).mint(1, { value: PRICE }))
        .to.be.revertedWithCustomError(nft, "MaxPerWalletReached");
    });
    
    it("should generate on-chain metadata", async function () {
      await nft.connect(alice).mint(1, { value: PRICE });
      
      const uri = await nft.tokenURI(0);
      expect(uri).to.include("data:application/json;base64,");
      
      // Decode base64
      const base64Data = uri.replace("data:application/json;base64,", "");
      const decoded = Buffer.from(base64Data, "base64").toString();
      const metadata = JSON.parse(decoded);
      
      expect(metadata.name).to.include("PixelArt #0");
      expect(metadata.image).to.include("data:image/svg+xml;base64,");
      expect(metadata.attributes).to.have.length(4);
    });
  });
  
  describe("Royalties", function () {
    it("should return correct royalty info", async function () {
      const salePrice = ethers.parseEther("1");
      const [receiver, royaltyAmount] = await nft.royaltyInfo(0, salePrice);
      
      expect(receiver).to.equal(await owner.getAddress());
      expect(royaltyAmount).to.equal(salePrice * 500n / 10000n); // 5%
    });
  });
  
  describe("Transfers", function () {
    beforeEach(async function () {
      await nft.openMint();
      await nft.connect(alice).mint(1, { value: PRICE });
    });
    
    it("should transfer NFT", async function () {
      const aliceAddr = await alice.getAddress();
      const bobAddr = await bob.getAddress();
      
      await nft.connect(alice).transferFrom(aliceAddr, bobAddr, 0);
      
      expect(await nft.ownerOf(0)).to.equal(bobAddr);
      expect(await nft.balanceOf(aliceAddr)).to.equal(0);
      expect(await nft.balanceOf(bobAddr)).to.equal(1);
    });
    
    it("should approve and transfer", async function () {
      const aliceAddr = await alice.getAddress();
      const bobAddr = await bob.getAddress();
      
      await nft.connect(alice).approve(bobAddr, 0);
      expect(await nft.getApproved(0)).to.equal(bobAddr);
      
      await nft.connect(bob).transferFrom(aliceAddr, bobAddr, 0);
      expect(await nft.ownerOf(0)).to.equal(bobAddr);
    });
    
    it("should setApprovalForAll", async function () {
      const aliceAddr = await alice.getAddress();
      const bobAddr = await bob.getAddress();
      
      await nft.connect(alice).setApprovalForAll(bobAddr, true);
      expect(await nft.isApprovedForAll(aliceAddr, bobAddr)).to.be.true;
      
      // Bob can transfer all Alice's tokens
      await nft.connect(bob).transferFrom(aliceAddr, bobAddr, 0);
      expect(await nft.ownerOf(0)).to.equal(bobAddr);
    });
  });
});
```

---

## สรุป Part 12

ERC-721 NFT Standard ที่เรียนรู้:
- ✅ ERC-721 interface และ events
- ✅ Full ERC-721 implementation
- ✅ ERC-165 supportsInterface
- ✅ EIP-2981 Royalty Standard
- ✅ On-chain SVG metadata
- ✅ Generative NFT with randomness
- ✅ Minting patterns

## Quiz

1. `safeTransferFrom` ต่างจาก `transferFrom` อย่างไร?
2. ERC-165 ใช้สำหรับอะไร?
3. On-chain metadata มีข้อดีกว่า IPFS อย่างไร?
4. ทำไม `prevrandao` ถึงไม่ปลอดภัยสำหรับ randomness?

---

## Next: Part 13 - ERC-1155 Multi-Token Standard
