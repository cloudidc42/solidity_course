# Part 92: Capstone - Core Contract Implementation

## บทนำ

ใน Part 92 เราจะ implement **core contracts** ของ OmniYield จากที่ออกแบบไว้ใน Part 91 ทุก contract จะใช้ patterns ที่ดีที่สุด:
- **ERC-4626** สำหรับ vault standard
- **EIP-7201** สำหรับ storage layout
- **Foundry** สำหรับ testing
- **OpenZeppelin** สำหรับ base contracts

---

## 1. OmniRegistry

เริ่มจาก Registry ก่อนเพราะ contracts อื่น ๆ ต้องการมัน

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Ownable2Step, Ownable} from "@openzeppelin/contracts/access/Ownable2Step.sol";
import {Pausable} from "@openzeppelin/contracts/utils/Pausable.sol";

/// @title OmniRegistry - Contract registry สำหรับ OmniYield Protocol
/// @notice เก็บ addresses ของ contracts ทั้งหมดใน protocol พร้อม version control
/// @dev ใช้ Ownable2Step เพื่อป้องกัน accidental ownership transfer
contract OmniRegistry is Ownable2Step {
    // ============================================================
    // Structs
    // ============================================================

    struct ContractRecord {
        address contractAddress;
        uint256 version;
        uint256 deployedAt;
        bool deprecated;
        string description;
    }

    // ============================================================
    // State Variables
    // ============================================================

    /// @notice latest record สำหรับแต่ละ key
    mapping(bytes32 => ContractRecord) private _records;

    /// @notice history ทั้งหมด (key => array of records)
    mapping(bytes32 => ContractRecord[]) private _history;

    /// @notice reverse lookup: address => key
    mapping(address => bytes32) private _addressToKey;

    /// @notice keys ทั้งหมดที่ register
    bytes32[] private _keys;

    // ============================================================
    // Events
    // ============================================================

    event ContractRegistered(
        bytes32 indexed key,
        address indexed contractAddress,
        uint256 version,
        string description
    );

    event ContractUpgraded(
        bytes32 indexed key,
        address indexed oldAddress,
        address indexed newAddress,
        uint256 newVersion
    );

    event ContractDeprecated(bytes32 indexed key, address indexed contractAddress);

    // ============================================================
    // Errors
    // ============================================================

    error AlreadyRegistered(bytes32 key);
    error NotRegistered(bytes32 key);
    error ZeroAddress();
    error AlreadyDeprecated(bytes32 key);

    // ============================================================
    // Constructor
    // ============================================================

    constructor(address initialOwner) Ownable(initialOwner) {}

    // ============================================================
    // View Functions
    // ============================================================

    /// @notice คืน address ปัจจุบันของ contract
    function getAddress(bytes32 key) external view returns (address) {
        ContractRecord storage record = _records[key];
        require(record.contractAddress != address(0), "Not registered");
        require(!record.deprecated, "Deprecated");
        return record.contractAddress;
    }

    /// @notice คืน record ปัจจุบัน (รวม deprecated)
    function getRecord(bytes32 key) external view returns (ContractRecord memory) {
        return _records[key];
    }

    /// @notice คืน history ทั้งหมดของ key
    function getHistory(bytes32 key) external view returns (ContractRecord[] memory) {
        return _history[key];
    }

    /// @notice ตรวจว่า address นี้ถูก register อยู่
    function isRegistered(address contractAddress) external view returns (bool) {
        bytes32 key = _addressToKey[contractAddress];
        if (key == bytes32(0)) return false;
        return _records[key].contractAddress == contractAddress && !_records[key].deprecated;
    }

    /// @notice คืน keys ทั้งหมด
    function getAllKeys() external view returns (bytes32[] memory) {
        return _keys;
    }

    /// @notice คืน version ของ contract
    function getVersion(bytes32 key) external view returns (uint256) {
        return _records[key].version;
    }

    // ============================================================
    // Management Functions (Owner only)
    // ============================================================

    /// @notice Register contract ใหม่
    function register(
        bytes32 key,
        address contractAddress,
        string calldata description
    ) external onlyOwner {
        if (contractAddress == address(0)) revert ZeroAddress();
        if (_records[key].contractAddress != address(0)) revert AlreadyRegistered(key);

        ContractRecord memory record = ContractRecord({
            contractAddress: contractAddress,
            version: 1,
            deployedAt: block.timestamp,
            deprecated: false,
            description: description
        });

        _records[key] = record;
        _history[key].push(record);
        _addressToKey[contractAddress] = key;
        _keys.push(key);

        emit ContractRegistered(key, contractAddress, 1, description);
    }

    /// @notice Upgrade contract เป็น version ใหม่
    function upgrade(bytes32 key, address newAddress) external onlyOwner {
        if (newAddress == address(0)) revert ZeroAddress();
        ContractRecord storage current = _records[key];
        if (current.contractAddress == address(0)) revert NotRegistered(key);

        address oldAddress = current.contractAddress;
        uint256 newVersion = current.version + 1;

        // Update reverse lookup
        delete _addressToKey[oldAddress];
        _addressToKey[newAddress] = key;

        // Update record
        ContractRecord memory newRecord = ContractRecord({
            contractAddress: newAddress,
            version: newVersion,
            deployedAt: block.timestamp,
            deprecated: false,
            description: current.description
        });

        _records[key] = newRecord;
        _history[key].push(newRecord);

        emit ContractUpgraded(key, oldAddress, newAddress, newVersion);
    }

    /// @notice Deprecate contract (ยังเก็บ history ไว้)
    function deprecate(bytes32 key) external onlyOwner {
        ContractRecord storage record = _records[key];
        if (record.contractAddress == address(0)) revert NotRegistered(key);
        if (record.deprecated) revert AlreadyDeprecated(key);

        record.deprecated = true;

        emit ContractDeprecated(key, record.contractAddress);
    }
}
```

---

## 2. StrategyBase

Abstract base class ที่ทุก strategy ต้อง inherit

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import {IStrategy} from "../interfaces/IStrategy.sol";

/// @title StrategyBase - Abstract base class สำหรับ OmniYield strategies
/// @notice ให้ lifecycle hooks, access control, emergency patterns
/// @dev Concrete strategies ต้อง implement _deposit, _withdraw, _harvest, _totalAssets
abstract contract StrategyBase is IStrategy, ReentrancyGuard {
    using SafeERC20 for IERC20;

    // ============================================================
    // Immutables
    // ============================================================

    /// @notice Vault ที่ strategy ทำงานให้
    address public immutable override vault;

    /// @notice Underlying asset
    address public immutable override asset;

    // ============================================================
    // State Variables
    // ============================================================

    /// @notice Emergency exit flag
    bool public override emergencyExit;

    /// @notice Timestamp ของ harvest ล่าสุด
    uint256 public lastHarvest;

    /// @notice Total assets ที่ deploy ไปแล้ว (ใน protocol นั้น)
    uint256 internal _deployedAssets;

    // ============================================================
    // Modifiers
    // ============================================================

    modifier onlyVault() {
        require(msg.sender == vault, "StrategyBase: not vault");
        _;
    }

    modifier notEmergency() {
        require(!emergencyExit, "StrategyBase: emergency exit");
        _;
    }

    // ============================================================
    // Events
    // ============================================================

    event Deposited(uint256 amount, uint256 deployedTotal);
    event Withdrawn(uint256 requested, uint256 actual);
    event Harvested(uint256 profit, uint256 loss, uint256 debtPayment);
    event EmergencyExitSet(bool value);
    event Migrated(address indexed newStrategy, uint256 amount);

    // ============================================================
    // Constructor
    // ============================================================

    constructor(address _vault, address _asset) {
        require(_vault != address(0), "StrategyBase: zero vault");
        require(_asset != address(0), "StrategyBase: zero asset");
        vault = _vault;
        asset = _asset;
    }

    // ============================================================
    // External Functions (Vault only)
    // ============================================================

    /// @notice ฝาก assets เข้า strategy
    /// @dev Vault transfer assets มาก่อน แล้วค่อยเรียก deposit()
    function deposit(uint256 amount) external override onlyVault nonReentrant notEmergency {
        require(amount > 0, "StrategyBase: zero amount");

        // ตรวจว่า asset ถูก transfer มาแล้ว
        uint256 balanceBefore = IERC20(asset).balanceOf(address(this));
        require(balanceBefore >= amount, "StrategyBase: insufficient balance");

        _deposit(amount);
        _deployedAssets += amount;

        emit Deposited(amount, _deployedAssets);
    }

    /// @notice ถอน assets ออกจาก strategy
    function withdraw(uint256 amount)
        external
        override
        onlyVault
        nonReentrant
        returns (uint256 withdrawn)
    {
        require(amount > 0, "StrategyBase: zero amount");

        uint256 balanceBefore = IERC20(asset).balanceOf(address(this));
        _withdraw(amount);
        uint256 balanceAfter = IERC20(asset).balanceOf(address(this));

        withdrawn = balanceAfter - balanceBefore;

        if (withdrawn > _deployedAssets) {
            _deployedAssets = 0;
        } else {
            _deployedAssets -= withdrawn;
        }

        // Transfer withdrawn assets กลับไป vault
        IERC20(asset).safeTransfer(vault, withdrawn);

        emit Withdrawn(amount, withdrawn);
    }

    /// @notice Harvest yield
    function harvest()
        external
        override
        onlyVault
        nonReentrant
        returns (uint256 profit, uint256 loss)
    {
        (profit, loss) = _harvest();
        lastHarvest = block.timestamp;

        // Transfer profit ไป vault
        if (profit > 0) {
            IERC20(asset).safeTransfer(vault, profit);
        }

        emit Harvested(profit, loss, 0);
    }

    /// @notice Emergency withdraw ทุกอย่างออก
    function emergencyWithdraw() external override onlyVault nonReentrant {
        emergencyExit = true;

        uint256 amount = _totalAssets();
        if (amount > 0) {
            _emergencyWithdraw();
        }

        uint256 balance = IERC20(asset).balanceOf(address(this));
        if (balance > 0) {
            IERC20(asset).safeTransfer(vault, balance);
        }

        _deployedAssets = 0;
        emit EmergencyExitSet(true);
    }

    /// @notice Migrate ไปยัง strategy ใหม่
    function migrate(address newStrategy) external override onlyVault nonReentrant {
        require(newStrategy != address(0), "StrategyBase: zero address");

        _migrate(newStrategy);

        uint256 balance = IERC20(asset).balanceOf(address(this));
        if (balance > 0) {
            IERC20(asset).safeTransfer(vault, balance);
        }

        emit Migrated(newStrategy, balance);
    }

    // ============================================================
    // View Functions
    // ============================================================

    /// @notice Total assets ที่ strategy manage (deployed + pending yield)
    function totalAssets() external view override returns (uint256) {
        return _totalAssets();
    }

    /// @notice Estimated total รวม unrealized yield
    function estimatedTotalAssets() external view override returns (uint256) {
        return _estimatedTotalAssets();
    }

    // ============================================================
    // Internal Abstract Functions (ต้อง implement ใน subclass)
    // ============================================================

    /// @dev Implement: deploy `amount` ไปยัง external protocol
    function _deposit(uint256 amount) internal virtual;

    /// @dev Implement: ถอน `amount` ออกจาก external protocol
    function _withdraw(uint256 amount) internal virtual;

    /// @dev Implement: collect yield, return (profit, loss)
    function _harvest() internal virtual returns (uint256 profit, uint256 loss);

    /// @dev Implement: ถอนทุกอย่างออก (emergency)
    function _emergencyWithdraw() internal virtual;

    /// @dev Implement: migrate ไป newStrategy
    function _migrate(address newStrategy) internal virtual;

    /// @dev Implement: คืน total assets ที่ deploy อยู่
    function _totalAssets() internal view virtual returns (uint256);

    /// @dev Implement: คืน estimated total รวม yield
    function _estimatedTotalAssets() internal view virtual returns (uint256);
}
```

