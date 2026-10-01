# Part 58: Advanced NFT Patterns

## บทนำ

NFT (Non-Fungible Token) ได้พัฒนาจากแค่ digital art ไปสู่ infrastructure ที่ซับซ้อนกว่ามาก บทนี้ครอบคลุม:

1. **Dynamic NFT** - metadata เปลี่ยนตาม on-chain state พร้อม base64 SVG
2. **NFT Staking** - stake NFT เพื่อรับ ERC-20 rewards
3. **Fractional Ownership** - แบ่ง NFT เป็น ERC-20 tokens พร้อม buyout
4. **Soulbound Token (SBT)** - EIP-5192 non-transferable identity token
5. **Royalty Standard EIP-2981** - automatic royalty payment

---

## 1. Dynamic NFT (dNFT)

### ทฤษฎี Dynamic NFT

Dynamic NFT คือ NFT ที่ metadata (image, attributes) เปลี่ยนแปลงได้ตาม:
- **On-chain state**: ระดับ, ประสบการณ์, สุขภาพ (ใน games)
- **Time**: ฤดูกาล, วันเวลา
- **External data**: ราคา token, ผลการแข่งขัน (ผ่าน oracle)

**Base64 Encoding**: tokenURI return base64-encoded JSON โดยตรงแทน IPFS URL ทำให้:
- Fully on-chain (ไม่พึ่ง external storage)
- ไม่มี single point of failure
- Metadata เปลี่ยนได้ทันทีตาม state

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Enumerable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Base64.sol";
import "@openzeppelin/contracts/utils/Strings.sol";

/**
 * @title DynamicNFT
 * @notice NFT ที่ metadata เปลี่ยนตาม on-chain state
 * @dev ใช้ Base64-encoded SVG ที่ generate on-chain
 */
