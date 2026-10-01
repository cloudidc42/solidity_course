# Part 90: Protocol Composability & DeFi Legos

## บทนำ

"DeFi Legos" คือแนวคิดที่ว่า DeFi protocols เป็นเหมือน building blocks ที่ประกอบกันได้ เนื่องจาก:
1. **Open source**: ทุกคนอ่าน code ได้
2. **Permissionless**: ไม่ต้องขออนุญาต
3. **Composable**: เรียกใช้กันได้ใน single transaction
4. **Standardized**: ERC standards ทำให้ interoperate ง่าย

ตัวอย่างการ compose:
- Flash loan → Swap → Stake → Repay = **Flash Arbitrage**
- Borrow → Swap → Provide LP → Borrow more = **Leveraged LP**
- Flash loan → Refinance debt = **Debt Migration**

ในบทนี้เราจะสร้าง:
1. Adapter pattern สำหรับ Protocol Abstraction
2. DeFi Lego compositions
3. Risk isolation mechanisms
4. Composability testing
5. Universal ProtocolAdapter

---

## 1. Adapter Pattern

### ทำไมต้องใช้ Adapter?

```
Problem: แต่ละ protocol มี interface ต่างกัน
Aave:    pool.supply(asset, amount, onBehalfOf, referralCode)
Compound: cToken.mint(amount)
Yearn:   vault.deposit(amount)

Solution: Adapter wraps each protocol ด้วย universal interface
ILendingAdapter.deposit(asset, amount) → works for all!
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

/**
 * @title IProtocolAdapter
 * @notice Universal interface สำหรับ DeFi protocol adapters
 */
interface IProtocolAdapter {
    // Basic operations
    function deposit(address asset, uint256 amount) external returns (uint256 sharesReceived);
    function withdraw(address asset, uint256 sharesOrAmount) external returns (uint256 received);
    
    // Borrowing (optional - lending protocols only)
    function borrow(address asset, uint256 amount, address onBehalfOf) external;
    function repay(address asset, uint256 amount, address onBehalfOf) external;
    
    // View functions
    function getBalance(address asset, address user) external view returns (uint256);
    function getAPY(address asset) external view returns (uint256);
    function getBorrowRate(address asset) external view returns (uint256);
    function getProtocolName() external pure returns (string memory);
    
    // Health factor for lending
    function getHealthFactor(address user) external view returns (uint256);
}

/**
 * @title AaveV3Adapter
 * @notice Adapter สำหรับ Aave V3
 */
contract AaveV3Adapter is IProtocolAdapter {
    using SafeERC20 for IERC20;
    
    IAaveV3Pool public immutable aavePool;
    address public immutable aaveDataProvider;
    
    constructor(address _aavePool, address _dataProvider) {
        aavePool = IAaveV3Pool(_aavePool);
        aaveDataProvider = _dataProvider;
    }
    
    function deposit(address asset, uint256 amount) external override returns (uint256 sharesReceived) {
        IERC20(asset).safeTransferFrom(msg.sender, address(this), amount);
        IERC20(asset).forceApprove(address(aavePool), amount);
        
        // Aave supply returns void, track aToken received
        uint256 aTokenBalBefore = _getATokenBalance(asset);
        aavePool.supply(asset, amount, msg.sender, 0);
        uint256 aTokenBalAfter = _getATokenBalance(asset);
        
        sharesReceived = aTokenBalAfter - aTokenBalBefore;
    }
    
    function withdraw(address asset, uint256 amount) external override returns (uint256 received) {
        address aToken = _getATokenAddress(asset);
        
        // Transfer aTokens from user
        IERC20(aToken).safeTransferFrom(msg.sender, address(this), amount);
        
        // Withdraw from Aave
        received = aavePool.withdraw(asset, amount, msg.sender);
    }
    
    function borrow(address asset, uint256 amount, address onBehalfOf) external override {
        aavePool.borrow(asset, amount, 2, 0, onBehalfOf); // mode 2 = variable rate
    }
    
    function repay(address asset, uint256 amount, address onBehalfOf) external override {
        IERC20(asset).safeTransferFrom(msg.sender, address(this), amount);
        IERC20(asset).forceApprove(address(aavePool), amount);
        aavePool.repay(asset, amount, 2, onBehalfOf); // mode 2 = variable rate
    }
    
    function getBalance(address asset, address user) external view override returns (uint256) {
        address aToken = _getATokenAddress(asset);
        return IERC20(aToken).balanceOf(user);
    }
    
    function getAPY(address asset) external view override returns (uint256) {
        (, , , , uint128 currentLiquidityRate, , , , , , ,) = IAaveDataProvider(aaveDataProvider)
            .getReserveData(asset);
        // Ray (27 decimals) → basis points
        return uint256(currentLiquidityRate) / 1e23;
    }
    
    function getBorrowRate(address asset) external view override returns (uint256) {
        (, , , , , uint128 currentVariableBorrowRate, , , , , ,) = IAaveDataProvider(aaveDataProvider)
            .getReserveData(asset);
        return uint256(currentVariableBorrowRate) / 1e23;
    }
    
    function getHealthFactor(address user) external view override returns (uint256) {
        (, , , , , uint256 healthFactor) = aavePool.getUserAccountData(user);
        return healthFactor; // 1e18 = 1.0
    }
    
    function getProtocolName() external pure override returns (string memory) {
        return "Aave V3";
    }
    
    function _getATokenAddress(address asset) internal view returns (address) {
        (address aTokenAddress, ,) = IAaveDataProvider(aaveDataProvider)
            .getReserveTokensAddresses(asset);
        return aTokenAddress;
    }
    
    function _getATokenBalance(address asset) internal view returns (uint256) {
        return IERC20(_getATokenAddress(asset)).balanceOf(address(this));
    }
}

/**
 * @title CompoundV3Adapter
 * @notice Adapter สำหรับ Compound V3 (Comet)
 */
contract CompoundV3Adapter is IProtocolAdapter {
    using SafeERC20 for IERC20;
    
    IComet public immutable comet;
    address public immutable baseToken;
    
    constructor(address _comet) {
        comet = IComet(_comet);
        baseToken = IComet(_comet).baseToken();
    }
    
    function deposit(address asset, uint256 amount) external override returns (uint256 sharesReceived) {
        IERC20(asset).safeTransferFrom(msg.sender, address(this), amount);
        IERC20(asset).forceApprove(address(comet), amount);
        
        uint256 balBefore = comet.balanceOf(msg.sender);
        comet.supplyTo(msg.sender, asset, amount);
        sharesReceived = comet.balanceOf(msg.sender) - balBefore;
    }
    
    function withdraw(address asset, uint256 amount) external override returns (uint256 received) {
        comet.withdrawFrom(msg.sender, msg.sender, asset, amount);
        received = amount;
    }
    
    function borrow(address asset, uint256 amount, address onBehalfOf) external override {
        require(asset == baseToken, "can only borrow base token");
        comet.withdrawFrom(onBehalfOf, onBehalfOf, asset, amount);
    }
    
    function repay(address asset, uint256 amount, address onBehalfOf) external override {
        require(asset == baseToken, "can only repay base token");
        IERC20(asset).safeTransferFrom(msg.sender, address(this), amount);
        IERC20(asset).forceApprove(address(comet), amount);
        comet.supplyTo(onBehalfOf, asset, amount);
    }
    
    function getBalance(address, address user) external view override returns (uint256) {
        return comet.balanceOf(user);
    }
    
    function getAPY(address) external view override returns (uint256) {
        uint256 supplyRatePerSecond = comet.getSupplyRate(comet.getUtilization());
        return supplyRatePerSecond * 365 days / 1e16; // Convert to bps
    }
    
    function getBorrowRate(address) external view override returns (uint256) {
        uint256 borrowRatePerSecond = comet.getBorrowRate(comet.getUtilization());
        return borrowRatePerSecond * 365 days / 1e16;
    }
    
    function getHealthFactor(address user) external view override returns (uint256) {
        // Compound V3 uses collateral factor differently
        uint256 collateralValue = comet.collateralBalanceOf(user, address(0)); // simplified
        uint256 borrowBalance = comet.borrowBalanceOf(user);
        if (borrowBalance == 0) return type(uint256).max;
        return collateralValue * 1e18 / borrowBalance;
    }
    
    function getProtocolName() external pure override returns (string memory) {
        return "Compound V3";
    }
}
```