---

## 3. AaveStrategy

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {StrategyBase} from "./StrategyBase.sol";

/// @notice Aave V3 interfaces (minimal)
interface IAavePool {
    function supply(address asset, uint256 amount, address onBehalfOf, uint16 referralCode) external;
    function withdraw(address asset, uint256 amount, address to) external returns (uint256);
    function getReserveData(address asset) external view returns (AaveDataTypes.ReserveData memory);
}

interface IAaveDataTypes {
    struct ReserveData {
        uint128 currentLiquidityRate;  // Ray units (1e27)
        address aTokenAddress;
        // ... other fields
    }
}

interface IAToken {
    function balanceOf(address account) external view returns (uint256);
    function scaledBalanceOf(address account) external view returns (uint256);
}

/// @title AaveStrategy - Strategy สำหรับ Aave V3
/// @notice Deposit underlying asset เข้า Aave V3 เพื่อรับ lending yield
contract AaveStrategy is StrategyBase {
    using SafeERC20 for IERC20;

    // ============================================================
    // Immutables
    // ============================================================

    /// @notice Aave V3 Pool
    IAavePool public immutable aavePool;

    /// @notice aToken (receipt token จาก Aave)
    IAToken public immutable aToken;

    // ============================================================
    // State
    // ============================================================

    /// @notice ใช้ track basis สำหรับคำนวณ profit
    uint256 private _basisAssets;

    // ============================================================
    // Constructor
    // ============================================================

    constructor(
        address _vault,
        address _aavePool,
        address _aToken,
        address _asset
    ) StrategyBase(_vault, _asset) {
        require(_aavePool != address(0), "AaveStrategy: zero pool");
        require(_aToken != address(0), "AaveStrategy: zero aToken");
        aavePool = IAavePool(_aavePool);
        aToken = IAToken(_aToken);
    }

    // ============================================================
    // View Functions
    // ============================================================

    function name() external pure override returns (string memory) {
        return "OmniYield Aave V3 Strategy";
    }

    /// @notice APR จาก Aave (basis points)
    function apr() external view override returns (uint256) {
        // Aave ใช้ Ray units (1e27)
        // currentLiquidityRate / 1e27 * 10000 (basis points)
        // ตัวอย่าง: 50000000000000000000000000 Ray = 5% APY
        // คำนวณจาก: rate * 10000 / 1e27
        // สมมติ rate = 0.05 * 1e27 = 5e25
        // apr = 5e25 * 10000 / 1e27 = 500 bps = 5%

        // Note: ใน production ต้อง query จาก Aave pool จริง
        // แต่ interface นี้ simplified
        return 500; // placeholder 5% APR
    }

    // ============================================================
    // Internal Implementations
    // ============================================================

    function _deposit(uint256 amount) internal override {
        // Approve Aave pool
        IERC20(asset).approve(address(aavePool), amount);

        // Supply ไป Aave
        aavePool.supply(asset, amount, address(this), 0);

        _basisAssets += amount;
    }

    function _withdraw(uint256 amount) internal override {
        // Withdraw จาก Aave (amount ที่ต้องการ)
        uint256 available = aToken.balanceOf(address(this));
        uint256 toWithdraw = amount > available ? available : amount;

        if (toWithdraw > 0) {
            aavePool.withdraw(asset, toWithdraw, address(this));
        }

        if (_basisAssets >= toWithdraw) {
            _basisAssets -= toWithdraw;
        } else {
            _basisAssets = 0;
        }
    }

    function _harvest() internal override returns (uint256 profit, uint256 loss) {
        uint256 currentBalance = aToken.balanceOf(address(this));
        uint256 basis = _basisAssets;

        if (currentBalance > basis) {
            profit = currentBalance - basis;
            // Withdraw เฉพาะ profit ออกมา
            aavePool.withdraw(asset, profit, address(this));
            // basis ยังเท่าเดิม (principal ยังอยู่)
        } else if (currentBalance < basis) {
            // Loss scenario (rare ใน Aave แต่ handle ไว้)
            loss = basis - currentBalance;
            _basisAssets = currentBalance;
        }
    }

    function _emergencyWithdraw() internal override {
        uint256 balance = aToken.balanceOf(address(this));
        if (balance > 0) {
            aavePool.withdraw(asset, type(uint256).max, address(this));
        }
        _basisAssets = 0;
    }

    function _migrate(address newStrategy) internal override {
        // ถอนทุกอย่างออก แล้ว vault จะ transfer ไป newStrategy
        uint256 balance = aToken.balanceOf(address(this));
        if (balance > 0) {
            aavePool.withdraw(asset, type(uint256).max, address(this));
        }
        _basisAssets = 0;
    }

    function _totalAssets() internal view override returns (uint256) {
        return aToken.balanceOf(address(this));
    }

    function _estimatedTotalAssets() internal view override returns (uint256) {
        // aToken balance = principal + accrued interest (auto-compounds)
        return aToken.balanceOf(address(this));
    }
}
```

---

## 4. UniswapV3Strategy

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {StrategyBase} from "./StrategyBase.sol";

/// @notice Uniswap V3 interfaces (minimal)
interface INonfungiblePositionManager {
    struct MintParams {
        address token0;
        address token1;
        uint24 fee;
        int24 tickLower;
        int24 tickUpper;
        uint256 amount0Desired;
        uint256 amount1Desired;
        uint256 amount0Min;
        uint256 amount1Min;
        address recipient;
        uint256 deadline;
    }

    struct IncreaseLiquidityParams {
        uint256 tokenId;
        uint256 amount0Desired;
        uint256 amount1Desired;
        uint256 amount0Min;
        uint256 amount1Min;
        uint256 deadline;
    }

    struct DecreaseLiquidityParams {
        uint256 tokenId;
        uint128 liquidity;
        uint256 amount0Min;
        uint256 amount1Min;
        uint256 deadline;
    }

    struct CollectParams {
        uint256 tokenId;
        address recipient;
        uint128 amount0Max;
        uint128 amount1Max;
    }

    function mint(MintParams calldata params) external payable returns (
        uint256 tokenId,
        uint128 liquidity,
        uint256 amount0,
        uint256 amount1
    );

    function increaseLiquidity(IncreaseLiquidityParams calldata params) external payable returns (
        uint128 liquidity,
        uint256 amount0,
        uint256 amount1
    );

    function decreaseLiquidity(DecreaseLiquidityParams calldata params) external payable returns (
        uint256 amount0,
        uint256 amount1
    );

    function collect(CollectParams calldata params) external payable returns (
        uint256 amount0,
        uint256 amount1
    );

    function burn(uint256 tokenId) external payable;

    function positions(uint256 tokenId) external view returns (
        uint96 nonce,
        address operator,
        address token0,
        address token1,
        uint24 fee,
        int24 tickLower,
        int24 tickUpper,
        uint128 liquidity,
        uint256 feeGrowthInside0LastX128,
        uint256 feeGrowthInside1LastX128,
        uint128 tokensOwed0,
        uint128 tokensOwed1
    );
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

    function exactInputSingle(ExactInputSingleParams calldata params) external payable returns (uint256 amountOut);
}

/// @title UniswapV3Strategy - Strategy สำหรับ Uniswap V3 LP
/// @notice Provide liquidity ใน Uniswap V3 pool เพื่อรับ trading fees
/// @dev ใช้ concentrated liquidity ใน full-range position (ง่ายกว่า)
contract UniswapV3Strategy is StrategyBase {
    using SafeERC20 for IERC20;

    // ============================================================
    // Immutables
    // ============================================================

    INonfungiblePositionManager public immutable positionManager;
    ISwapRouter public immutable swapRouter;

    /// @notice Token อีกตัวใน pair (เช่น WETH ถ้า asset = USDC)
    address public immutable quoteToken;

    /// @notice Fee tier ของ pool (500 = 0.05%, 3000 = 0.3%)
    uint24 public immutable poolFee;

    // ============================================================
    // State
    // ============================================================

    /// @notice NFT token ID ของ LP position
    uint256 public tokenId;

    /// @notice Tick range สำหรับ liquidity
    int24 public tickLower;
    int24 public tickUpper;

    /// @notice เปิด/ปิด auto-rebalance
    bool public autoRebalance;

    // ============================================================
    // Constants
    // ============================================================

    /// @notice Full range ticks
    int24 private constant TICK_LOWER = -887272;
    int24 private constant TICK_UPPER = 887272;

    // ============================================================
    // Constructor
    // ============================================================

    constructor(
        address _vault,
        address _positionManager,
        address _swapRouter,
        address _asset,
        address _quoteToken,
        uint24 _poolFee
    ) StrategyBase(_vault, _asset) {
        positionManager = INonfungiblePositionManager(_positionManager);
        swapRouter = ISwapRouter(_swapRouter);
        quoteToken = _quoteToken;
        poolFee = _poolFee;

        // ใช้ full-range เป็น default
        tickLower = TICK_LOWER;
        tickUpper = TICK_UPPER;
    }

    // ============================================================
    // View Functions
    // ============================================================

    function name() external pure override returns (string memory) {
        return "OmniYield Uniswap V3 Strategy";
    }

    function apr() external pure override returns (uint256) {
        // ใน production จะคำนวณจาก historical fees
        return 800; // placeholder 8% APR
    }

    // ============================================================
    // Internal Implementations
    // ============================================================

    function _deposit(uint256 amount) internal override {
        // แปลง 50% ของ asset ไปเป็น quoteToken
        uint256 halfAmount = amount / 2;
        uint256 quoteAmount = _swapToQuoteToken(halfAmount);

        uint256 assetAmount = amount - halfAmount;

        // Determine token order (Uniswap requires token0 < token1)
        (address token0, address token1, uint256 amount0, uint256 amount1) =
            _sortTokens(assetAmount, quoteAmount);

        // Approve
        IERC20(token0).approve(address(positionManager), amount0);
        IERC20(token1).approve(address(positionManager), amount1);

        if (tokenId == 0) {
            // สร้าง position ใหม่
            (uint256 newTokenId,,,) = positionManager.mint(
                INonfungiblePositionManager.MintParams({
                    token0: token0,
                    token1: token1,
                    fee: poolFee,
                    tickLower: tickLower,
                    tickUpper: tickUpper,
                    amount0Desired: amount0,
                    amount1Desired: amount1,
                    amount0Min: 0,
                    amount1Min: 0,
                    recipient: address(this),
                    deadline: block.timestamp + 300
                })
            );
            tokenId = newTokenId;
        } else {
            // เพิ่ม liquidity ใน position ที่มีอยู่
            positionManager.increaseLiquidity(
                INonfungiblePositionManager.IncreaseLiquidityParams({
                    tokenId: tokenId,
                    amount0Desired: amount0,
                    amount1Desired: amount1,
                    amount0Min: 0,
                    amount1Min: 0,
                    deadline: block.timestamp + 300
                })
            );
        }
    }

    function _withdraw(uint256 amount) internal override {
        if (tokenId == 0) return;

        (,,,,,,,uint128 liquidity,,,,) = positionManager.positions(tokenId);

        if (liquidity == 0) return;

        // คำนวณ liquidity ที่ต้องถอน proportional
        uint256 totalValue = _estimatedTotalAssets();
        uint128 liquidityToRemove;

        if (amount >= totalValue) {
            liquidityToRemove = liquidity;
        } else {
            liquidityToRemove = uint128(uint256(liquidity) * amount / totalValue);
        }

        // Decrease liquidity
        positionManager.decreaseLiquidity(
            INonfungiblePositionManager.DecreaseLiquidityParams({
                tokenId: tokenId,
                liquidity: liquidityToRemove,
                amount0Min: 0,
                amount1Min: 0,
                deadline: block.timestamp + 300
            })
        );

        // Collect tokens
        (uint256 collected0, uint256 collected1) = positionManager.collect(
            INonfungiblePositionManager.CollectParams({
                tokenId: tokenId,
                recipient: address(this),
                amount0Max: type(uint128).max,
                amount1Max: type(uint128).max
            })
        );

        // แปลง quoteToken กลับเป็น asset
        _convertToAsset(collected0, collected1);
    }

    function _harvest() internal override returns (uint256 profit, uint256 loss) {
        if (tokenId == 0) return (0, 0);

        // Collect fees โดยไม่ถอน liquidity
        (uint256 fee0, uint256 fee1) = positionManager.collect(
            INonfungiblePositionManager.CollectParams({
                tokenId: tokenId,
                recipient: address(this),
                amount0Max: type(uint128).max,
                amount1Max: type(uint128).max
            })
        );

        if (fee0 == 0 && fee1 == 0) return (0, 0);

        // แปลง fees ทั้งหมดเป็น asset
        profit = _convertToAsset(fee0, fee1);
    }

    function _emergencyWithdraw() internal override {
        if (tokenId == 0) return;

        (,,,,,,,uint128 liquidity,,,,) = positionManager.positions(tokenId);

        if (liquidity > 0) {
            positionManager.decreaseLiquidity(
                INonfungiblePositionManager.DecreaseLiquidityParams({
                    tokenId: tokenId,
                    liquidity: liquidity,
                    amount0Min: 0,
                    amount1Min: 0,
                    deadline: block.timestamp + 300
                })
            );
        }

        positionManager.collect(
            INonfungiblePositionManager.CollectParams({
                tokenId: tokenId,
                recipient: address(this),
                amount0Max: type(uint128).max,
                amount1Max: type(uint128).max
            })
        );

        positionManager.burn(tokenId);
        tokenId = 0;

        // Convert everything to asset
        uint256 quoteBalance = IERC20(quoteToken).balanceOf(address(this));
        if (quoteBalance > 0) {
            _swapFromQuoteToken(quoteBalance);
        }
    }

    function _migrate(address /*newStrategy*/) internal override {
        _emergencyWithdraw();
    }

    function _totalAssets() internal view override returns (uint256) {
        return IERC20(asset).balanceOf(address(this)) + _deployedAssets;
    }

    function _estimatedTotalAssets() internal view override returns (uint256) {
        if (tokenId == 0) return IERC20(asset).balanceOf(address(this));
        // Simplified: return deployed assets (ใน production ต้องคำนวณจาก position value)
        return _deployedAssets + IERC20(asset).balanceOf(address(this));
    }

    // ============================================================
    // Internal Helpers
    // ============================================================

    function _swapToQuoteToken(uint256 assetAmount) internal returns (uint256 quoteAmount) {
        IERC20(asset).approve(address(swapRouter), assetAmount);

        quoteAmount = swapRouter.exactInputSingle(
            ISwapRouter.ExactInputSingleParams({
                tokenIn: asset,
                tokenOut: quoteToken,
                fee: poolFee,
                recipient: address(this),
                deadline: block.timestamp + 300,
                amountIn: assetAmount,
                amountOutMinimum: 0, // NOTE: production ต้องใช้ price oracle
                sqrtPriceLimitX96: 0
            })
        );
    }

    function _swapFromQuoteToken(uint256 quoteAmount) internal returns (uint256 assetAmount) {
        IERC20(quoteToken).approve(address(swapRouter), quoteAmount);

        assetAmount = swapRouter.exactInputSingle(
            ISwapRouter.ExactInputSingleParams({
                tokenIn: quoteToken,
                tokenOut: asset,
                fee: poolFee,
                recipient: address(this),
                deadline: block.timestamp + 300,
                amountIn: quoteAmount,
                amountOutMinimum: 0,
                sqrtPriceLimitX96: 0
            })
        );
    }

    function _convertToAsset(uint256 amount0, uint256 amount1) internal returns (uint256 total) {
        (address token0,) = asset < quoteToken ? (asset, quoteToken) : (quoteToken, asset);

        if (token0 == asset) {
            // amount0 = asset, amount1 = quoteToken
            total = amount0 + _swapFromQuoteToken(amount1);
        } else {
            // amount0 = quoteToken, amount1 = asset
            total = amount1 + _swapFromQuoteToken(amount0);
        }
    }

    function _sortTokens(uint256 assetAmt, uint256 quoteAmt) internal view returns (
        address token0,
        address token1,
        uint256 amount0,
        uint256 amount1
    ) {
        if (asset < quoteToken) {
            return (asset, quoteToken, assetAmt, quoteAmt);
        } else {
            return (quoteToken, asset, quoteAmt, assetAmt);
        }
    }
}
```