contract DynamicNFT is ERC721Enumerable, Ownable {
    using Strings for uint256;

    // ============ Enums ============
    enum Level { BRONZE, SILVER, GOLD, PLATINUM, DIAMOND }

    // ============ Structs ============
    struct CharacterStats {
        string name;
        uint256 experience;
        uint256 strength;
        uint256 defense;
        uint256 speed;
        uint256 lastBattleTime;
        Level level;
    }

    // ============ State ============
    mapping(uint256 => CharacterStats) public characters;
    uint256 private _tokenIdCounter;

    // XP thresholds ต่อ level
    uint256[5] public levelThresholds = [0, 100, 500, 2000, 10000];

    // Color themes ต่อ level
    string[5] public levelColors = ["#CD7F32", "#C0C0C0", "#FFD700", "#E5E4E2", "#00D4FF"];
    string[5] public levelNames = ["Bronze", "Silver", "Gold", "Platinum", "Diamond"];

    // ============ Events ============
    event ExperienceGained(uint256 indexed tokenId, uint256 amount, uint256 totalXP);
    event LevelUp(uint256 indexed tokenId, Level newLevel);
    event BattleResult(uint256 indexed tokenId, bool won, uint256 xpGained);

    // ============ Constructor ============
    constructor() ERC721("Dynamic Character NFT", "DCNFT") Ownable(msg.sender) {}

    // ============ Minting ============

    /**
     * @notice Mint NFT ใหม่พร้อม character stats เริ่มต้น
     * @param to ผู้รับ NFT
     * @param characterName ชื่อ character
     */
    function mint(address to, string memory characterName) external onlyOwner returns (uint256) {
        uint256 tokenId = ++_tokenIdCounter;

        characters[tokenId] = CharacterStats({
            name: characterName,
            experience: 0,
            strength: 10 + (uint256(keccak256(abi.encodePacked(tokenId, "str"))) % 10),
            defense: 5 + (uint256(keccak256(abi.encodePacked(tokenId, "def"))) % 10),
            speed: 8 + (uint256(keccak256(abi.encodePacked(tokenId, "spd"))) % 8),
            lastBattleTime: block.timestamp,
            level: Level.BRONZE
        });

        _safeMint(to, tokenId);
        return tokenId;
    }

    // ============ Game Mechanics ============

    /**
     * @notice เพิ่ม XP ให้ character
     * @param tokenId Token ID
     * @param xpAmount จำนวน XP ที่ได้รับ
     */
    function gainExperience(uint256 tokenId, uint256 xpAmount) external {
        require(ownerOf(tokenId) == msg.sender, "DynamicNFT: not owner");

        CharacterStats storage stats = characters[tokenId];
        stats.experience += xpAmount;

        // ตรวจสอบ level up
        _checkLevelUp(tokenId);

        emit ExperienceGained(tokenId, xpAmount, stats.experience);
    }

    /**
     * @notice Battle ระหว่าง characters
     * @param attackerTokenId Token ID ของผู้โจมตี
     * @param defenderTokenId Token ID ของผู้ป้องกัน (ไม่จำเป็นต้องเป็น owner)
     */
    function battle(uint256 attackerTokenId, uint256 defenderTokenId) external {
        require(ownerOf(attackerTokenId) == msg.sender, "DynamicNFT: not attacker owner");
        require(
            block.timestamp >= characters[attackerTokenId].lastBattleTime + 1 hours,
            "DynamicNFT: battle cooldown"
        );

        CharacterStats storage attacker = characters[attackerTokenId];
        CharacterStats storage defender = characters[defenderTokenId];

        // คำนวณ battle score
        uint256 attackScore = attacker.strength * 2 + attacker.speed;
        uint256 defendScore = defender.defense * 2 + defender.speed;

        // เพิ่ม randomness จาก block
        uint256 rand = uint256(keccak256(abi.encodePacked(
            block.timestamp, block.prevrandao, attackerTokenId, defenderTokenId
        ))) % 100;

        bool won = (attackScore + rand % 20) > (defendScore + rand % 10);

        uint256 xpGained = won ? 50 + rand % 50 : 10 + rand % 20;
        attacker.experience += xpGained;
        attacker.lastBattleTime = block.timestamp;

        // Stats เติบโตตาม battle
        if (won && attacker.strength < 100) {
            attacker.strength += 1;
        }
        if (!won && attacker.defense < 100) {
            attacker.defense += 1;
        }

        _checkLevelUp(attackerTokenId);

        emit BattleResult(attackerTokenId, won, xpGained);
    }

    function _checkLevelUp(uint256 tokenId) internal {
        CharacterStats storage stats = characters[tokenId];
        Level currentLevel = stats.level;
        Level newLevel = currentLevel;

        for (uint256 i = 4; i > uint256(currentLevel); i--) {
            if (stats.experience >= levelThresholds[i]) {
                newLevel = Level(i);
                break;
            }
        }

        if (newLevel != currentLevel) {
            stats.level = newLevel;
            // Bonus stats เมื่อ level up
            stats.strength += 5 * (uint256(newLevel) - uint256(currentLevel));
            stats.defense += 3 * (uint256(newLevel) - uint256(currentLevel));
            emit LevelUp(tokenId, newLevel);
        }
    }

    // ============ Dynamic tokenURI ============

    /**
     * @notice สร้าง metadata JSON พร้อม SVG image แบบ on-chain
     * @dev Return base64-encoded JSON data URI
     */
    function tokenURI(uint256 tokenId) public view override returns (string memory) {
        require(_ownerOf(tokenId) != address(0), "DynamicNFT: nonexistent token");

        CharacterStats memory stats = characters[tokenId];
        string memory svg = _generateSVG(tokenId, stats);
        string memory json = _generateJSON(tokenId, stats, svg);

        return string(abi.encodePacked(
            "data:application/json;base64,",
            Base64.encode(bytes(json))
        ));
    }

    /**
     * @notice สร้าง SVG image ที่แสดง character stats
     */
    function _generateSVG(
        uint256 tokenId,
        CharacterStats memory stats
    ) internal view returns (string memory) {
        string memory color = levelColors[uint256(stats.level)];
        string memory levelName = levelNames[uint256(stats.level)];

        // คำนวณ XP bar width (0-200 pixels)
        uint256 nextLevelXP = uint256(stats.level) < 4
            ? levelThresholds[uint256(stats.level) + 1]
            : levelThresholds[4];
        uint256 currentLevelXP = levelThresholds[uint256(stats.level)];
        uint256 xpProgress = nextLevelXP > currentLevelXP
            ? (stats.experience - currentLevelXP) * 200 / (nextLevelXP - currentLevelXP)
            : 200;
        if (xpProgress > 200) xpProgress = 200;

        return string(abi.encodePacked(
            '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 300 400" width="300" height="400">',
            '<defs>',
            '<linearGradient id="bg" x1="0%" y1="0%" x2="100%" y2="100%">',
            '<stop offset="0%" style="stop-color:#1a1a2e;stop-opacity:1" />',
            '<stop offset="100%" style="stop-color:#16213e;stop-opacity:1" />',
            '</linearGradient>',
            '</defs>',
            '<rect width="300" height="400" fill="url(#bg)" rx="15"/>',
            // Border
            '<rect x="5" y="5" width="290" height="390" fill="none" stroke="', color, '" stroke-width="2" rx="12"/>',
            // Character circle
            '<circle cx="150" cy="120" r="60" fill="none" stroke="', color, '" stroke-width="3"/>',
            '<text x="150" y="130" text-anchor="middle" font-size="50" fill="', color, '">&#9876;</text>',
            // Level badge
            '<rect x="110" y="170" width="80" height="25" rx="12" fill="', color, '"/>',
            '<text x="150" y="187" text-anchor="middle" font-size="12" fill="#000" font-weight="bold">', levelName, '</text>',
            // Name
            '<text x="150" y="220" text-anchor="middle" font-size="16" fill="white" font-weight="bold">', stats.name, '</text>',
            // Stats
            _generateStatBars(stats, color),
            // XP Bar
            '<text x="20" y="355" font-size="11" fill="#888">XP Progress</text>',
            '<rect x="20" y="360" width="260" height="8" rx="4" fill="#333"/>',
            '<rect x="20" y="360" width="', xpProgress.toString(), '" height="8" rx="4" fill="', color, '"/>',
            // Token ID
            '<text x="150" y="390" text-anchor="middle" font-size="10" fill="#555">#', tokenId.toString(), '</text>',
            '</svg>'
        ));
    }

    function _generateStatBars(
        CharacterStats memory stats,
        string memory color
    ) internal pure returns (string memory) {
        return string(abi.encodePacked(
            // STR
            '<text x="20" y="245" font-size="11" fill="#aaa">STR</text>',
            '<rect x="60" y="233" width="220" height="10" rx="5" fill="#333"/>',
            '<rect x="60" y="233" width="', _statWidth(stats.strength).toString(), '" height="10" rx="5" fill="', color, '"/>',
            '<text x="285" y="243" text-anchor="end" font-size="11" fill="white">', stats.strength.toString(), '</text>',
            // DEF
            '<text x="20" y="265" font-size="11" fill="#aaa">DEF</text>',
            '<rect x="60" y="253" width="220" height="10" rx="5" fill="#333"/>',
            '<rect x="60" y="253" width="', _statWidth(stats.defense).toString(), '" height="10" rx="5" fill="', color, '"/>',
            '<text x="285" y="263" text-anchor="end" font-size="11" fill="white">', stats.defense.toString(), '</text>',
            // SPD
            '<text x="20" y="285" font-size="11" fill="#aaa">SPD</text>',
            '<rect x="60" y="273" width="220" height="10" rx="5" fill="#333"/>',
            '<rect x="60" y="273" width="', _statWidth(stats.speed).toString(), '" height="10" rx="5" fill="', color, '"/>',
            '<text x="285" y="283" text-anchor="end" font-size="11" fill="white">', stats.speed.toString(), '</text>'
        ));
    }

    function _statWidth(uint256 statValue) internal pure returns (uint256) {
        return statValue > 100 ? 220 : statValue * 220 / 100;
    }

    function _generateJSON(
        uint256 tokenId,
        CharacterStats memory stats,
        string memory svg
    ) internal view returns (string memory) {
        return string(abi.encodePacked(
            '{"name":"', stats.name, ' #', tokenId.toString(), '",',
            '"description":"A dynamic on-chain character NFT that evolves through battles.",',
            '"image":"data:image/svg+xml;base64,', Base64.encode(bytes(svg)), '",',
            '"attributes":[',
            '{"trait_type":"Level","value":"', levelNames[uint256(stats.level)], '"},',
            '{"trait_type":"Experience","value":', stats.experience.toString(), '},',
            '{"trait_type":"Strength","value":', stats.strength.toString(), '},',
            '{"trait_type":"Defense","value":', stats.defense.toString(), '},',
            '{"trait_type":"Speed","value":', stats.speed.toString(), '}',
            ']}'
        ));
    }
}
```

---

## 2. NFT Staking

### ทฤษฎี NFT Staking

NFT Staking ให้ผู้ถือ NFT ได้รับ ERC-20 rewards โดย:
- Lock NFT ไว้ใน staking contract
- Earn rewards ต่อวินาที/บล็อก
- Unstake เมื่อต้องการ (รับ rewards สะสม)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/token/ERC721/utils/ERC721Holder.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title NFTStaking
 * @notice Stake NFT เพื่อรับ ERC-20 rewards
 * @dev รองรับ multiple NFT collections และ reward multipliers
 */
contract NFTStaking is ERC721Holder, Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ============ Structs ============
    struct StakeInfo {
        address owner;          // ผู้ stake
        uint256 tokenId;        // NFT token ID
        uint256 startTime;      // เวลาที่ stake
        uint256 lastClaimTime;  // เวลา claim rewards ล่าสุด
        uint256 accumulatedRewards; // rewards ที่สะสมแต่ยังไม่ claim
    }

    struct CollectionConfig {
        uint256 rewardPerDay;   // rewards ต่อวัน (18 decimals)
        uint256 multiplier;     // multiplier เพิ่มเติม (basis points, 10000 = 1x)
        bool isActive;          // collection นี้ยังรับ stake หรือไม่
    }

    // ============ State ============
    IERC20 public immutable rewardToken;

    mapping(address => CollectionConfig) public collectionConfigs;
    mapping(address => mapping(uint256 => StakeInfo)) public stakes;
    mapping(address => uint256[]) public userStakedTokens; // user → [tokenIds]
    mapping(address => mapping(uint256 => uint256)) private _stakedTokenIndex; // tokenId → index

    uint256 public totalStaked;

    // ============ Events ============
    event Staked(
        address indexed user,
        address indexed collection,
        uint256 indexed tokenId,
        uint256 timestamp
    );
    event Unstaked(
        address indexed user,
        address indexed collection,
        uint256 indexed tokenId,
        uint256 rewardsClaimed
    );
    event RewardsClaimed(
        address indexed user,
        address indexed collection,
        uint256 indexed tokenId,
        uint256 amount
    );
    event CollectionConfigUpdated(
        address indexed collection,
        uint256 rewardPerDay,
        uint256 multiplier
    );

    // ============ Constructor ============
    constructor(address _rewardToken) Ownable(msg.sender) {
        rewardToken = IERC20(_rewardToken);
    }

    // ============ Admin Functions ============

    /**
     * @notice ตั้งค่า reward config สำหรับ NFT collection
     * @param collection Address ของ NFT contract
     * @param rewardPerDay จำนวน reward ต่อวัน (in wei)
     * @param multiplier Multiplier ใน basis points (10000 = 1x, 15000 = 1.5x)
     */
    function setCollectionConfig(
        address collection,
        uint256 rewardPerDay,
        uint256 multiplier,
        bool isActive
    ) external onlyOwner {
        require(collection != address(0), "NFTStaking: zero address");
        require(multiplier >= 10000, "NFTStaking: multiplier < 1x");

        collectionConfigs[collection] = CollectionConfig({
            rewardPerDay: rewardPerDay,
            multiplier: multiplier,
            isActive: isActive
        });

        emit CollectionConfigUpdated(collection, rewardPerDay, multiplier);
    }

    // ============ Core Functions ============

    /**
     * @notice Stake NFT
     * @param collection NFT collection address
     * @param tokenId Token ID ที่ต้องการ stake
     */
    function stake(
        address collection,
        uint256 tokenId
    ) external nonReentrant {
        CollectionConfig storage config = collectionConfigs[collection];
        require(config.isActive, "NFTStaking: collection not active");
        require(
            stakes[collection][tokenId].owner == address(0),
            "NFTStaking: already staked"
        );

        IERC721(collection).safeTransferFrom(msg.sender, address(this), tokenId);

        stakes[collection][tokenId] = StakeInfo({
            owner: msg.sender,
            tokenId: tokenId,
            startTime: block.timestamp,
            lastClaimTime: block.timestamp,
            accumulatedRewards: 0
        });

        // Track staked tokens ของ user
        _stakedTokenIndex[collection][tokenId] = userStakedTokens[msg.sender].length;
        userStakedTokens[msg.sender].push(tokenId);

        totalStaked++;
        emit Staked(msg.sender, collection, tokenId, block.timestamp);
    }

    /**
     * @notice Unstake NFT และรับ rewards ทั้งหมด
     * @param collection NFT collection address
     * @param tokenId Token ID ที่ต้องการ unstake
     */
    function unstake(
        address collection,
        uint256 tokenId
    ) external nonReentrant {
        StakeInfo storage stakeInfo = stakes[collection][tokenId];
        require(stakeInfo.owner == msg.sender, "NFTStaking: not owner");

        // คำนวณและ claim rewards ก่อน unstake
        uint256 pending = _calculatePendingRewards(collection, tokenId);
        uint256 totalRewards = stakeInfo.accumulatedRewards + pending;

        // ลบออกจาก user's staked list
        _removeStakedToken(msg.sender, collection, tokenId);

        delete stakes[collection][tokenId];
        totalStaked--;

        // คืน NFT
        IERC721(collection).safeTransferFrom(address(this), msg.sender, tokenId);

        // จ่าย rewards
        if (totalRewards > 0) {
            rewardToken.safeTransfer(msg.sender, totalRewards);
        }

        emit Unstaked(msg.sender, collection, tokenId, totalRewards);
    }

    /**
     * @notice Claim rewards โดยไม่ unstake
     * @param collection NFT collection address
     * @param tokenId Token ID
     */
    function claimRewards(
        address collection,
        uint256 tokenId
    ) external nonReentrant {
        StakeInfo storage stakeInfo = stakes[collection][tokenId];
        require(stakeInfo.owner == msg.sender, "NFTStaking: not owner");

        uint256 pending = _calculatePendingRewards(collection, tokenId);
        uint256 totalRewards = stakeInfo.accumulatedRewards + pending;

        require(totalRewards > 0, "NFTStaking: no rewards");

        stakeInfo.lastClaimTime = block.timestamp;
        stakeInfo.accumulatedRewards = 0;

        rewardToken.safeTransfer(msg.sender, totalRewards);

        emit RewardsClaimed(msg.sender, collection, tokenId, totalRewards);
    }

    /**
     * @notice Batch claim rewards สำหรับหลาย tokens
     */
    function batchClaimRewards(
        address[] calldata collections,
        uint256[] calldata tokenIds
    ) external nonReentrant {
        require(collections.length == tokenIds.length, "NFTStaking: length mismatch");

        uint256 totalRewards = 0;

        for (uint256 i = 0; i < collections.length; i++) {
            StakeInfo storage stakeInfo = stakes[collections[i]][tokenIds[i]];
            require(stakeInfo.owner == msg.sender, "NFTStaking: not owner");

            uint256 pending = _calculatePendingRewards(collections[i], tokenIds[i]);
            totalRewards += stakeInfo.accumulatedRewards + pending;

            stakeInfo.lastClaimTime = block.timestamp;
            stakeInfo.accumulatedRewards = 0;
        }

        if (totalRewards > 0) {
            rewardToken.safeTransfer(msg.sender, totalRewards);
        }
    }

    // ============ Internal Functions ============

    function _calculatePendingRewards(
        address collection,
        uint256 tokenId
    ) internal view returns (uint256) {
        StakeInfo storage stakeInfo = stakes[collection][tokenId];
        if (stakeInfo.owner == address(0)) return 0;

        CollectionConfig storage config = collectionConfigs[collection];
        uint256 timeElapsed = block.timestamp - stakeInfo.lastClaimTime;
        uint256 baseReward = config.rewardPerDay * timeElapsed / 1 days;

        return baseReward * config.multiplier / 10000;
    }

    function _removeStakedToken(
        address user,
        address collection,
        uint256 tokenId
    ) internal {
        uint256 index = _stakedTokenIndex[collection][tokenId];
        uint256 lastTokenId = userStakedTokens[user][userStakedTokens[user].length - 1];

        userStakedTokens[user][index] = lastTokenId;
        _stakedTokenIndex[collection][lastTokenId] = index;
        userStakedTokens[user].pop();
        delete _stakedTokenIndex[collection][tokenId];
    }

    // ============ View Functions ============

    /**
     * @notice ดู pending rewards ของ token
     */
    function pendingRewards(
        address collection,
        uint256 tokenId
    ) external view returns (uint256) {
        StakeInfo storage stakeInfo = stakes[collection][tokenId];
        return stakeInfo.accumulatedRewards + _calculatePendingRewards(collection, tokenId);
    }

    /**
     * @notice ดู staked tokens ทั้งหมดของ user
     */
    function getUserStakedTokens(address user) external view returns (uint256[] memory) {
        return userStakedTokens[user];
    }

    /**
     * @notice ดู total pending rewards ของ user ทุก token
     */
    function getTotalPendingRewards(
        address user,
        address collection
    ) external view returns (uint256 total) {
        uint256[] memory tokenIds = userStakedTokens[user];
        for (uint256 i = 0; i < tokenIds.length; i++) {
            StakeInfo storage info = stakes[collection][tokenIds[i]];
            if (info.owner == user) {
                total += info.accumulatedRewards + _calculatePendingRewards(collection, tokenIds[i]);
            }
        }
    }
}
```

