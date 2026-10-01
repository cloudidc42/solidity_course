# Part 22: Oracles และ Chainlink

## สารบัญ
1. Oracle Problem
2. Chainlink Price Feeds
3. Chainlink VRF (Verifiable Random)
4. Chainlink Automation (Keepers)
5. Workshop: Price-based Contract

---

## 1. Oracle Problem

```
Oracle Problem:
Smart contracts ไม่สามารถเข้าถึงข้อมูลนอก blockchain ได้โดยตรง
- ราคาทองคำ, ดอกเบี้ย, ผลกีฬา
- Random numbers
- ข้อมูล API

วิธีแก้:
1. Centralized Oracle: เชื่อถือ 1 แหล่ง (ง่าย แต่ single point of failure)
2. Decentralized Oracle (Chainlink): หลาย nodes → consensus
3. TWAP (Time-Weighted Average Price): ราคาเฉลี่ยจาก DEX
4. Merkle Proof: ข้อมูล off-chain พิสูจน์ on-chain

Chainlink Architecture:
User Contract → Chainlink Aggregator → Multiple Oracle Nodes → External APIs
```

---

## 2. Chainlink Price Feeds

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Chainlink Price Feed Integration
 * 
 * AggregatorV3Interface: interface สำหรับ Chainlink price feeds
 * 
 * Feed Addresses (Ethereum Mainnet):
 * ETH/USD: 0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419
 * BTC/USD: 0xF4030086522a5bEEa4988F8cA5B36dbC97BeE88c
 * 
 * latestRoundData returns:
 * - roundId: ID ของ round นี้
 * - answer: ราคา (8 decimals สำหรับ USD pairs)
 * - startedAt: เวลาเริ่ม round
 * - updatedAt: เวลาอัพเดทล่าสุด
 * - answeredInRound: round ที่ตอบ
 */
interface AggregatorV3Interface {
    function decimals() external view returns (uint8);
    function description() external view returns (string memory);
    function version() external view returns (uint256);
    
    function latestRoundData()
        external
        view
        returns (
            uint80 roundId,
            int256 answer,
            uint256 startedAt,
            uint256 updatedAt,
            uint80 answeredInRound
        );
    
    function getRoundData(uint80 _roundId)
        external
        view
        returns (
            uint80 roundId,
            int256 answer,
            uint256 startedAt,
            uint256 updatedAt,
            uint80 answeredInRound
        );
}

