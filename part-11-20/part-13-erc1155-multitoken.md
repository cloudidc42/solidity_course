# Part 13: ERC-1155 Multi-Token Standard

## สารบัญ
1. ERC-1155 คืออะไร
2. Interface และ Events
3. Implementation สมบูรณ์
4. Batch Operations
5. Semi-Fungible Tokens
6. Gaming Use Case
7. Workshop: Game Items Contract

---

## 1. ERC-1155 คืออะไร

ERC-1155 คือมาตรฐาน **Multi-Token** ที่รวม ERC-20 และ ERC-721 ไว้ในตัวเดียว

```
ERC-20:   1 contract = 1 token type (fungible)
ERC-721:  1 contract = 1 token type (non-fungible, unique ids)
ERC-1155: 1 contract = หลาย token types (fungible + non-fungible)

Token ID 1: Gold Coin (fungible) - อาจมี 10,000 ชิ้น
Token ID 2: Silver Sword (semi-fungible) - อาจมี 100 ชิ้น
Token ID 3: Legendary Dragon (non-fungible) - มีชิ้นเดียว
```

**ข้อดีหลักของ ERC-1155:**
- **Gas Efficient**: Batch operations ใน 1 transaction
- **Flexible**: Fungible + Non-fungible ในสัญญาเดียว
- **Gaming**: เหมาะสำหรับ game items
- **Less Storage**: ใช้ mapping แทน ownership tracking แบบ ERC-721

---

## 2. Interface และ Events

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface IERC1155 is IERC165 {
    
    // Events
    event TransferSingle(
        address indexed operator,
        address indexed from,
        address indexed to,
        uint256 id,
        uint256 value
    );
    
    event TransferBatch(
        address indexed operator,
        address indexed from,
        address indexed to,
        uint256[] ids,
        uint256[] values
    );
    
    event ApprovalForAll(
        address indexed account,
        address indexed operator,
        bool approved
    );
    
    event URI(string value, uint256 indexed id);
    
    // Functions
    function balanceOf(address account, uint256 id) external view returns (uint256);
    function balanceOfBatch(address[] calldata accounts, uint256[] calldata ids) external view returns (uint256[] memory);
    function setApprovalForAll(address operator, bool approved) external;
    function isApprovedForAll(address account, address operator) external view returns (bool);
    function safeTransferFrom(address from, address to, uint256 id, uint256 amount, bytes calldata data) external;
    function safeBatchTransferFrom(address from, address to, uint256[] calldata ids, uint256[] calldata amounts, bytes calldata data) external;
}

interface IERC1155Receiver is IERC165 {
    function onERC1155Received(
        address operator,
        address from,
        uint256 id,
        uint256 value,
        bytes calldata data
    ) external returns (bytes4);
    
    function onERC1155BatchReceived(
        address operator,
        address from,
        uint256[] calldata ids,
        uint256[] calldata values,
        bytes calldata data
    ) external returns (bytes4);
}