---

## 3. NFT Fractional Ownership

### ทฤษฎี Fractionalization

Fractional ownership ช่วยให้ NFT ราคาสูง (เช่น Bored Ape) สามารถเป็นเจ้าของร่วมกันได้:
1. Lock NFT ใน vault
2. Mint ERC-20 tokens แทน shares
3. ใครก็ตามสามารถ buyout โดยจ่าย ETH/token ให้ครบ

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/token/ERC721/utils/ERC721Holder.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title FractionalNFTVault
 * @notice แปลง ERC-721 เป็น ERC-20 fractional shares
 * @dev ผู้ใดก็ได้สามารถ buyout โดยจ่าย ETH ให้ครบตามราคา
 */
contract FractionalNFTVault is ERC20, ERC721Holder, ReentrancyGuard {

    // ============ Enums ============
    enum State { FRACTIONALIZED, BUYOUT_IN_PROGRESS, REDEEMED }

    // ============ State Variables ============
    IERC721 public immutable nft;
    uint256 public immutable tokenId;
    address public immutable curator;  // ผู้สร้าง vault

    State public vaultState;
    uint256 public listingPrice;       // ราคา buyout ที่ตั้งไว้ (in wei)
    uint256 public buyoutPrice;        // ราคา buyout ปัจจุบัน (vote-weighted)

    // Buyout tracking
    address public buyoutBidder;
    uint256 public buyoutBid;
    uint256 public buyoutEndTime;
    uint256 public constant BUYOUT_PERIOD = 2 days;

    // Governance: vote สำหรับ listing price
    mapping(address => uint256) public priceVotes;
    uint256 public totalVotedSupply;

    // ============ Events ============
    event Fractionalized(
        address indexed nft,
        uint256 indexed tokenId,
        uint256 supply,
        uint256 listingPrice
    );
    event BuyoutStarted(address indexed bidder, uint256 bid);
    event BuyoutFinalized(address indexed buyer, uint256 amount);
    event BuyoutCancelled(address indexed bidder);
    event Redeemed(address indexed redeemer, uint256 fractionsBurned);
    event PriceVoteCast(address indexed voter, uint256 price, uint256 weight);

    // ============ Constructor ============
    constructor(
        address _nft,
        uint256 _tokenId,
        uint256 _fractionalSupply,
        uint256 _initialListingPrice,
        string memory _name,
        string memory _symbol,
        address _curator
    ) ERC20(_name, _symbol) {
        require(_fractionalSupply > 0, "Vault: zero supply");
        require(_initialListingPrice > 0, "Vault: zero price");

        nft = IERC721(_nft);
        tokenId = _tokenId;
        curator = _curator;
        listingPrice = _initialListingPrice;
        buyoutPrice = _initialListingPrice;
        vaultState = State.FRACTIONALIZED;

        _mint(_curator, _fractionalSupply);

        emit Fractionalized(_nft, _tokenId, _fractionalSupply, _initialListingPrice);
    }

    // ============ Price Governance ============

    /**
     * @notice Vote สำหรับ listing price
     * @dev ใช้ token balance เป็น voting weight
     * @param price ราคาที่ต้องการ vote (in wei)
     */
    function updatePrice(uint256 price) external {
        require(vaultState == State.FRACTIONALIZED, "Vault: not active");
        require(price > 0, "Vault: zero price");

        uint256 weight = balanceOf(msg.sender);
        require(weight > 0, "Vault: no tokens");

        // ลบ vote เก่าออกก่อน
        if (priceVotes[msg.sender] > 0) {
            uint256 oldPrice = priceVotes[msg.sender];
            // weighted average adjustment: ลบน้ำหนักเก่าออก
            // simplified: just track votes and recalculate
        }

        priceVotes[msg.sender] = price;
        _updateReservationPrice();

        emit PriceVoteCast(msg.sender, price, weight);
    }

    /**
     * @notice คำนวณ weighted average listing price
     */
    function _updateReservationPrice() internal {
        // Simplified: curator price เป็น baseline
        // Real implementation จะ weighted average จาก votes ทั้งหมด
        uint256 supply = totalSupply();
        if (supply == 0) return;

        // Calculate vote-weighted average
        // นี่เป็น simplified version - production จะซับซ้อนกว่า
        buyoutPrice = listingPrice;
    }

    // ============ Buyout Mechanism ============

    /**
     * @notice เริ่ม buyout โดยส่ง ETH เท่ากับ listing price
     * @dev ต้องส่ง ETH ตาม buyoutPrice
     */
    function startBuyout() external payable nonReentrant {
        require(vaultState == State.FRACTIONALIZED, "Vault: not active");
        require(msg.value >= buyoutPrice, "Vault: insufficient ETH");

        vaultState = State.BUYOUT_IN_PROGRESS;
        buyoutBidder = msg.sender;
        buyoutBid = msg.value;
        buyoutEndTime = block.timestamp + BUYOUT_PERIOD;

        emit BuyoutStarted(msg.sender, msg.value);
    }

    /**
     * @notice Finalize buyout หลังผ่าน 2 วัน (ไม่มี opposition)
     * @dev ผู้ถือ fractions สามารถ redeem ETH ได้หลัง buyout
     */
    function finalizeBuyout() external nonReentrant {
        require(vaultState == State.BUYOUT_IN_PROGRESS, "Vault: not in buyout");
        require(block.timestamp >= buyoutEndTime, "Vault: buyout period not ended");

        address buyer = buyoutBidder;
        vaultState = State.REDEEMED;

        // โอน NFT ให้ buyer
        nft.safeTransferFrom(address(this), buyer, tokenId);

        emit BuyoutFinalized(buyer, buyoutBid);
    }

    /**
     * @notice Opposition: outbid buyout
     * @dev ผู้ถือ fractions สามารถส่ง ETH มากกว่าเพื่อ outbid
     */
    function outbidBuyout() external payable nonReentrant {
        require(vaultState == State.BUYOUT_IN_PROGRESS, "Vault: not in buyout");
        require(block.timestamp < buyoutEndTime, "Vault: buyout ended");
        require(msg.value > buyoutBid * 110 / 100, "Vault: insufficient outbid"); // 10% higher

        address previousBidder = buyoutBidder;
        uint256 previousBid = buyoutBid;

        buyoutBidder = msg.sender;
        buyoutBid = msg.value;
        buyoutEndTime = block.timestamp + BUYOUT_PERIOD; // reset timer

        // คืน ETH ให้ previous bidder
        (bool success, ) = previousBidder.call{value: previousBid}("");
        require(success, "Vault: ETH refund failed");

        emit BuyoutStarted(msg.sender, msg.value);
    }

    /**
     * @notice Cancel buyout (เฉพาะ bidder เท่านั้น)
     */
    function cancelBuyout() external nonReentrant {
        require(vaultState == State.BUYOUT_IN_PROGRESS, "Vault: not in buyout");
        require(msg.sender == buyoutBidder, "Vault: not bidder");
        require(block.timestamp < buyoutEndTime, "Vault: cannot cancel after end");

        vaultState = State.FRACTIONALIZED;
        uint256 refundAmount = buyoutBid;
        buyoutBidder = address(0);
        buyoutBid = 0;

        (bool success, ) = msg.sender.call{value: refundAmount}("");
        require(success, "Vault: ETH refund failed");

        emit BuyoutCancelled(msg.sender);
    }

    /**
     * @notice Redeem ETH โดย burn fraction tokens (หลัง buyout)
     * @param fractionAmount จำนวน fraction tokens ที่ต้องการ burn
     */
    function redeem(uint256 fractionAmount) external nonReentrant {
        require(vaultState == State.REDEEMED, "Vault: buyout not finalized");
        require(fractionAmount > 0, "Vault: zero amount");
        require(balanceOf(msg.sender) >= fractionAmount, "Vault: insufficient balance");

        uint256 supply = totalSupply();
        uint256 ethAmount = buyoutBid * fractionAmount / supply;

        _burn(msg.sender, fractionAmount);

        (bool success, ) = msg.sender.call{value: ethAmount}("");
        require(success, "Vault: ETH transfer failed");

        emit Redeemed(msg.sender, fractionAmount);
    }

    /**
     * @notice ผู้ถือ fraction ทั้งหมด vote cancel → คืน NFT
     */
    function kickBuyout() external nonReentrant {
        require(vaultState == State.BUYOUT_IN_PROGRESS, "Vault: not in buyout");
        // Simplified: 50%+ ของ supply ต้องโทรหา → จริงๆ ต้องมี governance
        // ตัวอย่างนี้ให้ curator kick ได้
        require(msg.sender == curator, "Vault: only curator");
        require(block.timestamp < buyoutEndTime, "Vault: too late");

        vaultState = State.FRACTIONALIZED;
        address bidder = buyoutBidder;
        uint256 bid = buyoutBid;
        buyoutBidder = address(0);
        buyoutBid = 0;

        (bool success, ) = bidder.call{value: bid}("");
        require(success, "Vault: ETH refund failed");

        emit BuyoutCancelled(bidder);
    }
}

