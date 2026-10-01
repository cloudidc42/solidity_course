# Part 78: Advanced Protocol Economics (เศรษฐศาสตร์โปรโตคอลขั้นสูง)

## บทนำ

เศรษฐศาสตร์โปรโตคอล (Protocol Economics) คือหัวใจของ DeFi ที่แท้จริง การออกแบบระบบแรงจูงใจที่ดีไม่ใช่แค่เรื่องของ tokenomics แต่เป็นการประยุกต์ใช้ทฤษฎีเกม (Game Theory) การเงินเชิงคณิตศาสตร์ และพฤติกรรมเศรษฐศาสตร์เข้าด้วยกัน

ใน Part นี้เราจะเรียนรู้:
- **Game Theory ใน DeFi**: Nash Equilibria, Schelling Points, และการโจมตีด้วย governance
- **คณิตศาสตร์ LP Token**: การคำนวณ fair value ของ Uniswap V2 และ Curve
- **Impermanent Loss**: สูตรคำนวณและการ hedge ด้วย options
- **Liquidity Incentive Decay**: โครงสร้าง emission schedule และ lockup boost
- **Protocol Revenue Model**: การวิเคราะห์รายได้และความยั่งยืน

---

## 1. Game Theory ใน DeFi

### 1.1 แนวคิด Nash Equilibrium

Nash Equilibrium คือสถานการณ์ที่ไม่มีผู้เล่นคนไหนได้ประโยชน์จากการเปลี่ยนกลยุทธ์ฝ่ายเดียว ใน DeFi เราพบ Nash Equilibria ในหลายบริบท

**ตัวอย่างที่ 1: Liquidity Provision Game**

สมมติมี LP อยู่ 2 คน แต่ละคนเลือกได้ว่าจะ provide liquidity หรือไม่:
- ถ้าทั้งคู่ provide: แต่ละคนได้ fee ครึ่งหนึ่ง = 50 units
- ถ้าคนเดียว provide: คนที่ provide ได้ 100 units, อีกคนได้ 0
- ถ้าไม่มีใคร provide: ทั้งคู่ได้ 0

```
         | Provide  | Not Provide |
---------|----------|-------------|
Provide  | (50, 50) | (100, 0)   |
Not Prov | (0, 100) | (0, 0)     |
```

Nash Equilibrium คือ (Provide, Provide) และ (Not Provide, Not Provide) — แต่ game นี้มี coordination problem

**ตัวอย่างที่ 2: Governance Attack (51% Attack)**

ใน governance token ถ้า attacker สามารถยืม token ชั่วคราวเพื่อผ่าน proposal ที่เป็นประโยชน์ตนเอง นี่คือ governance attack

### 1.2 Schelling Points

Schelling Point คือ "focal point" ที่ผู้เล่นหลายคนจะเลือกโดยไม่ต้องสื่อสารกัน ใน DeFi:
- Price oracle อ้างอิง Schelling point ของราคา "จริง"
- UMA protocol ใช้ Schelling point mechanism สำหรับ dispute resolution
- Curve gauge weight voting มี Schelling points ที่ pools ที่มี liquidity สูงอยู่แล้ว

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title SchellingPointOracle
 * @dev ตัวอย่าง Schelling Point Oracle สำหรับการ resolve ราคา
 * ผู้รายงาน (reporters) แต่ละคนส่งราคาที่ตนเชื่อว่า "ถูก"
 * ผู้ที่รายงานใกล้เคียง median มากที่สุดจะได้รับรางวัล
 */
