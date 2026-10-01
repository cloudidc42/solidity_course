# Part 09: Inheritance และ Interfaces

## สารบัญ
1. Inheritance พื้นฐาน
2. Multiple Inheritance (C3 Linearization)
3. Abstract Contracts
4. Interfaces
5. Super Calls
6. Override Rules
7. Contract Composition Patterns
8. Libraries
9. Workshop: DeFi Protocol Base

---

## 1. Inheritance พื้นฐาน

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// Base Contract
contract Ownable {
    
    address private _owner;
    
    event OwnershipTransferred(
        address indexed previousOwner,
        address indexed newOwner
    );
    
    error OwnableUnauthorizedAccount(address account);
    error OwnableInvalidOwner(address owner);
    
    constructor(address initialOwner) {
        if (initialOwner == address(0)) {
            revert OwnableInvalidOwner(address(0));
        }
        _transferOwnership(initialOwner);
    }
    
    modifier onlyOwner() {
        _checkOwner();
        _;
    }
    
    function owner() public view virtual returns (address) {
        return _owner;
    }
    
    function _checkOwner() internal view virtual {
        if (owner() != msg.sender) {
            revert OwnableUnauthorizedAccount(msg.sender);
        }
    }
    
    function renounceOwnership() public virtual onlyOwner {
        _transferOwnership(address(0));
    }
    
    function transferOwnership(address newOwner) public virtual onlyOwner {
        if (newOwner == address(0)) {
            revert OwnableInvalidOwner(address(0));
        }
        _transferOwnership(newOwner);
    }
    
    function _transferOwnership(address newOwner) internal virtual {
        address oldOwner = _owner;
        _owner = newOwner;
        emit OwnershipTransferred(oldOwner, newOwner);
    }
}

// Child Contract ที่ inherit Ownable
contract SimpleToken is Ownable {
    
    string public name;
    string public symbol;
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    
    constructor(
        string memory _name,
        string memory _symbol,
        address initialOwner
    ) Ownable(initialOwner) {
        name = _name;
        symbol = _symbol;
    }
    
    // ใช้ onlyOwner จาก Ownable
    function mint(address to, uint256 amount) public onlyOwner {
        totalSupply += amount;
        balanceOf[to] += amount;
    }
    
    // Override parent function
    function renounceOwnership() public override onlyOwner {
        require(totalSupply == 0, "Cannot renounce with supply");
        super.renounceOwnership();
    }
}
```

---

## 2. Multiple Inheritance (C3 Linearization)

```solidity
contract A {
    function hello() public virtual pure returns (string memory) {
        return "A";
    }
    
    function greet() public virtual pure returns (string memory) {
        return "Hello from A";
    }
}

contract B is A {
    function hello() public virtual override pure returns (string memory) {
        return "B";
    }
}

contract C is A {
    function hello() public virtual override pure returns (string memory) {
        return "C";
    }
}

// ลำดับ inheritance: D → C → B → A (C3 linearization)
// เรียก super.hello() ใน D จะเรียก C
// เรียก super.hello() ใน C จะเรียก B
// เรียก super.hello() ใน B จะเรียก A

contract D is B, C {
    function hello() public override(B, C) pure returns (string memory) {
        return string.concat("D+", super.hello()); // calls C.hello()
    }
}

// MRO (Method Resolution Order):
// D → C → B → A

// ตัวอย่าง ERC-20 กับ Multiple Inheritance

