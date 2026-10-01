# Part 06: Arrays และ Mappings เชิงลึก

## สารบัญ
1. Arrays เชิงลึก
2. Array Manipulation Patterns
3. Mappings เชิงลึก
4. Enumerable Mappings
5. Nested Data Structures
6. IterableMapping Library
7. Packed Storage Patterns
8. Gas-Efficient Data Structures
9. OpenZeppelin EnumerableSet/EnumerableMap
10. Workshop: Token Distribution System

---

## 1. Arrays เชิงลึก

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract AdvancedArrays {
    
    // === Array Types ===
    
    // Static (fixed-size)
    uint256[10] public fixedArray;
    address[3] public topAddresses;
    
    // Dynamic
    uint256[] public dynamicArray;
    
    // 2D Arrays
    uint256[][] public matrix;
    uint256[3][3] public fixedMatrix; // 3x3 fixed
    
    // Array ของ structs
    struct Token {
        address addr;
        string symbol;
        uint256 decimals;
    }
    Token[] public tokens;
    
    // Array ของ addresses
    address[] public participants;
    
    // === Array Operations ===
    
    // Push (dynamic only)
    function pushElement(uint256 value) public {
        dynamicArray.push(value);
    }
    
    // Pop (dynamic only) - ลบตัวสุดท้าย
    function popElement() public returns (uint256 removed) {
        require(dynamicArray.length > 0, "Empty array");
        removed = dynamicArray[dynamicArray.length - 1];
        dynamicArray.pop();
    }
    
    // Update element
    function updateElement(uint256 index, uint256 newValue) public {
        require(index < dynamicArray.length, "Index out of bounds");
        dynamicArray[index] = newValue;
    }
    
    // Delete element (ตั้งค่าเป็น default, length ไม่เปลี่ยน)
    function deleteElement(uint256 index) public {
        require(index < dynamicArray.length, "Index out of bounds");
        delete dynamicArray[index]; // = 0
    }
    
    // Remove and maintain order O(n)
    function removeOrdered(uint256 index) public {
        require(index < dynamicArray.length, "Index out of bounds");
        
        for (uint256 i = index; i < dynamicArray.length - 1; i++) {
            dynamicArray[i] = dynamicArray[i + 1];
        }
        dynamicArray.pop();
    }
    
    // Remove without maintaining order O(1) - swap with last
    function removeSwap(uint256 index) public {
        require(index < dynamicArray.length, "Index out of bounds");
        
        dynamicArray[index] = dynamicArray[dynamicArray.length - 1];
        dynamicArray.pop();
    }
    
    // === Slice (calldata only) ===
    
    function processSlice(uint256[] calldata data, uint256 start, uint256 end) 
        public pure returns (uint256 sum) 
    {
        uint256[] calldata slice = data[start:end];
        for (uint256 i = 0; i < slice.length; i++) {
            sum += slice[i];
        }
    }
    
    // === Sorting ===
    
    // Bubble Sort (สำหรับ arrays เล็กๆ)
    function bubbleSort(uint256[] memory arr) public pure returns (uint256[] memory) {
        uint256 n = arr.length;
        
        for (uint256 i = 0; i < n - 1; i++) {
            for (uint256 j = 0; j < n - i - 1; j++) {
                if (arr[j] > arr[j + 1]) {
                    (arr[j], arr[j + 1]) = (arr[j + 1], arr[j]);
                }
            }
        }
        
        return arr;
    }
    
    // Quick Sort (efficient สำหรับ on-chain)
    function quickSort(uint256[] memory arr, uint256 left, uint256 right) 
        public pure returns (uint256[] memory) 
    {
        if (left < right) {
            uint256 pivot = arr[right];
            uint256 i = left;
            
            for (uint256 j = left; j < right; j++) {
                if (arr[j] <= pivot) {
                    (arr[i], arr[j]) = (arr[j], arr[i]);
                    i++;
                }
            }
            
            (arr[i], arr[right]) = (arr[right], arr[i]);
            
            if (i > 0) quickSort(arr, left, i - 1);
            quickSort(arr, i + 1, right);
        }
        
        return arr;
    }
    
    // === Deduplication ===
    
    function unique(uint256[] memory arr) public pure returns (uint256[] memory result) {
        if (arr.length == 0) return result;
        
        // Sort first
        arr = quickSort(arr, 0, arr.length - 1);
        
        // Count unique
        uint256 count = 1;
        for (uint256 i = 1; i < arr.length; i++) {
            if (arr[i] != arr[i - 1]) count++;
        }
        
        // Fill result
        result = new uint256[](count);
        result[0] = arr[0];
        uint256 idx = 1;
        
        for (uint256 i = 1; i < arr.length; i++) {
            if (arr[i] != arr[i - 1]) {
                result[idx++] = arr[i];
            }
        }
    }
    
    // === Searching ===
    
    function indexOf(uint256[] memory arr, uint256 value) public pure returns (int256) {
        for (uint256 i = 0; i < arr.length; i++) {
            if (arr[i] == value) return int256(i);
        }
        return -1;
    }
    
    function contains(uint256[] memory arr, uint256 value) public pure returns (bool) {
        return indexOf(arr, value) >= 0;
    }
    
    // Binary search (sorted array)
    function binarySearch(
        uint256[] memory sortedArr, 
        uint256 target
    ) public pure returns (int256) {
        if (sortedArr.length == 0) return -1;
        
        uint256 low = 0;
        uint256 high = sortedArr.length - 1;
        
        while (low <= high) {
            uint256 mid = low + (high - low) / 2;
            
            if (sortedArr[mid] == target) return int256(mid);
            if (sortedArr[mid] < target) {
                low = mid + 1;
            } else {
                if (mid == 0) break;
                high = mid - 1;
            }
        }
        
        return -1;
    }
}
```

---

## 2. Array Manipulation Patterns

```solidity
contract ArrayPatterns {
    
    // === Pattern 1: Paginated Array ===
    
    uint256[] public allItems;
    
    function getPage(uint256 page, uint256 pageSize) 
        public view returns (uint256[] memory result, uint256 total) 
    {
        total = allItems.length;
        
        uint256 start = page * pageSize;
        if (start >= total) return (new uint256[](0), total);
        
        uint256 end = start + pageSize;
        if (end > total) end = total;
        
        result = new uint256[](end - start);
        for (uint256 i = start; i < end; i++) {
            result[i - start] = allItems[i];
        }
    }
    
    // === Pattern 2: Filtered Array ===
    
    struct Item {
        uint256 id;
        address owner;
        uint256 price;
        bool forSale;
    }
    
    Item[] public items;
    
    function getForSaleItems() public view returns (Item[] memory result) {
        // นับก่อน
        uint256 count = 0;
        for (uint256 i = 0; i < items.length; i++) {
            if (items[i].forSale) count++;
        }
        
        // สร้างผลลัพธ์
        result = new Item[](count);
        uint256 idx = 0;
        for (uint256 i = 0; i < items.length; i++) {
            if (items[i].forSale) {
                result[idx++] = items[i];
            }
        }
    }
    
    function getItemsByOwner(address owner) public view returns (Item[] memory result) {
        uint256 count = 0;
        for (uint256 i = 0; i < items.length; i++) {
            if (items[i].owner == owner) count++;
        }
        
        result = new Item[](count);
        uint256 idx = 0;
        for (uint256 i = 0; i < items.length; i++) {
            if (items[i].owner == owner) {
                result[idx++] = items[i];
            }
        }
    }
    
    // === Pattern 3: Ring Buffer ===
    
    uint256 public constant RING_SIZE = 10;
    uint256[RING_SIZE] public ringBuffer;
    uint256 public ringHead;
    uint256 public ringCount;
    
    function pushRing(uint256 value) public {
        ringBuffer[(ringHead + ringCount) % RING_SIZE] = value;
        
        if (ringCount < RING_SIZE) {
            ringCount++;
        } else {
            ringHead = (ringHead + 1) % RING_SIZE;
        }
    }
    
    function getRingBuffer() public view returns (uint256[] memory result) {
        result = new uint256[](ringCount);
        for (uint256 i = 0; i < ringCount; i++) {
            result[i] = ringBuffer[(ringHead + i) % RING_SIZE];
        }
    }
    
    // === Pattern 4: Batch Operations ===
    
    mapping(address => uint256) public balances;
    
    function batchTransfer(
        address[] calldata recipients,
        uint256[] calldata amounts
    ) external {
        require(recipients.length == amounts.length, "Length mismatch");
        
        uint256 totalAmount = 0;
        for (uint256 i = 0; i < amounts.length; i++) {
            totalAmount += amounts[i];
        }
        require(balances[msg.sender] >= totalAmount, "Insufficient balance");
        
        balances[msg.sender] -= totalAmount;
        
        for (uint256 i = 0; i < recipients.length; i++) {
            if (amounts[i] > 0 && recipients[i] != address(0)) {
                balances[recipients[i]] += amounts[i];
            }
        }
    }
    
    function batchSetBalances(
        address[] calldata users,
        uint256[] calldata amounts
    ) external {
        require(users.length == amounts.length, "Length mismatch");
        require(users.length <= 500, "Too many users"); // Prevent out of gas
        
        for (uint256 i = 0; i < users.length; i++) {
            balances[users[i]] = amounts[i];
        }
    }
}
```

---

## 3. Mappings เชิงลึก

```solidity
contract AdvancedMappings {
    
    // === Basic Mapping ===
    
    mapping(address => uint256) public balanceOf;
    
    // === Nested Mapping ===
    
    // allowances[owner][spender] = amount
    mapping(address => mapping(address => uint256)) public allowance;
    
    function approve(address spender, uint256 amount) public returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }
    
    function transferFrom(
        address from,
        address to,
        uint256 amount
    ) public returns (bool) {
        require(allowance[from][msg.sender] >= amount, "Not approved");
        require(balanceOf[from] >= amount, "Insufficient balance");
        
        allowance[from][msg.sender] -= amount;
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        
        return true;
    }
    
    // === Mapping กับ Struct ===
    
    struct UserProfile {
        string username;
        uint256 reputation;
        uint256 joinedAt;
        bool verified;
        address[] following;
    }
    
    mapping(address => UserProfile) public profiles;
    
    function createProfile(string calldata username) public {
        require(bytes(profiles[msg.sender].username).length == 0, "Already has profile");
        require(bytes(username).length >= 3, "Username too short");
        
        profiles[msg.sender] = UserProfile({
            username: username,
            reputation: 0,
            joinedAt: block.timestamp,
            verified: false,
            following: new address[](0)
        });
    }
    
    function follow(address user) public {
        require(user != msg.sender, "Cannot follow self");
        
        address[] storage followingList = profiles[msg.sender].following;
        
        // ตรวจสอบว่าติดตามแล้วหรือยัง
        for (uint256 i = 0; i < followingList.length; i++) {
            require(followingList[i] != user, "Already following");
        }
        
        followingList.push(user);
    }
    
    // === Double Mapping Patterns ===
    
    // ตรวจสอบ vote ซ้ำ
    mapping(uint256 => mapping(address => bool)) public hasVoted; // proposalId => voter => voted
    
    function vote(uint256 proposalId) public {
        require(!hasVoted[proposalId][msg.sender], "Already voted");
        hasVoted[proposalId][msg.sender] = true;
    }
    
    // NFT ownership
    // ownerOf[tokenId] = owner
    mapping(uint256 => address) public ownerOf;
    // ownedTokens[owner] = [tokenId1, tokenId2, ...]
    mapping(address => uint256[]) public ownedTokens;
    // tokenIndex[tokenId] = index in ownedTokens[owner]
    mapping(uint256 => uint256) public tokenIndex;
    
    function mint(address to, uint256 tokenId) internal {
        ownerOf[tokenId] = to;
        tokenIndex[tokenId] = ownedTokens[to].length;
        ownedTokens[to].push(tokenId);
    }
    
    function burn(uint256 tokenId) internal {
        address owner = ownerOf[tokenId];
        
        // Remove from ownedTokens efficiently
        uint256 lastIndex = ownedTokens[owner].length - 1;
        uint256 tokenIdx = tokenIndex[tokenId];
        
        if (tokenIdx != lastIndex) {
            uint256 lastToken = ownedTokens[owner][lastIndex];
            ownedTokens[owner][tokenIdx] = lastToken;
            tokenIndex[lastToken] = tokenIdx;
        }
        
        ownedTokens[owner].pop();
        delete ownerOf[tokenId];
        delete tokenIndex[tokenId];
    }
}
```

---

## 4. Enumerable Mappings

```solidity
contract EnumerableMapping {
    
    // Mapping ไม่สามารถ iterate ได้โดยตรง
    // แก้ด้วยการเก็บ keys แยก
    
    // Pattern: mapping + array of keys
    
    mapping(address => uint256) private _balances;
    address[] private _holders;
    mapping(address => uint256) private _holderIndex; // index ใน _holders + 1
    
    event BalanceSet(address indexed account, uint256 balance);
    event BalanceRemoved(address indexed account);
    
    function setBalance(address account, uint256 amount) public {
        if (amount == 0) {
            _removeHolder(account);
        } else {
            _addHolder(account);
            _balances[account] = amount;
        }
        emit BalanceSet(account, amount);
    }
    
    function getBalance(address account) public view returns (uint256) {
        return _balances[account];
    }
    
    function isHolder(address account) public view returns (bool) {
        return _holderIndex[account] != 0;
    }
    
    function holderCount() public view returns (uint256) {
        return _holders.length;
    }
    
    function holderAt(uint256 index) public view returns (address) {
        require(index < _holders.length, "Index out of bounds");
        return _holders[index];
    }
    
    function getAllHolders() public view returns (address[] memory) {
        return _holders;
    }
    
    function getTotalBalance() public view returns (uint256 total) {
        for (uint256 i = 0; i < _holders.length; i++) {
            total += _balances[_holders[i]];
        }
    }
    
    // Internal helpers
    function _addHolder(address account) internal {
        if (_holderIndex[account] == 0) {
            _holders.push(account);
            _holderIndex[account] = _holders.length; // +1 offset
        }
    }
    
    function _removeHolder(address account) internal {
        uint256 index = _holderIndex[account];
        if (index == 0) return; // not a holder
        
        uint256 lastIndex = _holders.length - 1;
        uint256 actualIndex = index - 1;
        
        if (actualIndex != lastIndex) {
            address lastHolder = _holders[lastIndex];
            _holders[actualIndex] = lastHolder;
            _holderIndex[lastHolder] = index;
        }
        
        _holders.pop();
        delete _holderIndex[account];
        delete _balances[account];
        
        emit BalanceRemoved(account);
    }
    
    // Paginated iteration
    function getHolderPage(
        uint256 page,
        uint256 pageSize
    ) public view returns (
        address[] memory holders,
        uint256[] memory balances,
        uint256 total
    ) {
        total = _holders.length;
        
        uint256 start = page * pageSize;
        if (start >= total) {
            return (new address[](0), new uint256[](0), total);
        }
        
        uint256 end = start + pageSize > total ? total : start + pageSize;
        uint256 size = end - start;
        
        holders = new address[](size);
        balances = new uint256[](size);
        
        for (uint256 i = 0; i < size; i++) {
            holders[i] = _holders[start + i];
            balances[i] = _balances[_holders[start + i]];
        }
    }
}
```

---

## 5. Nested Data Structures

```solidity
contract NestedStructures {
    
    // === Complex Nested Struct ===
    
    struct Address {
        string street;
        string city;
        string country;
        uint256 postalCode;
    }
    
    struct Contact {
        string email;
        string phone;
        Address homeAddress;
        Address workAddress;
    }
    
    struct Employee {
        uint256 id;
        string name;
        uint256 salary;
        Contact contact;
        uint256[] projectIds;
        mapping(uint256 => uint256) projectHours; // projectId => hours
    }
    
    // Mapping ของ struct ที่มี mapping ข้างใน
    mapping(uint256 => Employee) public employees;
    uint256 public employeeCount;
    
    function addEmployee(
        string calldata name,
        uint256 salary,
        string calldata email
    ) public returns (uint256 id) {
        id = ++employeeCount;
        
        Employee storage emp = employees[id];
        emp.id = id;
        emp.name = name;
        emp.salary = salary;
        emp.contact.email = email;
    }
    
    function addProjectHours(
        uint256 employeeId,
        uint256 projectId,
        uint256 hours
    ) public {
        Employee storage emp = employees[employeeId];
        
        // เพิ่ม project ถ้ายังไม่มี
        if (emp.projectHours[projectId] == 0) {
            emp.projectIds.push(projectId);
        }
        
        emp.projectHours[projectId] += hours;
    }
    
    function getEmployeeProjects(uint256 employeeId) 
        public view returns (uint256[] memory projectIds, uint256[] memory hours) 
    {
        Employee storage emp = employees[employeeId];
        uint256 len = emp.projectIds.length;
        
        projectIds = emp.projectIds;
        hours = new uint256[](len);
        
        for (uint256 i = 0; i < len; i++) {
            hours[i] = emp.projectHours[emp.projectIds[i]];
        }
    }
    
    // === Tree Structure ===
    
    struct TreeNode {
        uint256 id;
        uint256 value;
        uint256 parentId;
        uint256[] childIds;
    }
    
    mapping(uint256 => TreeNode) public tree;
    uint256 public nodeCount;
    
    function addNode(uint256 parentId, uint256 value) public returns (uint256 nodeId) {
        nodeId = ++nodeCount;
        
        tree[nodeId] = TreeNode({
            id: nodeId,
            value: value,
            parentId: parentId,
            childIds: new uint256[](0)
        });
        
        if (parentId != 0) {
            tree[parentId].childIds.push(nodeId);
        }
    }
    
    function getChildren(uint256 nodeId) public view returns (uint256[] memory) {
        return tree[nodeId].childIds;
    }
    
    // DFS traversal (ระวัง stack overflow สำหรับ tree ลึกมาก)
    function sumSubtree(uint256 nodeId) public view returns (uint256 sum) {
        sum = tree[nodeId].value;
        
        uint256[] memory children = tree[nodeId].childIds;
        for (uint256 i = 0; i < children.length; i++) {
            sum += sumSubtree(children[i]);
        }
    }
}
```

---

## 6. Gas-Efficient Data Structures

```solidity
contract GasEfficientStructures {
    
    // === Bitmap for Boolean Arrays ===
    // แทนที่จะใช้ mapping(uint => bool) ซึ่งแต่ละ bit ใช้ 1 slot
    // ใช้ bitmap เก็บ 256 booleans ใน 1 slot
    
    mapping(uint256 => uint256) private _bitmap;
    
    function setBit(uint256 index, bool value) public {
        uint256 wordIndex = index / 256;
        uint256 bitIndex = index % 256;
        
        if (value) {
            _bitmap[wordIndex] |= (1 << bitIndex);
        } else {
            _bitmap[wordIndex] &= ~(1 << bitIndex);
        }
    }
    
    function getBit(uint256 index) public view returns (bool) {
        uint256 wordIndex = index / 256;
        uint256 bitIndex = index % 256;
        
        return (_bitmap[wordIndex] & (1 << bitIndex)) != 0;
    }
    
    // === Packed User Data ===
    // เก็บข้อมูลหลายค่าใน 1 slot
    
    // แทนที่จะใช้หลาย state variables:
    // uint256 balance;      // slot 0
    // uint32 lastActivity;  // slot 1
    // uint8 tier;           // slot 2
    // bool active;          // slot 3
    
    // ใช้ packed encoding:
    // bits 0-127:   balance (uint128)
    // bits 128-159: lastActivity (uint32)
    // bits 160-167: tier (uint8)
    // bit 168:      active (bool)
    
    mapping(address => uint256) private _userData;
    
    uint256 private constant BALANCE_MASK = type(uint128).max;
    uint256 private constant LAST_ACTIVITY_SHIFT = 128;
    uint256 private constant LAST_ACTIVITY_MASK = uint256(type(uint32).max) << LAST_ACTIVITY_SHIFT;
    uint256 private constant TIER_SHIFT = 160;
    uint256 private constant TIER_MASK = uint256(type(uint8).max) << TIER_SHIFT;
    uint256 private constant ACTIVE_SHIFT = 168;
    uint256 private constant ACTIVE_MASK = 1 << ACTIVE_SHIFT;
    
    function setUserData(
        address user,
        uint128 balance,
        uint32 lastActivity,
        uint8 tier,
        bool active
    ) public {
        uint256 packed = uint256(balance);
        packed |= uint256(lastActivity) << LAST_ACTIVITY_SHIFT;
        packed |= uint256(tier) << TIER_SHIFT;
        if (active) packed |= ACTIVE_MASK;
        
        _userData[user] = packed;
    }
    
    function getUserBalance(address user) public view returns (uint128) {
        return uint128(_userData[user] & BALANCE_MASK);
    }
    
    function getUserTier(address user) public view returns (uint8) {
        return uint8((_userData[user] & TIER_MASK) >> TIER_SHIFT);
    }
    
    function isUserActive(address user) public view returns (bool) {
        return (_userData[user] & ACTIVE_MASK) != 0;
    }
    
    // === Merkle Proof สำหรับ Large Lists ===
    // แทนที่จะเก็บ whitelist ทั้งหมดใน storage
    // เก็บแค่ root hash
    
    bytes32 public merkleRoot;
    
    function setMerkleRoot(bytes32 root) external {
        merkleRoot = root;
    }
    
    function isWhitelisted(
        address addr,
        bytes32[] calldata proof
    ) public view returns (bool) {
        bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(addr))));
        return _verifyMerkle(proof, merkleRoot, leaf);
    }
    
    function _verifyMerkle(
        bytes32[] calldata proof,
        bytes32 root,
        bytes32 leaf
    ) internal pure returns (bool) {
        bytes32 computedHash = leaf;
        
        for (uint256 i = 0; i < proof.length; i++) {
            bytes32 proofElement = proof[i];
            
            if (computedHash <= proofElement) {
                computedHash = keccak256(abi.encodePacked(computedHash, proofElement));
            } else {
                computedHash = keccak256(abi.encodePacked(proofElement, computedHash));
            }
        }
        
        return computedHash == root;
    }
}
```

---

## 7. OpenZeppelin EnumerableSet และ EnumerableMap

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/utils/structs/EnumerableSet.sol";
import "@openzeppelin/contracts/utils/structs/EnumerableMap.sol";

contract OZDataStructures {
    
    using EnumerableSet for EnumerableSet.AddressSet;
    using EnumerableSet for EnumerableSet.UintSet;
    using EnumerableSet for EnumerableSet.Bytes32Set;
    using EnumerableMap for EnumerableMap.UintToAddressMap;
    using EnumerableMap for EnumerableMap.AddressToUintMap;
    
    // === EnumerableSet ===
    
    EnumerableSet.AddressSet private _admins;
    EnumerableSet.UintSet private _tokenIds;
    
    function addAdmin(address admin) public {
        _admins.add(admin); // returns bool (true if added)
    }
    
    function removeAdmin(address admin) public {
        _admins.remove(admin);
    }
    
    function isAdmin(address admin) public view returns (bool) {
        return _admins.contains(admin);
    }
    
    function adminCount() public view returns (uint256) {
        return _admins.length();
    }
    
    function getAdmin(uint256 index) public view returns (address) {
        return _admins.at(index);
    }
    
    function getAllAdmins() public view returns (address[] memory result) {
        uint256 len = _admins.length();
        result = new address[](len);
        for (uint256 i = 0; i < len; i++) {
            result[i] = _admins.at(i);
        }
    }
    
    // === EnumerableMap ===
    
    EnumerableMap.AddressToUintMap private _userScores;
    
    function setScore(address user, uint256 score) public {
        _userScores.set(user, score); // true if new key
    }
    
    function removeScore(address user) public {
        _userScores.remove(user);
    }
    
    function getScore(address user) public view returns (uint256) {
        return _userScores.get(user);
    }
    
    function hasScore(address user) public view returns (bool) {
        return _userScores.contains(user);
    }
    
    function scoredUserCount() public view returns (uint256) {
        return _userScores.length();
    }
    
    function getScoreAtIndex(uint256 index) public view returns (address user, uint256 score) {
        return _userScores.at(index);
    }
    
    function getTopScorers(uint256 n) public view returns (
        address[] memory users,
        uint256[] memory scores
    ) {
        uint256 len = _userScores.length();
        uint256 count = n < len ? n : len;
        
        // Get all
        address[] memory allUsers = new address[](len);
        uint256[] memory allScores = new uint256[](len);
        
        for (uint256 i = 0; i < len; i++) {
            (allUsers[i], allScores[i]) = _userScores.at(i);
        }
        
        // Sort by score (bubble sort for simplicity)
        for (uint256 i = 0; i < len - 1; i++) {
            for (uint256 j = 0; j < len - i - 1; j++) {
                if (allScores[j] < allScores[j + 1]) {
                    (allScores[j], allScores[j + 1]) = (allScores[j + 1], allScores[j]);
                    (allUsers[j], allUsers[j + 1]) = (allUsers[j + 1], allUsers[j]);
                }
            }
        }
        
        users = new address[](count);
        scores = new uint256[](count);
        
        for (uint256 i = 0; i < count; i++) {
            users[i] = allUsers[i];
            scores[i] = allScores[i];
        }
    }
}
```

