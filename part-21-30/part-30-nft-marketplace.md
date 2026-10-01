# Part 30: NFT Marketplace

## สารบัญ
1. Marketplace Architecture
2. Listing และ Order Book
3. Auction System
4. Royalty Distribution
5. Workshop: Full Marketplace

---

## 1. Marketplace Architecture

```
NFT Marketplace Components:
1. Listing: เจ้าของวาง NFT ขาย + กำหนดราคา
2. Offer: ผู้ซื้อเสนอราคา
3. Auction: English/Dutch auction
4. Royalties: creator ได้ % จากทุก secondary sale
5. Fee: platform ได้ % จาก transaction

Standards:
- ERC-2981: royalty standard (on-chain)
- EIP-712: typed structured signatures
- Seaport: OpenSea's marketplace protocol

Security:
- Signature-based orders (off-chain order book)
- Cancel listing via nonce invalidation
- Expiry timestamps
- Re-entrancy protection
```

---

## 2. Fixed-Price Marketplace

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Simple NFT Marketplace
 * - Direct listings (fixed price)
 * - ETH payments
 * - ERC-2981 royalty support
 * - Platform fee
 */
contract NFTMarketplace {
    
    struct Listing {
        address seller;
        uint256 price;
        bool active;
    }
    
    // nftAddress → tokenId → Listing
    mapping(address => mapping(uint256 => Listing)) public listings;
    
    uint256 public platformFee = 250; // 2.5%
    uint256 public constant FEE_DENOMINATOR = 10000;
    address public feeRecipient;
    
    event Listed(address indexed nft, uint256 indexed tokenId, address indexed seller, uint256 price);
    event Sold(address indexed nft, uint256 indexed tokenId, address indexed buyer, uint256 price);
    event Delisted(address indexed nft, uint256 indexed tokenId);
    event OfferMade(address indexed nft, uint256 indexed tokenId, address indexed buyer, uint256 amount);
    
    error NotOwner();
    error NotListed();
    error InsufficientPayment(uint256 required, uint256 sent);
    error AlreadyListed();
    
    constructor(address _feeRecipient) {
        feeRecipient = _feeRecipient;
    }
    
    function list(address nft, uint256 tokenId, uint256 price) external {
        IERC721 nftContract = IERC721(nft);
        
        if (nftContract.ownerOf(tokenId) != msg.sender) revert NotOwner();
        if (listings[nft][tokenId].active) revert AlreadyListed();
        require(
            nftContract.isApprovedForAll(msg.sender, address(this)) ||
            nftContract.getApproved(tokenId) == address(this),
            "Not approved"
        );
        require(price > 0, "Price must be > 0");
        
        listings[nft][tokenId] = Listing({
            seller: msg.sender,
            price: price,
            active: true
        });
        
        emit Listed(nft, tokenId, msg.sender, price);
    }
    
    function buy(address nft, uint256 tokenId) external payable {
        Listing storage listing = listings[nft][tokenId];
        
        if (!listing.active) revert NotListed();
        if (msg.value < listing.price) revert InsufficientPayment(listing.price, msg.value);
        
        address seller = listing.seller;
        uint256 price = listing.price;
        
        listing.active = false;
        
        // Calculate fees
        uint256 platformFeeAmount = (price * platformFee) / FEE_DENOMINATOR;
        
        // Check ERC-2981 royalty
        uint256 royaltyAmount = 0;
        address royaltyRecipient = address(0);
        
        try IERC2981(nft).royaltyInfo(tokenId, price) returns (
            address recipient,
            uint256 royalty
        ) {
            if (royalty > 0 && recipient != address(0)) {
                royaltyRecipient = recipient;
                royaltyAmount = royalty;
            }
        } catch {}
        
        uint256 sellerAmount = price - platformFeeAmount - royaltyAmount;
        
        // Transfer NFT first (CEI)
        IERC721(nft).safeTransferFrom(seller, msg.sender, tokenId);
        
        // Distribute payments
        payable(feeRecipient).transfer(platformFeeAmount);
        
        if (royaltyAmount > 0) {
            payable(royaltyRecipient).transfer(royaltyAmount);
        }
        
        payable(seller).transfer(sellerAmount);
        
        // Refund excess
        if (msg.value > price) {
            payable(msg.sender).transfer(msg.value - price);
        }
        
        emit Sold(nft, tokenId, msg.sender, price);
    }
    
    function delist(address nft, uint256 tokenId) external {
        Listing storage listing = listings[nft][tokenId];
        
        if (!listing.active) revert NotListed();
        if (listing.seller != msg.sender) revert NotOwner();
        
        listing.active = false;
        
        emit Delisted(nft, tokenId);
    }
    
    function updatePrice(address nft, uint256 tokenId, uint256 newPrice) external {
        Listing storage listing = listings[nft][tokenId];
        
        if (!listing.active) revert NotListed();
        if (listing.seller != msg.sender) revert NotOwner();
        require(newPrice > 0, "Price must be > 0");
        
        listing.price = newPrice;
        
        emit Listed(nft, tokenId, msg.sender, newPrice);
    }
}