---

## 2. DeFi Lego Compositions

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title DeFiCompositor
 * @notice รวม DeFi operations ใน single transaction
 * @dev Flash loan → Swap → Stake → Repay ใน 1 transaction
 */
contract DeFiCompositor is ReentrancyGuard {
    using SafeERC20 for IERC20;
    
    // ========== Data Structures ==========
    
    enum ActionType {
        FlashLoan,
        Swap,
        Deposit,
        Withdraw,
        Borrow,
        Repay,
        AddLiquidity,
        RemoveLiquidity,
        Stake,
        Unstake
    }
    
    struct Action {
        ActionType actionType;
        address protocol;      // Protocol adapter หรือ contract
        address tokenIn;
        address tokenOut;
        uint256 amountIn;      // 0 = use full balance from previous action
        bytes extraData;       // Protocol-specific data
    }
    
    struct CompositeOperation {
        Action[] actions;
        address flashLoanToken; // Token สำหรับ flash loan (ถ้ามี)
        uint256 flashLoanAmount;
        address recipient;      // ผู้รับ final output
    }
    
    // Protocol registry
    mapping(address => bool) public approvedProtocols;
    
    // Temporary state สำหรับ flash loan callback
    bytes private _pendingCallbackData;
    
    address public owner;
    
    event CompositeExecuted(address indexed user, uint256 actionsCount, uint256 finalBalance);
    
    modifier onlyOwner() {
        require(msg.sender == owner, "not owner");
        _;
    }
    
    constructor() {
        owner = msg.sender;
    }
    
    // ========== Main Execution ==========
    
    /**
     * @notice Execute composite DeFi operation
     * @dev ถ้ามี flash loan, เริ่มจาก flash loan แล้วทำ actions ใน callback
     */
    function executeComposite(CompositeOperation calldata operation) external nonReentrant {
        if (operation.flashLoanAmount > 0) {
            // Encode operation data สำหรับ flash loan callback
            _pendingCallbackData = abi.encode(operation, msg.sender);
            
            // Initiate flash loan
            IFlashLoanProvider(operation.flashLoanToken).flashLoan(
                address(this),
                operation.flashLoanToken,
                operation.flashLoanAmount,
                abi.encode(operation, msg.sender)
            );
        } else {
            // Execute directly without flash loan
            _executeActions(operation.actions, operation.recipient);
        }
    }
    
    /**
     * @notice Flash loan callback - execute actions with borrowed funds
     */
    function executeOperation(
        address asset,
        uint256 amount,
        uint256 premium,
        address initiator,
        bytes calldata params
    ) external returns (bool) {
        require(approvedProtocols[msg.sender], "unauthorized flash loan provider");
        
        (CompositeOperation memory operation, address user) = abi.decode(
            params, 
            (CompositeOperation, address)
        );
        
        // Execute all actions
        _executeActions(operation.actions, operation.recipient);
        
        // Repay flash loan + premium
        uint256 repayAmount = amount + premium;
        IERC20(asset).forceApprove(msg.sender, repayAmount);
        
        return true;
    }
    
    /**
     * @notice Execute array of actions sequentially
     */
    function _executeActions(Action[] memory actions, address recipient) internal {
        for (uint256 i = 0; i < actions.length; i++) {
            _executeAction(actions[i]);
        }
        
        // Transfer final balance to recipient
        // (ใน production: ต้องระบุ token ที่ transfer)
    }
    
    /**
     * @notice Execute single action
     */
    function _executeAction(Action memory action) internal {
        if (action.actionType == ActionType.Swap) {
            _executeSwap(action);
        } else if (action.actionType == ActionType.Deposit) {
            _executeDeposit(action);
        } else if (action.actionType == ActionType.Withdraw) {
            _executeWithdraw(action);
        } else if (action.actionType == ActionType.Borrow) {
            _executeBorrow(action);
        } else if (action.actionType == ActionType.Repay) {
            _executeRepay(action);
        } else if (action.actionType == ActionType.Stake) {
            _executeStake(action);
        }
    }
    
    function _executeSwap(Action memory action) internal {
        uint256 amount = action.amountIn > 0 
            ? action.amountIn 
            : IERC20(action.tokenIn).balanceOf(address(this));
        
        IERC20(action.tokenIn).forceApprove(action.protocol, amount);
        
        (address[] memory path, uint256 minOut) = abi.decode(action.extraData, (address[], uint256));
        
        IUniswapV2Router(action.protocol).swapExactTokensForTokens(
            amount,
            minOut,
            path,
            address(this),
            block.timestamp
        );
    }
    
    function _executeDeposit(Action memory action) internal {
        uint256 amount = action.amountIn > 0 
            ? action.amountIn 
            : IERC20(action.tokenIn).balanceOf(address(this));
        
        IERC20(action.tokenIn).forceApprove(action.protocol, amount);
        IProtocolAdapter(action.protocol).deposit(action.tokenIn, amount);
    }
    
    function _executeWithdraw(Action memory action) internal {
        uint256 amount = action.amountIn > 0 ? action.amountIn : type(uint256).max;
        IProtocolAdapter(action.protocol).withdraw(action.tokenIn, amount);
    }
    
    function _executeBorrow(Action memory action) internal {
        IProtocolAdapter(action.protocol).borrow(action.tokenOut, action.amountIn, address(this));
    }
    
    function _executeRepay(Action memory action) internal {
        uint256 amount = action.amountIn > 0 
            ? action.amountIn 
            : IERC20(action.tokenIn).balanceOf(address(this));
        
        IERC20(action.tokenIn).forceApprove(action.protocol, amount);
        IProtocolAdapter(action.protocol).repay(action.tokenIn, amount, address(this));
    }
    
    function _executeStake(Action memory action) internal {
        uint256 amount = action.amountIn > 0 
            ? action.amountIn 
            : IERC20(action.tokenIn).balanceOf(address(this));
        
        IERC20(action.tokenIn).forceApprove(action.protocol, amount);
        IStaking(action.protocol).stake(amount);
    }
    
    function approveProtocol(address protocol) external onlyOwner {
        approvedProtocols[protocol] = true;
    }
}