contract ERC20 {
    mapping(address => uint256) internal _balances;
    mapping(address => mapping(address => uint256)) internal _allowances;
    uint256 internal _totalSupply;
    string internal _name;
    string internal _symbol;
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    constructor(string memory name_, string memory symbol_) {
        _name = name_;
        _symbol = symbol_;
    }
    
    function name() public view virtual returns (string memory) { return _name; }
    function symbol() public view virtual returns (string memory) { return _symbol; }
    function decimals() public view virtual returns (uint8) { return 18; }
    function totalSupply() public view virtual returns (uint256) { return _totalSupply; }
    function balanceOf(address account) public view virtual returns (uint256) { return _balances[account]; }
    
    function transfer(address to, uint256 amount) public virtual returns (bool) {
        _transfer(msg.sender, to, amount);
        return true;
    }
    
    function allowance(address owner, address spender) public view virtual returns (uint256) {
        return _allowances[owner][spender];
    }
    
    function approve(address spender, uint256 amount) public virtual returns (bool) {
        _approve(msg.sender, spender, amount);
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount) public virtual returns (bool) {
        _spendAllowance(from, msg.sender, amount);
        _transfer(from, to, amount);
        return true;
    }
    
    function _transfer(address from, address to, uint256 amount) internal virtual {
        require(from != address(0), "Transfer from zero");
        require(to != address(0), "Transfer to zero");
        
        uint256 fromBalance = _balances[from];
        require(fromBalance >= amount, "Insufficient balance");
        
        unchecked {
            _balances[from] = fromBalance - amount;
            _balances[to] += amount;
        }
        
        emit Transfer(from, to, amount);
    }
    
    function _mint(address account, uint256 amount) internal virtual {
        require(account != address(0), "Mint to zero");
        _totalSupply += amount;
        unchecked { _balances[account] += amount; }
        emit Transfer(address(0), account, amount);
    }
    
    function _burn(address account, uint256 amount) internal virtual {
        require(account != address(0), "Burn from zero");
        uint256 accountBalance = _balances[account];
        require(accountBalance >= amount, "Burn exceeds balance");
        unchecked { _balances[account] = accountBalance - amount; }
        _totalSupply -= amount;
        emit Transfer(account, address(0), amount);
    }
    
    function _approve(address owner, address spender, uint256 amount) internal virtual {
        require(owner != address(0), "Approve from zero");
        require(spender != address(0), "Approve to zero");
        _allowances[owner][spender] = amount;
        emit Approval(owner, spender, amount);
    }
    
    function _spendAllowance(address owner, address spender, uint256 amount) internal virtual {
        uint256 currentAllowance = allowance(owner, spender);
        if (currentAllowance != type(uint256).max) {
            require(currentAllowance >= amount, "Insufficient allowance");
            unchecked { _approve(owner, spender, currentAllowance - amount); }
        }
    }
}

contract Pausable {
    
    bool private _paused;
    
    event Paused(address account);
    event Unpaused(address account);
    
    error EnforcedPause();
    error ExpectedPause();
    
    constructor() {
        _paused = false;
    }
    
    modifier whenNotPaused() {
        _requireNotPaused();
        _;
    }
    
    modifier whenPaused() {
        _requirePaused();
        _;
    }
    
    function paused() public view virtual returns (bool) { return _paused; }
    
    function _requireNotPaused() internal view virtual {
        if (paused()) revert EnforcedPause();
    }
    
    function _requirePaused() internal view virtual {
        if (!paused()) revert ExpectedPause();
    }
    
    function _pause() internal virtual whenNotPaused {
        _paused = true;
        emit Paused(msg.sender);
    }
    
    function _unpause() internal virtual whenPaused {
        _paused = false;
        emit Unpaused(msg.sender);
    }
}

// ERC-20 + Ownable + Pausable
contract MyToken is ERC20, Ownable, Pausable {
    
    uint256 public constant MAX_SUPPLY = 100_000_000 * 1e18;
    
    constructor() 
        ERC20("My Token", "MTK")
        Ownable(msg.sender)
    {}
    
    function mint(address to, uint256 amount) public onlyOwner {
        require(_totalSupply + amount <= MAX_SUPPLY, "Exceeds max supply");
        _mint(to, amount);
    }
    
    function burn(uint256 amount) public {
        _burn(msg.sender, amount);
    }
    
    function pause() public onlyOwner {
        _pause();
    }
    
    function unpause() public onlyOwner {
        _unpause();
    }
    
    // Override transfer to add pause check
    function _transfer(
        address from,
        address to,
        uint256 amount
    ) internal virtual override whenNotPaused {
        super._transfer(from, to, amount);
    }
}
```

---

## 3. Abstract Contracts

```solidity
// Abstract: มี function ที่ยังไม่มี implementation
abstract contract BaseStrategy {
    
    string public name;
    address public asset;
    uint256 public totalAssets;
    
    constructor(string memory _name, address _asset) {
        name = _name;
        asset = _asset;
    }
    
    // Abstract functions - child MUST implement
    function invest(uint256 amount) external virtual;
    function withdraw(uint256 amount) external virtual;
    function getAPY() external view virtual returns (uint256);
    
    // Concrete functions - child can override
    function getInfo() external view virtual returns (
        string memory strategyName,
        address strategyAsset,
        uint256 assets,
        uint256 apy
    ) {
        return (name, asset, totalAssets, this.getAPY());
    }
    
    // Internal helper
    function _validateAmount(uint256 amount) internal pure {
        require(amount > 0, "Zero amount");
    }
}