---

## 8. Workshop: Token Distribution System

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/utils/structs/EnumerableSet.sol";

/**
 * @title TokenDistribution
 * @dev ระบบแจก Token ให้ผู้ถือ NFT/Token
 */
contract TokenDistribution {
    
    using EnumerableSet for EnumerableSet.AddressSet;
    
    // === Types ===
    
    enum DistributionType {
        Equal,        // แจกเท่ากันทุกคน
        ProRata,      // แจกตามสัดส่วน balance
        Tiered        // แจกตาม tier
    }
    
    struct Distribution {
        uint256 id;
        string name;
        DistributionType distType;
        uint256 totalAmount;
        uint256 claimedAmount;
        uint256 snapshotTime;
        uint256 claimDeadline;
        bool active;
        mapping(address => uint256) allocations;
        mapping(address => bool) claimed;
        uint256 recipientCount;
    }
    
    struct Snapshot {
        uint256 timestamp;
        mapping(address => uint256) balances;
        address[] holders;
        uint256 totalSupply;
    }
    
    // === State ===
    
    address public immutable ADMIN;
    
    mapping(uint256 => Distribution) public distributions;
    uint256 public distributionCount;
    
    // Token balances (simulated - จริงๆ ใช้ ERC-20)
    mapping(address => uint256) public tokenBalance;
    EnumerableSet.AddressSet private _tokenHolders;
    uint256 public totalTokenSupply;
    
    // Snapshots
    mapping(uint256 => Snapshot) private _snapshots;
    uint256 public snapshotCount;
    
    // === Events ===
    
    event DistributionCreated(
        uint256 indexed distId,
        string name,
        DistributionType distType,
        uint256 totalAmount
    );
    event AllocationSet(uint256 indexed distId, address indexed recipient, uint256 amount);
    event Claimed(uint256 indexed distId, address indexed recipient, uint256 amount);
    event SnapshotTaken(uint256 indexed snapshotId, uint256 timestamp, uint256 holderCount);
    
    // === Custom Errors ===
    
    error NotAdmin();
    error DistributionNotFound(uint256 id);
    error DistributionInactive(uint256 id);
    error AlreadyClaimed(address user, uint256 distId);
    error DeadlineExpired(uint256 deadline, uint256 currentTime);
    error NoAllocation(address user, uint256 distId);
    error InsufficientFunds();
    
    // === Constructor ===
    
    constructor() {
        ADMIN = msg.sender;
    }
    
    // === Modifiers ===
    
    modifier onlyAdmin() {
        if (msg.sender != ADMIN) revert NotAdmin();
        _;
    }
    
    // === Token Management (Simplified) ===
    
    function mint(address to, uint256 amount) external onlyAdmin {
        require(amount > 0, "Amount must be > 0");
        
        tokenBalance[to] += amount;
        totalTokenSupply += amount;
        
        _tokenHolders.add(to);
    }
    
    function burn(address from, uint256 amount) external onlyAdmin {
        require(tokenBalance[from] >= amount, "Insufficient balance");
        
        tokenBalance[from] -= amount;
        totalTokenSupply -= amount;
        
        if (tokenBalance[from] == 0) {
            _tokenHolders.remove(from);
        }
    }
    
    // === Snapshot ===
    
    function takeSnapshot() external onlyAdmin returns (uint256 snapshotId) {
        snapshotId = ++snapshotCount;
        Snapshot storage snap = _snapshots[snapshotId];
        snap.timestamp = block.timestamp;
        snap.totalSupply = totalTokenSupply;
        
        uint256 holderCount = _tokenHolders.length();
        snap.holders = new address[](holderCount);
        
        for (uint256 i = 0; i < holderCount; i++) {
            address holder = _tokenHolders.at(i);
            snap.holders[i] = holder;
            snap.balances[holder] = tokenBalance[holder];
        }
        
        emit SnapshotTaken(snapshotId, block.timestamp, holderCount);
    }
    
    function getSnapshotBalance(uint256 snapshotId, address account) 
        external view returns (uint256) 
    {
        require(snapshotId <= snapshotCount, "Snapshot not found");
        return _snapshots[snapshotId].balances[account];
    }
    
    // === Distribution Creation ===
    
    function createDistribution(
        string calldata name,
        DistributionType distType,
        uint256 snapshotId,
        uint256 claimDeadline
    ) external payable onlyAdmin returns (uint256 distId) {
        require(msg.value > 0, "Must provide ETH");
        require(snapshotId <= snapshotCount, "Invalid snapshot");
        require(claimDeadline > block.timestamp, "Deadline must be future");
        
        distId = ++distributionCount;
        Distribution storage dist = distributions[distId];
        
        dist.id = distId;
        dist.name = name;
        dist.distType = distType;
        dist.totalAmount = msg.value;
        dist.snapshotTime = _snapshots[snapshotId].timestamp;
        dist.claimDeadline = claimDeadline;
        dist.active = true;
        
        // Set allocations based on snapshot
        _setAllocations(distId, snapshotId, distType, msg.value);
        
        emit DistributionCreated(distId, name, distType, msg.value);
    }
    
    function _setAllocations(
        uint256 distId,
        uint256 snapshotId,
        DistributionType distType,
        uint256 totalAmount
    ) internal {
        Snapshot storage snap = _snapshots[snapshotId];
        Distribution storage dist = distributions[distId];
        
        address[] memory holders = snap.holders;
        uint256 holderCount = holders.length;
        
        if (holderCount == 0) return;
        
        if (distType == DistributionType.Equal) {
            uint256 perPerson = totalAmount / holderCount;
            
            for (uint256 i = 0; i < holderCount; i++) {
                if (snap.balances[holders[i]] > 0) {
                    dist.allocations[holders[i]] = perPerson;
                    dist.recipientCount++;
                    emit AllocationSet(distId, holders[i], perPerson);
                }
            }
            
        } else if (distType == DistributionType.ProRata) {
            uint256 totalSupply = snap.totalSupply;
            if (totalSupply == 0) return;
            
            for (uint256 i = 0; i < holderCount; i++) {
                address holder = holders[i];
                uint256 balance = snap.balances[holder];
                
                if (balance > 0) {
                    uint256 allocation = (totalAmount * balance) / totalSupply;
                    dist.allocations[holder] = allocation;
                    dist.recipientCount++;
                    emit AllocationSet(distId, holder, allocation);
                }
            }
            
        } else if (distType == DistributionType.Tiered) {
            // Tier 1: < 100 tokens = 1x
            // Tier 2: 100-999 tokens = 2x
            // Tier 3: >= 1000 tokens = 4x
            
            uint256 totalWeight = 0;
            
            for (uint256 i = 0; i < holderCount; i++) {
                uint256 balance = snap.balances[holders[i]];
                if (balance >= 1000) totalWeight += 4;
                else if (balance >= 100) totalWeight += 2;
                else if (balance > 0) totalWeight += 1;
            }
            
            if (totalWeight == 0) return;
            
            uint256 perWeight = totalAmount / totalWeight;
            
            for (uint256 i = 0; i < holderCount; i++) {
                address holder = holders[i];
                uint256 balance = snap.balances[holder];
                uint256 weight;
                
                if (balance >= 1000) weight = 4;
                else if (balance >= 100) weight = 2;
                else if (balance > 0) weight = 1;
                
                if (weight > 0) {
                    uint256 allocation = perWeight * weight;
                    dist.allocations[holder] = allocation;
                    dist.recipientCount++;
                    emit AllocationSet(distId, holder, allocation);
                }
            }
        }
    }
    
    // === Claiming ===
    
    function claim(uint256 distId) external {
        Distribution storage dist = distributions[distId];
        
        if (distId == 0 || distId > distributionCount) revert DistributionNotFound(distId);
        if (!dist.active) revert DistributionInactive(distId);
        if (block.timestamp > dist.claimDeadline) {
            revert DeadlineExpired(dist.claimDeadline, block.timestamp);
        }
        if (dist.claimed[msg.sender]) revert AlreadyClaimed(msg.sender, distId);
        
        uint256 allocation = dist.allocations[msg.sender];
        if (allocation == 0) revert NoAllocation(msg.sender, distId);
        
        dist.claimed[msg.sender] = true;
        dist.claimedAmount += allocation;
        
        (bool success, ) = msg.sender.call{value: allocation}("");
        require(success, "Transfer failed");
        
        emit Claimed(distId, msg.sender, allocation);
    }
    
    function batchClaim(uint256[] calldata distIds) external {
        for (uint256 i = 0; i < distIds.length; i++) {
            uint256 distId = distIds[i];
            Distribution storage dist = distributions[distId];
            
            if (!dist.active) continue;
            if (block.timestamp > dist.claimDeadline) continue;
            if (dist.claimed[msg.sender]) continue;
            
            uint256 allocation = dist.allocations[msg.sender];
            if (allocation == 0) continue;
            
            dist.claimed[msg.sender] = true;
            dist.claimedAmount += allocation;
            
            (bool success, ) = msg.sender.call{value: allocation}("");
            if (!success) continue;
            
            emit Claimed(distId, msg.sender, allocation);
        }
    }
    
    // === Admin: Reclaim unclaimed ===
    
    function reclaimUnclaimed(uint256 distId) external onlyAdmin {
        Distribution storage dist = distributions[distId];
        require(distId > 0 && distId <= distributionCount, "Not found");
        require(block.timestamp > dist.claimDeadline, "Not expired");
        require(dist.active, "Already reclaimed");
        
        dist.active = false;
        
        uint256 unclaimed = dist.totalAmount - dist.claimedAmount;
        if (unclaimed > 0) {
            (bool success, ) = ADMIN.call{value: unclaimed}("");
            require(success, "Transfer failed");
        }
    }
    
    // === View Functions ===
    
    function getAllocation(uint256 distId, address account) 
        external view returns (uint256) 
    {
        return distributions[distId].allocations[account];
    }
    
    function hasClaimed(uint256 distId, address account) 
        external view returns (bool) 
    {
        return distributions[distId].claimed[account];
    }
    
    function getDistributionStatus(uint256 distId) 
        external view returns (
            string memory name,
            uint256 total,
            uint256 claimed,
            uint256 recipients,
            bool active,
            uint256 deadline
        ) 
    {
        Distribution storage dist = distributions[distId];
        return (
            dist.name,
            dist.totalAmount,
            dist.claimedAmount,
            dist.recipientCount,
            dist.active,
            dist.claimDeadline
        );
    }
    
    function getHolderCount() external view returns (uint256) {
        return _tokenHolders.length();
    }
    
    function getHolderAt(uint256 index) external view returns (address) {
        return _tokenHolders.at(index);
    }
    
    receive() external payable {}
}
```

---

## สรุป Part 06

Data Structures ที่เรียนรู้:
- ✅ Arrays เชิงลึก (static, dynamic, 2D)
- ✅ Array manipulation patterns (sort, filter, paginate)
- ✅ Mappings เชิงลึก (nested, with structs)
- ✅ Enumerable mappings (iterate ได้)
- ✅ Nested data structures
- ✅ Bitmap สำหรับ boolean arrays
- ✅ Packed storage optimization
- ✅ Merkle proof
- ✅ OpenZeppelin EnumerableSet/Map
- ✅ Token Distribution System

## Quiz

1. ทำไม Mapping ถึง iterate ไม่ได้? และแก้อย่างไร?
2. Bitmap เหมาะกับ use case ไหน?
3. Pro-rata distribution คำนวณอย่างไร?
4. ความแตกต่างระหว่าง `delete array[i]` และ `removeSwap(i)`?

---

## Next: Part 07 - Structs และ Enums