contract SchellingPointOracle {
    struct Report {
        address reporter;
        uint256 price;
        uint256 stake;
        bool rewarded;
    }

    struct Round {
        uint256 startTime;
        uint256 endTime;
        uint256 resolveTime;
        Report[] reports;
        uint256 resolvedPrice;
        bool resolved;
    }

    mapping(uint256 => Round) public rounds;
    uint256 public currentRound;
    uint256 public constant ROUND_DURATION = 1 hours;
    uint256 public constant RESOLVE_DELAY = 30 minutes;
    uint256 public constant MIN_STAKE = 0.1 ether;
    uint256 public constant REWARD_BAND = 5; // 5% tolerance band

    address public immutable token;

    event RoundStarted(uint256 indexed roundId, uint256 startTime);
    event PriceReported(uint256 indexed roundId, address indexed reporter, uint256 price, uint256 stake);
    event RoundResolved(uint256 indexed roundId, uint256 resolvedPrice);
    event RewardClaimed(uint256 indexed roundId, address indexed reporter, uint256 reward);

    constructor(address _token) {
        token = _token;
        _startNewRound();
    }

    function _startNewRound() internal {
        currentRound++;
        Round storage r = rounds[currentRound];
        r.startTime = block.timestamp;
        r.endTime = block.timestamp + ROUND_DURATION;
        r.resolveTime = r.endTime + RESOLVE_DELAY;
        emit RoundStarted(currentRound, block.timestamp);
    }

    /**
     * @dev รายงานราคาพร้อม stake เพื่อรับรางวัล
     * ยิ่ง stake มาก ยิ่งมีน้ำหนักใน weighted median
     */
    function reportPrice(uint256 price) external payable {
        require(msg.value >= MIN_STAKE, "Insufficient stake");
        Round storage r = rounds[currentRound];
        require(block.timestamp < r.endTime, "Round ended");

        r.reports.push(Report({
            reporter: msg.sender,
            price: price,
            stake: msg.value,
            rewarded: false
        }));

        emit PriceReported(currentRound, msg.sender, price, msg.value);
    }

    /**
     * @dev คำนวณ weighted median เป็น Schelling Point
     * ผู้ที่รายงานใกล้ median จะได้รับรางวัลตาม stake
     */
    function resolveRound(uint256 roundId) external {
        Round storage r = rounds[roundId];
        require(!r.resolved, "Already resolved");
        require(block.timestamp >= r.resolveTime, "Too early");

        uint256 median = _calculateWeightedMedian(roundId);
        r.resolvedPrice = median;
        r.resolved = true;

        emit RoundResolved(roundId, median);
    }

    function _calculateWeightedMedian(uint256 roundId) internal view returns (uint256) {
        Round storage r = rounds[roundId];
        uint256 n = r.reports.length;
        if (n == 0) return 0;

        // Simple sort by price (insertion sort for small arrays)
        uint256[] memory prices = new uint256[](n);
        uint256[] memory stakes = new uint256[](n);

        for (uint256 i = 0; i < n; i++) {
            prices[i] = r.reports[i].price;
            stakes[i] = r.reports[i].stake;
        }

        // Sort by price
        for (uint256 i = 1; i < n; i++) {
            uint256 keyPrice = prices[i];
            uint256 keyStake = stakes[i];
            int256 j = int256(i) - 1;
            while (j >= 0 && prices[uint256(j)] > keyPrice) {
                prices[uint256(j + 1)] = prices[uint256(j)];
                stakes[uint256(j + 1)] = stakes[uint256(j)];
                j--;
            }
            prices[uint256(j + 1)] = keyPrice;
            stakes[uint256(j + 1)] = keyStake;
        }

        // Find weighted median
        uint256 totalStake = 0;
        for (uint256 i = 0; i < n; i++) {
            totalStake += stakes[i];
        }

        uint256 halfStake = totalStake / 2;
        uint256 cumStake = 0;
        for (uint256 i = 0; i < n; i++) {
            cumStake += stakes[i];
            if (cumStake >= halfStake) {
                return prices[i];
            }
        }

        return prices[n - 1];
    }

    /**
     * @dev ผู้รายงานใกล้ median ภายใน REWARD_BAND% จะได้รับ stake คืน + reward
     */
    function claimReward(uint256 roundId, uint256 reportIndex) external {
        Round storage r = rounds[roundId];
        require(r.resolved, "Not resolved");
        Report storage report = r.reports[reportIndex];
        require(report.reporter == msg.sender, "Not your report");
        require(!report.rewarded, "Already rewarded");

        uint256 deviation = _absDiff(report.price, r.resolvedPrice) * 100 / r.resolvedPrice;

        if (deviation <= REWARD_BAND) {
            report.rewarded = true;
            // คืน stake + 10% bonus
            uint256 reward = report.stake * 110 / 100;
            payable(msg.sender).transfer(reward);
            emit RewardClaimed(roundId, msg.sender, reward);
        }
        // ถ้า deviation เกิน band จะ lose stake (ไปเป็น reward pool)
    }

    function _absDiff(uint256 a, uint256 b) internal pure returns (uint256) {
        return a > b ? a - b : b - a;
    }
}
```

### 1.3 Governance Attack Protection

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title AntiGovernanceAttack
 * @dev ระบบป้องกันการโจมตี governance ด้วยวิธีต่างๆ:
 * 1. Voting power snapshot (ใช้ balance ณ เวลาที่ proposal ถูกสร้าง)
 * 2. Minimum holding period ก่อน vote
 * 3. Quorum requirement สูงพอ
 * 4. Timelock delay ให้ community ตรวจสอบก่อน execute
 */
contract AntiGovernanceAttack {
    // ใช้ ERC20Votes snapshot เพื่อป้องกัน flash loan attack
    // voting power ถูก lock ณ เวลา proposal สร้าง

    struct Proposal {
        uint256 id;
        address proposer;
        uint256 snapshotBlock; // block ที่ snapshot voting power
        uint256 startTime;
        uint256 endTime;
        uint256 executionTime; // earliest execution time (after timelock)
        uint256 forVotes;
        uint256 againstVotes;
        uint256 quorumRequired;
        bool executed;
        bool cancelled;
        bytes callData;
        address target;
    }

    mapping(uint256 => Proposal) public proposals;
    mapping(uint256 => mapping(address => bool)) public hasVoted;
    mapping(address => uint256) public lastVoteTime; // สำหรับ cooldown

    uint256 public proposalCount;
    uint256 public constant VOTING_PERIOD = 3 days;
    uint256 public constant TIMELOCK_DELAY = 2 days;
    uint256 public constant MIN_HOLDING_PERIOD = 7 days; // ต้องถือ token ก่อน vote
    uint256 public constant QUORUM_BPS = 1000; // 10% of total supply
    uint256 public constant VOTE_COOLDOWN = 1 days;

    address public immutable token; // ERC20Votes token

    event ProposalCreated(uint256 indexed proposalId, address indexed proposer);
    event Voted(uint256 indexed proposalId, address indexed voter, bool support, uint256 weight);
    event ProposalExecuted(uint256 indexed proposalId);
    event ProposalCancelled(uint256 indexed proposalId);

    constructor(address _token) {
        token = _token;
    }

    /**
     * @dev สร้าง proposal พร้อม snapshot voting power ณ block ปัจจุบัน
     * Flash loan attack จะไม่ได้ผล เพราะ voting power ถูก lock แล้ว
     */
    function propose(
        address target,
        bytes calldata callData,
        uint256 quorumRequired
    ) external returns (uint256 proposalId) {
        // ตรวจสอบว่าผู้ propose มี voting power เพียงพอ
        uint256 proposerVotes = _getVotes(msg.sender, block.number - 1);
        require(proposerVotes >= _getProposalThreshold(), "Insufficient voting power");

        proposalId = ++proposalCount;
        proposals[proposalId] = Proposal({
            id: proposalId,
            proposer: msg.sender,
            snapshotBlock: block.number - 1, // snapshot ก่อนหน้า 1 block
            startTime: block.timestamp,
            endTime: block.timestamp + VOTING_PERIOD,
            executionTime: block.timestamp + VOTING_PERIOD + TIMELOCK_DELAY,
            forVotes: 0,
            againstVotes: 0,
            quorumRequired: quorumRequired,
            executed: false,
            cancelled: false,
            callData: callData,
            target: target
        });

        emit ProposalCreated(proposalId, msg.sender);
    }

    /**
     * @dev Vote โดยใช้ voting power ณ snapshot block
     * ไม่สามารถใช้ token ที่เพิ่งซื้อ/ยืมมาได้
     */
    function vote(uint256 proposalId, bool support) external {
        Proposal storage p = proposals[proposalId];
        require(block.timestamp >= p.startTime, "Voting not started");
        require(block.timestamp <= p.endTime, "Voting ended");
        require(!hasVoted[proposalId][msg.sender], "Already voted");

        // ตรวจสอบ cooldown period
        require(
            block.timestamp >= lastVoteTime[msg.sender] + VOTE_COOLDOWN,
            "Vote cooldown active"
        );

        // ดึง voting power ณ snapshot block (ป้องกัน flash loan attack)
        uint256 weight = _getVotes(msg.sender, p.snapshotBlock);
        require(weight > 0, "No voting power");

        hasVoted[proposalId][msg.sender] = true;
        lastVoteTime[msg.sender] = block.timestamp;

        if (support) {
            p.forVotes += weight;
        } else {
            p.againstVotes += weight;
        }

        emit Voted(proposalId, msg.sender, support, weight);
    }

    function execute(uint256 proposalId) external {
        Proposal storage p = proposals[proposalId];
        require(!p.executed, "Already executed");
        require(!p.cancelled, "Cancelled");
        require(block.timestamp >= p.executionTime, "Timelock not expired");
        require(p.forVotes > p.againstVotes, "Proposal failed");
        require(p.forVotes + p.againstVotes >= p.quorumRequired, "Quorum not met");

        p.executed = true;

        (bool success,) = p.target.call(p.callData);
        require(success, "Execution failed");

        emit ProposalExecuted(proposalId);
    }

    function _getVotes(address account, uint256 blockNumber) internal view returns (uint256) {
        // Interface call to ERC20Votes
        // ใน production ใช้ IVotes(token).getPastVotes(account, blockNumber)
        blockNumber; // suppress unused warning in example
        return IERC20(token).balanceOf(account); // simplified
    }

    function _getProposalThreshold() internal view returns (uint256) {
        return IERC20(token).totalSupply() / 100; // 1% of total supply
    }
}

interface IERC20 {
    function balanceOf(address account) external view returns (uint256);
    function totalSupply() external view returns (uint256);
}
```

---

## 2. LP Token Fair Value Mathematics

### 2.1 Uniswap V2 LP Token Valuation

ใน Uniswap V2 มูลค่า LP token คำนวณได้จาก geometric mean ของ reserves:

**สูตร:**
```
LP_value = 2 * sqrt(reserve0 * reserve1) * sqrt(price0)
```

โดย:
- `reserve0`, `reserve1` = จำนวน token ใน pool
- `price0` = ราคา token0 ใน USD
- ปัจจัย 2 มาจากการที่ LP token แทน "สองฝั่ง" ของ liquidity