contract YieldFarmingStrategy is BaseStrategy {
    
    address public farm;
    uint256 public rewardRate;
    
    constructor(address _asset, address _farm, uint256 _rewardRate) 
        BaseStrategy("Yield Farming", _asset) 
    {
        farm = _farm;
        rewardRate = _rewardRate;
    }
    
    function invest(uint256 amount) external override {
        _validateAmount(amount);
        // Deposit to farm
        totalAssets += amount;
    }
    
    function withdraw(uint256 amount) external override {
        _validateAmount(amount);
        require(totalAssets >= amount, "Insufficient");
        // Withdraw from farm
        totalAssets -= amount;
    }
    
    function getAPY() external view override returns (uint256) {
        return rewardRate * 365; // Simplified
    }
}

contract LendingStrategy is BaseStrategy {
    
    address public lendingPool;
    
    constructor(address _asset, address _pool) 
        BaseStrategy("Lending", _asset) 
    {
        lendingPool = _pool;
    }
    
    function invest(uint256 amount) external override {
        _validateAmount(amount);
        // Deposit to lending pool
        totalAssets += amount;
    }
    
    function withdraw(uint256 amount) external override {
        _validateAmount(amount);
        require(totalAssets >= amount, "Insufficient");
        totalAssets -= amount;
    }
    
    function getAPY() external view override returns (uint256) {
        return 500; // 5% APY in basis points
    }
}
```

---

## 4. Interfaces

```solidity
// Interface: กำหนด ABI เท่านั้น ไม่มี implementation
interface IERC20 {
    
    // Events
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    // Functions (ทุก function ต้องเป็น external)
    function name() external view returns (string memory);
    function symbol() external view returns (string memory);
    function decimals() external view returns (uint8);
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
}

interface IERC20Metadata is IERC20 {
    function name() external view returns (string memory);
    function symbol() external view returns (string memory);
    function decimals() external view returns (uint8);
}

// ใช้ Interface สำหรับ type safety
contract TokenInteractor {
    
    function getBalance(address token, address account) external view returns (uint256) {
        return IERC20(token).balanceOf(account);
    }
    
    function transferToken(
        address token,
        address to,
        uint256 amount
    ) external returns (bool) {
        return IERC20(token).transfer(to, amount);
    }
    
    function transferFromToken(
        address token,
        address from,
        address to,
        uint256 amount
    ) external returns (bool) {
        return IERC20(token).transferFrom(from, to, amount);
    }
    
    // Approve และ call
    function approveAndCall(
        address token,
        address spender,
        uint256 amount
    ) external {
        IERC20(token).approve(spender, amount);
        // แล้วทำอะไรบางอย่างกับ spender
    }
    
    // Safe transfer (handle ERC-20 ที่ไม่ return bool)
    function safeTransfer(address token, address to, uint256 amount) external {
        (bool success, bytes memory data) = token.call(
            abi.encodeWithSelector(IERC20.transfer.selector, to, amount)
        );
        require(
            success && (data.length == 0 || abi.decode(data, (bool))),
            "SafeTransfer failed"
        );
    }
}

// Interface กับ ERC-165 (supportsInterface)
interface IERC165 {
    function supportsInterface(bytes4 interfaceId) external view returns (bool);
}