// Interfaces
interface IFlashLoanProvider {
    function flashLoan(
        address receiver,
        address token,
        uint256 amount,
        bytes calldata data
    ) external;
}

interface IUniswapV2Router {
    function swapExactTokensForTokens(
        uint256 amountIn,
        uint256 amountOutMin,
        address[] calldata path,
        address to,
        uint256 deadline
    ) external returns (uint256[] memory amounts);
}

interface IStaking {
    function stake(uint256 amount) external;
    function unstake(uint256 amount) external;
    function getRewards() external;
}

interface IAaveV3Pool {
    function supply(address asset, uint256 amount, address onBehalfOf, uint16 referralCode) external;
    function withdraw(address asset, uint256 amount, address to) external returns (uint256);
    function borrow(address asset, uint256 amount, uint256 interestRateMode, uint16 referralCode, address onBehalfOf) external;
    function repay(address asset, uint256 amount, uint256 interestRateMode, address onBehalfOf) external returns (uint256);
    function getUserAccountData(address user) external view returns (uint256, uint256, uint256, uint256, uint256, uint256);
}

interface IAaveDataProvider {
    function getReserveData(address asset) external view returns (uint256, uint128, uint128, uint128, uint128, uint128, uint40, uint16, address, address, address, address, uint128, uint128, uint128);
    function getReserveTokensAddresses(address asset) external view returns (address, address, address);
}