interface IERC721 {
    function ownerOf(uint256) external view returns (address);
    function safeTransferFrom(address, address, uint256) external;
    function isApprovedForAll(address, address) external view returns (bool);
    function getApproved(uint256) external view returns (address);
}

interface IERC2981 {
    function royaltyInfo(uint256 tokenId, uint256 salePrice)
        external view returns (address receiver, uint256 royaltyAmount);
}
```

---

## 3. English Auction

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * English Auction (ascending price)
 * - ผู้ขายกำหนด starting price และ duration
 * - ผู้ซื้อ bid สูงขึ้นเรื่อยๆ
 * - เมื่อหมดเวลา highest bidder ชนะ
 */
contract EnglishAuction {
    
    struct Auction {
        address seller;
        IERC721 nft;
        uint256 tokenId;
        uint256 startingPrice;
        uint256 reservePrice;
        uint256 highestBid;
        address highestBidder;
        uint256 endTime;
        bool ended;
    }
    
    uint256 public auctionCount;
    mapping(uint256 => Auction) public auctions;
    
    // Pending returns for outbid users
    mapping(uint256 => mapping(address => uint256)) public pendingReturns;
    
    uint256 public constant MIN_BID_INCREMENT = 500; // 5%
    uint256 public constant FEE = 250; // 2.5%
    address public feeRecipient;
    
    event AuctionCreated(uint256 indexed auctionId, address indexed nft, uint256 tokenId, uint256 endTime);
    event BidPlaced(uint256 indexed auctionId, address indexed bidder, uint256 amount);
    event AuctionEnded(uint256 indexed auctionId, address indexed winner, uint256 amount);
    event BidWithdrawn(address indexed bidder, uint256 amount);
    
    error AuctionNotActive();
    error BidTooLow(uint256 minimum, uint256 sent);
    error AuctionStillActive();
    error NotSeller();
    
    constructor(address _feeRecipient) {
        feeRecipient = _feeRecipient;
    }
    
    function createAuction(
        address nft,
        uint256 tokenId,
        uint256 startingPrice,
        uint256 reservePrice,
        uint256 duration
    ) external returns (uint256 auctionId) {
        require(duration >= 1 hours && duration <= 30 days, "Invalid duration");
        
        IERC721 nftContract = IERC721(nft);
        require(nftContract.ownerOf(tokenId) == msg.sender, "Not owner");
        
        nftContract.transferFrom(msg.sender, address(this), tokenId);
        
        auctionId = auctionCount++;
        
        auctions[auctionId] = Auction({
            seller: msg.sender,
            nft: nftContract,
            tokenId: tokenId,
            startingPrice: startingPrice,
            reservePrice: reservePrice,
            highestBid: 0,
            highestBidder: address(0),
            endTime: block.timestamp + duration,
            ended: false
        });
        
        emit AuctionCreated(auctionId, nft, tokenId, block.timestamp + duration);
    }
    
    function bid(uint256 auctionId) external payable {
        Auction storage auction = auctions[auctionId];
        
        if (block.timestamp >= auction.endTime || auction.ended) revert AuctionNotActive();
        
        uint256 minBid = auction.highestBid == 0
            ? auction.startingPrice
            : auction.highestBid + (auction.highestBid * MIN_BID_INCREMENT / 10000);
        
        if (msg.value < minBid) revert BidTooLow(minBid, msg.value);
        
        // Return previous highest bid
        if (auction.highestBidder != address(0)) {
            pendingReturns[auctionId][auction.highestBidder] += auction.highestBid;
        }
        
        auction.highestBid = msg.value;
        auction.highestBidder = msg.sender;
        
        // Extend auction if bid near end (anti-sniping)
        if (auction.endTime - block.timestamp < 15 minutes) {
            auction.endTime += 15 minutes;
        }
        
        emit BidPlaced(auctionId, msg.sender, msg.value);
    }
    
    // Withdraw previous losing bids
    function withdrawBid(uint256 auctionId) external {
        uint256 amount = pendingReturns[auctionId][msg.sender];
        require(amount > 0, "Nothing to withdraw");
        
        pendingReturns[auctionId][msg.sender] = 0;
        payable(msg.sender).transfer(amount);
        
        emit BidWithdrawn(msg.sender, amount);
    }
    
    function endAuction(uint256 auctionId) external {
        Auction storage auction = auctions[auctionId];
        
        if (block.timestamp < auction.endTime) revert AuctionStillActive();
        if (auction.ended) revert AuctionNotActive();
        
        auction.ended = true;
        
        if (auction.highestBidder != address(0) && 
            auction.highestBid >= auction.reservePrice) {
            // Reserve met: transfer NFT and payment
            uint256 fee = (auction.highestBid * FEE) / 10000;
            uint256 sellerAmount = auction.highestBid - fee;
            
            auction.nft.safeTransferFrom(address(this), auction.highestBidder, auction.tokenId);
            
            payable(feeRecipient).transfer(fee);
            payable(auction.seller).transfer(sellerAmount);
            
            emit AuctionEnded(auctionId, auction.highestBidder, auction.highestBid);
        } else {
            // Reserve not met: return NFT to seller
            if (auction.highestBidder != address(0)) {
                pendingReturns[auctionId][auction.highestBidder] += auction.highestBid;
            }
            auction.nft.safeTransferFrom(address(this), auction.seller, auction.tokenId);
            
            emit AuctionEnded(auctionId, address(0), 0);
        }
    }
}

interface IERC721 {
    function ownerOf(uint256) external view returns (address);
    function transferFrom(address, address, uint256) external;
    function safeTransferFrom(address, address, uint256) external;
}
```

