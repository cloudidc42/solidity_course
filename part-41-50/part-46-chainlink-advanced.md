# Part 46: Chainlink Advanced Integration

## สารบัญ
1. Data Streams (Low Latency Feeds)
2. Functions (Serverless Compute)
3. CCIP (Cross-Chain Messaging)
4. Proof of Reserve
5. Workshop: Multi-Oracle Protocol

---

## 1. Chainlink Data Streams

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Chainlink Data Streams:
 * Low-latency price feeds สำหรับ perps/derivatives
 * ต่างจาก Price Feeds ทั่วไป:
 * - Push-based vs Pull-based
 * - < 500ms latency (vs 1 block)
 * - Reports verified on-chain
 */
interface IVerifierProxy {
    function verify(
        bytes calldata payload,
        bytes calldata parameterPayload
    ) external payable returns (bytes memory verifierResponse);
}

interface IFeeManager {
    function getFeeAndReward(
        address subscriber,
        bytes memory report,
        address quoteAddress
    ) external view returns (
        IFeeManager.Asset memory,
        IFeeManager.Asset memory,
        uint256
    );
    
    struct Asset {
        address assetAddress;
        uint256 amount;
    }
}

/**
 * Perp engine using Data Streams for low-latency prices
 */
contract PerpEngineWithStreams {
    
    IVerifierProxy public immutable verifier;
    
    // Report struct for Data Streams v3
    struct Report {
        bytes32 feedId;
        uint32 validFromTimestamp;
        uint32 observationsTimestamp;
        uint192 nativeFee;
        uint192 linkFee;
        uint32 expiresAt;
        int192 price;
        int192 bid;
        int192 ask;
    }
    
    bytes32 constant ETH_USD_FEED_ID = 0x000359843a543ee2fe414dc14c7e7920ef10f4372990b79d6361cdc0dd1ba782;
    
    struct Position {
        int256 size;
        uint256 entryPrice;
        uint256 margin;
        uint256 timestamp;
    }
    
    mapping(address => Position) public positions;
    
    constructor(address _verifier) {
        verifier = IVerifierProxy(_verifier);
    }
    
    /**
     * Open position with verified on-chain report
     * Report comes from Chainlink Data Streams API
     */
    function openPosition(
        bytes calldata reportData,
        bool isLong,
        uint256 leverage
    ) external payable {
        // Verify and decode report
        bytes memory verifiedReport = verifier.verify{value: msg.value}(reportData, abi.encode(ETH_USD_FEED_ID));
        
        Report memory report = abi.decode(verifiedReport, (Report));
        
        // Validate report freshness
        require(block.timestamp - report.observationsTimestamp < 60, "Stale report");
        require(report.expiresAt >= block.timestamp, "Report expired");
        
        uint256 price = uint256(uint192(report.price));
        
        // Open position using verified price
        positions[msg.sender] = Position({
            size: isLong ? int256(leverage) : -int256(leverage),
            entryPrice: price,
            margin: msg.value,
            timestamp: block.timestamp
        });
    }
}
```

---

## 2. Chainlink Functions

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@chainlink/contracts/src/v0.8/functions/v1_0_0/FunctionsClient.sol";
import "@chainlink/contracts/src/v0.8/functions/v1_0_0/libraries/FunctionsRequest.sol";

/**
 * Chainlink Functions:
 * Run serverless JavaScript ใน Chainlink DON (Decentralized Oracle Network)
 * - Fetch data from any API
 * - Compute off-chain
 * - Return result on-chain
 * 
 * Use cases:
 * - Custom API integrations
 * - Complex calculations
 * - Privacy-preserving attestations
 */
contract ChainlinkFunctionsExample is FunctionsClient {
    
    using FunctionsRequest for FunctionsRequest.Request;
    
    bytes32 public latestRequestId;
    bytes public latestResponse;
    bytes public latestError;
    
    uint64 public subscriptionId;
    uint32 public gasLimit = 300000;
    bytes32 public donId;
    
    // Custom result
    uint256 public latestPrice;
    
    event RequestSent(bytes32 indexed requestId, uint8 numWords);
    event ResponseReceived(bytes32 indexed requestId, uint256 price, bytes err);
    
    constructor(
        address router,
        bytes32 _donId,
        uint64 _subscriptionId
    ) FunctionsClient(router) {
        donId = _donId;
        subscriptionId = _subscriptionId;
    }
    
    /**
     * Fetch custom price from an API that Chainlink Price Feeds doesn't support
     */
    function requestCustomPrice(string calldata symbol) external returns (bytes32 requestId) {
        FunctionsRequest.Request memory req;
        req.initializeRequestForInlineJavaScript(
            // JavaScript to run in DON
            string(abi.encodePacked(
                "const response = await Functions.makeHttpRequest({",
                "url: `https://api.coingecko.com/api/v3/simple/price?ids=",
                symbol,
                "&vs_currencies=usd`",
                "});",
                "if (response.error) { throw Error(response.error); }",
                "const price = response.data['",
                symbol,
                "']['usd'];",
                "return Functions.encodeUint256(Math.round(price * 1e8));"
            ))
        );
        
        // No arguments needed for inline JS
        requestId = _sendRequest(
            req.encodeCBOR(),
            subscriptionId,
            gasLimit,
            donId
        );
        
        latestRequestId = requestId;
        emit RequestSent(requestId, 0);
    }
    
    /**
     * Callback function
     * Called by the DON when the request is fulfilled
     */
    function fulfillRequest(
        bytes32 requestId,
        bytes memory response,
        bytes memory err
    ) internal override {
        latestResponse = response;
        latestError = err;
        
        if (err.length == 0 && response.length > 0) {
            latestPrice = abi.decode(response, (uint256));
        }
        
        emit ResponseReceived(requestId, latestPrice, err);
    }
}
```

---

## 3. CCIP Advanced

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IRouterClient} from "@chainlink/contracts-ccip/src/v0.8/ccip/interfaces/IRouterClient.sol";
import {Client} from "@chainlink/contracts-ccip/src/v0.8/ccip/libraries/Client.sol";
import {CCIPReceiver} from "@chainlink/contracts-ccip/src/v0.8/ccip/applications/CCIPReceiver.sol";

/**
 * Cross-Chain DeFi with CCIP:
 * - Deposit on Chain A
 * - Yield strategy executes on Chain B  
 * - Withdraw back to Chain A
 */
contract CrossChainVault is CCIPReceiver {
    
    IRouterClient public immutable router;
    address public immutable linkToken;
    
    uint64 public constant ARBITRUM_CHAIN_SELECTOR = 4949039107694359620;
    uint64 public constant OPTIMISM_CHAIN_SELECTOR = 2664363617261496610;
    
    // Track cross-chain deposits
    mapping(bytes32 => bool) public processedMessages;
    mapping(address => uint256) public deposits;
    
    enum MessageType { DEPOSIT, WITHDRAW, YIELD }
    
    struct CrossChainMessage {
        MessageType messageType;
        address user;
        uint256 amount;
        bytes extraData;
    }
    
    event CrossChainDeposit(address indexed user, uint256 amount, uint64 destinationChain);
    event CrossChainWithdraw(address indexed user, uint256 amount);
    
    constructor(address _router, address _link) CCIPReceiver(_router) {
        router = IRouterClient(_router);
        linkToken = _link;
    }
    
    /**
     * Send tokens to another chain + message
     */
    function depositCrossChain(
        uint64 destinationChain,
        address destinationVault,
        address token,
        uint256 amount
    ) external returns (bytes32 messageId) {
        
        CrossChainMessage memory message = CrossChainMessage({
            messageType: MessageType.DEPOSIT,
            user: msg.sender,
            amount: amount,
            extraData: ""
        });
        
        // Prepare CCIP message
        Client.EVM2AnyMessage memory ccipMessage = Client.EVM2AnyMessage({
            receiver: abi.encode(destinationVault),
            data: abi.encode(message),
            tokenAmounts: new Client.EVMTokenAmount[](1),
            extraArgs: Client._argsToBytes(
                Client.EVMExtraArgsV1({gasLimit: 200_000})
            ),
            feeToken: linkToken
        });
        
        ccipMessage.tokenAmounts[0] = Client.EVMTokenAmount({
            token: token,
            amount: amount
        });
        
        // Calculate and pay fees
        uint256 fees = router.getFee(destinationChain, ccipMessage);
        
        IERC20(linkToken).transferFrom(msg.sender, address(this), fees);
        IERC20(linkToken).approve(address(router), fees);
        
        IERC20(token).transferFrom(msg.sender, address(this), amount);
        IERC20(token).approve(address(router), amount);
        
        // Send
        messageId = router.ccipSend(destinationChain, ccipMessage);
        
        emit CrossChainDeposit(msg.sender, amount, destinationChain);
    }
    
    /**
     * Receive cross-chain message
     */
    function _ccipReceive(Client.Any2EVMMessage memory message) internal override {
        bytes32 messageId = message.messageId;
        
        // Prevent replay
        require(!processedMessages[messageId], "Already processed");
        processedMessages[messageId] = true;
        
        CrossChainMessage memory decoded = abi.decode(message.data, (CrossChainMessage));
        
        if (decoded.messageType == MessageType.DEPOSIT) {
            deposits[decoded.user] += decoded.amount;
        } else if (decoded.messageType == MessageType.WITHDRAW) {
            // Process withdrawal on source chain
            require(deposits[decoded.user] >= decoded.amount, "Insufficient");
            deposits[decoded.user] -= decoded.amount;
        }
    }
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
    function approve(address, uint256) external returns (bool);
}
```