/**
 * @title FractionalVaultFactory
 * @notice Factory สำหรับสร้าง FractionalNFTVault
 */
contract FractionalVaultFactory {
    mapping(address => mapping(uint256 => address)) public vaults;
    address[] public allVaults;

    event VaultCreated(
        address indexed vault,
        address indexed nft,
        uint256 indexed tokenId,
        address curator
    );

    /**
     * @notice สร้าง vault ใหม่สำหรับ NFT
     * @param nft NFT contract address
     * @param tokenId Token ID
     * @param fractionalSupply จำนวน fraction tokens (e.g., 1,000,000)
     * @param listingPrice ราคาเริ่มต้นสำหรับ buyout (in wei)
     * @param name ชื่อ fraction token
     * @param symbol Symbol ของ fraction token
     */
    function createVault(
        address nft,
        uint256 tokenId,
        uint256 fractionalSupply,
        uint256 listingPrice,
        string calldata name,
        string calldata symbol
    ) external returns (address vault) {
        require(vaults[nft][tokenId] == address(0), "Factory: vault exists");

        vault = address(new FractionalNFTVault(
            nft,
            tokenId,
            fractionalSupply,
            listingPrice,
            name,
            symbol,
            msg.sender
        ));

        // Transfer NFT to vault
        IERC721(nft).safeTransferFrom(msg.sender, vault, tokenId);

        vaults[nft][tokenId] = vault;
        allVaults.push(vault);

        emit VaultCreated(vault, nft, tokenId, msg.sender);
    }

    function getVault(address nft, uint256 tokenId) external view returns (address) {
        return vaults[nft][tokenId];
    }

    function getVaultCount() external view returns (uint256) {
        return allVaults.length;
    }
}
```

---

## 4. Soulbound Token (SBT) - EIP-5192

### ทฤษฎี Soulbound Token

SBT (Soulbound Token) คือ non-transferable NFT ที่ represent:
- **Identity**: KYC verification, membership
- **Credentials**: Academic certificates, professional licenses
- **Reputation**: Achievement badges, contribution records

**EIP-5192** กำหนด interface:
- `locked(uint256 tokenId)` → returns true เสมอสำหรับ pure SBT
- `Locked` event เมื่อ token ถูก lock
- `Unlocked` event เมื่อ token ถูก unlock (สำหรับ semi-soulbound)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/utils/Base64.sol";
import "@openzeppelin/contracts/utils/Strings.sol";

/**
 * @title IERC5192
 * @notice EIP-5192: Minimal Soulbound NFT interface
 */
interface IERC5192 {
    /// @notice Emitted when the locking status is changed to locked
    event Locked(uint256 tokenId);

    /// @notice Emitted when the locking status is changed to unlocked
    event Unlocked(uint256 tokenId);

    /// @notice Returns the locking status of an Soulbound Token
    function locked(uint256 tokenId) external view returns (bool);
}

/**
 * @title SoulboundToken
 * @notice EIP-5192 compliant Soulbound Token for identity/credentials
 * @dev Non-transferable NFT with issuer-controlled revocation
 */
contract SoulboundToken is ERC721URIStorage, AccessControl, IERC5192 {
    using Strings for uint256;

    bytes32 public constant ISSUER_ROLE = keccak256("ISSUER_ROLE");
    bytes32 public constant REVOKER_ROLE = keccak256("REVOKER_ROLE");

    // ============ Structs ============
    struct CredentialData {
        string credentialType; // "KYC", "MEMBERSHIP", "CERTIFICATE", etc.
        string issuer;         // Organization name
        uint256 issuedAt;      // Issue timestamp
        uint256 expiresAt;     // 0 = never expires
        string metadata;       // Additional JSON metadata
        bool isRevoked;
    }

    // ============ State ============
    mapping(uint256 => CredentialData) public credentials;
    mapping(address => uint256[]) public holderTokens;
    mapping(uint256 => bool) private _isLocked;

    uint256 private _tokenIdCounter;

    // ============ Events ============
    event CredentialIssued(
        uint256 indexed tokenId,
        address indexed holder,
        string credentialType,
        string issuer,
        uint256 expiresAt
    );
    event CredentialRevoked(
        uint256 indexed tokenId,
        address indexed holder,
        string reason
    );
    event CredentialExpired(uint256 indexed tokenId);

    // ============ Constructor ============
    constructor(string memory name, string memory symbol) ERC721(name, symbol) {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(ISSUER_ROLE, msg.sender);
        _grantRole(REVOKER_ROLE, msg.sender);
    }

    // ============ Issuing ============

    /**
     * @notice Issue credential SBT ให้ address
     * @param to ผู้รับ SBT
     * @param credentialType ประเภท credential
     * @param issuerName ชื่อ issuer
     * @param expiresAt วันหมดอายุ (0 = ไม่หมดอายุ)
     * @param metadata JSON metadata เพิ่มเติม
     */
    function issueCredential(
        address to,
        string calldata credentialType,
        string calldata issuerName,
        uint256 expiresAt,
        string calldata metadata
    ) external onlyRole(ISSUER_ROLE) returns (uint256 tokenId) {
        require(to != address(0), "SBT: zero address");
        if (expiresAt != 0) {
            require(expiresAt > block.timestamp, "SBT: invalid expiry");
        }

        tokenId = ++_tokenIdCounter;

        credentials[tokenId] = CredentialData({
            credentialType: credentialType,
            issuer: issuerName,
            issuedAt: block.timestamp,
            expiresAt: expiresAt,
            metadata: metadata,
            isRevoked: false
        });

        _isLocked[tokenId] = true;
        _safeMint(to, tokenId);

        holderTokens[to].push(tokenId);

        emit CredentialIssued(tokenId, to, credentialType, issuerName, expiresAt);
        emit Locked(tokenId); // EIP-5192

        return tokenId;
    }

    /**
     * @notice Revoke credential
     * @param tokenId Token ที่ต้องการ revoke
     * @param reason เหตุผล
     */
    function revokeCredential(
        uint256 tokenId,
        string calldata reason
    ) external onlyRole(REVOKER_ROLE) {
        require(_ownerOf(tokenId) != address(0), "SBT: nonexistent token");
        require(!credentials[tokenId].isRevoked, "SBT: already revoked");

        credentials[tokenId].isRevoked = true;
        address holder = ownerOf(tokenId);

        emit CredentialRevoked(tokenId, holder, reason);
    }

    // ============ EIP-5192 ============

    /**
     * @notice EIP-5192: Returns true (SBT is always locked)
     */
    function locked(uint256 tokenId) external view returns (bool) {
        require(_ownerOf(tokenId) != address(0), "SBT: nonexistent token");
        return _isLocked[tokenId];
    }

    // ============ Override Transfers ============

    /**
     * @notice Block transfers สำหรับ locked tokens
     */
    function _update(
        address to,
        uint256 tokenId,
        address auth
    ) internal override returns (address) {
        address from = _ownerOf(tokenId);

        // อนุญาตเฉพาะ mint (from == address(0)) และ burn (to == address(0))
        if (from != address(0) && to != address(0)) {
            require(!_isLocked[tokenId], "SBT: token is soulbound");
        }

        return super._update(to, tokenId, auth);
    }

    // ============ tokenURI ============

    function tokenURI(uint256 tokenId) public view override returns (string memory) {
        require(_ownerOf(tokenId) != address(0), "SBT: nonexistent token");

        CredentialData storage cred = credentials[tokenId];
        bool isValid = !cred.isRevoked &&
            (cred.expiresAt == 0 || block.timestamp < cred.expiresAt);

        string memory status = isValid ? "VALID" : (cred.isRevoked ? "REVOKED" : "EXPIRED");
        string memory statusColor = isValid ? "#00ff88" : "#ff4444";

        string memory svg = string(abi.encodePacked(
            '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 300 200" width="300" height="200">',
            '<rect width="300" height="200" fill="#0d1117" rx="10"/>',
            '<rect x="3" y="3" width="294" height="194" fill="none" stroke="', statusColor, '" stroke-width="1.5" rx="8"/>',
            '<text x="150" y="35" text-anchor="middle" font-size="14" fill="', statusColor, '" font-weight="bold">&#128274; SOULBOUND</text>',
            '<text x="150" y="65" text-anchor="middle" font-size="18" fill="white" font-weight="bold">', cred.credentialType, '</text>',
            '<text x="150" y="90" text-anchor="middle" font-size="11" fill="#888">Issued by: ', cred.issuer, '</text>',
            '<rect x="90" y="105" width="120" height="22" rx="11" fill="', statusColor, '" opacity="0.2"/>',
            '<text x="150" y="120" text-anchor="middle" font-size="12" fill="', statusColor, '" font-weight="bold">', status, '</text>',
            '<text x="150" y="150" text-anchor="middle" font-size="10" fill="#555">Issued: ', _formatTimestamp(cred.issuedAt), '</text>',
            cred.expiresAt > 0
                ? string(abi.encodePacked('<text x="150" y="165" text-anchor="middle" font-size="10" fill="#555">Expires: ', _formatTimestamp(cred.expiresAt), '</text>'))
                : '<text x="150" y="165" text-anchor="middle" font-size="10" fill="#555">No Expiration</text>',
            '<text x="150" y="188" text-anchor="middle" font-size="9" fill="#333">Token #', tokenId.toString(), ' | EIP-5192</text>',
            '</svg>'
        ));

        string memory json = string(abi.encodePacked(
            '{"name":"', cred.credentialType, ' #', tokenId.toString(), '",',
            '"description":"Soulbound credential issued by ', cred.issuer, '",',
            '"image":"data:image/svg+xml;base64,', Base64.encode(bytes(svg)), '",',
            '"attributes":[',
            '{"trait_type":"Type","value":"', cred.credentialType, '"},',
            '{"trait_type":"Status","value":"', status, '"},',
            '{"trait_type":"Issuer","value":"', cred.issuer, '"},',
            '{"trait_type":"Soulbound","value":true}',
            ']}'
        ));

        return string(abi.encodePacked("data:application/json;base64,", Base64.encode(bytes(json))));
    }

    function _formatTimestamp(uint256 timestamp) internal pure returns (string memory) {
        // Simplified date format: just return unix timestamp as string
        return timestamp.toString();
    }

    function isValidCredential(uint256 tokenId) external view returns (bool) {
        if (_ownerOf(tokenId) == address(0)) return false;
        CredentialData storage cred = credentials[tokenId];
        return !cred.isRevoked && (cred.expiresAt == 0 || block.timestamp < cred.expiresAt);
    }

    function getHolderCredentials(address holder) external view returns (uint256[] memory) {
        return holderTokens[holder];
    }

    function supportsInterface(bytes4 interfaceId) public view override(ERC721URIStorage, AccessControl) returns (bool) {
        return interfaceId == type(IERC5192).interfaceId || super.supportsInterface(interfaceId);
    }
}
```