**ทำไมต้อง geometric mean ไม่ใช่ arithmetic mean?**

Uniswap ใช้ invariant: `x * y = k`  
ณ จุดสมดุล: `x = sqrt(k/p)`, `y = sqrt(k*p)` (p = ราคา x ใน terms of y)  
ดังนั้น: `total_value = 2 * sqrt(x*y) * price_y`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/utils/math/Math.sol";

/**
 * @title UniswapV2LPValuation
 * @dev คำนวณมูลค่า fair value ของ Uniswap V2 LP token
 * ใช้สำหรับ oracle pricing ของ LP token เป็น collateral ใน lending protocol
 */
contract UniswapV2LPValuation {
    // Precision สำหรับการคำนวณ
    uint256 private constant PRECISION = 1e18;
    uint256 private constant SQRT_PRECISION = 1e9; // sqrt(1e18)

    struct LPTokenInfo {
        address pair;
        address token0;
        address token1;
        uint256 reserve0;
        uint256 reserve1;
        uint256 totalSupply;
        uint8 decimals0;
        uint8 decimals1;
    }

    /**
     * @dev คำนวณ fair value ของ LP token
     * @param pair address ของ Uniswap V2 pair
     * @param price0USD ราคา token0 ใน USD (scaled by 1e18)
     * @param price1USD ราคา token1 ใน USD (scaled by 1e18)
     * @return lpPrice มูลค่า LP token ต่อ unit ใน USD (scaled by 1e18)
     *
     * สูตร: LP_price = 2 * sqrt(reserve0 * reserve1 * price0 * price1) / totalSupply
     * ซึ่ง equivalent กับ: 2 * sqrt(reserve0 * price0 * reserve1 * price1) / totalSupply
     */
    function getLPFairValue(
        address pair,
        uint256 price0USD,
        uint256 price1USD
    ) external view returns (uint256 lpPrice) {
        IUniswapV2Pair pairContract = IUniswapV2Pair(pair);

        (uint256 reserve0, uint256 reserve1,) = pairContract.getReserves();
        uint256 totalSupply = pairContract.totalSupply();

        require(totalSupply > 0, "Empty pool");

        // Normalize reserves to 18 decimals
        // ใน production ต้องดึง decimals จาก token contract
        uint256 normalizedR0 = reserve0; // assume 18 decimals
        uint256 normalizedR1 = reserve1; // assume 18 decimals

        // คำนวณ sqrt(reserve0 * price0 * reserve1 * price1)
        // เพื่อหลีกเลี่ยง overflow ใช้ sqrtProduct ทีละขั้น
        uint256 sqrtK = _sqrt(normalizedR0 * normalizedR1);

        // sqrt(price0 * price1)
        uint256 sqrtPrices = _sqrt(price0USD * price1USD / PRECISION);

        // LP price = 2 * sqrtK * sqrtPrices / totalSupply
        // ปรับ scale: sqrtK มี unit = sqrt(token^2) = token, sqrtPrices มี unit = sqrt(USD^2) = USD
        lpPrice = 2 * sqrtK * sqrtPrices / totalSupply;
    }

    /**
     * @dev คำนวณ LP token price แบบ alternative โดยใช้ manipulation-resistant formula
     * อ้างอิง: https://blog.alphafinance.io/fair-lp-token-pricing/
     *
     * LP_fair_price = (2 * sqrt(k)) / totalSupply * sqrt(price0 * price1)
     * where k = reserve0 * reserve1 (the constant product)
     */
    function getLPFairValueManipResistant(
        uint256 reserve0,
        uint256 reserve1,
        uint256 totalSupply,
        uint256 price0USD,
        uint256 price1USD
    ) external pure returns (uint256) {
        // k = reserve0 * reserve1
        // sqrt(k) ต้องระวัง overflow
        uint256 sqrtK = _sqrt(reserve0) * _sqrt(reserve1) / SQRT_PRECISION;

        // sqrt(price0 * price1)
        uint256 sqrtP = _sqrt(price0USD) * _sqrt(price1USD) / SQRT_PRECISION;

        // LP_price = 2 * sqrtK * sqrtP / totalSupply
        return 2 * sqrtK * sqrtP / totalSupply;
    }

    /**
     * @dev แสดงการแตกส่วน LP token ออกเป็น token0 และ token1
     * สำหรับคำนวณ underlying value
     */
    function getLPUnderlying(
        address pair,
        uint256 lpAmount
    ) external view returns (uint256 amount0, uint256 amount1) {
        IUniswapV2Pair pairContract = IUniswapV2Pair(pair);

        (uint256 reserve0, uint256 reserve1,) = pairContract.getReserves();
        uint256 totalSupply = pairContract.totalSupply();

        // LP ส่วนแบ่ง proportional ใน reserves
        amount0 = lpAmount * reserve0 / totalSupply;
        amount1 = lpAmount * reserve1 / totalSupply;
    }

    function _sqrt(uint256 x) internal pure returns (uint256 y) {
        if (x == 0) return 0;
        uint256 z = (x + 1) / 2;
        y = x;
        while (z < y) {
            y = z;
            z = (x / z + z) / 2;
        }
    }
}

interface IUniswapV2Pair {
    function getReserves() external view returns (uint112 reserve0, uint112 reserve1, uint32 blockTimestampLast);
    function totalSupply() external view returns (uint256);
    function token0() external view returns (address);
    function token1() external view returns (address);
}
```

### 2.2 Curve LP Token Virtual Price

Curve ใช้ **virtual_price** ซึ่งเป็นค่าที่บอกว่า LP token มี "backing" มากน้อยแค่ไหนเมื่อเทียบกับ ideal state:

```
virtual_price = D / total_supply
```

โดย D คือ Curve's invariant (คำนวณจาก StableSwap equation)

**StableSwap Invariant:**
```
A * n^n * sum(x_i) + D = A * D * n^n + D^(n+1) / (n^n * prod(x_i))
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title CurveLPValuation
 * @dev การคำนวณ virtual_price ของ Curve LP token
 * และ fair value สำหรับ stableswap pools
 */