interface IERC721 is IERC165 {
    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);
    
    function balanceOf(address owner) external view returns (uint256 balance);
    function ownerOf(uint256 tokenId) external view returns (address owner);
    function safeTransferFrom(address from, address to, uint256 tokenId, bytes calldata data) external;
    function safeTransferFrom(address from, address to, uint256 tokenId) external;
    function transferFrom(address from, address to, uint256 tokenId) external;
    function approve(address to, uint256 tokenId) external;
    function setApprovalForAll(address operator, bool approved) external;
    function getApproved(uint256 tokenId) external view returns (address operator);
    function isApprovedForAll(address owner, address operator) external view returns (bool);
}
```

---

## 5. Libraries

```solidity
// Library: code ที่ใช้ร่วมกันได้
library SafeERC20 {
    
    using Address for address;
    
    function safeTransfer(IERC20 token, address to, uint256 value) internal {
        _callOptionalReturn(token, abi.encodeCall(token.transfer, (to, value)));
    }
    
    function safeTransferFrom(IERC20 token, address from, address to, uint256 value) internal {
        _callOptionalReturn(token, abi.encodeCall(token.transferFrom, (from, to, value)));
    }
    
    function safeApprove(IERC20 token, address spender, uint256 value) internal {
        require(
            (value == 0) || (token.allowance(address(this), spender) == 0),
            "SafeERC20: approve from non-zero"
        );
        _callOptionalReturn(token, abi.encodeCall(token.approve, (spender, value)));
    }
    
    function _callOptionalReturn(IERC20 token, bytes memory data) private {
        bytes memory returndata = address(token).functionCall(data);
        require(
            returndata.length == 0 || abi.decode(returndata, (bool)),
            "SafeERC20: ERC20 operation did not succeed"
        );
    }
}

library Address {
    function isContract(address account) internal view returns (bool) {
        return account.code.length > 0;
    }
    
    function functionCall(address target, bytes memory data) internal returns (bytes memory) {
        (bool success, bytes memory returndata) = target.call(data);
        return verifyCallResultFromTarget(target, success, returndata);
    }
    
    function verifyCallResultFromTarget(
        address target,
        bool success,
        bytes memory returndata
    ) internal view returns (bytes memory) {
        if (success) {
            if (returndata.length == 0 && !isContract(target)) {
                revert("Address: call to non-contract");
            }
            return returndata;
        } else {
            _revert(returndata);
        }
    }
    
    function _revert(bytes memory returndata) private pure {
        if (returndata.length > 0) {
            assembly {
                let returndata_size := mload(returndata)
                revert(add(32, returndata), returndata_size)
            }
        } else {
            revert("Address: low-level call failed");
        }
    }
}

library Math {
    function max(uint256 a, uint256 b) internal pure returns (uint256) {
        return a > b ? a : b;
    }
    
    function min(uint256 a, uint256 b) internal pure returns (uint256) {
        return a < b ? a : b;
    }
    
    function average(uint256 a, uint256 b) internal pure returns (uint256) {
        return (a & b) + (a ^ b) / 2;
    }
    
    function sqrt(uint256 a) internal pure returns (uint256) {
        if (a == 0) return 0;
        
        uint256 result = 1 << (log2(a) >> 1);
        
        unchecked {
            result = (result + a / result) >> 1;
            result = (result + a / result) >> 1;
            result = (result + a / result) >> 1;
            result = (result + a / result) >> 1;
            result = (result + a / result) >> 1;
            result = (result + a / result) >> 1;
            result = (result + a / result) >> 1;
            return min(result, a / result);
        }
    }
    
    function log2(uint256 value) internal pure returns (uint256) {
        uint256 result = 0;
        unchecked {
            if (value >> 128 > 0) { value >>= 128; result += 128; }
            if (value >> 64 > 0) { value >>= 64; result += 64; }
            if (value >> 32 > 0) { value >>= 32; result += 32; }
            if (value >> 16 > 0) { value >>= 16; result += 16; }
            if (value >> 8 > 0) { value >>= 8; result += 8; }
            if (value >> 4 > 0) { value >>= 4; result += 4; }
            if (value >> 2 > 0) { value >>= 2; result += 2; }
            if (value >> 1 > 0) result += 1;
        }
        return result;
    }
    
    function mulDiv(uint256 x, uint256 y, uint256 denominator) internal pure returns (uint256 result) {
        uint256 prod0 = x * y;
        uint256 prod1;
        
        assembly {
            let mm := mulmod(x, y, not(0))
            prod1 := sub(sub(mm, prod0), lt(mm, prod0))
        }
        
        if (prod1 == 0) {
            require(denominator > 0);
            assembly { result := div(prod0, denominator) }
            return result;
        }
        
        require(denominator > prod1);
        
        uint256 remainder;
        assembly {
            remainder := mulmod(x, y, denominator)
            prod1 := sub(prod1, gt(remainder, prod0))
            prod0 := sub(prod0, remainder)
        }
        
        uint256 twos = denominator & (~denominator + 1);
        
        assembly {
            denominator := div(denominator, twos)
            prod0 := div(prod0, twos)
            twos := add(div(sub(0, twos), twos), 1)
        }
        
        prod0 |= prod1 * twos;
        
        uint256 inverse = (3 * denominator) ^ 2;
        
        inverse *= 2 - denominator * inverse;
        inverse *= 2 - denominator * inverse;
        inverse *= 2 - denominator * inverse;
        inverse *= 2 - denominator * inverse;
        inverse *= 2 - denominator * inverse;
        inverse *= 2 - denominator * inverse;
        
        result = prod0 * inverse;
        return result;
    }
}

