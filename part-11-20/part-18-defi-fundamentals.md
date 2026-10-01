# Part 18: DeFi Fundamentals

## สารบัญ
1. DeFi คืออะไร
2. AMM (Automated Market Maker)
3. Liquidity Pools
4. Yield Farming / Liquidity Mining
5. Flash Loans
6. Stablecoins
7. Workshop: Mini DEX

---

## 1. DeFi คืออะไร

```
DeFi = Decentralized Finance
     = บริการทางการเงินบน blockchain โดยไม่มี intermediary

Traditional Finance:        DeFi:
Bank → holds funds          Smart Contract → holds funds
Exchange → matches orders   AMM → algorithmic pricing
Broker → manages assets     Vault → automated strategies
Loan → requires KYC         Flash Loan → no collateral needed

Key Protocols (TVL ใหญ่สุด):
- Uniswap: DEX (Decentralized Exchange)
- Aave: Lending/Borrowing
- Compound: Lending/Borrowing
- Curve: Stablecoin DEX
- MakerDAO: Stablecoin (DAI)
- Lido: Liquid Staking
```

---

## 2. AMM (Automated Market Maker)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Constant Product AMM (Uniswap V2 style)
 * x * y = k
 * 
 * ตัวอย่าง: Pool มี 100 ETH และ 100,000 USDC
 * k = 100 * 100,000 = 10,000,000
 * 
 * ถ้าซื้อ ETH ด้วย 1000 USDC:
 * ETH ใหม่ = k / (USDC ใหม่) = 10,000,000 / 101,000 ≈ 99.009 ETH
 * ได้ ETH = 100 - 99.009 = 0.991 ETH (ราคา ~1009 USDC/ETH)
 */