---

## 4. Workshop: Multi-Oracle Price Aggregator

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Multi-Oracle Aggregator:
 * รวม Chainlink + Uniswap TWAP + Pyth
 * ลด single point of failure
 */
contract MultiOracleAggregator {
    
    struct OracleSource {
        address oracle;
        OracleType oracleType;
        uint256 weight;       // weight in aggregation
        uint256 maxStaleness; // max allowed age
        bool active;
    }
    
    enum OracleType { CHAINLINK, UNISWAP_TWAP, PYTH }
    
    OracleSource[] public sources;
    uint256 public constant MAX_DEVIATION = 500; // 5% max deviation between oracles
    uint256 public constant BASIS = 10000;
    
    event PriceAggregated(uint256 price, uint256 numSources);
    event OracleDeviation(uint256 price1, uint256 price2, uint256 deviation);
    
    function addOracle(
        address oracle,
        OracleType oracleType,
        uint256 weight,
        uint256 maxStaleness
    ) external {
        sources.push(OracleSource({
            oracle: oracle,
            oracleType: oracleType,
            weight: weight,
            maxStaleness: maxStaleness,
            active: true
        }));
    }
    
    function getAggregatedPrice() external returns (uint256 price) {
        uint256 weightedSum;
        uint256 totalWeight;
        uint256 validCount;
        
        uint256[] memory prices = new uint256[](sources.length);
        
        for (uint256 i; i < sources.length; i++) {
            OracleSource storage source = sources[i];
            if (!source.active) continue;
            
            try this._getPrice(source) returns (uint256 p) {
                prices[validCount] = p;
                weightedSum += p * source.weight;
                totalWeight += source.weight;
                validCount++;
            } catch {
                // Oracle failed, skip it
            }
        }
        
        require(validCount >= 2, "Insufficient oracles");
        
        price = weightedSum / totalWeight;
        
        // Check for anomalous prices
        for (uint256 i; i < validCount; i++) {
            if (prices[i] == 0) continue;
            uint256 deviation = prices[i] > price 
                ? (prices[i] - price) * BASIS / price
                : (price - prices[i]) * BASIS / price;
            
            if (deviation > MAX_DEVIATION) {
                emit OracleDeviation(prices[i], price, deviation);
                // Could disable the outlier oracle
            }
        }
        
        emit PriceAggregated(price, validCount);
    }
    
    function _getPrice(OracleSource memory source) external view returns (uint256) {
        if (source.oracleType == OracleType.CHAINLINK) {
            (,int256 answer,,uint256 updatedAt,) = AggregatorV3Interface(source.oracle).latestRoundData();
            require(block.timestamp - updatedAt <= source.maxStaleness, "Stale");
            return uint256(answer);
        } else if (source.oracleType == OracleType.UNISWAP_TWAP) {
            return ITWAPOracle(source.oracle).getTWAP(1800); // 30 min TWAP
        }
        revert("Unknown oracle type");
    }
}

interface AggregatorV3Interface {
    function latestRoundData() external view returns (
        uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound
    );
}

interface ITWAPOracle {
    function getTWAP(uint256 period) external view returns (uint256);
}
```

---

## สรุป Part 46

Chainlink Advanced ที่เรียนรู้:
- ✅ Data Streams (low latency, pull-based)
- ✅ Chainlink Functions (serverless JS)
- ✅ CCIP advanced patterns
- ✅ Multi-oracle aggregator
- ✅ Oracle security (staleness, deviation)

## Next: Part 47 - Layer 2 Advanced (zkSync, Starknet)