contract CurveLPValuation {
    uint256 private constant PRECISION = 1e18;
    uint256 private constant A_PRECISION = 100; // Amplification coefficient precision
    uint256 private constant N_COINS = 3; // สำหรับ 3-coin pool (3CRV)
    uint256 private constant MAX_ITERATIONS = 255;

    /**
     * @dev คำนวณ D invariant จาก StableSwap equation
     * D คือ ผลรวม virtual balance ของ liquidity ทั้งหมด
     * ใช้ Newton's method ในการ solve
     */
    function calculateD(
        uint256[N_COINS] memory balances,
        uint256 amp
    ) public pure returns (uint256) {
        uint256 S = 0;
        for (uint256 i = 0; i < N_COINS; i++) {
            S += balances[i];
        }
        if (S == 0) return 0;

        uint256 Dprev = 0;
        uint256 D = S;
        uint256 Ann = amp * N_COINS;

        for (uint256 i = 0; i < MAX_ITERATIONS; i++) {
            uint256 D_P = D;
            for (uint256 j = 0; j < N_COINS; j++) {
                // D_P = D_P * D / (balances[j] * N_COINS)
                D_P = D_P * D / (balances[j] * N_COINS);
            }

            Dprev = D;
            // Newton step:
            // D = (Ann * S + D_P * N_COINS) * D / ((Ann - 1) * D + (N_COINS + 1) * D_P)
            D = (Ann * S / A_PRECISION + D_P * N_COINS) * D /
                ((Ann / A_PRECISION - 1) * D + (N_COINS + 1) * D_P);

            if (_absDiff(D, Dprev) <= 1) break;
        }

        return D;
    }

    /**
     * @dev คำนวณ virtual_price ของ Curve LP token
     * virtual_price เพิ่มขึ้นตามเวลาเมื่อ pool สะสม fees
     *
     * @param balances balances ของแต่ละ coin ใน pool
     * @param amp Amplification parameter (A)
     * @param totalSupply จำนวน LP token ทั้งหมด
     * @return virtualPrice = D / totalSupply
     */
    function getVirtualPrice(
        uint256[N_COINS] memory balances,
        uint256 amp,
        uint256 totalSupply
    ) external pure returns (uint256) {
        uint256 D = calculateD(balances, amp);
        return D * PRECISION / totalSupply;
    }

    /**
     * @dev คำนวณ LP token value ใน USD โดยใช้ virtual_price
     * สำหรับ stablecoin pool ทุก coin มีราคา ~$1
     *
     * ข้อดีของ virtual_price:
     * - Monotonically increasing (ไม่ลดลง ยกเว้น admin action)
     * - Manipulation resistant (ต้องมี real liquidity เปลี่ยนแปลง)
     * - Accumulates fees automatically
     */
    function getLPValueUSD(
        address curvePool,
        uint256 lpAmount
    ) external view returns (uint256) {
        ICurvePool pool = ICurvePool(curvePool);
        uint256 vp = pool.get_virtual_price(); // normalized to 1e18

        // For stablecoin pools: 1 virtual unit ≈ $1
        // LP value = lpAmount * virtualPrice / 1e18
        return lpAmount * vp / PRECISION;
    }

    /**
     * @dev คำนวณ LP token value สำหรับ non-stable pool
     * ต้องใช้ price oracle สำหรับแต่ละ coin
     */
    function getTriCryptoLPValue(
        address curvePool,
        uint256 lpAmount,
        uint256[] memory tokenPricesUSD
    ) external view returns (uint256) {
        ICurvePool pool = ICurvePool(curvePool);

        // ดึง balances
        uint256 totalUSDValue = 0;
        uint256 totalSupply = pool.totalSupply();

        for (uint256 i = 0; i < tokenPricesUSD.length; i++) {
            uint256 balance = pool.balances(i);
            uint256 lpShare = balance * lpAmount / totalSupply;
            totalUSDValue += lpShare * tokenPricesUSD[i] / PRECISION;
        }

        return totalUSDValue;
    }

    function _absDiff(uint256 a, uint256 b) internal pure returns (uint256) {
        return a > b ? a - b : b - a;
    }
}

interface ICurvePool {
    function get_virtual_price() external view returns (uint256);
    function totalSupply() external view returns (uint256);
    function balances(uint256 i) external view returns (uint256);
    function A() external view returns (uint256);
}
```

---

## 3. Impermanent Loss: สูตรและการ Hedge

### 3.1 สูตร Impermanent Loss

IL (Impermanent Loss) เกิดจากการเปลี่ยนแปลงราคา relative ของ token ใน pair:

**สูตร:**
```
IL = 2*sqrt(p) / (1 + p) - 1

โดย p = price_ratio_new / price_ratio_old
```

ตัวอย่าง:
- ถ้าราคา ETH เพิ่มขึ้น 2x: p = 2, IL = 2*sqrt(2)/(1+2) - 1 ≈ -5.7%
- ถ้าราคา ETH เพิ่มขึ้น 4x: p = 4, IL = 2*sqrt(4)/(1+4) - 1 = -20%
- ถ้าราคา ETH ลดลง 50%: p = 0.5, IL = 2*sqrt(0.5)/(1+0.5) - 1 ≈ -5.7%

**หมายเหตุ**: IL symmetric กับ log(p) — ดังนั้นเพิ่ม 100% และลด 50% ให้ IL เท่ากัน

### 3.2 การ Hedge IL ด้วย Options

**กลยุทธ์ Straddle เพื่อ Hedge IL:**

เนื่องจาก IL เกิดจาก price movement ในทิศทางใดก็ได้ การซื้อ straddle (long call + long put) จะ offset IL ได้บางส่วน:

```
IL payoff: สูญเสียเมื่อราคาเปลี่ยนแปลง (concave)
Straddle payoff: กำไรเมื่อราคาเปลี่ยนแปลง (convex)
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title ImpermanentLossCalculator
 * @dev คำนวณ Impermanent Loss และประเมิน hedge cost
 *
 * สูตร IL = 2*sqrt(p)/(1+p) - 1
 * โดย p = ratio ราคาใหม่ต่อราคาเก่า
 */