---

## 5. Royalty Standard EIP-2981

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/interfaces/IERC2981.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title RoyaltyNFT
 * @notice ERC-721 พร้อม EIP-2981 royalty standard
 * @dev รองรับทั้ง global royalty และ per-token royalty
 */
contract RoyaltyNFT is ERC721, Ownable, IERC2981 {
    using Strings for uint256;

    // ============ Structs ============
    struct RoyaltyInfo {
        address receiver;
        uint96 royaltyFraction; // out of 10000 (basis points)
    }

    // ============ State ============
    RoyaltyInfo private _defaultRoyaltyInfo;
    mapping(uint256 => RoyaltyInfo) private _tokenRoyaltyInfo;

    uint256 private _tokenIdCounter;
    uint256 public constant MAX_ROYALTY = 1000; // 10% maximum

    // ============ Events ============
    event DefaultRoyaltySet(address indexed receiver, uint96 feeNumerator);
    event TokenRoyaltySet(uint256 indexed tokenId, address indexed receiver, uint96 feeNumerator);

    // ============ Constructor ============
    constructor(
        string memory name,
        string memory symbol,
        address defaultRoyaltyReceiver,
        uint96 defaultRoyaltyFraction
    ) ERC721(name, symbol) Ownable(msg.sender) {
        _setDefaultRoyalty(defaultRoyaltyReceiver, defaultRoyaltyFraction);
    }

    // ============ EIP-2981 ============

    /**
     * @notice EIP-2981: คำนวณ royalty ที่ต้องจ่าย
     * @param tokenId Token ID
     * @param salePrice ราคาขาย
     * @return receiver Address ที่รับ royalty
     * @return royaltyAmount จำนวน royalty ที่ต้องจ่าย
     */
    function royaltyInfo(
        uint256 tokenId,
        uint256 salePrice
    ) external view override returns (address receiver, uint256 royaltyAmount) {
        RoyaltyInfo memory royalty = _tokenRoyaltyInfo[tokenId].receiver != address(0)
            ? _tokenRoyaltyInfo[tokenId]
            : _defaultRoyaltyInfo;

        receiver = royalty.receiver;
        royaltyAmount = (salePrice * royalty.royaltyFraction) / 10000;
    }

    // ============ Admin ============

    function setDefaultRoyalty(
        address receiver,
        uint96 feeNumerator
    ) external onlyOwner {
        _setDefaultRoyalty(receiver, feeNumerator);
    }

    function setTokenRoyalty(
        uint256 tokenId,
        address receiver,
        uint96 feeNumerator
    ) external onlyOwner {
        require(feeNumerator <= MAX_ROYALTY, "RoyaltyNFT: royalty too high");
        require(receiver != address(0), "RoyaltyNFT: zero receiver");

        _tokenRoyaltyInfo[tokenId] = RoyaltyInfo({
            receiver: receiver,
            royaltyFraction: feeNumerator
        });

        emit TokenRoyaltySet(tokenId, receiver, feeNumerator);
    }

    function resetTokenRoyalty(uint256 tokenId) external onlyOwner {
        delete _tokenRoyaltyInfo[tokenId];
    }

    function _setDefaultRoyalty(address receiver, uint96 feeNumerator) internal {
        require(feeNumerator <= MAX_ROYALTY, "RoyaltyNFT: royalty too high");
        require(receiver != address(0), "RoyaltyNFT: zero receiver");

        _defaultRoyaltyInfo = RoyaltyInfo({
            receiver: receiver,
            royaltyFraction: feeNumerator
        });

        emit DefaultRoyaltySet(receiver, feeNumerator);
    }

    // ============ Minting ============

    function mint(address to) external onlyOwner returns (uint256) {
        uint256 tokenId = ++_tokenIdCounter;
        _safeMint(to, tokenId);
        return tokenId;
    }

    function mintWithCustomRoyalty(
        address to,
        address royaltyReceiver,
        uint96 royaltyFraction
    ) external onlyOwner returns (uint256) {
        uint256 tokenId = ++_tokenIdCounter;
        _safeMint(to, tokenId);
        setTokenRoyalty(tokenId, royaltyReceiver, royaltyFraction);
        return tokenId;
    }

    // ============ Interface Support ============

    function supportsInterface(bytes4 interfaceId) public view override(ERC721, IERC165) returns (bool) {
        return interfaceId == type(IERC2981).interfaceId || super.supportsInterface(interfaceId);
    }
}
```

---

## Workshop และ Exercises

### Exercise 1: Level System สำหรับ Dynamic NFT

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title LevelSystem
 * @notice คำนวณ level progression สำหรับ game NFT
 */
contract LevelSystem {
    struct LevelInfo {
        uint256 level;
        uint256 xpRequired;     // XP ที่ต้องการถึง level นี้
        uint256 xpToNextLevel;  // XP ที่เหลือถึง level ถัดไป
        uint256 progress;       // % progress (0-100)
    }

    uint256[] public xpThresholds = [0, 100, 300, 700, 1500, 3000, 6000, 12000, 25000, 50000];

    function getLevelInfo(uint256 totalXP) external view returns (LevelInfo memory info) {
        info.level = 0;
        info.xpRequired = 0;

        for (uint256 i = xpThresholds.length - 1; i > 0; i--) {
            if (totalXP >= xpThresholds[i]) {
                info.level = i;
                info.xpRequired = xpThresholds[i];
                break;
            }
        }

        if (info.level < xpThresholds.length - 1) {
            uint256 nextThreshold = xpThresholds[info.level + 1];
            info.xpToNextLevel = nextThreshold - totalXP;
            uint256 levelXP = nextThreshold - info.xpRequired;
            uint256 progressXP = totalXP - info.xpRequired;
            info.progress = progressXP * 100 / levelXP;
        } else {
            info.xpToNextLevel = 0;
            info.progress = 100;
        }
    }

    function calcXPReward(
        bool won,
        uint256 levelDiff, // attacker level - defender level
        uint256 rarity     // 1-5
    ) external pure returns (uint256 xpReward) {
        uint256 base = won ? 50 : 15;

        // Bonus สำหรับ beating stronger opponent
        if (won && levelDiff == 0) {
            base += 20;
        } else if (won && int256(levelDiff) < 0) {
            base += uint256(-int256(levelDiff)) * 15; // beat higher level = more XP
        }

        // Rarity multiplier
        xpReward = base * rarity;
    }
}
```