interface IERC1155MetadataURI is IERC1155 {
    function uri(uint256 id) external view returns (string memory);
}
```

---

## 3. ERC-1155 Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract ERC1155 is IERC1155MetadataURI {
    
    // tokenId => (owner => balance)
    mapping(uint256 id => mapping(address account => uint256)) private _balances;
    
    // owner => operator => approved
    mapping(address account => mapping(address operator => bool)) private _operatorApprovals;
    
    string private _uri;
    
    error ERC1155InvalidOperator(address operator);
    error ERC1155InvalidSender(address sender);
    error ERC1155InvalidReceiver(address receiver);
    error ERC1155MissingApprovalForAll(address operator, address owner);
    error ERC1155InvalidArrayLength(uint256 idsLength, uint256 valuesLength);
    error ERC1155InsufficientBalance(address sender, uint256 balance, uint256 needed, uint256 tokenId);
    
    constructor(string memory uri_) {
        _setURI(uri_);
    }
    
    function supportsInterface(bytes4 interfaceId) public view virtual returns (bool) {
        return
            interfaceId == type(IERC1155).interfaceId ||
            interfaceId == type(IERC1155MetadataURI).interfaceId ||
            interfaceId == type(IERC165).interfaceId;
    }
    
    function uri(uint256 /* id */) public view virtual override returns (string memory) {
        return _uri;
    }
    
    function balanceOf(address account, uint256 id) public view virtual returns (uint256) {
        return _balances[id][account];
    }
    
    function balanceOfBatch(
        address[] memory accounts,
        uint256[] memory ids
    ) public view virtual returns (uint256[] memory) {
        if (accounts.length != ids.length) {
            revert ERC1155InvalidArrayLength(ids.length, accounts.length);
        }
        
        uint256[] memory batchBalances = new uint256[](accounts.length);
        for (uint256 i = 0; i < accounts.length; ++i) {
            batchBalances[i] = balanceOf(accounts[i], ids[i]);
        }
        return batchBalances;
    }
    
    function setApprovalForAll(address operator, bool approved) public virtual {
        _setApprovalForAll(msg.sender, operator, approved);
    }
    
    function isApprovedForAll(address account, address operator) public view virtual returns (bool) {
        return _operatorApprovals[account][operator];
    }
    
    function safeTransferFrom(
        address from,
        address to,
        uint256 id,
        uint256 value,
        bytes memory data
    ) public virtual {
        address sender = msg.sender;
        if (from != sender && !isApprovedForAll(from, sender)) {
            revert ERC1155MissingApprovalForAll(sender, from);
        }
        _safeTransferFrom(from, to, id, value, data);
    }
    
    function safeBatchTransferFrom(
        address from,
        address to,
        uint256[] memory ids,
        uint256[] memory values,
        bytes memory data
    ) public virtual {
        address sender = msg.sender;
        if (from != sender && !isApprovedForAll(from, sender)) {
            revert ERC1155MissingApprovalForAll(sender, from);
        }
        _safeBatchTransferFrom(from, to, ids, values, data);
    }
    
    function _safeTransferFrom(
        address from,
        address to,
        uint256 id,
        uint256 value,
        bytes memory data
    ) internal virtual {
        if (to == address(0)) revert ERC1155InvalidReceiver(address(0));
        if (from == address(0)) revert ERC1155InvalidSender(address(0));
        
        (uint256[] memory ids, uint256[] memory values) = _asSingletonArrays(id, value);
        _updateWithAcceptanceCheck(from, to, ids, values, data);
    }
    
    function _safeBatchTransferFrom(
        address from,
        address to,
        uint256[] memory ids,
        uint256[] memory values,
        bytes memory data
    ) internal virtual {
        if (to == address(0)) revert ERC1155InvalidReceiver(address(0));
        if (from == address(0)) revert ERC1155InvalidSender(address(0));
        _updateWithAcceptanceCheck(from, to, ids, values, data);
    }
    
    function _setURI(string memory newuri) internal virtual {
        _uri = newuri;
    }
    
    function _mint(address to, uint256 id, uint256 value, bytes memory data) internal virtual {
        if (to == address(0)) revert ERC1155InvalidReceiver(address(0));
        (uint256[] memory ids, uint256[] memory values) = _asSingletonArrays(id, value);
        _updateWithAcceptanceCheck(address(0), to, ids, values, data);
    }
    
    function _mintBatch(
        address to,
        uint256[] memory ids,
        uint256[] memory values,
        bytes memory data
    ) internal virtual {
        if (to == address(0)) revert ERC1155InvalidReceiver(address(0));
        _updateWithAcceptanceCheck(address(0), to, ids, values, data);
    }
    
    function _burn(address from, uint256 id, uint256 value) internal virtual {
        if (from == address(0)) revert ERC1155InvalidSender(address(0));
        (uint256[] memory ids, uint256[] memory values) = _asSingletonArrays(id, value);
        _updateWithAcceptanceCheck(from, address(0), ids, values, "");
    }
    
    function _burnBatch(
        address from,
        uint256[] memory ids,
        uint256[] memory values
    ) internal virtual {
        if (from == address(0)) revert ERC1155InvalidSender(address(0));
        _updateWithAcceptanceCheck(from, address(0), ids, values, "");
    }
    
    function _update(
        address from,
        address to,
        uint256[] memory ids,
        uint256[] memory values
    ) internal virtual {
        if (ids.length != values.length) {
            revert ERC1155InvalidArrayLength(ids.length, values.length);
        }
        
        address operator = msg.sender;
        
        for (uint256 i = 0; i < ids.length; ++i) {
            uint256 id = ids[i];
            uint256 value = values[i];
            
            if (from != address(0)) {
                uint256 fromBalance = _balances[id][from];
                if (fromBalance < value) {
                    revert ERC1155InsufficientBalance(from, fromBalance, value, id);
                }
                unchecked { _balances[id][from] = fromBalance - value; }
            }
            
            if (to != address(0)) {
                _balances[id][to] += value;
            }
        }
        
        if (ids.length == 1) {
            uint256 id = ids[0];
            uint256 value = values[0];
            emit TransferSingle(operator, from, to, id, value);
        } else {
            emit TransferBatch(operator, from, to, ids, values);
        }
    }
    
    function _updateWithAcceptanceCheck(
        address from,
        address to,
        uint256[] memory ids,
        uint256[] memory values,
        bytes memory data
    ) internal virtual {
        _update(from, to, ids, values);
        
        if (to != address(0)) {
            address operator = msg.sender;
            if (ids.length == 1) {
                uint256 id = ids[0];
                uint256 value = values[0];
                _doSafeTransferAcceptanceCheck(operator, from, to, id, value, data);
            } else {
                _doSafeBatchTransferAcceptanceCheck(operator, from, to, ids, values, data);
            }
        }
    }
    
    function _setApprovalForAll(address owner, address operator, bool approved) internal virtual {
        if (operator == address(0)) revert ERC1155InvalidOperator(address(0));
        _operatorApprovals[owner][operator] = approved;
        emit ApprovalForAll(owner, operator, approved);
    }
    
    function _doSafeTransferAcceptanceCheck(
        address operator,
        address from,
        address to,
        uint256 id,
        uint256 value,
        bytes memory data
    ) private {
        if (to.code.length > 0) {
            try IERC1155Receiver(to).onERC1155Received(operator, from, id, value, data) returns (bytes4 response) {
                if (response != IERC1155Receiver.onERC1155Received.selector) {
                    revert ERC1155InvalidReceiver(to);
                }
            } catch (bytes memory reason) {
                if (reason.length == 0) {
                    revert ERC1155InvalidReceiver(to);
                } else {
                    assembly { revert(add(32, reason), mload(reason)) }
                }
            }
        }
    }
    
    function _doSafeBatchTransferAcceptanceCheck(
        address operator,
        address from,
        address to,
        uint256[] memory ids,
        uint256[] memory values,
        bytes memory data
    ) private {
        if (to.code.length > 0) {
            try IERC1155Receiver(to).onERC1155BatchReceived(operator, from, ids, values, data) returns (bytes4 response) {
                if (response != IERC1155Receiver.onERC1155BatchReceived.selector) {
                    revert ERC1155InvalidReceiver(to);
                }
            } catch (bytes memory reason) {
                if (reason.length == 0) {
                    revert ERC1155InvalidReceiver(to);
                } else {
                    assembly { revert(add(32, reason), mload(reason)) }
                }
            }
        }
    }
    
    function _asSingletonArrays(
        uint256 element1,
        uint256 element2
    ) private pure returns (uint256[] memory array1, uint256[] memory array2) {
        assembly {
            array1 := mload(0x40)
            mstore(array1, 1)
            mstore(add(array1, 0x20), element1)
            array2 := add(array1, 0x40)
            mstore(array2, 1)
            mstore(add(array2, 0x20), element2)
            mstore(0x40, add(array2, 0x40))
        }
    }
}
```