contract PriceFeedConsumer {
    
    AggregatorV3Interface public immutable ethUsdFeed;
    AggregatorV3Interface public immutable btcUsdFeed;
    
    uint256 public constant STALENESS_THRESHOLD = 3600; // 1 hour
    
    error StalePrice(uint256 updatedAt, uint256 threshold);
    error InvalidPrice(int256 price);
    error RoundNotComplete(uint80 roundId, uint80 answeredInRound);
    
    constructor(address _ethUsdFeed, address _btcUsdFeed) {
        ethUsdFeed = AggregatorV3Interface(_ethUsdFeed);
        btcUsdFeed = AggregatorV3Interface(_btcUsdFeed);
    }
    
    function getETHPrice() public view returns (uint256) {
        return _getPrice(ethUsdFeed);
    }
    
    function getBTCPrice() public view returns (uint256) {
        return _getPrice(btcUsdFeed);
    }
    
    function _getPrice(AggregatorV3Interface feed) internal view returns (uint256) {
        (
            uint80 roundId,
            int256 answer,
            ,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = feed.latestRoundData();
        
        // 1. Check price is positive
        if (answer <= 0) revert InvalidPrice(answer);
        
        // 2. Check price is fresh (not stale)
        if (block.timestamp - updatedAt > STALENESS_THRESHOLD) {
            revert StalePrice(updatedAt, STALENESS_THRESHOLD);
        }
        
        // 3. Check round is complete
        if (answeredInRound < roundId) {
            revert RoundNotComplete(roundId, answeredInRound);
        }
        
        // Convert to 18 decimals
        uint8 feedDecimals = feed.decimals();
        uint256 price = uint256(answer);
        
        if (feedDecimals < 18) {
            price = price * 10 ** (18 - feedDecimals);
        } else if (feedDecimals > 18) {
            price = price / 10 ** (feedDecimals - 18);
        }
        
        return price;
    }
    
    // Calculate ETH value in USD
    function ethToUsd(uint256 ethAmount) external view returns (uint256) {
        uint256 ethPrice = getETHPrice();
        return (ethAmount * ethPrice) / 1e18;
    }
    
    // Calculate USD to ETH
    function usdToEth(uint256 usdAmount) external view returns (uint256) {
        uint256 ethPrice = getETHPrice();
        return (usdAmount * 1e18) / ethPrice;
    }
}
```

---

## 3. Chainlink VRF (Verifiable Random Function)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Chainlink VRF v2 - Verifiable Random Numbers
 * 
 * Cryptographically secure randomness จาก off-chain
 * ป้องกัน miner manipulation
 * 
 * Flow:
 * 1. Contract requests random number (pays LINK)
 * 2. Chainlink node generates VRF proof
 * 3. VRF Coordinator verifies proof and calls fulfillRandomWords
 * 4. Contract receives verifiably random number
 */
interface VRFCoordinatorV2Interface {
    function requestRandomWords(
        bytes32 keyHash,
        uint64 subId,
        uint16 minimumRequestConfirmations,
        uint32 callbackGasLimit,
        uint32 numWords
    ) external returns (uint256 requestId);
}

abstract contract VRFConsumerBaseV2 {
    address private immutable vrfCoordinator;
    
    constructor(address _vrfCoordinator) {
        vrfCoordinator = _vrfCoordinator;
    }
    
    function fulfillRandomWords(uint256 requestId, uint256[] memory randomWords) internal virtual;
    
    function rawFulfillRandomWords(uint256 requestId, uint256[] memory randomWords) external {
        require(msg.sender == vrfCoordinator, "Only VRF Coordinator");
        fulfillRandomWords(requestId, randomWords);
    }
}

contract NFTLottery is VRFConsumerBaseV2 {
    
    VRFCoordinatorV2Interface public immutable coordinator;
    
    // Chainlink VRF Config (Ethereum Sepolia)
    bytes32 public constant KEY_HASH = 0x474e34a077df58807dbe9c96d3c009b23b3c6d0cce433e59bbf5b34f823bc56c;
    uint64 public subscriptionId;
    uint16 public constant REQUEST_CONFIRMATIONS = 3;
    uint32 public constant CALLBACK_GAS_LIMIT = 100000;
    uint32 public constant NUM_WORDS = 1;
    
    // Lottery state
    address[] public participants;
    mapping(uint256 => bool) public requestPending;
    mapping(uint256 => address) public requestToSender;
    
    address public lastWinner;
    uint256 public prizePool;
    bool public lotteryOpen;
    
    event LotteryEntered(address indexed participant, uint256 totalParticipants);
    event RandomnessRequested(uint256 requestId);
    event WinnerSelected(address indexed winner, uint256 prize);
    
    error LotteryClosed();
    error InsufficientEntryFee();
    error NotEnoughParticipants();
    
    uint256 public constant ENTRY_FEE = 0.01 ether;
    
    constructor(address _vrfCoordinator, uint64 _subscriptionId)
        VRFConsumerBaseV2(_vrfCoordinator)
    {
        coordinator = VRFCoordinatorV2Interface(_vrfCoordinator);
        subscriptionId = _subscriptionId;
        lotteryOpen = true;
    }
    
    function enter() external payable {
        if (!lotteryOpen) revert LotteryClosed();
        if (msg.value < ENTRY_FEE) revert InsufficientEntryFee();
        
        participants.push(msg.sender);
        prizePool += msg.value;
        
        emit LotteryEntered(msg.sender, participants.length);
    }
    
    function pickWinner() external returns (uint256 requestId) {
        if (participants.length < 2) revert NotEnoughParticipants();
        
        lotteryOpen = false;
        
        requestId = coordinator.requestRandomWords(
            KEY_HASH,
            subscriptionId,
            REQUEST_CONFIRMATIONS,
            CALLBACK_GAS_LIMIT,
            NUM_WORDS
        );
        
        requestPending[requestId] = true;
        requestToSender[requestId] = msg.sender;
        
        emit RandomnessRequested(requestId);
    }
    
    function fulfillRandomWords(uint256 requestId, uint256[] memory randomWords) internal override {
        require(requestPending[requestId], "Request not pending");
        
        uint256 winnerIndex = randomWords[0] % participants.length;
        address winner = participants[winnerIndex];
        
        lastWinner = winner;
        requestPending[requestId] = false;
        
        uint256 prize = prizePool;
        prizePool = 0;
        
        // Reset lottery
        delete participants;
        lotteryOpen = true;
        
        // Pay winner
        payable(winner).transfer(prize);
        
        emit WinnerSelected(winner, prize);
    }
    
    function getParticipants() external view returns (address[] memory) {
        return participants;
    }
}
```

---

## 4. Chainlink Automation (Keepers)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Chainlink Automation (formerly Keepers)
 * 
 * ทำให้ contract ทำงานอัตโนมัติโดย Chainlink network
 * เช่น: harvest yield, rebalance portfolio, expire options
 * 
 * AutomationCompatibleInterface:
 * - checkUpkeep: ตรวจสอบว่าต้อง perform ไหม
 * - performUpkeep: รัน action จริง
 */
interface AutomationCompatibleInterface {
    function checkUpkeep(bytes calldata checkData)
        external
        view
        returns (bool upkeepNeeded, bytes memory performData);
    
    function performUpkeep(bytes calldata performData) external;
}

contract AutoHarvest is AutomationCompatibleInterface {
    
    IYieldFarm public immutable farm;
    
    uint256 public lastHarvest;
    uint256 public constant HARVEST_INTERVAL = 6 hours;
    uint256 public constant MIN_HARVEST_AMOUNT = 100e18; // 100 tokens
    
    address public owner;
    
    event Harvested(uint256 amount, uint256 timestamp);
    
    constructor(address _farm) {
        farm = IYieldFarm(_farm);
        owner = msg.sender;
        lastHarvest = block.timestamp;
    }
    
    // Called by Chainlink nodes to check if action needed
    function checkUpkeep(bytes calldata /* checkData */)
        external
        view
        override
        returns (bool upkeepNeeded, bytes memory performData)
    {
        bool intervalPassed = (block.timestamp - lastHarvest) >= HARVEST_INTERVAL;
        uint256 pendingRewards = farm.pendingRewards(address(this));
        bool enoughRewards = pendingRewards >= MIN_HARVEST_AMOUNT;
        
        upkeepNeeded = intervalPassed && enoughRewards;
        performData = abi.encode(pendingRewards);
    }
    
    // Called by Chainlink nodes when checkUpkeep returns true
    function performUpkeep(bytes calldata performData) external override {
        (uint256 expectedAmount) = abi.decode(performData, (uint256));
        
        // Re-verify conditions (защита от front-run)
        uint256 pendingRewards = farm.pendingRewards(address(this));
        require(pendingRewards >= MIN_HARVEST_AMOUNT, "Rewards too low");
        require((block.timestamp - lastHarvest) >= HARVEST_INTERVAL, "Too soon");
        
        lastHarvest = block.timestamp;
        
        uint256 harvested = farm.harvest();
        
        emit Harvested(harvested, block.timestamp);
    }
}

interface IYieldFarm {
    function pendingRewards(address user) external view returns (uint256);
    function harvest() external returns (uint256);
}
```

---

## 5. Workshop: Price-Gated NFT Mint

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * NFT ที่ราคา mint ขึ้นอยู่กับ ETH/USD price
 * ราคา NFT คงที่ $100 USD แต่จ่ายเป็น ETH
 */
contract PriceGatedNFT {
    
    AggregatorV3Interface public immutable priceFeed;
    
    uint256 public constant USD_PRICE = 100; // $100 USD
    uint256 public constant MAX_SUPPLY = 10000;
    
    mapping(address => uint256) public balanceOf;
    mapping(uint256 => address) public ownerOf;
    
    uint256 public totalSupply;
    
    event Transfer(address indexed from, address indexed to, uint256 tokenId);
    
    error InsufficientPayment(uint256 required, uint256 sent);
    error SoldOut();
    
    constructor(address _priceFeed) {
        priceFeed = AggregatorV3Interface(_priceFeed);
    }
    
    // คำนวณ ETH ที่ต้องจ่ายสำหรับ $100 USD
    function getMintPrice() public view returns (uint256) {
        (, int256 ethPrice, , uint256 updatedAt, ) = priceFeed.latestRoundData();
        
        require(ethPrice > 0, "Invalid price");
        require(block.timestamp - updatedAt <= 3600, "Stale price");
        
        // ETH price has 8 decimals: e.g., 200000000000 = $2000
        // We want: $100 / ETH_PRICE ETH
        // = 100 * 1e8 / ethPrice * 1e18
        // = 100 * 1e26 / ethPrice
        
        return (USD_PRICE * 1e26) / uint256(ethPrice);
    }
    
    function mint() external payable {
        if (totalSupply >= MAX_SUPPLY) revert SoldOut();
        
        uint256 required = getMintPrice();
        if (msg.value < required) revert InsufficientPayment(required, msg.value);
        
        uint256 tokenId = totalSupply++;
        ownerOf[tokenId] = msg.sender;
        balanceOf[msg.sender]++;
        
        emit Transfer(address(0), msg.sender, tokenId);
        
        // Refund excess
        if (msg.value > required) {
            payable(msg.sender).transfer(msg.value - required);
        }
    }
}

// Mock for testing
contract MockAggregator is AggregatorV3Interface {
    int256 private _answer;
    uint8 private _decimals = 8;
    uint256 private _updatedAt;
    
    constructor(int256 initialAnswer) {
        _answer = initialAnswer;
        _updatedAt = block.timestamp;
    }
    
    function setAnswer(int256 answer) external {
        _answer = answer;
        _updatedAt = block.timestamp;
    }
    
    function decimals() external view override returns (uint8) { return _decimals; }
    function description() external pure override returns (string memory) { return "Mock ETH/USD"; }
    function version() external pure override returns (uint256) { return 4; }
    
    function latestRoundData() external view override returns (
        uint80, int256, uint256, uint256, uint80
    ) {
        return (1, _answer, block.timestamp, _updatedAt, 1);
    }
    
    function getRoundData(uint80) external view override returns (
        uint80, int256, uint256, uint256, uint80
    ) {
        return (1, _answer, block.timestamp, _updatedAt, 1);
    }
}
```

---

## สรุป Part 22

Oracles ที่เรียนรู้:
- ✅ Oracle problem และวิธีแก้
- ✅ Chainlink Price Feeds + staleness check
- ✅ Chainlink VRF v2 (ตัวเลขสุ่มที่ยืนยันได้)
- ✅ Chainlink Automation (Keepers)
- ✅ Price-gated NFT mint

## Quiz

1. ทำไมต้อง check staleness สำหรับ price feeds?
2. VRF แตกต่างจาก `block.timestamp % N` อย่างไร?
3. `checkUpkeep` ต้องเป็น view function เพราะอะไร?
4. ถ้า ETH price เป็น $2000 จะต้องจ่าย ETH เท่าไหร่สำหรับ $100 NFT?

---

## Next: Part 23 - Layer 2 Solutions