### Exercise 2: Staking Rewards Calculator

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title StakingRewardsCalculator
 * @notice คำนวณ APY และ projected rewards สำหรับ NFT staking
 */
contract StakingRewardsCalculator {
    /**
     * @notice คำนวณ APY จาก reward rate
     */
    function calcAPY(
        uint256 rewardPerDay,   // tokens ต่อวัน
        uint256 nftFloorPrice,  // ราคา NFT ใน wei
        uint256 tokenPrice      // ราคา reward token ใน wei (per token)
    ) external pure returns (uint256 apy) {
        if (nftFloorPrice == 0) return 0;

        uint256 annualRewardValue = rewardPerDay * 365 * tokenPrice / 1e18;
        apy = annualRewardValue * 10000 / nftFloorPrice; // basis points
    }

    /**
     * @notice คำนวณ projected rewards ระยะเวลาหนึ่ง
     */
    function calcProjectedRewards(
        uint256 rewardPerDay,
        uint256 multiplier,    // basis points (10000 = 1x)
        uint256 durationDays
    ) external pure returns (uint256) {
        return rewardPerDay * durationDays * multiplier / 10000;
    }

    /**
     * @notice Break-even analysis: ใช้เวลากี่วันถึง recoup NFT cost
     */
    function breakEvenDays(
        uint256 nftPurchasePrice, // in wei
        uint256 rewardPerDay,     // tokens ต่อวัน
        uint256 tokenPrice        // ราคา 1 token ใน wei
    ) external pure returns (uint256 days_) {
        if (rewardPerDay == 0 || tokenPrice == 0) return type(uint256).max;

        uint256 dailyRewardValue = rewardPerDay * tokenPrice / 1e18;
        if (dailyRewardValue == 0) return type(uint256).max;

        days_ = nftPurchasePrice / dailyRewardValue;
    }
}
```

### Exercise 3: Credential Verification System

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title CredentialVerifier
 * @notice ตรวจสอบ SBT credentials สำหรับ access control
 */
contract CredentialVerifier {
    address public immutable sbtContract;

    constructor(address _sbtContract) {
        sbtContract = _sbtContract;
    }

    /**
     * @notice ตรวจสอบว่า address มี valid credential ประเภทหนึ่งหรือไม่
     */
    function hasValidCredential(
        address holder,
        string calldata credentialType
    ) external view returns (bool) {
        // Simplified: ตรวจผ่าน interface call
        // Production จะ implement interface call ไปยัง SBT contract
        (bool success, bytes memory data) = sbtContract.staticcall(
            abi.encodeWithSignature("getHolderCredentials(address)", holder)
        );

        if (!success) return false;
        uint256[] memory tokenIds = abi.decode(data, (uint256[]));

        for (uint256 i = 0; i < tokenIds.length; i++) {
            (bool credSuccess, bytes memory credData) = sbtContract.staticcall(
                abi.encodeWithSignature("isValidCredential(uint256)", tokenIds[i])
            );
            if (credSuccess) {
                bool isValid = abi.decode(credData, (bool));
                if (isValid) return true;
            }
        }
        return false;
    }
}
```