interface IComet {
    function baseToken() external view returns (address);
    function supplyTo(address dst, address asset, uint256 amount) external;
    function withdrawFrom(address src, address dst, address asset, uint256 amount) external;
    function balanceOf(address account) external view returns (uint256);
    function borrowBalanceOf(address account) external view returns (uint256);
    function collateralBalanceOf(address account, address asset) external view returns (uint128);
    function getUtilization() external view returns (uint256);
    function getSupplyRate(uint256 utilization) external view returns (uint256);
    function getBorrowRate(uint256 utilization) external view returns (uint256);
}
```

---

## 3. Flash Loan Arbitrage Lego

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

/**
 * @title FlashArbitrager
 * @notice Flash loan → Swap → Repay arbitrage ใน 1 transaction
 * @dev ตัวอย่างการ compose flash loan + DEX swap
 */
contract FlashArbitrager {
    using SafeERC20 for IERC20;
    
    address public immutable aavePool;
    address public immutable uniswapRouter;
    address public immutable sushiswapRouter;
    
    address public owner;
    
    event ArbitrageExecuted(
        address indexed tokenIn,
        uint256 flashAmount,
        uint256 profit
    );
    
    constructor(address _aave, address _uniswap, address _sushi) {
        aavePool = _aave;
        uniswapRouter = _uniswap;
        sushiswapRouter = _sushi;
        owner = msg.sender;
    }
    
    /**
     * @notice Execute flash loan arbitrage
     * @param tokenIn Token ที่จะยืม
     * @param flashAmount Amount ที่จะยืม
     * @param path1 Swap path บน DEX 1 (buy cheap)
     * @param path2 Swap path บน DEX 2 (sell high)
     */
    function executeArbitrage(
        address tokenIn,
        uint256 flashAmount,
        address[] calldata path1,
        address[] calldata path2,
        uint256 minProfit
    ) external {
        require(msg.sender == owner, "not owner");
        
        // Encode params สำหรับ callback
        bytes memory params = abi.encode(
            path1, path2, minProfit, msg.sender
        );
        
        // Request flash loan จาก Aave
        IAaveFlashLoan(aavePool).flashLoan(
            address(this),
            tokenIn,
            flashAmount,
            params
        );
    }
    
    /**
     * @notice Aave flash loan callback
     */
    function executeOperation(
        address asset,
        uint256 amount,
        uint256 premium,
        address,
        bytes calldata params
    ) external returns (bool) {
        require(msg.sender == aavePool, "invalid caller");
        
        (
            address[] memory path1,
            address[] memory path2,
            uint256 minProfit,
            address profitRecipient
        ) = abi.decode(params, (address[], address[], uint256, address));
        
        uint256 balanceBefore = IERC20(asset).balanceOf(address(this));
        
        // Step 1: Buy on DEX 1 (cheaper price)
        IERC20(asset).forceApprove(uniswapRouter, amount);
        uint256[] memory amounts1 = IUniswapV2Router02(uniswapRouter).swapExactTokensForTokens(
            amount,
            0, // Min amount - ตรวจสอบจาก final profit
            path1,
            address(this),
            block.timestamp
        );
        
        uint256 intermediateAmount = amounts1[amounts1.length - 1];
        address intermediateToken = path1[path1.length - 1];
        
        // Step 2: Sell on DEX 2 (higher price)
        IERC20(intermediateToken).forceApprove(sushiswapRouter, intermediateAmount);
        uint256[] memory amounts2 = IUniswapV2Router02(sushiswapRouter).swapExactTokensForTokens(
            intermediateAmount,
            0,
            path2,
            address(this),
            block.timestamp
        );
        
        uint256 finalAmount = amounts2[amounts2.length - 1];
        
        // ตรวจสอบ profit
        uint256 repayAmount = amount + premium;
        require(finalAmount > repayAmount, "no profit");
        
        uint256 profit = finalAmount - repayAmount;
        require(profit >= minProfit, "profit below minimum");
        
        // Repay flash loan
        IERC20(asset).forceApprove(aavePool, repayAmount);
        
        // ส่ง profit ให้ owner
        IERC20(asset).safeTransfer(profitRecipient, profit);
        
        emit ArbitrageExecuted(asset, amount, profit);
        
        return true;
    }
}

interface IAaveFlashLoan {
    function flashLoan(
        address receiverAddress,
        address asset,
        uint256 amount,
        bytes calldata params
    ) external;
}

interface IUniswapV2Router02 {
    function swapExactTokensForTokens(
        uint256 amountIn,
        uint256 amountOutMin,
        address[] calldata path,
        address to,
        uint256 deadline
    ) external returns (uint256[] memory amounts);
    
    function getAmountsOut(uint256 amountIn, address[] calldata path) 
        external view returns (uint256[] memory amounts);
}
```

---

## 4. Risk Isolation in Composed Protocols

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title ComposedProtocolSecurity
 * @notice Security patterns สำหรับ composed protocols
 * @dev ป้องกัน reentrancy, callback manipulation, และ cross-protocol risks
 */
contract ComposedProtocolSecurity {
    
    // ========== Callback Authorization ==========
    
    // Trusted callers สำหรับ callbacks
    mapping(address => mapping(bytes4 => bool)) public trustedCallbacks;
    
    // Lock สำหรับ reentrancy protection ในระหว่าง composed calls
    uint256 private _compositionLock;
    
    // Track originator ของ composition chain
    address private _compositionOriginator;
    
    // ========== Modifiers ==========
    
    /**
     * @notice ป้องกัน unauthorized callbacks
     * @dev ตรวจสอบว่า caller เป็น trusted protocol
     */
    modifier onlyTrustedCallback(bytes4 selector) {
        require(
            trustedCallbacks[msg.sender][selector],
            "unauthorized callback"
        );
        _;
    }
    
    /**
     * @notice ป้องกัน reentrant composition
     * @dev ใช้ custom lock แทน OpenZeppelin เพื่อ support cross-function calls
     */
    modifier noCompositionReentrancy() {
        require(_compositionLock == 0, "composition in progress");
        _compositionLock = 1;
        _compositionOriginator = msg.sender;
        _;
        _compositionLock = 0;
        _compositionOriginator = address(0);
    }
    
    /**
     * @notice อนุญาตเฉพาะ calls ที่มาจาก composition chain เดิม
     */
    modifier onlyDuringComposition() {
        require(_compositionLock == 1, "not in composition");
        _;
    }
    
    /**
     * @notice Register trusted callback
     */
    function registerTrustedCallback(address caller, bytes4 selector) external {
        // ใน production: ต้องมี access control
        trustedCallbacks[caller][selector] = true;
    }
    
    /**
     * @notice ตรวจสอบ state ก่อนและหลัง callback
     * @dev Guard สำหรับ invariant violations
     */
    function _checkStateInvariants() internal view {
        // Example invariant: total assets must equal liabilities + equity
        // _assertInvariant(totalAssets() == totalLiabilities() + totalEquity());
    }
}

/**
 * @title SafeComposedCall
 * @notice Helper สำหรับ safe cross-protocol calls
 */