---

## 5. OmniYieldVault (ERC-4626)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC4626, ERC20, IERC20, IERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import {Pausable} from "@openzeppelin/contracts/utils/Pausable.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";
import {IStrategy} from "./interfaces/IStrategy.sol";
import {OmniRegistry} from "./OmniRegistry.sol";

/// @title OmniYieldVault - ERC-4626 Vault พร้อม multi-strategy support
/// @notice รับ deposits และจัดสรรไปยัง strategies เพื่อ maximize yield
/// @dev ใช้ ERC-4626 standard เพื่อ composability
contract OmniYieldVault is ERC4626, ReentrancyGuard, Pausable {
    using SafeERC20 for IERC20;
    using Math for uint256;

    // ============================================================
    // Constants
    // ============================================================

    uint256 public constant MAX_BPS = 10_000;
    uint256 public constant MAX_STRATEGIES = 20;
    uint256 public constant HARVEST_COOLDOWN = 6 hours;

    // ============================================================
    // Structs
    // ============================================================

    struct StrategyParams {
        uint256 allocation;      // basis points (10000 = 100%)
        uint256 lastHarvest;     // timestamp
        uint256 totalDebt;       // assets ที่อยู่ใน strategy
        uint256 totalGain;       // cumulative gain
        uint256 totalLoss;       // cumulative loss
        bool active;
    }

    // ============================================================
    // State Variables
    // ============================================================

    /// @notice Registry สำหรับ lookup contracts
    OmniRegistry public immutable registry;

    /// @notice Governance address (timelock)
    address public governance;

    /// @notice Keeper address (สำหรับ harvest)
    address public keeper;

    /// @notice Emergency admin (multisig)
    address public emergencyAdmin;

    /// @notice Total debt = assets deployed to strategies
    uint256 public totalDebt;

    /// @notice Deposit limit (0 = no limit)
    uint256 public depositLimit;

    /// @notice Emergency shutdown flag
    bool public emergencyShutdown;

    /// @notice Performance fee (basis points)
    uint256 public performanceFee = 2_000; // 20%

    /// @notice Management fee (basis points per year)
    uint256 public managementFee = 200; // 2%

    /// @notice Fee recipient
    address public feeRecipient;

    /// @notice Last fee collection timestamp
    uint256 public lastFeeCollection;

    /// @notice Ordered list ของ strategies
    address[] public strategies;

    /// @notice Params ของแต่ละ strategy
    mapping(address => StrategyParams) public strategyParams;

    // ============================================================
    // Events
    // ============================================================

    event StrategyAdded(address indexed strategy, uint256 allocation);
    event StrategyRemoved(address indexed strategy, uint256 debtRepaid);
    event StrategyRebalanced(address[] strategies, uint256[] allocations);
    event Harvested(address indexed strategy, uint256 gain, uint256 loss);
    event HarvestAll(uint256 totalGain, uint256 totalLoss);
    event EmergencyShutdownSet(bool value);
    event GovernanceUpdated(address indexed newGovernance);
    event KeeperUpdated(address indexed newKeeper);
    event FeeCollected(uint256 amount, address indexed recipient);

    // ============================================================
    // Errors
    // ============================================================

    error NotGovernance();
    error NotKeeper();
    error NotEmergencyAdmin();
    error StrategyAlreadyAdded(address strategy);
    error StrategyNotFound(address strategy);
    error ExceedsDepositLimit(uint256 amount, uint256 limit);
    error EmergencyShutdownActive();
    error AllocationExceedsMax(uint256 total);
    error MaxStrategiesReached();
    error HarvestCooldown(uint256 nextHarvest);

    // ============================================================
    // Modifiers
    // ============================================================

    modifier onlyGovernance() {
        if (msg.sender != governance) revert NotGovernance();
        _;
    }

    modifier onlyKeeper() {
        if (msg.sender != keeper && msg.sender != governance) revert NotKeeper();
        _;
    }

    modifier onlyEmergencyAdmin() {
        if (msg.sender != emergencyAdmin && msg.sender != governance) revert NotEmergencyAdmin();
        _;
    }

    modifier notInEmergency() {
        if (emergencyShutdown) revert EmergencyShutdownActive();
        _;
    }

    // ============================================================
    // Constructor
    // ============================================================

    constructor(
        address _asset,
        address _registry,
        string memory _name,
        string memory _symbol
    )
        ERC4626(IERC20(_asset))
        ERC20(_name, _symbol)
    {
        require(_registry != address(0), "OmniYieldVault: zero registry");
        registry = OmniRegistry(_registry);
        governance = msg.sender;
        feeRecipient = msg.sender;
        lastFeeCollection = block.timestamp;
    }

    // ============================================================
    // ERC-4626 Overrides
    // ============================================================

    /// @notice Total assets = idle + deployed to strategies
    function totalAssets() public view override returns (uint256) {
        return IERC20(asset()).balanceOf(address(this)) + totalDebt;
    }

    /// @notice Deposit: override เพื่อเพิ่ม checks
    function deposit(uint256 assets, address receiver)
        public
        override
        nonReentrant
        whenNotPaused
        notInEmergency
        returns (uint256 shares)
    {
        if (depositLimit > 0) {
            uint256 newTotal = totalAssets() + assets;
            if (newTotal > depositLimit) revert ExceedsDepositLimit(assets, depositLimit);
        }

        // Collect management fee ก่อน deposit (เพื่อให้ calculation ถูกต้อง)
        _collectManagementFee();

        shares = super.deposit(assets, receiver);

        // Allocate ไปยัง strategies
        _allocateAssets();
    }

    /// @notice Withdraw: override เพื่อ pull จาก strategies ถ้าจำเป็น
    function withdraw(uint256 assets, address receiver, address owner)
        public
        override
        nonReentrant
        whenNotPaused
        returns (uint256 shares)
    {
        // ดึง assets จาก strategies ถ้า idle ไม่พอ
        _ensureLiquidity(assets);

        shares = super.withdraw(assets, receiver, owner);
    }

    /// @notice Redeem: override เพื่อ pull จาก strategies
    function redeem(uint256 shares, address receiver, address owner)
        public
        override
        nonReentrant
        whenNotPaused
        returns (uint256 assets)
    {
        uint256 assetsNeeded = previewRedeem(shares);
        _ensureLiquidity(assetsNeeded);

        assets = super.redeem(shares, receiver, owner);
    }

    // ============================================================
    // Strategy Management
    // ============================================================

    /// @notice เพิ่ม strategy ใหม่
    function addStrategy(address strategy, uint256 allocation)
        external
        onlyGovernance
    {
        if (strategies.length >= MAX_STRATEGIES) revert MaxStrategiesReached();
        if (strategyParams[strategy].active) revert StrategyAlreadyAdded(strategy);

        // ตรวจว่า total allocation ไม่เกิน 100%
        uint256 totalAlloc = _getTotalAllocation() + allocation;
        if (totalAlloc > MAX_BPS) revert AllocationExceedsMax(totalAlloc);

        strategyParams[strategy] = StrategyParams({
            allocation: allocation,
            lastHarvest: block.timestamp,
            totalDebt: 0,
            totalGain: 0,
            totalLoss: 0,
            active: true
        });

        strategies.push(strategy);

        emit StrategyAdded(strategy, allocation);

        // Allocate ทันที
        _allocateToStrategy(strategy);
    }

    /// @notice ลบ strategy
    function removeStrategy(address strategy)
        external
        onlyGovernance
    {
        if (!strategyParams[strategy].active) revert StrategyNotFound(strategy);

        // ถอน assets ออกจาก strategy
        uint256 debtRepaid = _withdrawFromStrategy(strategy, type(uint256).max);

        strategyParams[strategy].active = false;
        strategyParams[strategy].allocation = 0;

        // ลบออกจาก array
        for (uint256 i = 0; i < strategies.length; i++) {
            if (strategies[i] == strategy) {
                strategies[i] = strategies[strategies.length - 1];
                strategies.pop();
                break;
            }
        }

        emit StrategyRemoved(strategy, debtRepaid);
    }

    /// @notice ปรับ allocations
    function rebalance(
        address[] calldata _strategies,
        uint256[] calldata allocations
    ) external onlyGovernance {
        require(_strategies.length == allocations.length, "length mismatch");

        uint256 totalAlloc;
        for (uint256 i = 0; i < allocations.length; i++) {
            totalAlloc += allocations[i];
        }
        if (totalAlloc > MAX_BPS) revert AllocationExceedsMax(totalAlloc);

        for (uint256 i = 0; i < _strategies.length; i++) {
            if (!strategyParams[_strategies[i]].active) revert StrategyNotFound(_strategies[i]);
            strategyParams[_strategies[i]].allocation = allocations[i];
        }

        // Re-allocate assets
        _reallocateAll();

        emit StrategyRebalanced(_strategies, allocations);
    }

    // ============================================================
    // Harvest
    // ============================================================

    /// @notice Harvest จาก strategy ที่ระบุ
    function harvest(address strategy)
        external
        onlyKeeper
        returns (uint256 gain, uint256 loss)
    {
        StrategyParams storage params = strategyParams[strategy];
        if (!params.active) revert StrategyNotFound(strategy);

        if (block.timestamp < params.lastHarvest + HARVEST_COOLDOWN) {
            revert HarvestCooldown(params.lastHarvest + HARVEST_COOLDOWN);
        }

        uint256 beforeAssets = IERC20(asset()).balanceOf(address(this));
        (gain, loss) = IStrategy(strategy).harvest();
        uint256 afterAssets = IERC20(asset()).balanceOf(address(this));

        uint256 received = afterAssets - beforeAssets;

        if (received > 0) {
            // เก็บ performance fee
            uint256 fee = (received * performanceFee) / MAX_BPS;
            if (fee > 0 && feeRecipient != address(0)) {
                IERC20(asset()).safeTransfer(feeRecipient, fee);
                emit FeeCollected(fee, feeRecipient);
            }

            gain = received - fee;
        }

        params.lastHarvest = block.timestamp;
        params.totalGain += gain;
        if (loss > 0) {
            params.totalLoss += loss;
            if (params.totalDebt >= loss) {
                params.totalDebt -= loss;
                totalDebt -= loss;
            }
        }

        emit Harvested(strategy, gain, loss);

        // Re-allocate harvested gains
        if (gain > 0) {
            _allocateAssets();
        }
    }

    /// @notice Harvest ทุก strategies
    function harvestAll() external onlyKeeper {
        uint256 totalGain;
        uint256 totalLoss;

        for (uint256 i = 0; i < strategies.length; i++) {
            address strategy = strategies[i];
            StrategyParams storage params = strategyParams[strategy];

            if (block.timestamp < params.lastHarvest + HARVEST_COOLDOWN) continue;

            try IStrategy(strategy).harvest() returns (uint256 gain, uint256 loss) {
                totalGain += gain;
                totalLoss += loss;
                params.lastHarvest = block.timestamp;
                params.totalGain += gain;
            } catch {
                // log แต่ไม่ revert เพื่อไม่ให้ strategies อื่นถูกกระทบ
                emit Harvested(strategy, 0, 0);
            }
        }

        emit HarvestAll(totalGain, totalLoss);
    }

    // ============================================================
    // Emergency
    // ============================================================

    /// @notice Set emergency shutdown
    function setEmergencyShutdown(bool _shutdown) external onlyEmergencyAdmin {
        emergencyShutdown = _shutdown;

        if (_shutdown) {
            _pause();
        } else {
            _unpause();
        }

        emit EmergencyShutdownSet(_shutdown);
    }

    /// @notice Emergency withdraw จาก strategy
    function emergencyWithdrawFromStrategy(address strategy) external onlyEmergencyAdmin {
        IStrategy(strategy).emergencyWithdraw();

        uint256 debt = strategyParams[strategy].totalDebt;
        if (totalDebt >= debt) {
            totalDebt -= debt;
        } else {
            totalDebt = 0;
        }
        strategyParams[strategy].totalDebt = 0;
    }

    // ============================================================
    // Admin
    // ============================================================

    function setGovernance(address _governance) external onlyGovernance {
        governance = _governance;
        emit GovernanceUpdated(_governance);
    }

    function setKeeper(address _keeper) external onlyGovernance {
        keeper = _keeper;
        emit KeeperUpdated(_keeper);
    }

    function setEmergencyAdmin(address _admin) external onlyGovernance {
        emergencyAdmin = _admin;
    }

    function setDepositLimit(uint256 _limit) external onlyGovernance {
        depositLimit = _limit;
    }

    function setFees(uint256 _performanceFee, uint256 _managementFee) external onlyGovernance {
        require(_performanceFee <= 5_000, "performance fee too high"); // max 50%
        require(_managementFee <= 500, "management fee too high"); // max 5%
        performanceFee = _performanceFee;
        managementFee = _managementFee;
    }

    function setFeeRecipient(address _recipient) external onlyGovernance {
        feeRecipient = _recipient;
    }

    // ============================================================
    // View Functions
    // ============================================================

    function getStrategies() external view returns (address[] memory) {
        return strategies;
    }

    function getStrategyParams(address strategy)
        external
        view
        returns (StrategyParams memory)
    {
        return strategyParams[strategy];
    }

    function idleAssets() public view returns (uint256) {
        return IERC20(asset()).balanceOf(address(this));
    }

    // ============================================================
    // Internal Functions
    // ============================================================

    function _allocateAssets() internal {
        uint256 idle = idleAssets();
        if (idle == 0) return;

        for (uint256 i = 0; i < strategies.length; i++) {
            address strategy = strategies[i];
            _allocateToStrategy(strategy);
        }
    }

    function _allocateToStrategy(address strategy) internal {
        StrategyParams storage params = strategyParams[strategy];
        if (!params.active || params.allocation == 0) return;

        uint256 total = totalAssets();
        uint256 target = (total * params.allocation) / MAX_BPS;
        uint256 current = params.totalDebt;

        if (target > current) {
            uint256 toDeposit = target - current;
            uint256 available = idleAssets();
            if (toDeposit > available) toDeposit = available;

            if (toDeposit > 0) {
                IERC20(asset()).safeTransfer(strategy, toDeposit);
                IStrategy(strategy).deposit(toDeposit);
                params.totalDebt += toDeposit;
                totalDebt += toDeposit;
            }
        }
    }

    function _reallocateAll() internal {
        // ถอนทุกอย่างออกก่อน แล้วค่อย allocate ใหม่
        for (uint256 i = 0; i < strategies.length; i++) {
            address strategy = strategies[i];
            uint256 debt = strategyParams[strategy].totalDebt;
            if (debt > 0) {
                _withdrawFromStrategy(strategy, debt);
            }
        }

        // Allocate ใหม่ตาม allocation ใหม่
        _allocateAssets();
    }

    function _withdrawFromStrategy(address strategy, uint256 amount)
        internal
        returns (uint256 withdrawn)
    {
        uint256 maxWithdraw = strategyParams[strategy].totalDebt;
        if (amount > maxWithdraw) amount = maxWithdraw;

        if (amount == 0) return 0;

        withdrawn = IStrategy(strategy).withdraw(amount);

        if (strategyParams[strategy].totalDebt >= withdrawn) {
            strategyParams[strategy].totalDebt -= withdrawn;
        } else {
            strategyParams[strategy].totalDebt = 0;
        }

        if (totalDebt >= withdrawn) {
            totalDebt -= withdrawn;
        } else {
            totalDebt = 0;
        }
    }

    function _ensureLiquidity(uint256 amount) internal {
        uint256 idle = idleAssets();
        if (idle >= amount) return;

        uint256 needed = amount - idle;

        // ถอนจาก strategies proportionally
        for (uint256 i = 0; i < strategies.length && needed > 0; i++) {
            address strategy = strategies[i];
            uint256 debt = strategyParams[strategy].totalDebt;
            if (debt == 0) continue;

            uint256 toWithdraw = needed > debt ? debt : needed;
            uint256 withdrawn = _withdrawFromStrategy(strategy, toWithdraw);
            needed = withdrawn >= needed ? 0 : needed - withdrawn;
        }
    }

    function _collectManagementFee() internal {
        if (feeRecipient == address(0)) return;
        if (managementFee == 0) return;

        uint256 elapsed = block.timestamp - lastFeeCollection;
        if (elapsed == 0) return;

        uint256 assets = totalAssets();
        uint256 fee = (assets * managementFee * elapsed) / (MAX_BPS * 365 days);

        if (fee > 0 && fee <= idleAssets()) {
            IERC20(asset()).safeTransfer(feeRecipient, fee);
            emit FeeCollected(fee, feeRecipient);
        }

        lastFeeCollection = block.timestamp;
    }

    function _getTotalAllocation() internal view returns (uint256 total) {
        for (uint256 i = 0; i < strategies.length; i++) {
            if (strategyParams[strategies[i]].active) {
                total += strategyParams[strategies[i]].allocation;
            }
        }
    }
}
```

---

## 6. Foundry Test Suite

### 6.1 Unit Tests - OmniRegistry

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test, console2} from "forge-std/Test.sol";
import {OmniRegistry} from "../../src/OmniRegistry.sol";

contract OmniRegistryTest is Test {
    OmniRegistry registry;
    address owner = makeAddr("owner");
    address alice = makeAddr("alice");

    bytes32 constant VAULT_KEY = keccak256("VAULT");
    address constant VAULT_ADDR = address(0x1234);

    function setUp() public {
        vm.prank(owner);
        registry = new OmniRegistry(owner);
    }

    // ─── Register ────────────────────────────────────────────────

    function test_register_success() public {
        vm.prank(owner);
        registry.register(VAULT_KEY, VAULT_ADDR, "OmniYield Vault v1");

        assertEq(registry.getAddress(VAULT_KEY), VAULT_ADDR);
    }

    function test_register_emitsEvent() public {
        vm.expectEmit(true, true, false, true);
        emit OmniRegistry.ContractRegistered(VAULT_KEY, VAULT_ADDR, 1, "OmniYield Vault v1");

        vm.prank(owner);
        registry.register(VAULT_KEY, VAULT_ADDR, "OmniYield Vault v1");
    }

    function test_register_revert_zeroAddress() public {
        vm.prank(owner);
        vm.expectRevert(OmniRegistry.ZeroAddress.selector);
        registry.register(VAULT_KEY, address(0), "test");
    }

    function test_register_revert_alreadyRegistered() public {
        vm.startPrank(owner);
        registry.register(VAULT_KEY, VAULT_ADDR, "v1");

        vm.expectRevert(abi.encodeWithSelector(OmniRegistry.AlreadyRegistered.selector, VAULT_KEY));
        registry.register(VAULT_KEY, address(0x5678), "v2");
        vm.stopPrank();
    }

    function test_register_revert_onlyOwner() public {
        vm.prank(alice);
        vm.expectRevert();
        registry.register(VAULT_KEY, VAULT_ADDR, "test");
    }

    // ─── Upgrade ────────────────────────────────────────────────

    function test_upgrade_incrementsVersion() public {
        vm.startPrank(owner);
        registry.register(VAULT_KEY, VAULT_ADDR, "v1");
        registry.upgrade(VAULT_KEY, address(0x5678));
        vm.stopPrank();

        assertEq(registry.getRecord(VAULT_KEY).version, 2);
        assertEq(registry.getAddress(VAULT_KEY), address(0x5678));
    }

    function test_upgrade_keepsHistory() public {
        vm.startPrank(owner);
        registry.register(VAULT_KEY, VAULT_ADDR, "v1");
        registry.upgrade(VAULT_KEY, address(0x5678));
        registry.upgrade(VAULT_KEY, address(0x9abc));
        vm.stopPrank();

        OmniRegistry.ContractRecord[] memory history = registry.getHistory(VAULT_KEY);
        assertEq(history.length, 3); // v1 + v2 + v3
        assertEq(history[0].contractAddress, VAULT_ADDR);
    }

    // ─── isRegistered ────────────────────────────────────────────

    function test_isRegistered_true() public {
        vm.prank(owner);
        registry.register(VAULT_KEY, VAULT_ADDR, "v1");
        assertTrue(registry.isRegistered(VAULT_ADDR));
    }

    function test_isRegistered_false_afterDeprecate() public {
        vm.startPrank(owner);
        registry.register(VAULT_KEY, VAULT_ADDR, "v1");
        registry.deprecate(VAULT_KEY);
        vm.stopPrank();

        assertFalse(registry.isRegistered(VAULT_ADDR));
    }

    // ─── Fuzz Tests ──────────────────────────────────────────────

    function testFuzz_register_anyKey(bytes32 key, address addr) public {
        vm.assume(addr != address(0));
        vm.assume(key != bytes32(0));

        vm.prank(owner);
        registry.register(key, addr, "fuzz test");

        assertEq(registry.getAddress(key), addr);
    }
}
```

