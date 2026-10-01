# Part 05: Control Flow และ Loops

## สารบัญ
1. If-Else Statements
2. Ternary Operator
3. For Loops
4. While Loops
5. Do-While Loops
6. Break และ Continue
7. Require, Revert, Assert
8. Custom Errors
9. Try-Catch
10. Workshop: Simple Voting System

---

## 1. If-Else Statements

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract ControlFlow {
    
    // Basic if-else
    function classify(uint256 n) public pure returns (string memory) {
        if (n == 0) {
            return "zero";
        } else if (n < 10) {
            return "small";
        } else if (n < 100) {
            return "medium";
        } else if (n < 1000) {
            return "large";
        } else {
            return "very large";
        }
    }
    
    // if ไม่มี else
    uint256 public counter;
    
    function incrementIfPositive(int256 n) public {
        if (n > 0) {
            counter++;
        }
        // ถ้า n <= 0 ไม่ทำอะไร
    }
    
    // Nested if
    function advancedClassify(uint256 price, uint256 quantity) public pure returns (string memory) {
        if (quantity > 0) {
            if (price > 1000) {
                if (quantity > 100) {
                    return "high value bulk order";
                } else {
                    return "high value small order";
                }
            } else {
                return "standard order";
            }
        } else {
            return "empty order";
        }
    }
    
    // Boolean conditions
    function checkAccess(
        address user,
        bool isVerified,
        uint256 balance,
        uint256 minBalance
    ) public pure returns (bool) {
        if (user == address(0)) return false;
        if (!isVerified) return false;
        if (balance < minBalance) return false;
        return true;
        
        // หรือ: return user != address(0) && isVerified && balance >= minBalance;
    }
}
```

---

## 2. Ternary Operator

```solidity
contract TernaryOperator {
    
    // condition ? value_if_true : value_if_false
    
    function max(uint256 a, uint256 b) public pure returns (uint256) {
        return a > b ? a : b;
    }
    
    function min(uint256 a, uint256 b) public pure returns (uint256) {
        return a < b ? a : b;
    }
    
    function abs(int256 n) public pure returns (uint256) {
        return uint256(n >= 0 ? n : -n);
    }
    
    function isEven(uint256 n) public pure returns (bool) {
        return n % 2 == 0;
    }
    
    function greet(bool formal) public pure returns (string memory) {
        return formal ? "Good day, sir/madam." : "Hey!";
    }
    
    // Nested ternary (อ่านยาก แต่ทำได้)
    function grade(uint256 score) public pure returns (string memory) {
        return score >= 90 ? "A" :
               score >= 80 ? "B" :
               score >= 70 ? "C" :
               score >= 60 ? "D" : "F";
    }
    
    // ใช้ใน assignment
    function clamp(uint256 value, uint256 min_, uint256 max_) public pure returns (uint256) {
        uint256 clamped = value < min_ ? min_ : value;
        return clamped > max_ ? max_ : clamped;
    }
}
```

---

## 3. For Loops

```solidity
contract ForLoops {
    
    // Basic for loop
    function sumTo(uint256 n) public pure returns (uint256 sum) {
        for (uint256 i = 1; i <= n; i++) {
            sum += i;
        }
        // return ไม่จำเป็นเพราะใช้ named return
    }
    
    // Loop กับ array
    uint256[] public numbers;
    
    function addNumbers(uint256[] calldata nums) public {
        for (uint256 i = 0; i < nums.length; i++) {
            numbers.push(nums[i]);
        }
    }
    
    function sumArray(uint256[] calldata arr) public pure returns (uint256 sum) {
        for (uint256 i = 0; i < arr.length; i++) {
            sum += arr[i];
        }
    }
    
    // ✅ Cache length
    function sumArrayOptimized(uint256[] memory arr) public pure returns (uint256 sum) {
        uint256 len = arr.length; // cache ใน memory
        for (uint256 i = 0; i < len; i++) {
            sum += arr[i];
        }
    }
    
    // Loop กับ mapping + array
    address[] public userList;
    mapping(address => uint256) public userBalance;
    
    function getTotalBalance() public view returns (uint256 total) {
        for (uint256 i = 0; i < userList.length; i++) {
            total += userBalance[userList[i]];
        }
    }
    
    // Reverse loop
    function reverseArray(uint256[] memory arr) public pure returns (uint256[] memory) {
        uint256 len = arr.length;
        for (uint256 i = 0; i < len / 2; i++) {
            (arr[i], arr[len - 1 - i]) = (arr[len - 1 - i], arr[i]);
        }
        return arr;
    }
    
    // Step loop
    function evenNumbers(uint256 max) public pure returns (uint256[] memory) {
        uint256 count = max / 2;
        uint256[] memory result = new uint256[](count);
        
        uint256 idx = 0;
        for (uint256 i = 2; i <= max; i += 2) {
            result[idx++] = i;
        }
        
        return result;
    }
    
    // ⚠️ ระวัง: Loop ยาวๆ อาจ out of gas!
    // ต้องกำหนด max iteration
    
    function safeBatchProcess(uint256 maxItems) public {
        uint256 processed = 0;
        for (uint256 i = 0; i < numbers.length && processed < maxItems; i++) {
            // process numbers[i]
            processed++;
        }
    }
}
```

---

## 4. While Loops

```solidity
contract WhileLoops {
    
    // Basic while
    function factorial(uint256 n) public pure returns (uint256 result) {
        result = 1;
        while (n > 1) {
            result *= n;
            n--;
        }
    }
    
    // Binary search
    function binarySearch(
        uint256[] memory sortedArr,
        uint256 target
    ) public pure returns (int256) {
        uint256 left = 0;
        uint256 right = sortedArr.length;
        
        while (left < right) {
            uint256 mid = left + (right - left) / 2;
            
            if (sortedArr[mid] == target) {
                return int256(mid);
            } else if (sortedArr[mid] < target) {
                left = mid + 1;
            } else {
                right = mid;
            }
        }
        
        return -1; // not found
    }
    
    // sqrt ด้วย Newton's method
    function sqrt(uint256 x) public pure returns (uint256 y) {
        if (x == 0) return 0;
        
        y = x;
        uint256 z = (x + 1) / 2;
        
        while (z < y) {
            y = z;
            z = (x / z + z) / 2;
        }
    }
    
    // ⚠️ While loop อาจ infinite loop ได้!
    // ต้องแน่ใจว่าเงื่อนไขจะเป็น false ในที่สุด
    
    // ❌ อันตราย: อาจ infinite loop
    // function dangerous(uint256 x) public pure {
    //     while (true) { // ไม่มีวันออก!
    //         x++;
    //     }
    // }
}
```

---

## 5. Do-While Loops

```solidity
contract DoWhileLoops {
    
    // do-while รันอย่างน้อยหนึ่งครั้ง
    
    function processAtLeastOnce(uint256 n) public pure returns (uint256 result) {
        result = 0;
        do {
            result += n;
            n--;
        } while (n > 0);
        
        return result;
    }
    
    // Countdown
    function countdown(uint256 from) public pure returns (uint256[] memory) {
        uint256[] memory result = new uint256[](from + 1);
        uint256 idx = 0;
        
        uint256 i = from;
        do {
            result[idx++] = i;
        } while (i-- > 0);
        
        return result;
    }
    
    // ใช้ประโยชน์ do-while: รัน validation ก่อน loop หลัก
    function collectUntilFull(
        uint256[] calldata data,
        uint256 target
    ) public pure returns (uint256[] memory collected) {
        collected = new uint256[](data.length);
        uint256 count = 0;
        
        do {
            if (count >= data.length) break;
            collected[count] = data[count];
            count++;
        } while (count < target && count < data.length);
        
        // Trim array
        assembly {
            mstore(collected, count)
        }
    }
}
```

---

## 6. Break และ Continue

```solidity
contract BreakAndContinue {
    
    // break: ออกจาก loop ทันที
    function findFirst(
        uint256[] memory arr,
        uint256 target
    ) public pure returns (int256) {
        for (uint256 i = 0; i < arr.length; i++) {
            if (arr[i] == target) {
                return int256(i);
            }
        }
        return -1;
    }
    
    // หา maximum
    function findMax(uint256[] memory arr) public pure returns (uint256 maxVal) {
        require(arr.length > 0, "Empty array");
        
        maxVal = arr[0];
        for (uint256 i = 1; i < arr.length; i++) {
            if (arr[i] > maxVal) {
                maxVal = arr[i];
            }
        }
    }
    
    // continue: ข้ามไป iteration ถัดไป
    function filterEven(uint256[] memory arr) public pure returns (uint256[] memory) {
        // Count even numbers first
        uint256 count = 0;
        for (uint256 i = 0; i < arr.length; i++) {
            if (arr[i] % 2 == 0) count++;
        }
        
        uint256[] memory result = new uint256[](count);
        uint256 idx = 0;
        
        for (uint256 i = 0; i < arr.length; i++) {
            if (arr[i] % 2 != 0) continue; // ข้ามเลขคี่
            result[idx++] = arr[i];
        }
        
        return result;
    }
    
    // Nested loops กับ break
    function findPair(
        uint256[] memory arr,
        uint256 target
    ) public pure returns (int256 i, int256 j) {
        bool found = false;
        
        for (uint256 a = 0; a < arr.length && !found; a++) {
            for (uint256 b = a + 1; b < arr.length; b++) {
                if (arr[a] + arr[b] == target) {
                    i = int256(a);
                    j = int256(b);
                    found = true;
                    break; // ออกจาก inner loop
                }
            }
        }
        
        if (!found) return (-1, -1);
    }
    
    // Batch processing with limit
    uint256 public processedCount;
    uint256[] public pendingItems;
    
    function batchProcess(uint256 batchSize) public {
        uint256 processed = 0;
        
        for (uint256 i = 0; i < pendingItems.length; i++) {
            if (processed >= batchSize) break;
            
            // process item
            // ...
            
            processed++;
            processedCount++;
        }
    }
}
```

---

## 7. Require, Revert, Assert

```solidity
contract ErrorHandling {
    
    // === require() ===
    // ใช้สำหรับ input validation และ preconditions
    // ถ้าเป็น false → revert + คืน gas ที่เหลือ
    
    mapping(address => uint256) public balances;
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    function deposit() public payable {
        require(msg.value > 0, "Must send ETH");
        require(msg.value <= 100 ether, "Max 100 ETH");
        balances[msg.sender] += msg.value;
    }
    
    function withdraw(uint256 amount) public {
        require(amount > 0, "Amount must be > 0");
        require(balances[msg.sender] >= amount, "Insufficient balance");
        require(address(this).balance >= amount, "Contract has insufficient funds");
        
        balances[msg.sender] -= amount;
        
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
    
    // === revert() ===
    // เหมือน require แต่ flexible กว่า
    
    enum OrderStatus { Pending, Active, Completed, Cancelled }
    
    struct Order {
        address buyer;
        uint256 amount;
        OrderStatus status;
    }
    
    mapping(uint256 => Order) public orders;
    
    function cancelOrder(uint256 orderId) public {
        Order storage order = orders[orderId];
        
        if (order.buyer == address(0)) {
            revert("Order does not exist");
        }
        
        if (order.buyer != msg.sender) {
            revert("Not your order");
        }
        
        if (order.status != OrderStatus.Pending) {
            revert(string(abi.encodePacked(
                "Cannot cancel: status is ",
                uint2str(uint256(order.status))
            )));
        }
        
        order.status = OrderStatus.Cancelled;
    }
    
    // === assert() ===
    // ใช้สำหรับ invariants ที่ไม่ควรเป็น false เลย
    // ถ้าเป็น false → revert + ใช้ gas ทั้งหมด (ต่างจาก require)
    
    uint256 public totalSupply;
    mapping(address => uint256) public tokenBalances;
    
    function transfer(address to, uint256 amount) public {
        require(tokenBalances[msg.sender] >= amount, "Insufficient balance");
        
        uint256 balanceBefore = tokenBalances[msg.sender] + tokenBalances[to];
        
        tokenBalances[msg.sender] -= amount;
        tokenBalances[to] += amount;
        
        // Invariant: total balance ต้องไม่เปลี่ยน
        assert(tokenBalances[msg.sender] + tokenBalances[to] == balanceBefore);
    }
    
    // เมื่อควรใช้อะไร?
    // require → ตรวจสอบ input, preconditions, external data
    // revert  → complex error conditions, custom messages
    // assert  → invariants, "this should never happen"
    
    function uint2str(uint256 n) internal pure returns (string memory) {
        if (n == 0) return "0";
        uint256 j = n;
        uint256 len;
        while (j != 0) { len++; j /= 10; }
        bytes memory bstr = new bytes(len);
        uint256 k = len;
        while (n != 0) {
            k--;
            bstr[k] = bytes1(uint8(48 + n % 10));
            n /= 10;
        }
        return string(bstr);
    }
}
```

---

## 8. Custom Errors (Solidity 0.8.4+)

Custom Errors ถูกกว่า string error messages มาก!

```solidity
contract CustomErrors {
    
    // Define custom errors (ประหยัด gas มาก!)
    error NotOwner(address caller, address owner);
    error InsufficientBalance(uint256 available, uint256 required);
    error InvalidAddress(address addr);
    error AmountTooLarge(uint256 amount, uint256 maximum);
    error OrderNotFound(uint256 orderId);
    error OrderNotPending(uint256 orderId, uint8 currentStatus);
    error Unauthorized(address caller, bytes32 requiredRole);
    error DeadlineExpired(uint256 deadline, uint256 currentTime);
    
    address public owner;
    mapping(address => uint256) public balances;
    
    constructor() {
        owner = msg.sender;
    }
    
    // Custom error ใช้แทน require string
    modifier onlyOwner() {
        if (msg.sender != owner) {
            revert NotOwner(msg.sender, owner);
        }
        _;
    }
    
    function deposit() external payable {
        if (msg.value == 0) revert InsufficientBalance(0, 1);
        balances[msg.sender] += msg.value;
    }
    
    function withdraw(uint256 amount) external {
        uint256 balance = balances[msg.sender];
        if (balance < amount) {
            revert InsufficientBalance(balance, amount);
        }
        
        balances[msg.sender] -= amount;
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
    
    function transfer(address to, uint256 amount) external {
        if (to == address(0)) revert InvalidAddress(to);
        
        uint256 balance = balances[msg.sender];
        if (balance < amount) revert InsufficientBalance(balance, amount);
        
        balances[msg.sender] -= amount;
        balances[to] += amount;
    }
    
    // Gas comparison:
    // require("Insufficient balance") → ~50 gas per character
    // revert InsufficientBalance(...)  → ~5 gas per parameter
    
    // Custom errors ใน inheritance
    function adminAction() external onlyOwner {
        // Only owner can call this
    }
}

// Errors ใน interface สำหรับ reuse
interface IErrors {
    error Paused();
    error NotPaused();
    error ZeroAmount();
    error ZeroAddress();
}

contract PausableContract is IErrors {
    
    bool public paused;
    
    function pause() external {
        if (paused) revert Paused();
        paused = true;
    }
    
    function unpause() external {
        if (!paused) revert NotPaused();
        paused = false;
    }
    
    function execute(uint256 amount) external {
        if (paused) revert Paused();
        if (amount == 0) revert ZeroAmount();
        // ...
    }
}
```

---

## 9. Try-Catch

```solidity
interface IExternalContract {
    function riskyOperation(uint256 x) external returns (uint256);
    function getData() external view returns (string memory);
}

contract TryCatch {
    
    event Success(uint256 result);
    event Failed(bytes reason);
    event FailedString(string reason);
    
    IExternalContract public externalContract;
    
    constructor(address _external) {
        externalContract = IExternalContract(_external);
    }
    
    // try-catch สำหรับ external calls
    function callRiskyOperation(uint256 x) public returns (uint256 result) {
        try externalContract.riskyOperation(x) returns (uint256 value) {
            // ✅ Success
            result = value;
            emit Success(value);
        } catch Error(string memory reason) {
            // revert("reason") or require(false, "reason")
            emit FailedString(reason);
            result = 0;
        } catch (bytes memory reason) {
            // custom error หรือ panic
            emit Failed(reason);
            result = 0;
        }
    }
    
    // try-catch สำหรับ contract creation
    function safeDeployment() public returns (address) {
        try new SubContract() returns (SubContract sub) {
            return address(sub);
        } catch {
            return address(0);
        }
    }
    
    // Catching ประเภทต่างๆ
    function comprehensiveCatch(uint256 x) public {
        try externalContract.riskyOperation(x) returns (uint256) {
            // success
        } catch Error(string memory reason) {
            // require/revert with string message
        } catch Panic(uint256 errorCode) {
            // Panic errors (arithmetic overflow, etc.)
            // errorCode: 0x01=assert, 0x11=overflow, 0x12=divide by 0
        } catch (bytes memory lowLevelData) {
            // custom errors หรือ other
        }
    }
    
    // ตัวอย่าง: Safe token transfer
    function safeTransfer(address token, address to, uint256 amount) public returns (bool) {
        (bool success, bytes memory data) = token.call(
            abi.encodeWithSignature("transfer(address,uint256)", to, amount)
        );
        
        // Handle both:
        // 1. ERC-20 ที่ return bool
        // 2. ERC-20 เก่าที่ไม่ return อะไร
        return success && (data.length == 0 || abi.decode(data, (bool)));
    }
}

contract SubContract {
    uint256 public value;
    
    constructor() {
        value = 42;
    }
    
    function riskyOperation(uint256 x) external pure returns (uint256) {
        require(x > 0, "Must be positive");
        return x * 2;
    }
}
```

---

## 10. Workshop: Simple Voting System

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title Voting
 * @dev ระบบโหวตแบบง่าย
 */
contract Voting {
    
    // === Custom Errors ===
    error AlreadyVoted(address voter);
    error VotingEnded(uint256 endTime, uint256 currentTime);
    error VotingNotEnded(uint256 endTime, uint256 currentTime);
    error InvalidCandidate(uint256 candidateId);
    error NotAuthorized(address caller);
    error RegistrationClosed();
    error VotingNotStarted();
    
    // === Types ===
    
    struct Candidate {
        uint256 id;
        string name;
        string description;
        uint256 voteCount;
        bool active;
    }
    
    struct VoteRecord {
        uint256 candidateId;
        uint256 timestamp;
        bool hasVoted;
    }
    
    // === State ===
    
    address public immutable CHAIRPERSON;
    string public ballotTitle;
    
    uint256 public startTime;
    uint256 public endTime;
    uint256 public registrationDeadline;
    
    Candidate[] public candidates;
    mapping(address => VoteRecord) public voteRecords;
    mapping(address => bool) public registeredVoters;
    
    uint256 public totalVotes;
    uint256 public registeredVoterCount;
    
    bool public resultsPublished;
    
    // === Events ===
    
    event VoterRegistered(address indexed voter, uint256 timestamp);
    event VoteCast(address indexed voter, uint256 indexed candidateId, uint256 timestamp);
    event CandidateAdded(uint256 indexed candidateId, string name);
    event ResultsPublished(uint256 indexed winnerCandidateId, uint256 voteCount);
    
    // === Constructor ===
    
    constructor(
        string memory _title,
        uint256 _startTime,
        uint256 _endTime,
        uint256 _registrationDeadline
    ) {
        require(_startTime > block.timestamp, "Start must be in future");
        require(_endTime > _startTime, "End must be after start");
        require(_registrationDeadline <= _startTime, "Registration must close before start");
        
        CHAIRPERSON = msg.sender;
        ballotTitle = _title;
        startTime = _startTime;
        endTime = _endTime;
        registrationDeadline = _registrationDeadline;
    }
    
    // === Modifiers ===
    
    modifier onlyChairperson() {
        if (msg.sender != CHAIRPERSON) revert NotAuthorized(msg.sender);
        _;
    }
    
    modifier onlyDuringVoting() {
        if (block.timestamp < startTime) revert VotingNotStarted();
        if (block.timestamp > endTime) revert VotingEnded(endTime, block.timestamp);
        _;
    }
    
    modifier onlyAfterVoting() {
        if (block.timestamp <= endTime) revert VotingNotEnded(endTime, block.timestamp);
        _;
    }
    
    // === Setup Functions (onlyChairperson) ===
    
    function addCandidate(
        string calldata name,
        string calldata description
    ) external onlyChairperson {
        require(block.timestamp < startTime, "Voting already started");
        require(bytes(name).length > 0, "Name required");
        
        uint256 candidateId = candidates.length;
        
        candidates.push(Candidate({
            id: candidateId,
            name: name,
            description: description,
            voteCount: 0,
            active: true
        }));
        
        emit CandidateAdded(candidateId, name);
    }
    
    function addCandidates(
        string[] calldata names,
        string[] calldata descriptions
    ) external onlyChairperson {
        require(names.length == descriptions.length, "Length mismatch");
        require(block.timestamp < startTime, "Voting already started");
        
        for (uint256 i = 0; i < names.length; i++) {
            require(bytes(names[i]).length > 0, "Empty name");
            
            uint256 candidateId = candidates.length;
            candidates.push(Candidate({
                id: candidateId,
                name: names[i],
                description: descriptions[i],
                voteCount: 0,
                active: true
            }));
            
            emit CandidateAdded(candidateId, names[i]);
        }
    }
    
    function disableCandidate(uint256 candidateId) external onlyChairperson {
        if (candidateId >= candidates.length) revert InvalidCandidate(candidateId);
        require(block.timestamp < startTime, "Voting already started");
        candidates[candidateId].active = false;
    }
    
    // === Registration ===
    
    function register() external {
        if (block.timestamp > registrationDeadline) revert RegistrationClosed();
        require(!registeredVoters[msg.sender], "Already registered");
        
        registeredVoters[msg.sender] = true;
        registeredVoterCount++;
        
        emit VoterRegistered(msg.sender, block.timestamp);
    }
    
    function registerBatch(address[] calldata voters) external onlyChairperson {
        if (block.timestamp > registrationDeadline) revert RegistrationClosed();
        
        for (uint256 i = 0; i < voters.length; i++) {
            address voter = voters[i];
            if (voter != address(0) && !registeredVoters[voter]) {
                registeredVoters[voter] = true;
                registeredVoterCount++;
                emit VoterRegistered(voter, block.timestamp);
            }
        }
    }
    
    // === Voting ===
    
    function vote(uint256 candidateId) external onlyDuringVoting {
        // ตรวจสอบสิทธิ์
        if (!registeredVoters[msg.sender]) revert NotAuthorized(msg.sender);
        if (voteRecords[msg.sender].hasVoted) revert AlreadyVoted(msg.sender);
        if (candidateId >= candidates.length) revert InvalidCandidate(candidateId);
        if (!candidates[candidateId].active) revert InvalidCandidate(candidateId);
        
        // บันทึกการโหวต
        voteRecords[msg.sender] = VoteRecord({
            candidateId: candidateId,
            timestamp: block.timestamp,
            hasVoted: true
        });
        
        candidates[candidateId].voteCount++;
        totalVotes++;
        
        emit VoteCast(msg.sender, candidateId, block.timestamp);
    }
    
    // === Results ===
    
    function getWinner() public view onlyAfterVoting returns (
        uint256 winnerId,
        string memory winnerName,
        uint256 winnerVotes
    ) {
        require(candidates.length > 0, "No candidates");
        
        winnerId = 0;
        winnerVotes = 0;
        
        for (uint256 i = 0; i < candidates.length; i++) {
            if (candidates[i].active && candidates[i].voteCount > winnerVotes) {
                winnerId = i;
                winnerVotes = candidates[i].voteCount;
            }
        }
        
        winnerName = candidates[winnerId].name;
    }
    
    function getAllResults() external view onlyAfterVoting returns (
        uint256[] memory ids,
        string[] memory names,
        uint256[] memory votes
    ) {
        uint256 len = candidates.length;
        ids   = new uint256[](len);
        names = new string[](len);
        votes = new uint256[](len);
        
        for (uint256 i = 0; i < len; i++) {
            ids[i]   = candidates[i].id;
            names[i] = candidates[i].name;
            votes[i] = candidates[i].voteCount;
        }
    }
    
    function getTurnout() external view returns (
        uint256 totalRegistered,
        uint256 totalVoted,
        uint256 turnoutPercent
    ) {
        totalRegistered = registeredVoterCount;
        totalVoted = totalVotes;
        
        if (registeredVoterCount > 0) {
            turnoutPercent = (totalVotes * 100) / registeredVoterCount;
        }
    }
    
    // === View Functions ===
    
    function getCandidateCount() external view returns (uint256) {
        return candidates.length;
    }
    
    function isVotingActive() external view returns (bool) {
        return block.timestamp >= startTime && block.timestamp <= endTime;
    }
    
    function getVotingStatus() external view returns (
        bool registrationOpen,
        bool votingActive,
        bool votingEnded,
        uint256 timeRemaining
    ) {
        registrationOpen = block.timestamp <= registrationDeadline;
        votingActive = block.timestamp >= startTime && block.timestamp <= endTime;
        votingEnded = block.timestamp > endTime;
        
        if (votingActive) {
            timeRemaining = endTime - block.timestamp;
        }
    }
    
    function hasVoted(address voter) external view returns (bool) {
        return voteRecords[voter].hasVoted;
    }
    
    function getVoterChoice(address voter) external view returns (
        bool voted,
        uint256 candidateId,
        string memory candidateName
    ) {
        VoteRecord memory record = voteRecords[voter];
        voted = record.hasVoted;
        
        if (voted) {
            candidateId = record.candidateId;
            candidateName = candidates[record.candidateId].name;
        }
    }
}
```

### Deploy Script

```typescript
// scripts/deployVoting.ts
import { ethers } from "hardhat";

async function main() {
    const [deployer] = await ethers.getSigners();
    
    const now = Math.floor(Date.now() / 1000);
    const registrationDeadline = now + 3600;  // 1 hour from now
    const startTime = now + 3700;              // 1 hour and ~2 mins
    const endTime = now + 7200;                // 2 hours from now
    
    const Voting = await ethers.getContractFactory("Voting");
    const voting = await Voting.deploy(
        "Board Election 2024",
        startTime,
        endTime,
        registrationDeadline
    );
    
    await voting.waitForDeployment();
    console.log("Voting deployed to:", await voting.getAddress());
    
    // Add candidates
    await voting.addCandidate("Alice", "Experienced leader");
    await voting.addCandidate("Bob", "Innovation focused");
    await voting.addCandidate("Carol", "Community builder");
    
    console.log("Candidates added!");
}

main().catch(console.error);
```

---

## สรุป Part 05

Control Flow ที่เรียนรู้:
- ✅ If-else statements
- ✅ Ternary operator
- ✅ For loops (basic, optimized)
- ✅ While loops
- ✅ Do-while loops
- ✅ Break และ continue
- ✅ require, revert, assert
- ✅ Custom errors
- ✅ Try-catch
- ✅ Voting System

## Quiz

1. ความแตกต่างระหว่าง `require` และ `assert`?
2. ทำไม Custom Errors ถูกกว่า string messages?
3. `try-catch` ใช้ได้กับ internal calls หรือไม่?
4. ทำไมควร cache array length ใน for loop?

## แบบฝึกหัด

1. สร้าง `FizzBuzz` contract ที่ return "Fizz" สำหรับ 3, "Buzz" สำหรับ 5, "FizzBuzz" สำหรับ 15
2. สร้าง `Lottery` contract ที่:
   - รับ participants ที่ส่ง ETH
   - มี `drawWinner()` เลือกผู้ชนะแบบ pseudo-random
   - โอน prize ให้ผู้ชนะ
3. เพิ่ม custom errors ให้กับ Lottery contract

---

## Next: Part 06 - Arrays และ Mappings เชิงลึก
