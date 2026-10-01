# Part 39: Flash Loans Advanced

## สารบัญ
1. Flash Loan Architecture
2. ERC-3156 Standard
3. Flash Loan Arbitrage
4. Flash Loan Attacks
5. Workshop: Multi-Protocol Flash Loan

---

## 1. Flash Loan Architecture

```
Flash Loan คืออะไร:
- กู้ tokens ในจำนวนมาก
- ใช้ภายใน transaction เดียว
- คืน + fee ก่อน transaction จบ
- ถ้าไม่คืน → transaction revert ทั้งหมด

Use Cases ที่ถูกต้อง:
1. Arbitrage: ซื้อถูก ขายแพง ระหว่าง DEXs
2. Collateral Swap: เปลี่ยน collateral โดยไม่ต้องมีเงิน
3. Liquidation: liquidate positions โดยไม่ต้องมีทุน
4. Self-liquidation: ปิด position ตัวเอง

Protocols ที่ให้ Flash Loans:
- Aave: fee 0.09%
- dYdX: fee 0 (แต่ต้อง hold margin)
- Uniswap V3: flash swap (ใน pool)
- Balancer: fee 0.0001%

Formula:
borrowed + fee ≤ balance_after
fee = borrowed × 0.0009 (for Aave)
```

---

## 2. ERC-3156 Flash Loan Standard

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * ERC-3156 Flash Loan Standard Interfaces
 */
interface IERC3156FlashBorrower {
    function onFlashLoan(
        address initiator,
        address token,
        uint256 amount,
        uint256 fee,
        bytes calldata data
    ) external returns (bytes32);
}

interface IERC3156FlashLender {
    function maxFlashLoan(address token) external view returns (uint256);
    function flashFee(address token, uint256 amount) external view returns (uint256);
    function flashLoan(
        IERC3156FlashBorrower receiver,
        address token,
        uint256 amount,
        bytes calldata data
    ) external returns (bool);
}

/**
 * Flash Loan Lender Implementation
 */