### 6.2 Unit Tests - OmniYieldVault

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test, console2} from "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {OmniYieldVault} from "../../src/OmniYieldVault.sol";
import {OmniRegistry} from "../../src/OmniRegistry.sol";
import {MockERC20} from "../mocks/MockERC20.sol";
import {MockStrategy} from "../mocks/MockStrategy.sol";

contract OmniYieldVaultTest is Test {
    OmniYieldVault vault;
    OmniRegistry registry;
    MockERC20 usdc;
    MockStrategy strategy1;
    MockStrategy strategy2;

    address governance = makeAddr("governance");
    address keeper = makeAddr("keeper");
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");

    uint256 constant INITIAL_BALANCE = 1_000_000e6; // 1M USDC

    function setUp() public {
        // Deploy mocks
        usdc = new MockERC20("USD Coin", "USDC", 6);

        // Deploy registry
        registry = new OmniRegistry(governance);

        // Deploy vault
        vm.prank(governance);
        vault = new OmniYieldVault(
            address(usdc),
            address(registry),
            "OmniYield USDC",
            "omUSDC"
        );

        vm.startPrank(governance);
        vault.setKeeper(keeper);
        vm.stopPrank();

        // Deploy mock strategies
        strategy1 = new MockStrategy(address(vault), address(usdc));
        strategy2 = new MockStrategy(address(vault), address(usdc));

        // Mint USDC to users
        usdc.mint(alice, INITIAL_BALANCE);
        usdc.mint(bob, INITIAL_BALANCE);
    }

    // ─── Deposit ─────────────────────────────────────────────────

    function test_deposit_mintsCorrectShares() public {
        uint256 depositAmount = 100_000e6;

        vm.startPrank(alice);
        usdc.approve(address(vault), depositAmount);
        uint256 shares = vault.deposit(depositAmount, alice);
        vm.stopPrank();

        assertEq(shares, depositAmount); // 1:1 ตอนแรก
        assertEq(vault.balanceOf(alice), shares);
        assertEq(vault.totalAssets(), depositAmount);
    }

    function test_deposit_revert_emergencyShutdown() public {
        vm.prank(governance);
        vault.setEmergencyAdmin(governance);
        vault.setEmergencyShutdown(true);

        vm.startPrank(alice);
        usdc.approve(address(vault), 1000e6);
        vm.expectRevert(OmniYieldVault.EmergencyShutdownActive.selector);
        vault.deposit(1000e6, alice);
        vm.stopPrank();
    }

    function test_deposit_revert_exceedsLimit() public {
        vm.prank(governance);
        vault.setDepositLimit(500_000e6);

        vm.startPrank(alice);
        usdc.approve(address(vault), 600_000e6);
        vm.expectRevert(
            abi.encodeWithSelector(OmniYieldVault.ExceedsDepositLimit.selector, 600_000e6, 500_000e6)
        );
        vault.deposit(600_000e6, alice);
        vm.stopPrank();
    }

    // ─── Withdraw ─────────────────────────────────────────────────

    function test_withdraw_returnsAssets() public {
        uint256 depositAmount = 100_000e6;

        vm.startPrank(alice);
        usdc.approve(address(vault), depositAmount);
        vault.deposit(depositAmount, alice);
        vm.stopPrank();

        uint256 aliceBalanceBefore = usdc.balanceOf(alice);

        vm.prank(alice);
        vault.withdraw(depositAmount, alice, alice);

        uint256 aliceBalanceAfter = usdc.balanceOf(alice);
        assertEq(aliceBalanceAfter - aliceBalanceBefore, depositAmount);
    }

    // ─── Strategy Management ──────────────────────────────────────

    function test_addStrategy_success() public {
        vm.prank(governance);
        vault.addStrategy(address(strategy1), 5000);

        address[] memory strats = vault.getStrategies();
        assertEq(strats.length, 1);
        assertEq(strats[0], address(strategy1));
    }

    function test_addStrategy_allocatesImmediately() public {
        uint256 depositAmount = 100_000e6;

        vm.startPrank(alice);
        usdc.approve(address(vault), depositAmount);
        vault.deposit(depositAmount, alice);
        vm.stopPrank();

        vm.prank(governance);
        vault.addStrategy(address(strategy1), 5000); // 50%

        // Strategy ควรได้รับ 50% = 50,000 USDC
        assertApproxEqAbs(
            strategy1.totalAssets(),
            50_000e6,
            100 // allow 100 wei rounding
        );
    }

    function test_addStrategy_revert_duplicated() public {
        vm.startPrank(governance);
        vault.addStrategy(address(strategy1), 5000);

        vm.expectRevert(
            abi.encodeWithSelector(OmniYieldVault.StrategyAlreadyAdded.selector, address(strategy1))
        );
        vault.addStrategy(address(strategy1), 3000);
        vm.stopPrank();
    }

    function test_removeStrategy_returnsDebt() public {
        uint256 depositAmount = 100_000e6;

        vm.startPrank(alice);
        usdc.approve(address(vault), depositAmount);
        vault.deposit(depositAmount, alice);
        vm.stopPrank();

        vm.startPrank(governance);
        vault.addStrategy(address(strategy1), 5000);
        vault.removeStrategy(address(strategy1));
        vm.stopPrank();

        // Vault ควรได้ assets คืน
        assertEq(vault.idleAssets(), depositAmount);
        assertEq(vault.totalDebt(), 0);
    }

    // ─── Harvest ──────────────────────────────────────────────────

    function test_harvest_collectsYield() public {
        uint256 depositAmount = 100_000e6;

        vm.startPrank(alice);
        usdc.approve(address(vault), depositAmount);
        vault.deposit(depositAmount, alice);
        vm.stopPrank();

        vm.prank(governance);
        vault.addStrategy(address(strategy1), 5000);

        // Simulate yield: 1000 USDC profit
        usdc.mint(address(strategy1), 1_000e6);
        strategy1.setProfit(1_000e6);

        // Warp past harvest cooldown
        vm.warp(block.timestamp + 7 hours);

        uint256 totalBefore = vault.totalAssets();

        vm.prank(keeper);
        (uint256 gain,) = vault.harvest(address(strategy1));

        uint256 totalAfter = vault.totalAssets();

        assertTrue(gain > 0, "Should have gain");
        assertGt(totalAfter, totalBefore, "Total should increase");
    }

    // ─── Fee Collection ───────────────────────────────────────────

    function test_managementFee_collectedOnDeposit() public {
        vm.prank(governance);
        vault.setFeeRecipient(governance);

        uint256 depositAmount = 100_000e6;

        vm.startPrank(alice);
        usdc.approve(address(vault), depositAmount);
        vault.deposit(depositAmount, alice);
        vm.stopPrank();

        // Warp 1 year
        vm.warp(block.timestamp + 365 days);

        // Second deposit triggers fee collection
        vm.startPrank(bob);
        usdc.approve(address(vault), 1000e6);
        vault.deposit(1000e6, bob);
        vm.stopPrank();

        // Governance should have received ~2% of 100k = ~2000 USDC
        uint256 feeReceived = usdc.balanceOf(governance);
        assertApproxEqRel(feeReceived, 2_000e6, 0.01e18); // within 1%
    }

    // ─── Integration ──────────────────────────────────────────────

    function test_fullCycle() public {
        // 1. Alice deposits 100k
        vm.startPrank(alice);
        usdc.approve(address(vault), 100_000e6);
        vault.deposit(100_000e6, alice);
        vm.stopPrank();

        // 2. Add 2 strategies
        vm.startPrank(governance);
        vault.addStrategy(address(strategy1), 5000); // 50%
        vault.addStrategy(address(strategy2), 3000); // 30%
        vm.stopPrank();

        // 3. Bob deposits 50k
        vm.startPrank(bob);
        usdc.approve(address(vault), 50_000e6);
        vault.deposit(50_000e6, bob);
        vm.stopPrank();

        // 4. Simulate yield
        usdc.mint(address(strategy1), 2_000e6);
        strategy1.setProfit(2_000e6);
        usdc.mint(address(strategy2), 500e6);
        strategy2.setProfit(500e6);

        vm.warp(block.timestamp + 7 hours);

        // 5. Harvest all
        vm.prank(keeper);
        vault.harvestAll();

        // 6. Alice withdraws everything
        uint256 aliceShares = vault.balanceOf(alice);
        vm.prank(alice);
        uint256 aliceAssets = vault.redeem(aliceShares, alice, alice);

        // Alice should get more than she deposited (profit)
        assertGt(aliceAssets, 100_000e6, "Alice should profit");

        console2.log("Alice deposited: 100000 USDC");
        console2.log("Alice received:", aliceAssets / 1e6, "USDC");
    }
}
```

### 6.3 Mock Contracts

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MockERC20 is ERC20 {
    uint8 private _decimals;

    constructor(string memory name, string memory symbol, uint8 decimals_)
        ERC20(name, symbol)
    {
        _decimals = decimals_;
    }

    function decimals() public view override returns (uint8) {
        return _decimals;
    }

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }

    function burn(address from, uint256 amount) external {
        _burn(from, amount);
    }
}
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {IStrategy} from "../../src/interfaces/IStrategy.sol";

/// @notice Mock strategy สำหรับ testing
contract MockStrategy is IStrategy {
    using SafeERC20 for IERC20;

    address public override vault;
    address public override asset;
    bool public override emergencyExit;

    uint256 private _totalAssets;
    uint256 private _pendingProfit;

    constructor(address _vault, address _asset) {
        vault = _vault;
        asset = _asset;
    }

    function name() external pure override returns (string memory) {
        return "MockStrategy";
    }

    function totalAssets() external view override returns (uint256) {
        return _totalAssets;
    }

    function estimatedTotalAssets() external view override returns (uint256) {
        return _totalAssets + _pendingProfit;
    }

    function apr() external pure override returns (uint256) {
        return 500; // 5%
    }

    function deposit(uint256 amount) external override {
        require(msg.sender == vault, "not vault");
        _totalAssets += amount;
    }

    function withdraw(uint256 amount) external override returns (uint256 withdrawn) {
        require(msg.sender == vault, "not vault");
        withdrawn = amount > _totalAssets ? _totalAssets : amount;
        _totalAssets -= withdrawn;
        IERC20(asset).safeTransfer(vault, withdrawn);
    }

    function harvest() external override returns (uint256 profit, uint256 loss) {
        require(msg.sender == vault, "not vault");
        profit = _pendingProfit;
        _pendingProfit = 0;
        if (profit > 0) {
            IERC20(asset).safeTransfer(vault, profit);
        }
    }

    function emergencyWithdraw() external override {
        require(msg.sender == vault, "not vault");
        emergencyExit = true;
        uint256 balance = IERC20(asset).balanceOf(address(this));
        if (balance > 0) {
            IERC20(asset).safeTransfer(vault, balance);
        }
        _totalAssets = 0;
    }

    function migrate(address /*newStrategy*/) external override {
        require(msg.sender == vault, "not vault");
        uint256 balance = IERC20(asset).balanceOf(address(this));
        if (balance > 0) {
            IERC20(asset).safeTransfer(vault, balance);
        }
        _totalAssets = 0;
    }

    // Helper สำหรับ testing
    function setProfit(uint256 profit) external {
        _pendingProfit = profit;
    }
}
```