---

## 4. Off-Chain Order Book (Seaport-style)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Signature-based Orders
 * 
 * Order สร้างนอก chain (gasless)
 * เมื่อ fulfill → ตรวจ signature แล้วดำเนิน
 * 
 * ข้อดี:
 * - ไม่ต้องเสีย gas ในการ list
 * - Cancel ผ่าน nonce invalidation
 * - รองรับ batch orders
 */
contract SignatureMarketplace {
    
    struct Order {
        address offerer;
        address nft;
        uint256 tokenId;
        uint256 price;
        uint256 expiry;
        uint256 nonce;
    }
    
    mapping(address => uint256) public nonces;
    mapping(bytes32 => bool) public cancelledOrders;
    
    // EIP-712
    bytes32 public constant DOMAIN_TYPEHASH = keccak256(
        "EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"
    );
    
    bytes32 public constant ORDER_TYPEHASH = keccak256(
        "Order(address offerer,address nft,uint256 tokenId,uint256 price,uint256 expiry,uint256 nonce)"
    );
    
    bytes32 public immutable DOMAIN_SEPARATOR;
    
    event OrderFulfilled(bytes32 indexed orderHash, address indexed buyer, uint256 price);
    event OrderCancelled(bytes32 indexed orderHash);
    
    error OrderExpired();
    error InvalidSignature();
    error OrderAlreadyCancelled();
    error IncorrectPayment(uint256 required, uint256 sent);
    
    constructor() {
        DOMAIN_SEPARATOR = keccak256(abi.encode(
            DOMAIN_TYPEHASH,
            keccak256("NFT Marketplace"),
            keccak256("1"),
            block.chainid,
            address(this)
        ));
    }
    
    function fulfillOrder(
        Order calldata order,
        bytes calldata signature
    ) external payable {
        if (block.timestamp > order.expiry) revert OrderExpired();
        
        bytes32 orderHash = _hashOrder(order);
        
        if (cancelledOrders[orderHash]) revert OrderAlreadyCancelled();
        if (msg.value < order.price) revert IncorrectPayment(order.price, msg.value);
        
        // Verify signature
        address signer = _recoverSigner(orderHash, signature);
        if (signer != order.offerer) revert InvalidSignature();
        
        // Mark order as used (via nonce: safer is increment nonce)
        cancelledOrders[orderHash] = true;
        
        // Transfer NFT
        IERC721(order.nft).safeTransferFrom(order.offerer, msg.sender, order.tokenId);
        
        // Pay seller
        payable(order.offerer).transfer(order.price);
        
        // Refund excess
        if (msg.value > order.price) {
            payable(msg.sender).transfer(msg.value - order.price);
        }
        
        emit OrderFulfilled(orderHash, msg.sender, order.price);
    }
    
    function cancelOrder(Order calldata order) external {
        require(msg.sender == order.offerer, "Not offerer");
        
        bytes32 orderHash = _hashOrder(order);
        cancelledOrders[orderHash] = true;
        
        emit OrderCancelled(orderHash);
    }
    
    // Cancel ALL outstanding orders by incrementing nonce
    function cancelAllOrders() external {
        nonces[msg.sender]++;
    }
    
    function _hashOrder(Order calldata order) internal view returns (bytes32) {
        return keccak256(abi.encodePacked(
            "\x19\x01",
            DOMAIN_SEPARATOR,
            keccak256(abi.encode(
                ORDER_TYPEHASH,
                order.offerer,
                order.nft,
                order.tokenId,
                order.price,
                order.expiry,
                order.nonce
            ))
        ));
    }
    
    function _recoverSigner(bytes32 hash, bytes calldata sig) internal pure returns (address) {
        require(sig.length == 65, "Invalid sig");
        bytes32 r;
        bytes32 s;
        uint8 v;
        assembly {
            r := calldataload(sig.offset)
            s := calldataload(add(sig.offset, 32))
            v := byte(0, calldataload(add(sig.offset, 64)))
        }
        return ecrecover(hash, v, r, s);
    }
    
    function getOrderHash(Order calldata order) external view returns (bytes32) {
        return _hashOrder(order);
    }
}

interface IERC721 {
    function safeTransferFrom(address, address, uint256) external;
}
```

---

## สรุป Part 30

NFT Marketplace ที่เรียนรู้:
- ✅ Fixed-price listings
- ✅ ERC-2981 royalty integration
- ✅ English Auction with anti-sniping
- ✅ Off-chain order book (EIP-712 signatures)
- ✅ Nonce-based cancellation

## Quiz

1. Off-chain order book มีข้อดีเหนือ on-chain listing อะไร?
2. Anti-sniping mechanism ทำงานอย่างไร?
3. Reserve price ใน auction คืออะไร?
4. ทำไม EIP-712 ถึงดีกว่า raw signature?

---

## Next: Part 31 - Perpetuals และ Derivatives