contract FlashLoanProvider is IERC3156FlashLender {
    
    IERC20 public immutable supportedToken;
    uint256 public constant FEE_RATE = 9; // 0.09% = 9 / 10000
    uint256 public constant FEE_BASIS = 10000;
    
    bytes32 constant CALLBACK_SUCCESS = keccak256("ERC3156FlashBorrower.onFlashLoan");
    
    event FlashLoan(address indexed receiver, address indexed token, uint256 amount, uint256 fee);
    
    error UnsupportedToken(address token);
    error CallbackFailed();
    error RepaymentFailed();
    
    constructor(address _token) {
        supportedToken = IERC20(_token);
    }
    
    function maxFlashLoan(address token) external view override returns (uint256) {
        if (token != address(supportedToken)) return 0;
        return supportedToken.balanceOf(address(this));
    }
    
    function flashFee(address token, uint256 amount) external view override returns (uint256) {
        if (token != address(supportedToken)) revert UnsupportedToken(token);
        return (amount * FEE_RATE) / FEE_BASIS;
    }
    
    function flashLoan(
        IERC3156FlashBorrower receiver,
        address token,
        uint256 amount,
        bytes calldata data
    ) external override returns (bool) {
        if (token != address(supportedToken)) revert UnsupportedToken(token);
        
        uint256 fee = (amount * FEE_RATE) / FEE_BASIS;
        uint256 balanceBefore = supportedToken.balanceOf(address(this));
        
        // Transfer tokens to borrower
        supportedToken.transfer(address(receiver), amount);
        
        // Call borrower callback
        bytes32 result = receiver.onFlashLoan(msg.sender, token, amount, fee, data);
        if (result != CALLBACK_SUCCESS) revert CallbackFailed();
        
        // Verify repayment
        uint256 balanceAfter = supportedToken.balanceOf(address(this));
        if (balanceAfter < balanceBefore + fee) revert RepaymentFailed();
        
        emit FlashLoan(address(receiver), token, amount, fee);
        return true;
    }
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
    function balanceOf(address) external view returns (uint256);
}
```

---

## 3. Flash Loan Arbitrage

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Flash Loan Arbitrage:
 * 1. Borrow USDC from flash loan
 * 2. Buy ETH cheap on DEX A (lower price)
 * 3. Sell ETH expensive on DEX B (higher price)
 * 4. Repay USDC + fee
 * 5. Keep profit
 */
contract FlashArbBot is IERC3156FlashBorrower {
    
    IERC3156FlashLender public immutable lender;
    address public immutable owner;
    
    bytes32 constant CALLBACK_SUCCESS = keccak256("ERC3156FlashBorrower.onFlashLoan");
    
    event ArbExecuted(
        address indexed token,
        uint256 borrowed,
        uint256 profit
    );
    
    constructor(address _lender) {
        lender = IERC3156FlashLender(_lender);
        owner = msg.sender;
    }
    
    // Entry point: start the arbitrage
    function executeArb(
        address token,
        uint256 amount,
        address dexA,
        address dexB,
        address targetToken,
        uint256 minProfit
    ) external {
        require(msg.sender == owner, "Not owner");
        
        bytes memory data = abi.encode(dexA, dexB, targetToken, minProfit);
        lender.flashLoan(this, token, amount, data);
    }
    
    // Flash loan callback
    function onFlashLoan(
        address initiator,
        address token,
        uint256 amount,
        uint256 fee,
        bytes calldata data
    ) external override returns (bytes32) {
        require(msg.sender == address(lender), "Not lender");
        require(initiator == address(this), "Not self");
        
        (address dexA, address dexB, address targetToken, uint256 minProfit) = 
            abi.decode(data, (address, address, address, uint256));
        
        uint256 balanceBefore = IERC20(token).balanceOf(address(this));
        
        // Step 1: Buy targetToken with 'token' on dexA (cheaper)
        IERC20(token).approve(dexA, amount);
        uint256 targetAmount = _swap(dexA, token, targetToken, amount);
        
        // Step 2: Sell targetToken for 'token' on dexB (more expensive)
        IERC20(targetToken).approve(dexB, targetAmount);
        uint256 received = _swap(dexB, targetToken, token, targetAmount);
        
        // Calculate profit
        uint256 repayAmount = amount + fee;
        require(received >= repayAmount, "Arbitrage failed: not profitable");
        
        uint256 profit = received - repayAmount;
        require(profit >= minProfit, "Profit too low");
        
        // Approve repayment
        IERC20(token).approve(address(lender), repayAmount);
        
        emit ArbExecuted(token, amount, profit);
        
        return CALLBACK_SUCCESS;
    }
    
    function _swap(
        address dex,
        address tokenIn,
        address tokenOut,
        uint256 amountIn
    ) internal returns (uint256 amountOut) {
        // Generic DEX swap
        (bool success, bytes memory result) = dex.call(
            abi.encodeWithSignature(
                "swap(address,address,uint256,uint256,address)",
                tokenIn, tokenOut, amountIn, 0, address(this)
            )
        );
        require(success, "Swap failed");
        amountOut = abi.decode(result, (uint256));
    }
    
    // Withdraw profits
    function withdraw(address token) external {
        require(msg.sender == owner);
        uint256 balance = IERC20(token).balanceOf(address(this));
        IERC20(token).transfer(owner, balance);
    }
}

interface IERC3156FlashBorrower {
    function onFlashLoan(address, address, uint256, uint256, bytes calldata) external returns (bytes32);
}

interface IERC3156FlashLender {
    function flashLoan(IERC3156FlashBorrower, address, uint256, bytes calldata) external returns (bool);
}
```

---

## 4. Flash Loan Attack Defense

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Flash Loan Attack Patterns & Defenses
 * 
 * Common attacks:
 * 1. Price Oracle Manipulation
 *    - Flash loan large amount → move spot price → exploit
 *    - Defense: TWAP oracle ไม่ใช่ spot price
 * 
 * 2. Governance Attack
 *    - Flash loan → buy votes → pass malicious proposal → repay
 *    - Defense: snapshot votes ก่อน proposal, lock period
 * 
 * 3. Reentrancy with Flash Loans
 *    - Flash loan + reentrancy ใน callback
 *    - Defense: CEI + reentrancy guard
 */