### 6.4 Integration Test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test, console2} from "forge-std/Test.sol";
import {OmniYieldVault} from "../../src/OmniYieldVault.sol";
import {OmniRegistry} from "../../src/OmniRegistry.sol";
import {MockERC20} from "../mocks/MockERC20.sol";
import {MockStrategy} from "../mocks/MockStrategy.sol";

/// @notice Integration tests สำหรับ OmniYield core
contract CoreIntegrationTest is Test {
    OmniYieldVault vault;
    OmniRegistry registry;
    MockERC20 usdc;
    MockStrategy[] strategies;

    address governance = makeAddr("governance");
    address keeper = makeAddr("keeper");

    uint256 constant NUM_USERS = 10;
    uint256 constant NUM_STRATEGIES = 3;

    address[] users;

    function setUp() public {
        usdc = new MockERC20("USDC", "USDC", 6);

        registry = new OmniRegistry(governance);

        vm.prank(governance);
        vault = new OmniYieldVault(address(usdc), address(registry), "omUSDC", "omUSDC");

        vm.startPrank(governance);
        vault.setKeeper(keeper);
        vm.stopPrank();

        // Create strategies
        for (uint256 i = 0; i < NUM_STRATEGIES; i++) {
            MockStrategy s = new MockStrategy(address(vault), address(usdc));
            strategies.push(s);
        }

        // Create users
        for (uint256 i = 0; i < NUM_USERS; i++) {
            address user = makeAddr(string(abi.encodePacked("user", i)));
            users.push(user);
            usdc.mint(user, 100_000e6);
        }
    }

    function test_multiUserMultiStrategy() public {
        // Add strategies with different allocations
        vm.startPrank(governance);
        vault.addStrategy(address(strategies[0]), 4000); // 40%
        vault.addStrategy(address(strategies[1]), 3000); // 30%
        vault.addStrategy(address(strategies[2]), 2000); // 20%
        // 10% idle buffer
        vm.stopPrank();

        // All users deposit
        for (uint256 i = 0; i < NUM_USERS; i++) {
            vm.startPrank(users[i]);
            usdc.approve(address(vault), 100_000e6);
            vault.deposit(100_000e6, users[i]);
            vm.stopPrank();
        }

        uint256 totalDeposited = NUM_USERS * 100_000e6;
        assertApproxEqRel(vault.totalAssets(), totalDeposited, 0.001e18);

        // Simulate yield on all strategies
        for (uint256 i = 0; i < NUM_STRATEGIES; i++) {
            uint256 strategyBalance = strategies[i].totalAssets();
            uint256 profit = strategyBalance / 100; // 1% profit
            usdc.mint(address(strategies[i]), profit);
            strategies[i].setProfit(profit);
        }

        // Warp past cooldown
        vm.warp(block.timestamp + 7 hours);

        // Harvest all
        vm.prank(keeper);
        vault.harvestAll();

        // Verify total assets increased
        assertGt(vault.totalAssets(), totalDeposited);

        // All users withdraw
        for (uint256 i = 0; i < NUM_USERS; i++) {
            uint256 shares = vault.balanceOf(users[i]);
            uint256 balanceBefore = usdc.balanceOf(users[i]);

            vm.prank(users[i]);
            vault.redeem(shares, users[i], users[i]);

            uint256 received = usdc.balanceOf(users[i]) - balanceBefore;
            assertGe(received, 100_000e6, "User should get at least their deposit back");
        }

        assertEq(vault.totalSupply(), 0);
    }
}
```

---

## Workshop

### Workshop 7.1: เพิ่ม CompoundStrategy

**โจทย์**: Implement `CompoundV3Strategy` ที่:
1. ฝากเงินใน Compound V3 (Comet)
2. Harvest COMP rewards
3. Swap COMP → USDC

```solidity
// TODO: Implement CompoundV3Strategy
// interface IComet {
//     function supply(address asset, uint256 amount) external;
//     function withdraw(address asset, uint256 amount) external;
//     function balanceOf(address account) external view returns (uint256);
// }