---

## สรุป Part 58

- **Dynamic NFT**: tokenURI return base64-encoded JSON พร้อม SVG ที่ generate on-chain ทำให้ metadata เปลี่ยนได้ตาม on-chain state เช่น stats และ level ของ game character โดยไม่พึ่ง external storage
- **NFT Staking**: Lock NFT ใน contract และ earn ERC-20 rewards ต่อวัน รองรับ multipliers ตาม collection, batch claim, และ unstake พร้อม rewards
- **Fractional Ownership**: FractionalNFTVault แปลง ERC-721 เป็น ERC-20 shares ที่ tradeable, buyout mechanism ให้ใครก็ได้ซื้อ NFT โดยจ่าย ETH ให้ fraction holders, มี 2-day window สำหรับ counter-bid
- **Soulbound Token (EIP-5192)**: Non-transferable NFT สำหรับ identity/credentials, block transfers โดย override `_update()`, รองรับ revocation โดย issuer, metadata แสดง validity status
- **EIP-2981 Royalty**: Standard interface สำหรับ royalty `royaltyInfo(tokenId, salePrice)` รองรับ default royalty และ per-token override, marketplace ต้องอ่าน interface นี้และจ่าย royalty ให้ receiver

## Next: Part 59 - Cross-Chain Bridges