/**
 * Vulnerable AMM (for educational purposes)
 */
contract VulnerableAMM {
    
    IERC20 public tokenA;
    IERC20 public tokenB;
    
    uint256 public reserveA;
    uint256 public reserveB;
    
    // ❌ VULNERABLE: uses current spot price
    function getPrice() public view returns (uint256) {
        return reserveB * 1e18 / reserveA;
    }
    
    // Attacker can:
    // 1. Flash loan huge amount of A
    // 2. Sell A → moves reserveA up, reserveB down → price drops
    // 3. Buy at artificially low price in another protocol
    // 4. Sell back → price recovers
    // 5. Repay flash loan + profit
}

/**
 * Protected AMM with TWAP
 */
contract ProtectedAMM {
    
    IERC20 public tokenA;
    IERC20 public tokenB;
    
    uint256 public reserveA;
    uint256 public reserveB;
    
    // TWAP accumulators (Uniswap V2 style)
    uint256 public price0CumulativeLast;
    uint256 public price1CumulativeLast;
    uint32 public blockTimestampLast;
    
    // TWAP observations
    uint256[] public observations;
    uint32[] public observationTimestamps;
    
    function _updateTWAP() internal {
        uint32 blockTimestamp = uint32(block.timestamp % 2**32);
        uint32 timeElapsed = blockTimestamp - blockTimestampLast;
        
        if (timeElapsed > 0 && reserveA > 0 && reserveB > 0) {
            // Accumulate price × time
            price0CumulativeLast += (reserveB * 1e18 / reserveA) * timeElapsed;
            price1CumulativeLast += (reserveA * 1e18 / reserveB) * timeElapsed;
        }
        
        blockTimestampLast = blockTimestamp;
    }
    
    // ✅ SAFE: TWAP over 30 minutes
    function getTWAP(uint256 windowSize) public view returns (uint256) {
        // In production: use stored accumulator snapshots
        // Return average price over windowSize seconds
        // Manipulating this would require sustained large trades
        return price0CumulativeLast / block.timestamp;
    }
    
    // ✅ Protected oracle-based liquidation
    function liquidateWithTWAP(address user) external {
        uint256 price = getTWAP(30 minutes);
        // Use TWAP price, not spot price
        // Even if spot price is manipulated, TWAP is stable
    }
}

/**
 * Flash-Loan-Resistant Governance
 */
contract ResistantGovernance {
    
    IERC20Votes public immutable token;
    
    uint48 public votingDelay = 1 days; // 1 day delay before voting starts
    
    struct Proposal {
        uint256 id;
        uint256 snapshotBlock; // votes snapshot at this block
        uint256 voteStart;
        uint256 voteEnd;
        mapping(address => bool) hasVoted;
        uint256 forVotes;
        uint256 againstVotes;
    }
    
    mapping(uint256 => Proposal) public proposals;
    uint256 public proposalCount;
    
    function propose(address[] calldata targets) external returns (uint256 id) {
        id = proposalCount++;
        
        Proposal storage p = proposals[id];
        p.id = id;
        // SNAPSHOT at current block (BEFORE voting starts)
        p.snapshotBlock = block.number;
        p.voteStart = block.timestamp + votingDelay;
        p.voteEnd = p.voteStart + 5 days;
    }
    
    function castVote(uint256 proposalId, bool support) external {
        Proposal storage p = proposals[proposalId];
        require(block.timestamp >= p.voteStart, "Voting not started");
        require(block.timestamp <= p.voteEnd, "Voting ended");
        require(!p.hasVoted[msg.sender], "Already voted");
        
        // Use PAST votes at snapshot block!
        // Flash loan บน block ปัจจุบัน ไม่ช่วย!
        uint256 weight = token.getPastVotes(msg.sender, p.snapshotBlock);
        
        p.hasVoted[msg.sender] = true;
        
        if (support) {
            p.forVotes += weight;
        } else {
            p.againstVotes += weight;
        }
    }
}