contract UniswapV2Pair {
    
    using Math for uint256;
    
    address public token0;
    address public token1;
    
    uint112 private reserve0;
    uint112 private reserve1;
    uint32 private blockTimestampLast;
    
    // LP Token (อยู่ใน contract นี้เลย)
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    
    uint256 public constant MINIMUM_LIQUIDITY = 1000; // burn on first mint
    
    event Mint(address indexed sender, uint256 amount0, uint256 amount1);
    event Burn(address indexed sender, uint256 amount0, uint256 amount1, address indexed to);
    event Swap(
        address indexed sender,
        uint256 amount0In, uint256 amount1In,
        uint256 amount0Out, uint256 amount1Out,
        address indexed to
    );
    event Sync(uint112 reserve0, uint112 reserve1);
    
    error InsufficientLiquidity();
    error InsufficientInputAmount();
    error InsufficientOutputAmount();
    error InvalidK();
    
    uint256 private unlocked = 1;
    modifier lock() {
        require(unlocked == 1, "Locked");
        unlocked = 2;
        _;
        unlocked = 1;
    }
    
    constructor(address _token0, address _token1) {
        token0 = _token0;
        token1 = _token1;
    }
    
    function getReserves() public view returns (uint112 _reserve0, uint112 _reserve1, uint32 _blockTimestampLast) {
        _reserve0 = reserve0;
        _reserve1 = reserve1;
        _blockTimestampLast = blockTimestampLast;
    }
    
    // Add Liquidity
    function mint(address to) external lock returns (uint256 liquidity) {
        (uint112 _reserve0, uint112 _reserve1,) = getReserves();
        
        uint256 balance0 = IERC20(token0).balanceOf(address(this));
        uint256 balance1 = IERC20(token1).balanceOf(address(this));
        
        uint256 amount0 = balance0 - _reserve0;
        uint256 amount1 = balance1 - _reserve1;
        
        uint256 _totalSupply = totalSupply;
        
        if (_totalSupply == 0) {
            // First liquidity: sqrt(amount0 * amount1) - MINIMUM_LIQUIDITY
            liquidity = Math.sqrt(amount0 * amount1) - MINIMUM_LIQUIDITY;
            _mint(address(0), MINIMUM_LIQUIDITY); // burn forever (prevents price manipulation)
        } else {
            // Subsequent: min(amount0 / reserve0, amount1 / reserve1) * totalSupply
            liquidity = Math.min(
                (amount0 * _totalSupply) / _reserve0,
                (amount1 * _totalSupply) / _reserve1
            );
        }
        
        require(liquidity > 0, "Insufficient liquidity minted");
        _mint(to, liquidity);
        
        _update(balance0, balance1);
        
        emit Mint(msg.sender, amount0, amount1);
    }
    
    // Remove Liquidity
    function burn(address to) external lock returns (uint256 amount0, uint256 amount1) {
        uint256 balance0 = IERC20(token0).balanceOf(address(this));
        uint256 balance1 = IERC20(token1).balanceOf(address(this));
        
        uint256 liquidity = balanceOf[address(this)];
        
        uint256 _totalSupply = totalSupply;
        amount0 = (liquidity * balance0) / _totalSupply;
        amount1 = (liquidity * balance1) / _totalSupply;
        
        require(amount0 > 0 && amount1 > 0, "Insufficient liquidity burned");
        
        _burn(address(this), liquidity);
        
        IERC20(token0).transfer(to, amount0);
        IERC20(token1).transfer(to, amount1);
        
        balance0 = IERC20(token0).balanceOf(address(this));
        balance1 = IERC20(token1).balanceOf(address(this));
        
        _update(balance0, balance1);
        
        emit Burn(msg.sender, amount0, amount1, to);
    }
    
    // Swap
    function swap(uint256 amount0Out, uint256 amount1Out, address to, bytes calldata data) external lock {
        require(amount0Out > 0 || amount1Out > 0, "Insufficient output");
        
        (uint112 _reserve0, uint112 _reserve1,) = getReserves();
        
        require(amount0Out < _reserve0 && amount1Out < _reserve1, "Insufficient liquidity");
        
        if (amount0Out > 0) IERC20(token0).transfer(to, amount0Out);
        if (amount1Out > 0) IERC20(token1).transfer(to, amount1Out);
        
        // Flash swap callback
        if (data.length > 0) IUniswapV2Callee(to).uniswapV2Call(msg.sender, amount0Out, amount1Out, data);
        
        uint256 balance0 = IERC20(token0).balanceOf(address(this));
        uint256 balance1 = IERC20(token1).balanceOf(address(this));
        
        uint256 amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
        uint256 amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
        
        require(amount0In > 0 || amount1In > 0, "Insufficient input");
        
        // Verify k: (balance0 - 0.3% fee) * (balance1 - 0.3% fee) >= k
        uint256 balance0Adjusted = (balance0 * 1000) - (amount0In * 3);
        uint256 balance1Adjusted = (balance1 * 1000) - (amount1In * 3);
        
        require(
            balance0Adjusted * balance1Adjusted >= uint256(_reserve0) * uint256(_reserve1) * 1000**2,
            "K violated"
        );
        
        _update(balance0, balance1);
        
        emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
    }
    
    function _update(uint256 balance0, uint256 balance1) private {
        reserve0 = uint112(balance0);
        reserve1 = uint112(balance1);
        blockTimestampLast = uint32(block.timestamp);
        emit Sync(reserve0, reserve1);
    }
    
    function _mint(address to, uint256 amount) internal {
        totalSupply += amount;
        balanceOf[to] += amount;
    }
    
    function _burn(address from, uint256 amount) internal {
        balanceOf[from] -= amount;
        totalSupply -= amount;
    }
}

interface IUniswapV2Callee {
    function uniswapV2Call(address sender, uint256 amount0, uint256 amount1, bytes calldata data) external;
}

interface IERC20 {
    function balanceOf(address) external view returns (uint256);
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
}