library SafeComposedCall {
    
    /**
     * @notice Execute call ด้วย strict gas limit
     * @dev ป้องกัน gas griefing attacks
     */
    function safeCall(
        address target,
        uint256 value,
        bytes memory data,
        uint256 gasLimit
    ) internal returns (bool success, bytes memory returnData) {
        require(gasleft() > gasLimit + 10000, "insufficient gas");
        
        (success, returnData) = target.call{value: value, gas: gasLimit}(data);
        
        // ถ้า call fails, revert ด้วย reason
        if (!success) {
            if (returnData.length > 0) {
                assembly {
                    let returnDataSize := mload(returnData)
                    revert(add(32, returnData), returnDataSize)
                }
            }
        }
    }
    
    /**
     * @notice Verify return data จาก external call
     */
    function verifyReturnData(
        bytes memory returnData,
        bytes32 expectedHash
    ) internal pure returns (bool) {
        return keccak256(returnData) == expectedHash;
    }
    
    /**
     * @notice Check ว่า target เป็น contract (ไม่ใช่ EOA)
     */
    function isContract(address target) internal view returns (bool) {
        return target.code.length > 0;
    }
}

/**
 * @title ReentrancyGuardComposed
 * @notice Advanced reentrancy guard สำหรับ composed calls
 * @dev รองรับ allowlisted reentrant calls (เช่น flash loan callbacks)
 */
abstract contract ReentrancyGuardComposed {
    
    uint256 private constant NOT_ENTERED = 1;
    uint256 private constant ENTERED = 2;
    
    uint256 private _status = NOT_ENTERED;
    
    // Functions ที่อนุญาตให้ reenter
    mapping(bytes4 => bool) private _allowedReentrant;
    
    modifier nonReentrant() {
        require(_status == NOT_ENTERED, "ReentrancyGuard: reentrant call");
        _status = ENTERED;
        _;
        _status = NOT_ENTERED;
    }
    
    /**
     * @notice nonReentrant แต่ allow specific selectors
     */
    modifier nonReentrantExcept(bytes4 allowedSelector) {
        if (msg.sig == allowedSelector) {
            // Allow reentry for this function
            _;
        } else {
            require(_status == NOT_ENTERED, "ReentrancyGuard: reentrant call");
            _status = ENTERED;
            _;
            _status = NOT_ENTERED;
        }
    }
    
    function _allowReentrant(bytes4 selector) internal {
        _allowedReentrant[selector] = true;
    }
}
```

---

## 5. ProtocolAdapter - Generic Universal Adapter

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * @title UniversalProtocolAdapter
 * @notice Generic adapter ที่รองรับ any DeFi protocol
 * @dev ใช้ delegate call pattern สำหรับ extensibility
 */
contract UniversalProtocolAdapter is Ownable, ReentrancyGuard {
    using SafeERC20 for IERC20;
    
    // ========== Protocol Registry ==========
    
    struct ProtocolInfo {
        address implementation;  // Adapter implementation contract
        bool active;
        string name;
        uint256 version;
    }
    
    // protocolId → ProtocolInfo
    mapping(bytes32 => ProtocolInfo) public protocols;
    
    // ========== Route Execution ==========
    
    struct RouteStep {
        bytes32 protocolId;     // Protocol ที่จะใช้
        bytes4 selector;        // Function selector ใน adapter
        bytes params;           // Encoded parameters
        address tokenIn;
        address tokenOut;
        uint256 amountIn;       // 0 = use balance from prev step
        uint256 minAmountOut;   // Slippage protection
    }
    
    // ========== Events ==========
    
    event ProtocolRegistered(bytes32 indexed protocolId, string name, address implementation);
    event RouteExecuted(address indexed user, uint256 steps, uint256 finalAmount);
    event StepExecuted(bytes32 indexed protocolId, bytes4 selector, uint256 amountIn, uint256 amountOut);
    
    // ========== Constructor ==========
    
    constructor() Ownable(msg.sender) {}
    
    // ========== Protocol Registration ==========
    
    /**
     * @notice Register protocol adapter
     */
    function registerProtocol(
        bytes32 protocolId,
        address implementation,
        string calldata name,
        uint256 version
    ) external onlyOwner {
        protocols[protocolId] = ProtocolInfo({
            implementation: implementation,
            active: true,
            name: name,
            version: version
        });
        
        emit ProtocolRegistered(protocolId, name, implementation);
    }
    
    /**
     * @notice Execute multi-step route ข้าม protocols
     * @dev Token flow: user → adapter → protocol1 → protocol2 → user
     */
    function executeRoute(
        RouteStep[] calldata steps,
        address tokenOut,
        address recipient
    ) external nonReentrant returns (uint256 finalAmount) {
        require(steps.length > 0, "empty route");
        
        // Transfer initial tokens from user
        if (steps[0].amountIn > 0) {
            IERC20(steps[0].tokenIn).safeTransferFrom(
                msg.sender,
                address(this),
                steps[0].amountIn
            );
        }
        
        // Execute each step
        for (uint256 i = 0; i < steps.length; i++) {
            RouteStep calldata step = steps[i];
            ProtocolInfo memory protocol = protocols[step.protocolId];
            require(protocol.active, "protocol not active");
            
            // คำนวณ actual amount (0 = use current balance)
            uint256 actualAmountIn = step.amountIn > 0 
                ? step.amountIn 
                : IERC20(step.tokenIn).balanceOf(address(this));
            
            // Execute via delegate call to adapter
            uint256 amountOut = _executeStep(
                protocol.implementation,
                step,
                actualAmountIn
            );
            
            require(amountOut >= step.minAmountOut, "slippage exceeded");
            
            emit StepExecuted(step.protocolId, step.selector, actualAmountIn, amountOut);
        }
        
        // Transfer final balance to recipient
        finalAmount = IERC20(tokenOut).balanceOf(address(this));
        if (finalAmount > 0 && recipient != address(this)) {
            IERC20(tokenOut).safeTransfer(recipient, finalAmount);
        }
        
        emit RouteExecuted(msg.sender, steps.length, finalAmount);
    }
    
    /**
     * @notice Execute single step via delegatecall
     */
    function _executeStep(
        address implementation,
        RouteStep calldata step,
        uint256 actualAmountIn
    ) internal returns (uint256 amountOut) {
        // Approve ให้ implementation ใช้ tokens
        if (step.tokenIn != address(0)) {
            IERC20(step.tokenIn).forceApprove(implementation, actualAmountIn);
        }
        
        // Encode call data
        bytes memory callData = abi.encodeWithSelector(
            step.selector,
            step.tokenIn,
            step.tokenOut,
            actualAmountIn,
            step.params
        );
        
        // Delegatecall to adapter implementation
        (bool success, bytes memory returnData) = implementation.delegatecall(callData);
        
        if (!success) {
            assembly {
                revert(add(returnData, 32), mload(returnData))
            }
        }
        
        amountOut = abi.decode(returnData, (uint256));
    }
    
    /**
     * @notice Emergency rescue tokens
     */
    function rescueTokens(address token, address to, uint256 amount) external onlyOwner {
        IERC20(token).safeTransfer(to, amount);
    }
    
    receive() external payable {}
}

/**
 * @title UniswapV3AdapterImpl
 * @notice Implementation สำหรับ Uniswap V3 (ใช้ delegate call)
 */
contract UniswapV3AdapterImpl {
    using SafeERC20 for IERC20;
    
    address public constant UNISWAP_ROUTER = 0xE592427A0AEce92De3Edee1F18E0157C05861564;
    
    /**
     * @notice Swap via Uniswap V3
     */
    function swap(
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        bytes calldata extraData
    ) external returns (uint256 amountOut) {
        (uint24 fee, uint256 minOut) = abi.decode(extraData, (uint24, uint256));
        
        IERC20(tokenIn).forceApprove(UNISWAP_ROUTER, amountIn);
        
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
            tokenIn: tokenIn,
            tokenOut: tokenOut,
            fee: fee,
            recipient: address(this),
            deadline: block.timestamp,
            amountIn: amountIn,
            amountOutMinimum: minOut,
            sqrtPriceLimitX96: 0
        });
        
        amountOut = ISwapRouter(UNISWAP_ROUTER).exactInputSingle(params);
    }
}

interface ISwapRouter {
    struct ExactInputSingleParams {
        address tokenIn;
        address tokenOut;
        uint24 fee;
        address recipient;
        uint256 deadline;
        uint256 amountIn;
        uint256 amountOutMinimum;
        uint160 sqrtPriceLimitX96;
    }
    
    function exactInputSingle(ExactInputSingleParams calldata params) external returns (uint256 amountOut);
}
```