contract ImpermanentLossCalculator {
    uint256 private constant PRECISION = 1e18;
    uint256 private constant ONE = 1e18;

    /**
     * @dev คำนวณ IL เป็น percentage (basis points, 1% = 100)
     * @param priceRatioNew อัตราส่วนราคา ณ ปัจจุบัน (scaled 1e18)
     * @param priceRatioOld อัตราส่วนราคา ณ ตอนเข้า LP (scaled 1e18)
     * @return ilBps IL เป็น basis points (ค่าลบ = การสูญเสีย)
     */
    function calculateIL(
        uint256 priceRatioNew,
        uint256 priceRatioOld
    ) external pure returns (int256 ilBps) {
        // p = priceRatioNew / priceRatioOld
        // ใช้ fixed-point arithmetic
        uint256 p = priceRatioNew * PRECISION / priceRatioOld;

        // sqrt(p) ใน fixed-point
        uint256 sqrtP = _sqrt(p * PRECISION); // sqrt(p * 1e18) = sqrt(p) * 1e9

        // IL = 2*sqrt(p)/(1+p) - 1
        // ทั้งหมด scaled by 1e18
        uint256 numerator = 2 * sqrtP * PRECISION; // 2 * sqrt(p) * 1e27
        uint256 denominator = (ONE + p) * 1e9;     // (1 + p) * 1e27

        uint256 ilFraction = numerator / denominator; // ผล = 2*sqrt(p)/(1+p), scaled 1e18

        // IL = ilFraction - 1 (ค่าลบ = สูญเสีย)
        if (ilFraction >= ONE) {
            ilBps = int256((ilFraction - ONE) * 10000 / ONE);
        } else {
            ilBps = -int256((ONE - ilFraction) * 10000 / ONE);
        }
    }

    /**
     * @dev คำนวณ breakeven fee สำหรับ cover IL
     * LP ต้องได้ fee มากกว่า IL จึงจะทำกำไร
     *
     * @param dailyVolume volume ต่อวัน (USD)
     * @param poolTVL TVL ของ pool (USD)
     * @param feeRate อัตรา fee (basis points, 30 = 0.3%)
     * @param daysHeld จำนวนวันที่ถือ LP
     * @return estimatedFeeEarnings ค่าธรรมเนียมที่คาดว่าได้รับ
     */
    function estimateFeeEarnings(
        uint256 dailyVolume,
        uint256 poolTVL,
        uint256 feeRate,
        uint256 daysHeld
    ) external pure returns (uint256) {
        // fee_earned = (LP_share * volume * fee_rate * days)
        // LP_share = LP_amount / TVL (สมมติ full LP)
        // ตัวอย่าง: TVL = $1M, volume = $500K/day, fee = 0.3%
        // fee per day = 500,000 * 0.003 = $1,500
        // LP share of $1M = 0.1% => earn $1.50/day

        uint256 dailyFee = dailyVolume * feeRate / 10000;
        uint256 totalFee = dailyFee * daysHeld;
        return totalFee * PRECISION / poolTVL; // fee rate per unit of liquidity
    }

    /**
     * @dev จำลอง LP position ที่มี IL hedge ด้วย options
     *
     * กลยุทธ์: ซื้อ long straddle ด้วย premium เท่ากับ expected IL
     * Net P&L = fee_earned - option_premium + option_payoff - IL
     *
     * @param lpAmount จำนวน LP (USD)
     * @param strikeDeltaPct ช่วง price change ที่ options ป้องกัน (เช่น 20 = ±20%)
     * @param optionPremiumBps ค่า premium ของ option (basis points)
     */
    function simulateHedgedLP(
        uint256 lpAmount,
        uint256 actualPriceChangePct, // ค่าบวก = ขึ้น, ใช้ uint เพื่อความง่าย (absolute)
        uint256 feesEarned,
        uint256 optionPremiumBps
    ) external pure returns (
        int256 unhedgedPnL,
        int256 hedgedPnL,
        uint256 ilAmount,
        uint256 optionPayoff
    ) {
        // คำนวณ p จาก price change
        // ถ้าราคาขึ้น 50%, p = 1.5
        uint256 p;
        if (actualPriceChangePct >= 100) {
            p = ONE + (actualPriceChangePct * ONE / 100);
        } else {
            // ถ้าลง, p < 1
            p = ONE - (actualPriceChangePct * ONE / 100);
        }

        // คำนวณ IL amount
        uint256 sqrtP = _sqrt(p * PRECISION);
        uint256 ilFraction = 2 * sqrtP * PRECISION / ((ONE + p) * 1e9);
        ilAmount = lpAmount * (ONE - ilFraction) / ONE;

        // Unhedged P&L
        unhedgedPnL = int256(feesEarned) - int256(ilAmount);

        // Option premium cost
        uint256 premiumCost = lpAmount * optionPremiumBps / 10000;

        // Option payoff (straddle pays |price_change| - strike)
        // สมมติ straddle with ATM strike
        optionPayoff = ilAmount * 120 / 100; // options typically over-compensate IL

        // Hedged P&L
        hedgedPnL = int256(feesEarned) - int256(ilAmount) - int256(premiumCost) + int256(optionPayoff);
    }

    function _sqrt(uint256 x) internal pure returns (uint256 y) {
        if (x == 0) return 0;
        uint256 z = (x + 1) / 2;
        y = x;
        while (z < y) {
            y = z;
            z = (x / z + z) / 2;
        }
    }
}
```

---

## 4. Liquidity Incentive Decay และ Emission Schedule

### 4.1 แนวคิด Emission Schedule

โปรโตคอลส่วนใหญ่ใช้ decreasing emission เพื่อ:
1. **Incentivize early adopters** ด้วย higher rewards
2. **Prevent inflation** ในระยะยาว
3. **Align long-term holders** ด้วย lockup boosts

Bitcoin halving เป็น inspiration สำหรับ 4-year halving cycle ใน DeFi

### 4.2 Step-Function Emission พร้อม Lockup Boost

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title LiquidityIncentiveDecay
 * @dev ระบบ emission schedule แบบ step-function ที่มี:
 * 1. 4-year halving cycle (คล้าย Bitcoin)
 * 2. Lockup boost multiplier (เหมือน veCRV)
 * 3. Anti-dilution mechanism
 *
 * Emission Schedule:
 * Year 1-4:   100 tokens/day
 * Year 5-8:    50 tokens/day  (halving)
 * Year 9-12:   25 tokens/day
 * Year 13-16:  12.5 tokens/day
 * ...
 *
 * Boost:
 * No lock:    1x
 * 6 months:   1.25x
 * 1 year:     1.5x
 * 2 years:    2x
 * 4 years:    2.5x (maximum)
 */
contract LiquidityIncentiveDecay is ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ==================== Constants ====================

    uint256 public constant HALVING_PERIOD = 4 * 365 days; // 4 years
    uint256 public constant BASE_EMISSION = 100e18;         // 100 tokens per day
    uint256 public constant EMISSION_PER_SECOND = BASE_EMISSION / 1 days;
    uint256 public constant MAX_BOOST = 250;    // 2.5x = 250%
    uint256 public constant BASE_BOOST = 100;   // 1x = 100%
    uint256 public constant MAX_LOCK_TIME = 4 * 365 days; // 4 years
    uint256 public constant PRECISION = 1e18;

    // ==================== State ====================

    IERC20 public immutable rewardToken;
    IERC20 public immutable stakingToken;

    uint256 public startTime;
    uint256 public totalStaked;
    uint256 public lastUpdateTime;
    uint256 public accRewardPerShare; // สะสม reward per share (scaled by 1e12)

    struct UserInfo {
        uint256 amount;          // จำนวน token ที่ stake
        uint256 rewardDebt;      // reward debt สำหรับ accurate calculation
        uint256 lockEndTime;     // เวลาสิ้นสุด lockup
        uint256 boostMultiplier; // boost ที่ได้รับ (100 = 1x)
        uint256 pendingRewards;  // rewards ที่สะสมไว้ก่อน harvest
    }

    mapping(address => UserInfo) public userInfo;

    // ==================== Events ====================

    event Deposited(address indexed user, uint256 amount, uint256 lockDuration);
    event Withdrawn(address indexed user, uint256 amount);
    event RewardClaimed(address indexed user, uint256 amount);
    event EmissionUpdated(uint256 newEmissionPerSecond);

    // ==================== Constructor ====================

    constructor(address _rewardToken, address _stakingToken) {
        rewardToken = IERC20(_rewardToken);
        stakingToken = IERC20(_stakingToken);
        startTime = block.timestamp;
        lastUpdateTime = block.timestamp;
    }

    // ==================== Core Functions ====================

    /**
     * @dev ฝาก token พร้อมเลือก lockup duration
     * lockup นานกว่า = boost สูงกว่า = reward มากกว่า
     */
    function deposit(uint256 amount, uint256 lockDuration) external nonReentrant {
        require(amount > 0, "Amount must be > 0");
        require(lockDuration == 0 ||
            (lockDuration >= 6 * 30 days && lockDuration <= MAX_LOCK_TIME),
            "Invalid lock duration"
        );

        _updatePool();

        UserInfo storage user = userInfo[msg.sender];

        // Harvest pending rewards ก่อน
        if (user.amount > 0) {
            _harvestRewards(msg.sender);
        }

        // คำนวณ boost จาก lock duration
        uint256 boost = _calculateBoost(lockDuration);

        // อัปเดต user state
        user.amount += amount;
        user.lockEndTime = block.timestamp + lockDuration;
        user.boostMultiplier = boost;
        user.rewardDebt = user.amount * boost * accRewardPerShare / PRECISION / BASE_BOOST;

        totalStaked += amount;

        stakingToken.safeTransferFrom(msg.sender, address(this), amount);

        emit Deposited(msg.sender, amount, lockDuration);
    }

    /**
     * @dev ถอน token ออก (ต้องรอ lockup หมดก่อน)
     */
    function withdraw(uint256 amount) external nonReentrant {
        UserInfo storage user = userInfo[msg.sender];
        require(user.amount >= amount, "Insufficient balance");
        require(block.timestamp >= user.lockEndTime, "Still locked");

        _updatePool();
        _harvestRewards(msg.sender);

        user.amount -= amount;
        totalStaked -= amount;

        // คำนวณ reward debt ใหม่
        user.rewardDebt = user.amount * user.boostMultiplier * accRewardPerShare / PRECISION / BASE_BOOST;

        stakingToken.safeTransfer(msg.sender, amount);

        emit Withdrawn(msg.sender, amount);
    }

    /**
     * @dev harvest reward โดยไม่ถอน stake
     */
    function harvest() external nonReentrant {
        _updatePool();
        _harvestRewards(msg.sender);
    }

    // ==================== Emission Calculation ====================

    /**
     * @dev คำนวณ emission rate ณ เวลาปัจจุบัน (step-function halving)
     * ทุก 4 ปี emission จะลดลงครึ่งหนึ่ง
     */
    function getCurrentEmissionRate() public view returns (uint256) {
        uint256 elapsed = block.timestamp - startTime;
        uint256 halvings = elapsed / HALVING_PERIOD;

        // หลังจาก 10 halvings emission จะน้อยมาก (< 0.1% ของ original)
        if (halvings >= 10) return 0;

        // Emission = BASE_EMISSION / 2^halvings
        return EMISSION_PER_SECOND >> halvings; // bit shift = divide by 2^halvings
    }

    /**
     * @dev คำนวณ reward ระหว่างสอง timestamp โดยคำนึงถึง halving boundaries
     */
    function calculateRewardBetween(uint256 from, uint256 to) public view returns (uint256) {
        if (from >= to) return 0;

        uint256 totalReward = 0;
        uint256 current = from;

        // แบ่งช่วงเวลาตาม halving boundaries
        while (current < to) {
            uint256 elapsedFromStart = current - startTime;
            uint256 currentHalving = elapsedFromStart / HALVING_PERIOD;
            uint256 nextHalvingTime = startTime + (currentHalving + 1) * HALVING_PERIOD;

            uint256 periodEnd = to < nextHalvingTime ? to : nextHalvingTime;
            uint256 duration = periodEnd - current;

            uint256 emissionRate = EMISSION_PER_SECOND >> currentHalving;
            totalReward += duration * emissionRate;

            current = periodEnd;
        }

        return totalReward;
    }

    // ==================== Internal Functions ====================

    function _updatePool() internal {
        if (block.timestamp <= lastUpdateTime) return;
        if (totalStaked == 0) {
            lastUpdateTime = block.timestamp;
            return;
        }

        uint256 reward = calculateRewardBetween(lastUpdateTime, block.timestamp);
        accRewardPerShare += reward * PRECISION / totalStaked;
        lastUpdateTime = block.timestamp;
    }

    function _harvestRewards(address account) internal {
        UserInfo storage user = userInfo[account];
        if (user.amount == 0) return;

        // Boosted reward = base reward * boost multiplier
        uint256 boostedDebt = user.amount * user.boostMultiplier * accRewardPerShare / PRECISION / BASE_BOOST;
        uint256 pending = boostedDebt - user.rewardDebt;

        if (pending > 0) {
            user.pendingRewards = 0;
            user.rewardDebt = boostedDebt;
            rewardToken.safeTransfer(account, pending);
            emit RewardClaimed(account, pending);
        }
    }

    /**
     * @dev คำนวณ boost multiplier จาก lock duration
     *
     * Linear interpolation ระหว่าง BASE_BOOST และ MAX_BOOST
     * 0 days   = 100 (1x)
     * MAX_LOCK = 250 (2.5x)
     */
    function _calculateBoost(uint256 lockDuration) internal pure returns (uint256) {
        if (lockDuration == 0) return BASE_BOOST;
        if (lockDuration >= MAX_LOCK_TIME) return MAX_BOOST;

        // Linear interpolation
        return BASE_BOOST + (MAX_BOOST - BASE_BOOST) * lockDuration / MAX_LOCK_TIME;
    }

    // ==================== View Functions ====================

    /**
     * @dev คำนวณ pending rewards ของ user รวม boost
     */
    function pendingReward(address account) external view returns (uint256) {
        UserInfo storage user = userInfo[account];
        if (user.amount == 0) return 0;

        uint256 _accRewardPerShare = accRewardPerShare;
        if (block.timestamp > lastUpdateTime && totalStaked > 0) {
            uint256 reward = calculateRewardBetween(lastUpdateTime, block.timestamp);
            _accRewardPerShare += reward * PRECISION / totalStaked;
        }

        uint256 boostedDebt = user.amount * user.boostMultiplier * _accRewardPerShare / PRECISION / BASE_BOOST;
        return boostedDebt - user.rewardDebt;
    }

    /**
     * @dev คำนวณ APR ณ ปัจจุบัน
     */
    function currentAPR(uint256 rewardTokenPriceUSD, uint256 stakingTokenPriceUSD) external view returns (uint256) {
        if (totalStaked == 0) return 0;

        uint256 yearlyEmission = getCurrentEmissionRate() * 365 days;
        uint256 yearlyValueUSD = yearlyEmission * rewardTokenPriceUSD / PRECISION;
        uint256 totalStakedUSD = totalStaked * stakingTokenPriceUSD / PRECISION;

        return yearlyValueUSD * 10000 / totalStakedUSD; // basis points
    }
}
```