library Math {
    function min(uint256 a, uint256 b) internal pure returns (uint256) { return a < b ? a : b; }
    function sqrt(uint256 y) internal pure returns (uint256 z) {
        if (y > 3) {
            z = y;
            uint256 x = y / 2 + 1;
            while (x < z) { z = x; x = (y / x + x) / 2; }
        } else if (y != 0) { z = 1; }
    }
}
```

---

## 3. Flash Loans

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Flash Loan: กู้เงินใน 1 transaction โดยไม่ต้องมี collateral
 * ต้องคืนภายใน same transaction หรือ revert ทั้งหมด
 * 
 * Use cases:
 * - Arbitrage
 * - Liquidations
 * - Collateral swaps
 * - Self-liquidation
 */

interface IFlashLoanReceiver {
    function executeOperation(
        address asset,
        uint256 amount,
        uint256 premium,
        address initiator,
        bytes calldata params
    ) external returns (bool);
}

contract FlashLoanProvider {
    
    mapping(address => uint256) public liquidity;
    uint256 public constant FLASH_LOAN_FEE = 9; // 0.09% = 9/10000
    
    event FlashLoan(address indexed receiver, address indexed asset, uint256 amount, uint256 premium);
    
    error InsufficientLiquidity();
    error FlashLoanFailed();
    error NotRepaid();
    
    function deposit(address asset, uint256 amount) external {
        IERC20(asset).transferFrom(msg.sender, address(this), amount);
        liquidity[asset] += amount;
    }
    
    function flashLoan(
        address receiverAddress,
        address asset,
        uint256 amount,
        bytes calldata params
    ) external {
        uint256 available = liquidity[asset];
        if (available < amount) revert InsufficientLiquidity();
        
        uint256 balanceBefore = IERC20(asset).balanceOf(address(this));
        uint256 premium = (amount * FLASH_LOAN_FEE) / 10000;
        
        // Transfer funds to receiver
        IERC20(asset).transfer(receiverAddress, amount);
        
        // Callback
        bool success = IFlashLoanReceiver(receiverAddress).executeOperation(
            asset, amount, premium, msg.sender, params
        );
        if (!success) revert FlashLoanFailed();
        
        // Verify repayment
        uint256 balanceAfter = IERC20(asset).balanceOf(address(this));
        if (balanceAfter < balanceBefore + premium) revert NotRepaid();
        
        liquidity[asset] = balanceAfter;
        
        emit FlashLoan(receiverAddress, asset, amount, premium);
    }
}

// Arbitrage using Flash Loan
contract FlashArbitrage is IFlashLoanReceiver {
    
    FlashLoanProvider public lender;
    address public owner;
    
    constructor(address _lender) {
        lender = FlashLoanProvider(_lender);
        owner = msg.sender;
    }
    
    function executeArbitrage(
        address asset,
        uint256 amount,
        address dexA,
        address dexB
    ) external {
        require(msg.sender == owner, "Not owner");
        
        bytes memory params = abi.encode(dexA, dexB);
        lender.flashLoan(address(this), asset, amount, params);
    }
    
    function executeOperation(
        address asset,
        uint256 amount,
        uint256 premium,
        address,
        bytes calldata params
    ) external override returns (bool) {
        require(msg.sender == address(lender), "Not lender");
        
        (address dexA, address dexB) = abi.decode(params, (address, address));
        
        // 1. Buy cheap on DEX A
        uint256 received = _swapOnDex(dexA, asset, amount);
        
        // 2. Sell expensive on DEX B
        uint256 profit = _swapOnDex(dexB, asset, received);
        
        // 3. Repay loan + fee
        uint256 repayAmount = amount + premium;
        require(profit >= repayAmount, "Arbitrage not profitable");
        
        IERC20(asset).transfer(address(lender), repayAmount);
        
        // 4. Keep profit
        uint256 netProfit = profit - repayAmount;
        if (netProfit > 0) {
            IERC20(asset).transfer(owner, netProfit);
        }
        
        return true;
    }
    
    function _swapOnDex(address dex, address asset, uint256 amountIn) internal returns (uint256) {
        // Simplified swap
        IERC20(asset).approve(dex, amountIn);
        // ... actual swap logic
        return amountIn; // placeholder
    }
}
```

---

## 4. Workshop: Mini DEX

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title MiniDEX
 * @dev Full-featured mini DEX with:
 * - Add/Remove Liquidity
 * - Token swaps (0.3% fee)
 * - LP tokens
 * - Price calculation
 */