---

## 4. Workshop: Game Items Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title GameItems
 * @dev ERC-1155 สำหรับ game items พร้อม crafting และ marketplace
 */
contract GameItems is ERC1155, Ownable {
    
    using Strings for uint256;
    
    // Item Categories
    uint8 public constant CATEGORY_CURRENCY = 0;
    uint8 public constant CATEGORY_WEAPON = 1;
    uint8 public constant CATEGORY_ARMOR = 2;
    uint8 public constant CATEGORY_POTION = 3;
    uint8 public constant CATEGORY_MATERIAL = 4;
    
    struct ItemInfo {
        string name;
        uint8 category;
        uint256 maxSupply;  // 0 = unlimited
        uint256 totalMinted;
        bool tradeable;
        bool burnable;
    }
    
    struct CraftingRecipe {
        uint256[] inputIds;
        uint256[] inputAmounts;
        uint256 outputId;
        uint256 outputAmount;
        bool active;
    }
    
    // Item Definitions
    uint256 public constant GOLD = 1;
    uint256 public constant WOOD = 2;
    uint256 public constant IRON = 3;
    uint256 public constant BRONZE_SWORD = 101;
    uint256 public constant IRON_SWORD = 102;
    uint256 public constant STEEL_ARMOR = 201;
    uint256 public constant HEALTH_POTION = 301;
    uint256 public constant MANA_POTION = 302;
    
    mapping(uint256 => ItemInfo) public items;
    mapping(bytes32 => CraftingRecipe) public recipes;
    bytes32[] public recipeIds;
    
    address public gameServer;
    
    event ItemDefined(uint256 indexed id, string name);
    event RecipeAdded(bytes32 indexed recipeId, uint256 outputId);
    event ItemCrafted(address indexed player, uint256 outputId, uint256 outputAmount);
    event ItemBurned(address indexed player, uint256 id, uint256 amount);
    
    error NotGameServer();
    error ItemNotDefined(uint256 id);
    error MaxSupplyReached(uint256 id);
    error NotTradeable(uint256 id);
    error NotBurnable(uint256 id);
    error RecipeNotFound();
    error InsufficientCraftingMaterials(uint256 itemId, uint256 required, uint256 available);
    
    modifier onlyGameServer() {
        if (msg.sender != gameServer && msg.sender != owner()) {
            revert NotGameServer();
        }
        _;
    }
    
    constructor(address _gameServer) 
        ERC1155("https://game.example.com/items/{id}.json")
        Ownable(msg.sender)
    {
        gameServer = _gameServer;
        _initializeItems();
        _initializeRecipes();
    }
    
    function _initializeItems() private {
        _defineItem(GOLD, "Gold", CATEGORY_CURRENCY, 0, true, true);
        _defineItem(WOOD, "Wood", CATEGORY_MATERIAL, 0, true, true);
        _defineItem(IRON, "Iron", CATEGORY_MATERIAL, 0, true, true);
        _defineItem(BRONZE_SWORD, "Bronze Sword", CATEGORY_WEAPON, 10000, true, false);
        _defineItem(IRON_SWORD, "Iron Sword", CATEGORY_WEAPON, 5000, true, false);
        _defineItem(STEEL_ARMOR, "Steel Armor", CATEGORY_ARMOR, 1000, true, false);
        _defineItem(HEALTH_POTION, "Health Potion", CATEGORY_POTION, 0, true, true);
        _defineItem(MANA_POTION, "Mana Potion", CATEGORY_POTION, 0, true, true);
    }
    
    function _initializeRecipes() private {
        // Bronze Sword: 5 Wood + 3 Iron = 1 Bronze Sword
        uint256[] memory inputs1 = new uint256[](2);
        inputs1[0] = WOOD; inputs1[1] = IRON;
        uint256[] memory amounts1 = new uint256[](2);
        amounts1[0] = 5; amounts1[1] = 3;
        _addRecipe("bronze_sword", inputs1, amounts1, BRONZE_SWORD, 1);
        
        // Iron Sword: 2 Bronze Sword + 5 Iron = 1 Iron Sword
        uint256[] memory inputs2 = new uint256[](2);
        inputs2[0] = BRONZE_SWORD; inputs2[1] = IRON;
        uint256[] memory amounts2 = new uint256[](2);
        amounts2[0] = 2; amounts2[1] = 5;
        _addRecipe("iron_sword", inputs2, amounts2, IRON_SWORD, 1);
        
        // Health Potion: 10 Gold = 5 Health Potions
        uint256[] memory inputs3 = new uint256[](1);
        inputs3[0] = GOLD;
        uint256[] memory amounts3 = new uint256[](1);
        amounts3[0] = 10;
        _addRecipe("health_potion", inputs3, amounts3, HEALTH_POTION, 5);
    }
    
    function _defineItem(
        uint256 id,
        string memory name,
        uint8 category,
        uint256 maxSupply,
        bool tradeable,
        bool burnable
    ) private {
        items[id] = ItemInfo({
            name: name,
            category: category,
            maxSupply: maxSupply,
            totalMinted: 0,
            tradeable: tradeable,
            burnable: burnable
        });
        emit ItemDefined(id, name);
    }
    
    function _addRecipe(
        string memory name,
        uint256[] memory inputIds,
        uint256[] memory inputAmounts,
        uint256 outputId,
        uint256 outputAmount
    ) private {
        bytes32 recipeId = keccak256(bytes(name));
        recipes[recipeId] = CraftingRecipe({
            inputIds: inputIds,
            inputAmounts: inputAmounts,
            outputId: outputId,
            outputAmount: outputAmount,
            active: true
        });
        recipeIds.push(recipeId);
        emit RecipeAdded(recipeId, outputId);
    }
    
    // === Minting (Game Server Only) ===
    
    function mintItem(address player, uint256 id, uint256 amount) external onlyGameServer {
        ItemInfo storage item = items[id];
        if (bytes(item.name).length == 0) revert ItemNotDefined(id);
        
        if (item.maxSupply > 0) {
            if (item.totalMinted + amount > item.maxSupply) {
                revert MaxSupplyReached(id);
            }
        }
        
        item.totalMinted += amount;
        _mint(player, id, amount, "");
    }
    
    function mintBatch(
        address player,
        uint256[] calldata ids,
        uint256[] calldata amounts
    ) external onlyGameServer {
        for (uint256 i = 0; i < ids.length; i++) {
            ItemInfo storage item = items[ids[i]];
            if (bytes(item.name).length == 0) revert ItemNotDefined(ids[i]);
            
            if (item.maxSupply > 0) {
                if (item.totalMinted + amounts[i] > item.maxSupply) {
                    revert MaxSupplyReached(ids[i]);
                }
            }
            item.totalMinted += amounts[i];
        }
        _mintBatch(player, ids, amounts, "");
    }
    
    // === Crafting ===
    
    function craft(string calldata recipeName) external {
        bytes32 recipeId = keccak256(bytes(recipeName));
        CraftingRecipe storage recipe = recipes[recipeId];
        
        if (!recipe.active) revert RecipeNotFound();
        
        // ตรวจสอบวัตถุดิบ
        for (uint256 i = 0; i < recipe.inputIds.length; i++) {
            uint256 balance = balanceOf(msg.sender, recipe.inputIds[i]);
            if (balance < recipe.inputAmounts[i]) {
                revert InsufficientCraftingMaterials(
                    recipe.inputIds[i],
                    recipe.inputAmounts[i],
                    balance
                );
            }
        }
        
        // เผาวัตถุดิบ
        for (uint256 i = 0; i < recipe.inputIds.length; i++) {
            _burn(msg.sender, recipe.inputIds[i], recipe.inputAmounts[i]);
        }
        
        // Mint ผลลัพธ์
        ItemInfo storage outputItem = items[recipe.outputId];
        if (outputItem.maxSupply > 0) {
            if (outputItem.totalMinted + recipe.outputAmount > outputItem.maxSupply) {
                revert MaxSupplyReached(recipe.outputId);
            }
        }
        outputItem.totalMinted += recipe.outputAmount;
        _mint(msg.sender, recipe.outputId, recipe.outputAmount, "");
        
        emit ItemCrafted(msg.sender, recipe.outputId, recipe.outputAmount);
    }
    
    // === Burn ===
    
    function burnItem(uint256 id, uint256 amount) external {
        if (!items[id].burnable) revert NotBurnable(id);
        _burn(msg.sender, id, amount);
        emit ItemBurned(msg.sender, id, amount);
    }
    
    // === Override Transfer (check tradeable) ===
    
    function safeTransferFrom(
        address from,
        address to,
        uint256 id,
        uint256 amount,
        bytes memory data
    ) public virtual override {
        if (from != address(0) && !items[id].tradeable) {
            revert NotTradeable(id);
        }
        super.safeTransferFrom(from, to, id, amount, data);
    }
    
    function safeBatchTransferFrom(
        address from,
        address to,
        uint256[] memory ids,
        uint256[] memory amounts,
        bytes memory data
    ) public virtual override {
        for (uint256 i = 0; i < ids.length; i++) {
            if (from != address(0) && !items[ids[i]].tradeable) {
                revert NotTradeable(ids[i]);
            }
        }
        super.safeBatchTransferFrom(from, to, ids, amounts, data);
    }
    
    // === URI ===
    
    function uri(uint256 id) public view override returns (string memory) {
        ItemInfo storage item = items[id];
        if (bytes(item.name).length == 0) revert ItemNotDefined(id);
        
        return string.concat(
            "https://game.example.com/items/",
            id.toString(),
            ".json"
        );
    }
    
    // === Admin ===
    
    function setGameServer(address _gameServer) external onlyOwner {
        gameServer = _gameServer;
    }
    
    function defineItem(
        uint256 id,
        string calldata name,
        uint8 category,
        uint256 maxSupply,
        bool tradeable,
        bool burnable
    ) external onlyOwner {
        _defineItem(id, name, category, maxSupply, tradeable, burnable);
    }
    
    function addRecipe(
        string calldata name,
        uint256[] calldata inputIds,
        uint256[] calldata inputAmounts,
        uint256 outputId,
        uint256 outputAmount
    ) external onlyOwner {
        _addRecipe(name, inputIds, inputAmounts, outputId, outputAmount);
    }
    
    function toggleRecipe(string calldata name, bool active) external onlyOwner {
        bytes32 recipeId = keccak256(bytes(name));
        recipes[recipeId].active = active;
    }
    
    function getItemInfo(uint256 id) external view returns (ItemInfo memory) {
        return items[id];
    }
    
    function getRecipeCount() external view returns (uint256) {
        return recipeIds.length;
    }
    
    function owner() public view returns (address) { return _owner; }
    address private _owner;
}
```

---

## 5. TypeScript Tests

```typescript
// test/GameItems.test.ts
import { expect } from "chai";
import { ethers } from "hardhat";
import { GameItems } from "../typechain-types";
import { Signer } from "ethers";