interface IERC20Votes {
    function getPastVotes(address account, uint256 blockNumber) external view returns (uint256);
}
```

---

## 5. Workshop: Multi-Protocol Flash Loan

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Multi-Protocol Flash Loan:
 * รวม Flash Loans จากหลาย sources
 * Use case: Debt migration (ย้าย debt จาก Compound ไป Aave)
 */
contract DebtMigrator is IERC3156FlashBorrower {
    
    // Protocol interfaces
    ILendingProtocol public immutable fromProtocol; // e.g., Compound
    ILendingProtocol public immutable toProtocol;   // e.g., Aave
    IERC3156FlashLender public immutable flashLender;
    
    address public immutable owner;
    
    bytes32 constant CALLBACK_SUCCESS = keccak256("ERC3156FlashBorrower.onFlashLoan");
    
    constructor(address _from, address _to, address _flashLender) {
        fromProtocol = ILendingProtocol(_from);
        toProtocol = ILendingProtocol(_to);
        flashLender = IERC3156FlashLender(_flashLender);
        owner = msg.sender;
    }
    
    /**
     * Migrate debt from fromProtocol to toProtocol
     * 
     * Flow:
     * 1. Flash loan USDC (equal to debt amount)
     * 2. Repay Compound debt with flash loan
     * 3. Withdraw collateral from Compound
     * 4. Deposit collateral into Aave
     * 5. Borrow USDC from Aave at better rate
     * 6. Repay flash loan
     */
    function migrateLoan(
        address debtToken,
        uint256 debtAmount,
        address collateralToken
    ) external {
        require(msg.sender == owner, "Not owner");
        
        bytes memory data = abi.encode(
            debtToken,
            debtAmount,
            collateralToken,
            msg.sender
        );
        
        flashLender.flashLoan(this, debtToken, debtAmount, data);
    }
    
    function onFlashLoan(
        address initiator,
        address token,
        uint256 amount,
        uint256 fee,
        bytes calldata data
    ) external override returns (bytes32) {
        require(msg.sender == address(flashLender));
        require(initiator == address(this));
        
        (address debtToken, uint256 debtAmount, address collateralToken, address user) =
            abi.decode(data, (address, uint256, address, address));
        
        // Step 2: Repay old protocol debt
        IERC20(debtToken).approve(address(fromProtocol), debtAmount);
        fromProtocol.repay(debtToken, debtAmount, user);
        
        // Step 3: Withdraw collateral from old protocol
        uint256 collateralAmount = fromProtocol.getCollateral(collateralToken, user);
        fromProtocol.withdrawCollateral(collateralToken, collateralAmount, user);
        
        // Step 4: Deposit collateral into new protocol
        IERC20(collateralToken).approve(address(toProtocol), collateralAmount);
        toProtocol.depositCollateral(collateralToken, collateralAmount, user);
        
        // Step 5: Borrow from new protocol to repay flash loan
        uint256 repayAmount = amount + fee;
        toProtocol.borrow(debtToken, repayAmount, user);
        
        // Approve flash loan repayment
        IERC20(debtToken).approve(address(flashLender), repayAmount);
        
        return CALLBACK_SUCCESS;
    }
}

interface ILendingProtocol {
    function repay(address token, uint256 amount, address onBehalfOf) external;
    function withdrawCollateral(address token, uint256 amount, address to) external;
    function depositCollateral(address token, uint256 amount, address onBehalfOf) external;
    function borrow(address token, uint256 amount, address onBehalfOf) external;
    function getCollateral(address token, address user) external view returns (uint256);
}
```

---

## สรุป Part 39

Flash Loans Advanced ที่เรียนรู้:
- ✅ ERC-3156 standard (lender + borrower interfaces)
- ✅ Flash loan arbitrage implementation
- ✅ Flash loan attack patterns (price oracle, governance)
- ✅ Defense mechanisms (TWAP, vote snapshot)
- ✅ Debt migration use case

## Quiz

1. Flash loan ปลอดภัยสำหรับ lender อย่างไร?
2. Spot price vs TWAP ต่างกันอย่างไรในแง่ security?
3. Governance snapshot ป้องกัน flash loan attack อย่างไร?
4. Debt migration ช่วยประหยัดเงินอย่างไร?

---

## Next: Part 40 - MEV และ Maximal Extractable Value