contract MiniDEX {
    
    IERC20 public immutable tokenA;
    IERC20 public immutable tokenB;
    
    uint256 public reserveA;
    uint256 public reserveB;
    
    // LP Token state
    uint256 public totalLPSupply;
    mapping(address => uint256) public lpBalance;
    
    uint256 private constant FEE_NUMERATOR = 997;   // 0.3% fee
    uint256 private constant FEE_DENOMINATOR = 1000;
    uint256 private constant MINIMUM_LIQUIDITY = 1000;
    
    event LiquidityAdded(address indexed provider, uint256 amountA, uint256 amountB, uint256 lpMinted);
    event LiquidityRemoved(address indexed provider, uint256 amountA, uint256 amountB, uint256 lpBurned);
    event Swapped(address indexed user, address tokenIn, uint256 amountIn, uint256 amountOut);
    
    error ZeroAmount();
    error InsufficientLiquidity();
    error SlippageExceeded(uint256 actual, uint256 minimum);
    error InsufficientLPTokens();
    
    constructor(address _tokenA, address _tokenB) {
        tokenA = IERC20(_tokenA);
        tokenB = IERC20(_tokenB);
    }
    
    // === Add Liquidity ===
    
    function addLiquidity(
        uint256 amountA,
        uint256 amountB,
        uint256 minAmountA,
        uint256 minAmountB
    ) external returns (uint256 lpMinted) {
        if (amountA == 0 || amountB == 0) revert ZeroAmount();
        
        uint256 actualAmountA = amountA;
        uint256 actualAmountB = amountB;
        
        if (reserveA > 0 && reserveB > 0) {
            // Adjust to maintain ratio
            uint256 amountBOptimal = (amountA * reserveB) / reserveA;
            
            if (amountBOptimal <= amountB) {
                actualAmountB = amountBOptimal;
            } else {
                uint256 amountAOptimal = (amountB * reserveA) / reserveB;
                actualAmountA = amountAOptimal;
            }
        }
        
        if (actualAmountA < minAmountA) revert SlippageExceeded(actualAmountA, minAmountA);
        if (actualAmountB < minAmountB) revert SlippageExceeded(actualAmountB, minAmountB);
        
        // Transfer tokens
        tokenA.transferFrom(msg.sender, address(this), actualAmountA);
        tokenB.transferFrom(msg.sender, address(this), actualAmountB);
        
        // Mint LP tokens
        if (totalLPSupply == 0) {
            lpMinted = _sqrt(actualAmountA * actualAmountB) - MINIMUM_LIQUIDITY;
            lpBalance[address(0)] += MINIMUM_LIQUIDITY; // burn
        } else {
            lpMinted = _min(
                (actualAmountA * totalLPSupply) / reserveA,
                (actualAmountB * totalLPSupply) / reserveB
            );
        }
        
        require(lpMinted > 0, "Zero LP");
        
        lpBalance[msg.sender] += lpMinted;
        totalLPSupply += lpMinted;
        
        reserveA += actualAmountA;
        reserveB += actualAmountB;
        
        emit LiquidityAdded(msg.sender, actualAmountA, actualAmountB, lpMinted);
    }
    
    // === Remove Liquidity ===
    
    function removeLiquidity(
        uint256 lpAmount,
        uint256 minAmountA,
        uint256 minAmountB
    ) external returns (uint256 amountA, uint256 amountB) {
        if (lpAmount == 0) revert ZeroAmount();
        if (lpBalance[msg.sender] < lpAmount) revert InsufficientLPTokens();
        
        amountA = (lpAmount * reserveA) / totalLPSupply;
        amountB = (lpAmount * reserveB) / totalLPSupply;
        
        if (amountA < minAmountA) revert SlippageExceeded(amountA, minAmountA);
        if (amountB < minAmountB) revert SlippageExceeded(amountB, minAmountB);
        
        // Burn LP tokens
        lpBalance[msg.sender] -= lpAmount;
        totalLPSupply -= lpAmount;
        
        reserveA -= amountA;
        reserveB -= amountB;
        
        tokenA.transfer(msg.sender, amountA);
        tokenB.transfer(msg.sender, amountB);
        
        emit LiquidityRemoved(msg.sender, amountA, amountB, lpAmount);
    }
    
    // === Swap ===
    
    function swapAtoB(uint256 amountIn, uint256 minAmountOut) external returns (uint256 amountOut) {
        if (amountIn == 0) revert ZeroAmount();
        if (reserveA == 0 || reserveB == 0) revert InsufficientLiquidity();
        
        amountOut = getAmountOut(amountIn, reserveA, reserveB);
        if (amountOut < minAmountOut) revert SlippageExceeded(amountOut, minAmountOut);
        
        tokenA.transferFrom(msg.sender, address(this), amountIn);
        tokenB.transfer(msg.sender, amountOut);
        
        reserveA += amountIn;
        reserveB -= amountOut;
        
        emit Swapped(msg.sender, address(tokenA), amountIn, amountOut);
    }
    
    function swapBtoA(uint256 amountIn, uint256 minAmountOut) external returns (uint256 amountOut) {
        if (amountIn == 0) revert ZeroAmount();
        if (reserveA == 0 || reserveB == 0) revert InsufficientLiquidity();
        
        amountOut = getAmountOut(amountIn, reserveB, reserveA);
        if (amountOut < minAmountOut) revert SlippageExceeded(amountOut, minAmountOut);
        
        tokenB.transferFrom(msg.sender, address(this), amountIn);
        tokenA.transfer(msg.sender, amountOut);
        
        reserveB += amountIn;
        reserveA -= amountOut;
        
        emit Swapped(msg.sender, address(tokenB), amountIn, amountOut);
    }
    
    // === Price Calculations ===
    
    // x * y = k formula with fee
    function getAmountOut(
        uint256 amountIn,
        uint256 reserveIn,
        uint256 reserveOut
    ) public pure returns (uint256 amountOut) {
        require(amountIn > 0, "Zero input");
        require(reserveIn > 0 && reserveOut > 0, "Zero reserves");
        
        uint256 amountInWithFee = amountIn * FEE_NUMERATOR;
        uint256 numerator = amountInWithFee * reserveOut;
        uint256 denominator = (reserveIn * FEE_DENOMINATOR) + amountInWithFee;
        amountOut = numerator / denominator;
    }
    
    function getAmountIn(
        uint256 amountOut,
        uint256 reserveIn,
        uint256 reserveOut
    ) public pure returns (uint256 amountIn) {
        require(amountOut > 0, "Zero output");
        require(reserveIn > 0 && reserveOut > 0, "Zero reserves");
        require(amountOut < reserveOut, "Insufficient liquidity");
        
        uint256 numerator = reserveIn * amountOut * FEE_DENOMINATOR;
        uint256 denominator = (reserveOut - amountOut) * FEE_NUMERATOR;
        amountIn = (numerator / denominator) + 1;
    }
    
    function getPriceAtoB() external view returns (uint256) {
        if (reserveA == 0) return 0;
        return (reserveB * 1e18) / reserveA;
    }
    
    function getPriceBtoA() external view returns (uint256) {
        if (reserveB == 0) return 0;
        return (reserveA * 1e18) / reserveB;
    }
    
    function _sqrt(uint256 y) internal pure returns (uint256 z) {
        if (y > 3) {
            z = y;
            uint256 x = y / 2 + 1;
            while (x < z) { z = x; x = (y / x + x) / 2; }
        } else if (y != 0) { z = 1; }
    }
    
    function _min(uint256 a, uint256 b) internal pure returns (uint256) {
        return a < b ? a : b;
    }
}
```

---

## สรุป Part 18

DeFi Fundamentals ที่เรียนรู้:
- ✅ AMM (x * y = k formula)
- ✅ Liquidity pools
- ✅ LP tokens
- ✅ Flash loans
- ✅ Mini DEX with slippage protection
- ✅ Price calculations

## Quiz

1. ทำไม AMM ถึงต้องมี `MINIMUM_LIQUIDITY` ที่ burn ทิ้ง?
2. Impermanent Loss คืออะไร?
3. Flash Loan ใช้ collateral ไหม? ทำงานยังไง?
4. ทำไม Slippage Protection ถึงสำคัญ?

---

## Next: Part 19 - Lending Protocols