// ใช้ Library
contract TokenWithLibrary {
    
    using SafeERC20 for IERC20;
    using Math for uint256;
    
    function swapTokens(
        IERC20 tokenIn,
        IERC20 tokenOut,
        address recipient,
        uint256 amountIn,
        uint256 amountOut
    ) external {
        tokenIn.safeTransferFrom(msg.sender, address(this), amountIn);
        tokenOut.safeTransfer(recipient, amountOut);
    }
    
    function calculateShares(
        uint256 amount,
        uint256 totalAssets,
        uint256 totalShares
    ) public pure returns (uint256) {
        if (totalAssets == 0) return amount;
        return amount.mulDiv(totalShares, totalAssets);
    }
    
    function getSqrt(uint256 n) public pure returns (uint256) {
        return n.sqrt();
    }
}
```

---

## 6. Workshop: DeFi Protocol Base

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// === Interfaces ===

interface IVault {
    event Deposit(address indexed caller, address indexed owner, uint256 assets, uint256 shares);
    event Withdraw(
        address indexed caller,
        address indexed receiver,
        address indexed owner,
        uint256 assets,
        uint256 shares
    );
    
    function asset() external view returns (address);
    function totalAssets() external view returns (uint256);
    function convertToShares(uint256 assets) external view returns (uint256);
    function convertToAssets(uint256 shares) external view returns (uint256);
    function maxDeposit(address receiver) external view returns (uint256);
    function previewDeposit(uint256 assets) external view returns (uint256);
    function deposit(uint256 assets, address receiver) external returns (uint256 shares);
    function maxWithdraw(address owner) external view returns (uint256);
    function previewWithdraw(uint256 assets) external view returns (uint256 shares);
    function withdraw(uint256 assets, address receiver, address owner) external returns (uint256 shares);
    function maxRedeem(address owner) external view returns (uint256);
    function previewRedeem(uint256 shares) external view returns (uint256);
    function redeem(uint256 shares, address receiver, address owner) external returns (uint256 assets);
}

interface IStrategy {
    function invest(uint256 amount) external returns (uint256);
    function divest(uint256 amount) external returns (uint256);
    function harvest() external returns (uint256 profit);
    function estimatedTotalAssets() external view returns (uint256);
}

// === Libraries ===

library WadMath {
    uint256 internal constant WAD = 1e18;
    uint256 internal constant RAY = 1e27;
    
    function wadMul(uint256 a, uint256 b) internal pure returns (uint256) {
        return (a * b + WAD / 2) / WAD;
    }
    
    function wadDiv(uint256 a, uint256 b) internal pure returns (uint256) {
        return (a * WAD + b / 2) / b;
    }
    
    function rayMul(uint256 a, uint256 b) internal pure returns (uint256) {
        return (a * b + RAY / 2) / RAY;
    }
    
    function rayDiv(uint256 a, uint256 b) internal pure returns (uint256) {
        return (a * RAY + b / 2) / b;
    }
    
    function rayToWad(uint256 a) internal pure returns (uint256) {
        return (a + 1e9 / 2) / 1e9;
    }
    
    function wadToRay(uint256 a) internal pure returns (uint256) {
        return a * 1e9;
    }
}

// === Base Contracts ===

abstract contract BaseVault is IVault, ERC20 {
    
    using WadMath for uint256;
    using SafeERC20 for IERC20;
    
    IERC20 private immutable _asset;
    
    constructor(IERC20 asset_, string memory name_, string memory symbol_)
        ERC20(name_, symbol_)
    {
        _asset = asset_;
    }
    
    function asset() public view virtual override returns (address) {
        return address(_asset);
    }
    
    function totalAssets() public view virtual override returns (uint256) {
        return _asset.balanceOf(address(this));
    }
    
    function convertToShares(uint256 assets) public view virtual override returns (uint256) {
        return _convertToShares(assets, Math.Rounding.Down);
    }
    
    function convertToAssets(uint256 shares) public view virtual override returns (uint256) {
        return _convertToAssets(shares, Math.Rounding.Down);
    }
    
    function maxDeposit(address) public view virtual override returns (uint256) {
        return type(uint256).max;
    }
    
    function previewDeposit(uint256 assets) public view virtual override returns (uint256) {
        return _convertToShares(assets, Math.Rounding.Down);
    }
    
    function deposit(uint256 assets, address receiver) public virtual override returns (uint256 shares) {
        require(assets <= maxDeposit(receiver), "ERC4626: deposit more than max");
        
        shares = previewDeposit(assets);
        _deposit(msg.sender, receiver, assets, shares);
        
        return shares;
    }
    
    function maxWithdraw(address owner) public view virtual override returns (uint256) {
        return _convertToAssets(balanceOf(owner), Math.Rounding.Down);
    }
    
    function previewWithdraw(uint256 assets) public view virtual override returns (uint256) {
        return _convertToShares(assets, Math.Rounding.Up);
    }
    
    function withdraw(
        uint256 assets,
        address receiver,
        address owner
    ) public virtual override returns (uint256 shares) {
        require(assets <= maxWithdraw(owner), "ERC4626: withdraw more than max");
        
        shares = previewWithdraw(assets);
        _withdraw(msg.sender, receiver, owner, assets, shares);
        
        return shares;
    }
    
    function maxRedeem(address owner) public view virtual override returns (uint256) {
        return balanceOf(owner);
    }
    
    function previewRedeem(uint256 shares) public view virtual override returns (uint256) {
        return _convertToAssets(shares, Math.Rounding.Down);
    }
    
    function redeem(
        uint256 shares,
        address receiver,
        address owner
    ) public virtual override returns (uint256 assets) {
        require(shares <= maxRedeem(owner), "ERC4626: redeem more than max");
        
        assets = previewRedeem(shares);
        _withdraw(msg.sender, receiver, owner, assets, shares);
        
        return assets;
    }
    
    function _convertToShares(uint256 assets, Math.Rounding rounding) internal view virtual returns (uint256) {
        return assets.mulDiv(totalSupply() + 10 ** decimals(), totalAssets() + 1, rounding);
    }
    
    function _convertToAssets(uint256 shares, Math.Rounding rounding) internal view virtual returns (uint256) {
        return shares.mulDiv(totalAssets() + 1, totalSupply() + 10 ** decimals(), rounding);
    }
    
    function _deposit(address caller, address receiver, uint256 assets, uint256 shares) internal virtual {
        _asset.safeTransferFrom(caller, address(this), assets);
        _mint(receiver, shares);
        emit Deposit(caller, receiver, assets, shares);
    }
    
    function _withdraw(
        address caller,
        address receiver,
        address owner,
        uint256 assets,
        uint256 shares
    ) internal virtual {
        if (caller != owner) {
            _spendAllowance(owner, caller, shares);
        }
        
        _burn(owner, shares);
        _asset.safeTransfer(receiver, assets);
        
        emit Withdraw(caller, receiver, owner, assets, shares);
    }
}

// === Concrete Vault with Strategy ===

contract StrategyVault is BaseVault, Ownable, Pausable {
    
    using SafeERC20 for IERC20;
    
    IStrategy public strategy;
    
    uint256 public performanceFee = 1000; // 10%
    uint256 public managementFee = 200;   // 2% per year
    uint256 public constant MAX_FEE = 5000; // 50% max
    
    uint256 public lastHarvest;
    address public feeRecipient;
    
    event StrategySet(address indexed strategy);
    event Harvested(uint256 profit, uint256 fee);
    event FeeUpdated(uint256 performance, uint256 management);
    
    constructor(
        IERC20 asset_,
        string memory name_,
        string memory symbol_,
        address feeRecipient_
    ) 
        BaseVault(asset_, name_, symbol_)
        Ownable(msg.sender)
    {
        feeRecipient = feeRecipient_;
        lastHarvest = block.timestamp;
    }
    
    function setStrategy(address newStrategy) external onlyOwner {
        require(newStrategy != address(0), "Zero address");
        strategy = IStrategy(newStrategy);
        emit StrategySet(newStrategy);
    }
    
    function harvest() external returns (uint256 profit) {
        require(address(strategy) != address(0), "No strategy");
        
        profit = strategy.harvest();
        
        if (profit > 0) {
            uint256 fee = (profit * performanceFee) / 10000;
            IERC20(asset()).safeTransfer(feeRecipient, fee);
            emit Harvested(profit, fee);
        }
        
        lastHarvest = block.timestamp;
    }
    
    function totalAssets() public view override returns (uint256) {
        uint256 idle = IERC20(asset()).balanceOf(address(this));
        uint256 invested = address(strategy) != address(0) 
            ? strategy.estimatedTotalAssets() 
            : 0;
        return idle + invested;
    }
    
    function setFees(uint256 performance, uint256 management) external onlyOwner {
        require(performance <= MAX_FEE && management <= MAX_FEE, "Fee too high");
        performanceFee = performance;
        managementFee = management;
        emit FeeUpdated(performance, management);
    }
    
    function _deposit(address caller, address receiver, uint256 assets, uint256 shares) 
        internal override whenNotPaused 
    {
        super._deposit(caller, receiver, assets, shares);
        
        // Invest in strategy if set
        if (address(strategy) != address(0)) {
            uint256 balance = IERC20(asset()).balanceOf(address(this));
            if (balance > 0) {
                IERC20(asset()).safeApprove(address(strategy), balance);
                strategy.invest(balance);
            }
        }
    }
    
    function _withdraw(
        address caller,
        address receiver,
        address owner,
        uint256 assets,
        uint256 shares
    ) internal override whenNotPaused {
        // Pull from strategy if needed
        uint256 idle = IERC20(asset()).balanceOf(address(this));
        if (idle < assets && address(strategy) != address(0)) {
            uint256 needed = assets - idle;
            strategy.divest(needed);
        }
        
        super._withdraw(caller, receiver, owner, assets, shares);
    }
    
    function pause() external onlyOwner { _pause(); }
    function unpause() external onlyOwner { _unpause(); }
}

// Placeholder imports for compilation
library Math {
    enum Rounding { Down, Up, Zero }
    
    function mulDiv(uint256 x, uint256 y, uint256 denominator, Rounding rounding) 
        internal pure returns (uint256 result) 
    {
        result = mulDiv(x, y, denominator);
        if (rounding == Rounding.Up && mulmod(x, y, denominator) > 0) {
            result += 1;
        }
    }
    
    function mulDiv(uint256 x, uint256 y, uint256 denominator) internal pure returns (uint256) {
        unchecked {
            uint256 prod0 = x * y;
            uint256 prod1;
            assembly {
                let mm := mulmod(x, y, not(0))
                prod1 := sub(sub(mm, prod0), lt(mm, prod0))
            }
            if (prod1 == 0) return prod0 / denominator;
            require(denominator > prod1, "Math: mulDiv overflow");
            uint256 remainder;
            assembly { remainder := mulmod(x, y, denominator) }
            assembly { prod1 := sub(prod1, gt(remainder, prod0)) prod0 := sub(prod0, remainder) }
            uint256 twos = denominator & (~denominator + 1);
            assembly { denominator := div(denominator, twos) prod0 := div(prod0, twos) twos := add(div(sub(0, twos), twos), 1) }
            prod0 |= prod1 * twos;
            uint256 inverse = (3 * denominator) ^ 2;
            inverse *= 2 - denominator * inverse;
            inverse *= 2 - denominator * inverse;
            inverse *= 2 - denominator * inverse;
            inverse *= 2 - denominator * inverse;
            inverse *= 2 - denominator * inverse;
            inverse *= 2 - denominator * inverse;
            result = prod0 * inverse;
        }
    }
}
```

---

## สรุป Part 09

Inheritance และ Interfaces ที่เรียนรู้:
- ✅ Single inheritance
- ✅ Multiple inheritance (C3 linearization)
- ✅ Abstract contracts
- ✅ Interfaces (ERC-20, ERC-721)
- ✅ Libraries (SafeERC20, Math)
- ✅ Super calls
- ✅ Override rules
- ✅ DeFi Protocol Base (ERC-4626 Vault)

## Quiz

1. C3 Linearization คืออะไร และทำไมสำคัญ?
2. ต่างกันอย่างไรระหว่าง Abstract Contract และ Interface?
3. Library functions เป็น `internal` vs `external` มีผลอะไร?
4. `using SafeERC20 for IERC20` ทำงานอย่างไร?

---

## Next: Part 10 - Contract Deployment