---

## 5. Protocol Revenue Model

### 5.1 Fee Structure Analysis

โปรโตคอล DeFi มีรายได้หลายชั้น:

```
Total Fee = Swap Fee + Protocol Fee + Referral Fee

ตัวอย่าง Uniswap V3:
- Tier 0.01%: สำหรับ stablecoin pairs
- Tier 0.05%: สำหรับ correlated assets
- Tier 0.3%: สำหรับ standard pairs
- Tier 1.0%: สำหรับ exotic pairs

Protocol Fee: 10-25% ของ Swap Fee (เปิดปิดได้โดย governance)
Referral: optional ตาม integration
```

### 5.2 Revenue vs Incentive Sustainability

**Token Emission ≠ Revenue**

ปัญหาของหลาย protocol:
- จ่าย $10M/ปีใน incentives
- แต่ได้ revenue จริงแค่ $1M/ปี
- Token price ค้ำยันด้วย dilution = death spiral

**Sustainable Protocol:**
- Revenue ≥ Incentives (หรืออย่างน้อย trending ไปทิศทางนั้น)
- Buyback & burn mechanism
- Real yield ให้ holders

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title ProtocolRevenueModel
 * @dev ระบบ revenue distribution ที่ซับซ้อน:
 *
 * Fee Distribution:
 * - 40% → Liquidity Providers (swap fee)
 * - 20% → Protocol Treasury (for development/buyback)
 * - 20% → veToken stakers (real yield)
 * - 10% → Referral partners
 * - 10% → Insurance fund
 *
 * Revenue Sustainability Check:
 * - ติดตาม protocol revenue vs token emission
 * - Emergency pause ถ้า ratio ต่ำเกินไป
 */