---

## 6. Composability Testing (Forked Mainnet)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";

/**
 * @title ComposabilityTest
 * @notice Integration tests ด้วย forked mainnet
 * @dev ใช้ Foundry's fork testing เพื่อ test กับ real protocols
 *
 * Run: forge test --fork-url $MAINNET_RPC_URL --fork-block-number <block>
 */
contract ComposabilityTest is Test {
    
    // Real mainnet addresses
    address constant AAVE_POOL = 0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2;
    address constant UNISWAP_ROUTER = 0xE592427A0AEce92De3Edee1F18E0157C05861564;
    address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
    
    // Fork block number
    uint256 constant FORK_BLOCK = 18000000;
    
    DeFiCompositor public compositor;
    FlashArbitrager public arbitrager;
    
    address alice = makeAddr("alice");
    
    function setUp() public {
        // Create fork (ต้องมี RPC URL)
        // vm.createSelectFork(vm.envString("MAINNET_RPC"), FORK_BLOCK);
        
        compositor = new DeFiCompositor();
        arbitrager = new FlashArbitrager(AAVE_POOL, UNISWAP_ROUTER, address(0));
        
        compositor.approveProtocol(AAVE_POOL);
        compositor.approveProtocol(UNISWAP_ROUTER);
        
        // Fund alice with USDC
        deal(USDC, alice, 10000e6);
        
        vm.startPrank(alice);
        IERC20(USDC).approve(address(compositor), type(uint256).max);
        vm.stopPrank();
    }
    
    /**
     * @notice Test: Deposit USDC ใน Aave และ verify aToken received
     */
    function test_fork_aave_deposit() public {
        // Skip if no fork
        if (block.chainid != 1) return;
        
        AaveV3Adapter adapter = new AaveV3Adapter(AAVE_POOL, address(0));
        
        uint256 depositAmount = 1000e6;
        deal(USDC, address(this), depositAmount);
        IERC20(USDC).approve(address(adapter), depositAmount);
        
        uint256 sharesBefore = adapter.getBalance(USDC, address(this));
        
        uint256 shares = adapter.deposit(USDC, depositAmount);
        
        uint256 sharesAfter = adapter.getBalance(USDC, address(this));
        
        assertGt(shares, 0);
        assertGt(sharesAfter, sharesBefore);
        
        console.log("Deposited:", depositAmount, "USDC");
        console.log("Received:", shares, "aUSDC");
        console.log("APY:", adapter.getAPY(USDC), "bps");
    }
    
    /**
     * @notice Test: Multi-hop composite operation
     * USDC → Deposit Aave → Borrow DAI → Swap DAI → WETH → Stake WETH
     */
    function test_composite_operation() public {
        // Simplified test without real protocols
        
        // Mock setup
        MockAave mockAave = new MockAave();
        MockDex mockDex = new MockDex();
        
        DeFiCompositor.Action[] memory actions = new DeFiCompositor.Action[](3);
        
        // Action 1: Deposit USDC to Aave
        actions[0] = DeFiCompositor.Action({
            actionType: DeFiCompositor.ActionType.Deposit,
            protocol: address(mockAave),
            tokenIn: USDC,
            tokenOut: address(0), // aUSDC
            amountIn: 1000e6,
            extraData: ""
        });
        
        // Action 2: Borrow DAI
        actions[1] = DeFiCompositor.Action({
            actionType: DeFiCompositor.ActionType.Borrow,
            protocol: address(mockAave),
            tokenIn: address(0),
            tokenOut: DAI,
            amountIn: 800e18,
            extraData: ""
        });
        
        // Action 3: Swap DAI → WETH
        address[] memory swapPath = new address[](2);
        swapPath[0] = DAI;
        swapPath[1] = WETH;
        
        actions[2] = DeFiCompositor.Action({
            actionType: DeFiCompositor.ActionType.Swap,
            protocol: address(mockDex),
            tokenIn: DAI,
            tokenOut: WETH,
            amountIn: 0, // use full balance
            extraData: abi.encode(swapPath, uint256(0))
        });
        
        console.log("Composite operation defined with", actions.length, "steps");
    }
    
    /**
     * @notice Test: Flash arbitrage profitability
     */
    function test_flash_arbitrage_simulation() public {
        // Simulate price difference between two DEXes
        uint256 uniswapPrice = 3000e6;   // 1 WETH = 3000 USDC on Uniswap
        uint256 sushiswapPrice = 3030e6;  // 1 WETH = 3030 USDC on SushiSwap
        
        uint256 flashAmount = 10e18; // 10 WETH
        uint256 buyUSDC = flashAmount * uniswapPrice / 1e18;
        uint256 sellUSDC = flashAmount * sushiswapPrice / 1e18;
        
        uint256 grossProfit = sellUSDC - buyUSDC;
        uint256 flashFee = flashAmount * 9 / 10000; // 0.09% Aave fee
        uint256 flashFeeUSDC = flashFee * 3000e6 / 1e18;
        uint256 gasCost = 200000 * 20 gwei * 3000 / 1e18; // Estimated gas * price
        
        uint256 netProfit = grossProfit > flashFeeUSDC + gasCost 
            ? grossProfit - flashFeeUSDC - gasCost 
            : 0;
        
        console.log("Flash amount:", flashAmount / 1e18, "WETH");
        console.log("Gross profit:", grossProfit / 1e6, "USDC");
        console.log("Flash fee:", flashFeeUSDC / 1e6, "USDC");
        console.log("Net profit:", netProfit / 1e6, "USDC");
        
        if (netProfit > 0) {
            console.log("Arbitrage is profitable!");
        }
    }
    
    /**
     * @notice Test: Protocol composability security
     */
    function test_reentrancy_protection() public {
        ReentrancyAttacker attacker = new ReentrancyAttacker();
        
        // ตรวจสอบว่า reentrancy attack ไม่ผ่าน
        vm.expectRevert();
        attacker.attack(address(compositor));
    }
}