describe("GameItems", function () {
  let items: GameItems;
  let owner: Signer;
  let gameServer: Signer;
  let player1: Signer;
  let player2: Signer;
  
  // Item IDs
  const GOLD = 1n;
  const WOOD = 2n;
  const IRON = 3n;
  const BRONZE_SWORD = 101n;
  const HEALTH_POTION = 301n;
  
  beforeEach(async function () {
    [owner, gameServer, player1, player2] = await ethers.getSigners();
    
    const GameItems = await ethers.getContractFactory("GameItems");
    items = await GameItems.deploy(await gameServer.getAddress());
  });
  
  describe("Minting", function () {
    it("should mint items as game server", async function () {
      const player1Addr = await player1.getAddress();
      
      await items.connect(gameServer).mintItem(player1Addr, GOLD, 100n);
      
      expect(await items.balanceOf(player1Addr, GOLD)).to.equal(100n);
    });
    
    it("should batch mint items", async function () {
      const player1Addr = await player1.getAddress();
      
      await items.connect(gameServer).mintBatch(
        player1Addr,
        [GOLD, WOOD, IRON],
        [100n, 50n, 20n]
      );
      
      const balances = await items.balanceOfBatch(
        [player1Addr, player1Addr, player1Addr],
        [GOLD, WOOD, IRON]
      );
      
      expect(balances[0]).to.equal(100n);
      expect(balances[1]).to.equal(50n);
      expect(balances[2]).to.equal(20n);
    });
    
    it("should reject minting from non-server", async function () {
      await expect(
        items.connect(player1).mintItem(await player1.getAddress(), GOLD, 100n)
      ).to.be.revertedWithCustomError(items, "NotGameServer");
    });
  });
  
  describe("Crafting", function () {
    beforeEach(async function () {
      const player1Addr = await player1.getAddress();
      // Give player materials for crafting
      await items.connect(gameServer).mintBatch(
        player1Addr,
        [WOOD, IRON, GOLD],
        [100n, 100n, 100n]
      );
    });
    
    it("should craft bronze sword with correct materials", async function () {
      const player1Addr = await player1.getAddress();
      
      const woodBefore = await items.balanceOf(player1Addr, WOOD);
      const ironBefore = await items.balanceOf(player1Addr, IRON);
      
      await items.connect(player1).craft("bronze_sword");
      
      expect(await items.balanceOf(player1Addr, BRONZE_SWORD)).to.equal(1n);
      expect(await items.balanceOf(player1Addr, WOOD)).to.equal(woodBefore - 5n);
      expect(await items.balanceOf(player1Addr, IRON)).to.equal(ironBefore - 3n);
    });
    
    it("should craft health potions", async function () {
      const player1Addr = await player1.getAddress();
      
      await items.connect(player1).craft("health_potion");
      
      expect(await items.balanceOf(player1Addr, HEALTH_POTION)).to.equal(5n);
      expect(await items.balanceOf(player1Addr, GOLD)).to.equal(90n); // 100 - 10
    });
    
    it("should fail crafting with insufficient materials", async function () {
      // Give player only 2 wood (needs 5)
      const player2Addr = await player2.getAddress();
      await items.connect(gameServer).mintBatch(
        player2Addr,
        [WOOD, IRON],
        [2n, 10n]
      );
      
      await expect(
        items.connect(player2).craft("bronze_sword")
      ).to.be.revertedWithCustomError(items, "InsufficientCraftingMaterials");
    });
    
    it("should emit ItemCrafted event", async function () {
      await expect(items.connect(player1).craft("bronze_sword"))
        .to.emit(items, "ItemCrafted")
        .withArgs(await player1.getAddress(), BRONZE_SWORD, 1n);
    });
  });
  
  describe("Transfers", function () {
    beforeEach(async function () {
      await items.connect(gameServer).mintItem(await player1.getAddress(), GOLD, 100n);
    });
    
    it("should transfer tradeable items", async function () {
      const p1 = await player1.getAddress();
      const p2 = await player2.getAddress();
      
      await items.connect(player1).safeTransferFrom(p1, p2, GOLD, 50n, "0x");
      
      expect(await items.balanceOf(p1, GOLD)).to.equal(50n);
      expect(await items.balanceOf(p2, GOLD)).to.equal(50n);
    });
    
    it("should revert transferring non-tradeable items", async function () {
      // Define a non-tradeable item
      await items.connect(owner).defineItem(
        999n, "Soulbound", 4, 0, false, false
      );
      await items.connect(gameServer).mintItem(await player1.getAddress(), 999n, 1n);
      
      await expect(
        items.connect(player1).safeTransferFrom(
          await player1.getAddress(),
          await player2.getAddress(),
          999n, 1n, "0x"
        )
      ).to.be.revertedWithCustomError(items, "NotTradeable");
    });
  });
  
  describe("Batch Operations Gas Comparison", function () {
    it("demonstrates batch transfer gas savings", async function () {
      const p1 = await player1.getAddress();
      const p2 = await player2.getAddress();
      
      // Mint multiple items
      await items.connect(gameServer).mintBatch(
        p1, [GOLD, WOOD, IRON], [100n, 100n, 100n]
      );
      
      // Batch transfer is much cheaper than 3 separate transfers
      const tx = await items.connect(player1).safeBatchTransferFrom(
        p1, p2, [GOLD, WOOD, IRON], [50n, 50n, 50n], "0x"
      );
      const receipt = await tx.wait();
      
      console.log(`Batch transfer gas: ${receipt?.gasUsed}`);
      
      // ตรวจสอบผลลัพธ์
      const balances = await items.balanceOfBatch(
        [p2, p2, p2], [GOLD, WOOD, IRON]
      );
      expect(balances[0]).to.equal(50n);
      expect(balances[1]).to.equal(50n);
      expect(balances[2]).to.equal(50n);
    });
  });
});
```

---

## สรุป Part 13

ERC-1155 Multi-Token Standard ที่เรียนรู้:
- ✅ ERC-1155 interface และ events
- ✅ Full ERC-1155 implementation
- ✅ Batch operations (mint, transfer, burn)
- ✅ Crafting system
- ✅ Soulbound items
- ✅ Game items with attributes

## ตารางเปรียบเทียบ Standards

| ฟีเจอร์ | ERC-20 | ERC-721 | ERC-1155 |
|--------|--------|---------|---------|
| Token Types | 1 (fungible) | 1 (non-fungible) | Multiple |
| Unique items | ❌ | ✅ | ✅ |
| Batch ops | ❌ | ❌ | ✅ |
| Gas efficiency | ✅ | ❌ | ✅ |
| Gaming | ❌ | ❌ | ✅ |

## Quiz

1. เมื่อไหร่ควรใช้ ERC-1155 แทน ERC-721?
2. `safeBatchTransferFrom` มี gas ถูกกว่า N ครั้ง `safeTransferFrom` อย่างไร?
3. Semi-fungible token คืออะไร ยกตัวอย่าง?
4. `onERC1155Received` และ `onERC1155BatchReceived` ทำงานอย่างไร?

---

## Next: Part 14 - Access Control และ Role Management