contract ProtocolRevenueModel is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ==================== Constants ====================

    uint256 public constant BPS_DENOMINATOR = 10000;

    // Fee tiers (in basis points)
    uint256 public constant SWAP_FEE_LP = 4000;         // 40% ของ total fee
    uint256 public constant SWAP_FEE_TREASURY = 2000;   // 20%
    uint256 public constant SWAP_FEE_STAKERS = 2000;    // 20%
    uint256 public constant SWAP_FEE_REFERRAL = 1000;   // 10%
    uint256 public constant SWAP_FEE_INSURANCE = 1000;  // 10%

    // Revenue sustainability threshold
    uint256 public constant MIN_REVENUE_RATIO = 5000; // 50% - revenue must cover 50% of emissions

    // ==================== State ====================

    address public treasury;
    address public stakersVault;
    address public insuranceFund;

    // Revenue tracking (30-day rolling)
    uint256 public totalRevenue30d;
    uint256 public totalEmissions30d;
    uint256 public lastRevenueUpdate;

    // Referral system
    mapping(address => address) public referrerOf;
    mapping(address => uint256) public referralEarnings;

    // Revenue breakdown by pool
    mapping(address => uint256) public poolRevenue;
    mapping(address => uint256) public poolVolume;

    // ==================== Events ====================

    event FeeCollected(
        address indexed pool,
        address indexed token,
        uint256 totalAmount,
        uint256 lpAmount,
        uint256 treasuryAmount,
        uint256 stakersAmount
    );

    event ReferralPaid(address indexed referrer, address indexed trader, uint256 amount);
    event SustainabilityAlert(uint256 revenueRatio);
    event BuybackExecuted(uint256 ethSpent, uint256 tokensBought);

    // ==================== Constructor ====================

    constructor(
        address _treasury,
        address _stakersVault,
        address _insuranceFund
    ) Ownable(msg.sender) {
        treasury = _treasury;
        stakersVault = _stakersVault;
        insuranceFund = _insuranceFund;
        lastRevenueUpdate = block.timestamp;
    }

    // ==================== Fee Distribution ====================

    /**
     * @dev กระจาย fee ที่เก็บได้จาก swap
     * @param pool address ของ liquidity pool
     * @param token token ที่ได้รับเป็น fee
     * @param totalFee จำนวน fee ทั้งหมด
     * @param trader ผู้ซื้อขาย (สำหรับ referral)
     */
    function distributeFee(
        address pool,
        address token,
        uint256 totalFee,
        address trader
    ) external nonReentrant {
        require(totalFee > 0, "No fee to distribute");

        // คำนวณแต่ละส่วน
        uint256 lpAmount = totalFee * SWAP_FEE_LP / BPS_DENOMINATOR;
        uint256 treasuryAmount = totalFee * SWAP_FEE_TREASURY / BPS_DENOMINATOR;
        uint256 stakersAmount = totalFee * SWAP_FEE_STAKERS / BPS_DENOMINATOR;
        uint256 referralAmount = totalFee * SWAP_FEE_REFERRAL / BPS_DENOMINATOR;
        uint256 insuranceAmount = totalFee - lpAmount - treasuryAmount - stakersAmount - referralAmount;

        IERC20 feeToken = IERC20(token);

        // Transfer to treasury
        feeToken.safeTransferFrom(msg.sender, treasury, treasuryAmount);

        // Transfer to stakers vault (real yield)
        feeToken.safeTransferFrom(msg.sender, stakersVault, stakersAmount);

        // Transfer to insurance fund
        feeToken.safeTransferFrom(msg.sender, insuranceFund, insuranceAmount);

        // Handle referral
        address referrer = referrerOf[trader];
        if (referrer != address(0)) {
            feeToken.safeTransferFrom(msg.sender, referrer, referralAmount);
            referralEarnings[referrer] += referralAmount;
            emit ReferralPaid(referrer, trader, referralAmount);
        } else {
            // ถ้าไม่มี referrer ส่ง referral amount ไป treasury แทน
            feeToken.safeTransferFrom(msg.sender, treasury, referralAmount);
        }

        // LP amount ถูก handle โดย pool contract เอง (ผ่าน accFeePerShare)

        // อัปเดต revenue tracking
        poolRevenue[pool] += treasuryAmount + stakersAmount;
        totalRevenue30d += treasuryAmount + stakersAmount;

        emit FeeCollected(pool, token, totalFee, lpAmount, treasuryAmount, stakersAmount);
    }

    /**
     * @dev ตรวจสอบ sustainability ของ protocol
     * Revenue / Emissions ratio ควร >= 50%
     */
    function checkSustainability() external returns (bool isSustainable) {
        // รีเซ็ต 30-day window
        if (block.timestamp - lastRevenueUpdate >= 30 days) {
            totalRevenue30d = 0;
            totalEmissions30d = 0;
            lastRevenueUpdate = block.timestamp;
        }

        if (totalEmissions30d == 0) return true; // ไม่มี emission = sustainable

        uint256 ratio = totalRevenue30d * BPS_DENOMINATOR / totalEmissions30d;
        isSustainable = ratio >= MIN_REVENUE_RATIO;

        if (!isSustainable) {
            emit SustainabilityAlert(ratio);
        }
    }

    /**
     * @dev คำนวณ "Real Yield" ที่ stakers ได้รับ
     * Real Yield = (Revenue จาก stakers) / (Total staked value) * annualized
     */
    function calculateRealYield(
        uint256 totalStakedValue,
        uint256 periodicRevenue,
        uint256 periodDays
    ) external pure returns (uint256 aprBps) {
        if (totalStakedValue == 0 || periodDays == 0) return 0;
        uint256 annualizedRevenue = periodicRevenue * 365 / periodDays;
        aprBps = annualizedRevenue * BPS_DENOMINATOR / totalStakedValue;
    }

    // ==================== Revenue Analytics ====================

    /**
     * @dev คำนวณ P/E ratio ของ protocol (Price-to-Earnings)
     * P/E = Market Cap / Annual Revenue
     * P/E ต่ำ = undervalued relative to revenue
     */
    function calculateProtocolPE(
        uint256 marketCap,
        uint256 annualRevenue
    ) external pure returns (uint256) {
        if (annualRevenue == 0) return type(uint256).max; // infinite P/E
        return marketCap / annualRevenue;
    }

    /**
     * @dev คำนวณ "Fully Diluted Valuation" vs Revenue
     * FDV/Revenue ratio
     */
    function calculateFDVRevenue(
        uint256 totalSupply,
        uint256 tokenPrice,
        uint256 annualRevenue
    ) external pure returns (uint256) {
        uint256 fdv = totalSupply * tokenPrice / 1e18;
        if (annualRevenue == 0) return type(uint256).max;
        return fdv / annualRevenue;
    }

    // ==================== Referral System ====================

    function setReferrer(address referee, address referrer) external onlyOwner {
        require(referrerOf[referee] == address(0), "Referrer already set");
        require(referee != referrer, "Self-referral not allowed");
        referrerOf[referee] = referrer;
    }

    // ==================== Admin ====================

    function recordEmission(uint256 amount) external onlyOwner {
        totalEmissions30d += amount;
    }
}
```

---

## Workshop: สร้าง Protocol Economics Dashboard

### เป้าหมาย
สร้าง contract ที่รวม:
1. LP fair value calculation (Uniswap V2 style)
2. IL calculator
3. Revenue sustainability check

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title ProtocolEconomicsDashboard
 * @dev Dashboard สรุป metrics ด้านเศรษฐศาสตร์ของ protocol
 *
 * Workshop Task:
 * 1. Deploy contract นี้บน testnet
 * 2. เพิ่ม pool เข้าไป
 * 3. คำนวณ IL หลังจากราคาเปลี่ยน
 * 4. ตรวจสอบ sustainability
 */
contract ProtocolEconomicsDashboard {
    uint256 private constant PRECISION = 1e18;
    uint256 private constant ONE = 1e18;

    struct PoolMetrics {
        address poolAddress;
        string name;
        uint256 tvl;
        uint256 volume24h;
        uint256 fee24h;
        uint256 lpPrice;
        uint256 virtualPrice;
        uint256 entryPrice0;  // ราคา token0 ตอนเข้า LP
        uint256 entryPrice1;  // ราคา token1 ตอนเข้า LP
        uint256 addedAt;
    }

    mapping(address => PoolMetrics) public poolMetrics;
    address[] public pools;

    event PoolAdded(address indexed pool, string name);
    event MetricsUpdated(address indexed pool, uint256 tvl, uint256 volume24h);

    /**
     * @dev เพิ่ม pool เข้า dashboard
     */
    function addPool(
        address pool,
        string calldata name,
        uint256 initialPrice0,
        uint256 initialPrice1
    ) external {
        require(poolMetrics[pool].addedAt == 0, "Pool already added");

        poolMetrics[pool] = PoolMetrics({
            poolAddress: pool,
            name: name,
            tvl: 0,
            volume24h: 0,
            fee24h: 0,
            lpPrice: 0,
            virtualPrice: ONE, // เริ่มต้นที่ 1.0
            entryPrice0: initialPrice0,
            entryPrice1: initialPrice1,
            addedAt: block.timestamp
        });

        pools.push(pool);
        emit PoolAdded(pool, name);
    }

    /**
     * @dev อัปเดต metrics ของ pool
     */
    function updateMetrics(
        address pool,
        uint256 tvl,
        uint256 volume24h,
        uint256 fee24h,
        uint256 currentReserve0,
        uint256 currentReserve1,
        uint256 totalSupply
    ) external {
        PoolMetrics storage m = poolMetrics[pool];
        require(m.addedAt > 0, "Pool not registered");

        m.tvl = tvl;
        m.volume24h = volume24h;
        m.fee24h = fee24h;

        // คำนวณ LP price จาก reserves
        if (currentReserve0 > 0 && currentReserve1 > 0 && totalSupply > 0) {
            uint256 sqrtK = _sqrt(currentReserve0 * currentReserve1);
            // ใช้ geometric mean pricing
            m.lpPrice = 2 * sqrtK * PRECISION / totalSupply;
        }

        // Virtual price เพิ่มขึ้นตาม fee accumulation
        if (totalSupply > 0) {
            m.virtualPrice = (currentReserve0 + currentReserve1) * PRECISION / totalSupply;
        }

        emit MetricsUpdated(pool, tvl, volume24h);
    }

    /**
     * @dev คำนวณ IL สำหรับ LP position
     * @param pool address ของ pool
     * @param currentPrice0 ราคา token0 ปัจจุบัน
     * @return ilBps IL ใน basis points (ค่าลบ = สูญเสีย)
     */
    function calculateCurrentIL(
        address pool,
        uint256 currentPrice0
    ) external view returns (int256 ilBps) {
        PoolMetrics storage m = poolMetrics[pool];
        require(m.addedAt > 0, "Pool not registered");

        // p = ราคาปัจจุบัน / ราคาตอนเข้า
        if (m.entryPrice0 == 0) return 0;

        uint256 p = currentPrice0 * PRECISION / m.entryPrice0;
        uint256 sqrtP = _sqrt(p * PRECISION); // sqrt(p) * 1e9

        // IL = 2*sqrt(p)/(1+p) - 1
        uint256 numerator = 2 * sqrtP * PRECISION;    // 2 * sqrt(p) * 1e27
        uint256 denominator = (ONE + p) * 1e9;        // (1 + p) * 1e27
        uint256 ilFraction = numerator / denominator;  // 2*sqrt(p)/(1+p), scaled 1e18

        if (ilFraction >= ONE) {
            ilBps = int256((ilFraction - ONE) * 10000 / ONE);
        } else {
            ilBps = -int256((ONE - ilFraction) * 10000 / ONE);
        }
    }

    /**
     * @dev คำนวณ breakeven days สำหรับ IL recovery จาก fees
     */
    function calculateBreakevenDays(
        address pool,
        uint256 currentPrice0,
        uint256 lpAmount
    ) external view returns (uint256 days_) {
        PoolMetrics storage m = poolMetrics[pool];
        if (m.tvl == 0 || m.fee24h == 0) return type(uint256).max;

        // คำนวณ IL amount
        int256 ilBps = this.calculateCurrentIL(pool, currentPrice0);
        if (ilBps >= 0) return 0; // No IL

        uint256 ilAmount = uint256(-ilBps) * lpAmount / 10000;

        // Daily fee income จาก LP
        uint256 lpShare = lpAmount * PRECISION / m.tvl;
        uint256 dailyFeeIncome = m.fee24h * lpShare / PRECISION;

        if (dailyFeeIncome == 0) return type(uint256).max;

        days_ = ilAmount / dailyFeeIncome;
    }

    /**
     * @dev Summary ของทุก pools
     */
    function getPortfolioSummary() external view returns (
        uint256 totalTVL,
        uint256 totalVolume24h,
        uint256 totalFee24h,
        uint256 poolCount
    ) {
        poolCount = pools.length;
        for (uint256 i = 0; i < poolCount; i++) {
            PoolMetrics storage m = poolMetrics[pools[i]];
            totalTVL += m.tvl;
            totalVolume24h += m.volume24h;
            totalFee24h += m.fee24h;
        }
    }

    function _sqrt(uint256 x) internal pure returns (uint256 y) {
        if (x == 0) return 0;
        uint256 z = (x + 1) / 2;
        y = x;
        while (z < y) {
            y = z;
            z = (x / z + z) / 2;
        }
    }
}
```

---

## สรุป Part 78

- **Game Theory ใน DeFi**: Nash Equilibria อธิบาย incentive alignment ใน liquidity provision และ governance; Schelling Points ใช้ใน oracle systems เพื่อ converge ไปที่ราคา "จริง"
- **LP Token Fair Value**: Uniswap V2 ใช้สูตร `2*sqrt(reserve0*reserve1)*sqrt(price0*price1)/totalSupply`; Curve ใช้ `virtual_price = D/totalSupply` ซึ่ง monotonically increasing และ manipulation-resistant
- **Impermanent Loss**: สูตร `IL = 2*sqrt(p)/(1+p) - 1` คำนวณ IL จาก price ratio; การ hedge ด้วย options (straddle) ช่วย offset IL แต่มี premium cost
- **Emission Schedule**: 4-year halving สร้าง predictable supply schedule; lockup boost (เช่น veCRV) จูงใจ long-term alignment และลด sell pressure
- **Revenue Sustainability**: protocol ที่ยั่งยืนต้องมี revenue ≥ incentives; Real Yield จาก fee revenue ดีกว่า inflationary rewards

## Next: Part 79 - Advanced Cross-Chain Messaging