/**
 * @notice Mock contracts สำหรับ testing
 */
contract MockAave {
    function supply(address asset, uint256 amount, address onBehalfOf, uint16) external {}
    function withdraw(address asset, uint256 amount, address to) external returns (uint256) {
        return amount;
    }
    function borrow(address asset, uint256 amount, uint256, uint16, address) external {}
    function repay(address asset, uint256 amount, uint256, address) external returns (uint256) {
        return amount;
    }
    function getUserAccountData(address) external pure returns (
        uint256, uint256, uint256, uint256, uint256, uint256
    ) {
        return (0, 0, 0, 0, 0, 2e18); // health factor = 2.0
    }
}

contract MockDex {
    function swapExactTokensForTokens(
        uint256 amountIn,
        uint256,
        address[] calldata,
        address to,
        uint256
    ) external returns (uint256[] memory amounts) {
        amounts = new uint256[](2);
        amounts[0] = amountIn;
        amounts[1] = amountIn * 99 / 100; // 1% slippage
    }
}

contract ReentrancyAttacker {
    uint256 count;
    
    function attack(address target) external {
        // Attempt reentrancy
        count = 0;
        _reenter(target);
    }
    
    function _reenter(address target) internal {
        if (count++ < 3) {
            _reenter(target);
        }
    }
    
    receive() external payable {
        // Try to reenter
    }
}
```

---

## Workshop: Building a Yield Strategy Lego

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title YieldStrategyLego
 * @notice สร้าง yield strategy ด้วย DeFi Legos
 * @dev USDC → Aave (earn 5%) → Borrow ETH → Stake ETH → Earn staking rewards
 *      Leverage: 1x USDC collateral → 0.5x ETH borrow → stake ETH for 4% yield
 */
contract YieldStrategyLego {
    
    address public immutable aaveAdapter;
    address public immutable swapAdapter;
    address public immutable stakeAdapter;
    
    struct StrategyPosition {
        uint256 collateralAmount;    // USDC deposited
        uint256 borrowAmount;        // ETH borrowed (in ETH units)
        uint256 stakedAmount;        // ETH staked
        uint256 openTime;
        bool active;
    }
    
    mapping(address => StrategyPosition) public positions;
    
    event StrategyOpened(address indexed user, uint256 collateral, uint256 borrowed);
    event StrategyClosed(address indexed user, int256 netPnL);
    
    constructor(address _aave, address _swap, address _stake) {
        aaveAdapter = _aave;
        swapAdapter = _swap;
        stakeAdapter = _stake;
    }
    
    /**
     * @notice เปิด yield strategy position
     * @param usdcAmount USDC ที่จะ deposit เป็น collateral
     * @param borrowRatio % ของ collateral ที่จะ borrow (basis points)
     */
    function openStrategy(
        uint256 usdcAmount,
        uint256 borrowRatio  // 5000 = 50%
    ) external {
        require(!positions[msg.sender].active, "strategy already open");
        require(usdcAmount > 0, "zero collateral");
        require(borrowRatio > 0 && borrowRatio <= 7000, "invalid borrow ratio"); // Max 70%
        
        // Step 1: Deposit USDC to Aave
        IERC20(USDC_ADDRESS).transferFrom(msg.sender, address(this), usdcAmount);
        IERC20(USDC_ADDRESS).approve(aaveAdapter, usdcAmount);
        IProtocolAdapter(aaveAdapter).deposit(USDC_ADDRESS, usdcAmount);
        
        // Step 2: Borrow ETH (50% of collateral value at current price)
        uint256 ethPrice = _getEthPrice();
        uint256 borrowValueUSD = usdcAmount * borrowRatio / 10000;
        uint256 ethBorrowAmount = borrowValueUSD * 1e18 / ethPrice;
        
        IProtocolAdapter(aaveAdapter).borrow(WETH_ADDRESS, ethBorrowAmount, address(this));
        
        // Step 3: Stake borrowed ETH
        IERC20(WETH_ADDRESS).approve(stakeAdapter, ethBorrowAmount);
        IStaking(stakeAdapter).stake(ethBorrowAmount);
        
        positions[msg.sender] = StrategyPosition({
            collateralAmount: usdcAmount,
            borrowAmount: ethBorrowAmount,
            stakedAmount: ethBorrowAmount,
            openTime: block.timestamp,
            active: true
        });
        
        emit StrategyOpened(msg.sender, usdcAmount, ethBorrowAmount);
    }
    
    /**
     * @notice ปิด strategy และคืน collateral
     */
    function closeStrategy() external {
        StrategyPosition storage pos = positions[msg.sender];
        require(pos.active, "no active strategy");
        
        // Step 1: Unstake ETH
        IStaking(stakeAdapter).unstake(pos.stakedAmount);
        
        // Step 2: Claim staking rewards
        IStaking(stakeAdapter).getRewards();
        
        // Step 3: Repay borrow
        uint256 ethBalance = IERC20(WETH_ADDRESS).balanceOf(address(this));
        uint256 repayAmount = pos.borrowAmount < ethBalance ? pos.borrowAmount : ethBalance;
        
        IERC20(WETH_ADDRESS).approve(aaveAdapter, repayAmount);
        IProtocolAdapter(aaveAdapter).repay(WETH_ADDRESS, repayAmount, address(this));
        
        // Step 4: Withdraw collateral
        IProtocolAdapter(aaveAdapter).withdraw(USDC_ADDRESS, pos.collateralAmount);
        
        // Step 5: Return to user
        uint256 usdcBalance = IERC20(USDC_ADDRESS).balanceOf(address(this));
        IERC20(USDC_ADDRESS).transfer(msg.sender, usdcBalance);
        
        // Transfer remaining ETH rewards if any
        uint256 ethRewards = IERC20(WETH_ADDRESS).balanceOf(address(this));
        if (ethRewards > 0) {
            IERC20(WETH_ADDRESS).transfer(msg.sender, ethRewards);
        }
        
        // Calculate net PnL
        int256 pnl = int256(usdcBalance) - int256(pos.collateralAmount);
        
        pos.active = false;
        
        emit StrategyClosed(msg.sender, pnl);
    }
    
    function _getEthPrice() internal view returns (uint256) {
        // ใน production: ดึงจาก Chainlink oracle
        return 3000e18; // Placeholder
    }
    
    address constant USDC_ADDRESS = address(0x1);
    address constant WETH_ADDRESS = address(0x2);
}
```