contract CompoundV3Strategy is StrategyBase {
    // เขียน implementation ที่นี่...
}
```

### Workshop 7.2: เพิ่ม Rebalance Logic ที่ดีขึ้น

**โจทย์**: แก้ `_reallocateAll()` ให้ไม่ต้องถอนทุกอย่างออกก่อน แต่คำนวณ delta แล้ว:
- ถ้า current > target: ถอนออก (delta)
- ถ้า current < target: ฝากเพิ่ม (delta)

```solidity
function _rebalanceOptimized() internal {
    // เขียน implementation ที่นี่...
    // Hint: คำนวณ delta สำหรับแต่ละ strategy ก่อน
    // แล้วถอนออกจากที่ over-allocated ก่อน
    // จากนั้น deposit เข้า under-allocated
}
```

---

## สรุป Part 92

- **OmniRegistry** ให้ version control และ audit trail สำหรับ contract addresses
- **StrategyBase** ให้ lifecycle hooks, access control, emergency patterns ที่ reusable
- **AaveStrategy** deposit ใน Aave V3 เพื่อรับ lending yield อัตโนมัติ
- **UniswapV3Strategy** provide liquidity ใน Uniswap V3 เพื่อรับ trading fees
- **OmniYieldVault** ใช้ ERC-4626 standard พร้อม multi-strategy allocation และ harvest
- **Test suite** ครอบคลุม unit tests, integration tests, และ mock contracts

## Next: Part 93 - Capstone Governance & Tokenomics

ใน Part 93 เราจะ implement:
- `OYT Token`: ERC-20 with permit + votes
- `veOYT`: vote-escrowed token สำหรับ governance power
- `OmniGovernor`: on-chain governance
- Full governance simulation