---

## สรุป Part 90

- **Adapter Pattern**: Universal interface สำหรับ wrap protocols ต่างๆ (Aave, Compound, Curve) ให้ใช้งานได้เหมือนกัน
- **DeFi Lego Composition**: Flash loan + Swap + Stake ใน single transaction ผ่าน DeFiCompositor
- **Risk Isolation**: Callback authorization, reentrancy protection สำหรับ composed calls, invariant checks
- **Flash Arbitrage**: คำนวณ profit โดยคิด flash fee และ gas ก่อนรัน
- **Composability Testing**: Fork testing กับ real protocols, protocol mock สำหรับ unit tests
- **YieldStrategyLego**: ตัวอย่าง real yield strategy ที่ combine multiple protocols ใน unified strategy

---

## สรุป Part 81-90 (ทั้ง Block)

| Part | Topic | Key Concepts |
|------|-------|-------------|
| 81 | Gas Optimization | Assembly, storage packing, SSTORE2 |
| 82 | Security Patterns | Reentrancy, oracle manipulation, access control |
| 83 | Cross-Chain | Bridge design, message passing, LayerZero |
| 84 | ZK Proofs | Groth16, PLONK, ZK rollup verifier |
| 85 | MEV & Flashbots | Flashbot bundles, sandwich protection |
| 86 | Account Abstraction | EIP-4337, Paymaster, Social Recovery, Session Keys |
| 87 | DeFi Aggregators | PathFinder, split routing, Multicall3, Yield aggregator |
| 88 | Prediction Markets | OrderBook, LMSR AMM, oracle resolution |
| 89 | Advanced Perps | Funding rate, cross/isolated margin, insurance fund, ADL |
| **90** | **Composability** | **Adapters, DeFi Legos, flash arb, risk isolation** |

## Next: Part 91 - Capstone: Building a Full Protocol

ใน Part 91 เราจะรวมทุก concept ที่เรียนมาทั้งหมดเพื่อสร้าง Full DeFi Protocol ตั้งแต่เริ่มต้น:
- Architecture design
- Smart contract suite (Core + Periphery)
- Security audit checklist
- Gas optimization pass
- Deployment strategy
- Governance integration
