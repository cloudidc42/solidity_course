# Part 85: Liquid Staking Protocols (Liquid Staking โปรโตคอล)

## บทนำ

**Liquid Staking** แก้ปัญหาหลักของ Ethereum staking ดั้งเดิม คือการที่ ETH ถูกล็อคและไม่สามารถใช้งานได้ในระหว่างที่ stake อยู่ ด้วย Liquid Staking ผู้ใช้จะได้รับ **Liquid Staking Token (LST)** ที่แทนมูลค่า staked ETH พร้อม rewards สะสม และสามารถนำไปใช้ใน DeFi protocol ต่างๆ ได้

**ข้อมูลตลาด 2024:**
- Lido Finance: ~$35B TVL, ครองตลาด ~30% ของ ETH ที่ stake ทั้งหมด
- Rocket Pool: ~$4B TVL, เน้น decentralization
- stETH เป็น collateral อันดับ 1 ใน Aave/MakerDAO

---

## 85.1 Lido Architecture

### สถาปัตยกรรมหลัก

```
Lido Architecture:
                                                        
User             Lido Contract        Oracle Committee   Beacon Chain
────             ─────────────        ────────────────   ────────────
                                                        
Stake ETH ──────► stETH minted                         
                  (rebasing)                            
                  │                                     
                  ▼                                     
                NodeOperators ◄── Approval ◄──────      Validators run
                (stake to          (DAO)                by Node Operators
                 validators)                            
                  │                                     
                  ▼                                     
                Oracle ◄── Report ◄─────────────────── Beacon State
                (daily rebase)     (32 Oracles)         (rewards)
                  │                                     
                  ▼                                     
                Rebase stETH ────────────────────────── Balance update
                (increase                               (all holders)
                 balance)                               
```

### 85.1.1 stETH Rebase Mechanics

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title stETH Rebase Mechanics (Simplified)
 * @notice stETH ใช้ "shares" model สำหรับ rebase
 * @dev แทนที่จะเก็บ balance โดยตรง เก็บ "shares"
 *      เมื่อ rebase เกิดขึ้น totalPooledEther เพิ่มขึ้น
 *      แต่ shares คงเดิม → balance ของแต่ละคนเพิ่มขึ้นอัตโนมัติ
 */
contract StETH {
    
    // ============ Storage ============
    
    // Total ETH ใน pool (รวม staking rewards)
    uint256 private _totalPooledEther;
    
    // Total shares ที่ outstanding
    uint256 private _totalShares;
    
    // User shares (ไม่ใช่ balance!)
    mapping(address => uint256) private _shares;
    
    // Allowances (ใช้ shares หรือ tokens?)
    // stETH ใช้ token-based allowance
    mapping(address => mapping(address => uint256)) private _allowances;
    
    // ============ Events ============
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    event TransferShares(address indexed from, address indexed to, uint256 sharesValue);
    event Rebase(
        uint256 preTotalShares,
        uint256 preTotalEther,
        uint256 postTotalShares,
        uint256 postTotalEther,
        uint256 sharesMintedAsFees
    );
    
    // ============ ERC-20 Compatible Functions ============
    
    function name() public pure returns (string memory) { return "Liquid staked Ether 2.0"; }
    function symbol() public pure returns (string memory) { return "stETH"; }
    function decimals() public pure returns (uint8) { return 18; }
    
    /**
     * @notice Total supply ใน ETH terms
     * @dev เพิ่มขึ้นตาม staking rewards
     */
    function totalSupply() public view returns (uint256) {
        return _totalPooledEther;
    }
    
    /**
     * @notice Balance ของ user ใน ETH terms
     * @dev คำนวณจาก shares * pooledEther / totalShares
     *      เพิ่มขึ้นอัตโนมัติเมื่อ rebase
     */
    function balanceOf(address _account) public view returns (uint256) {
        uint256 userShares = _shares[_account];
        return _getPooledEthByShares(userShares);
    }
    
    /**
     * @notice โอน stETH (โดย amount เป็น ETH terms)
     */
    function transfer(address _recipient, uint256 _amount) public returns (bool) {
        _transfer(msg.sender, _recipient, _amount);
        return true;
    }
    
    function approve(address _spender, uint256 _amount) public returns (bool) {
        _allowances[msg.sender][_spender] = _amount;
        emit Approval(msg.sender, _spender, _amount);
        return true;
    }
    
    function transferFrom(
        address _sender,
        address _recipient,
        uint256 _amount
    ) public returns (bool) {
        uint256 currentAllowance = _allowances[_sender][msg.sender];
        require(currentAllowance >= _amount, "Insufficient allowance");
        
        _allowances[_sender][msg.sender] = currentAllowance - _amount;
        _transfer(_sender, _recipient, _amount);
        return true;
    }
    
    function allowance(address _owner, address _spender) public view returns (uint256) {
        return _allowances[_owner][_spender];
    }
    
    // ============ Shares Functions ============
    
    /**
     * @notice Shares ของ user (ค่าคงที่ ไม่ rebase)
     */
    function sharesOf(address _account) public view returns (uint256) {
        return _shares[_account];
    }
    
    /**
     * @notice แปลง ETH amount เป็น shares
     */
    function getSharesByPooledEth(uint256 _ethAmount) public view returns (uint256) {
        return _getSharesByPooledEth(_ethAmount);
    }
    
    /**
     * @notice แปลง shares เป็น ETH amount
     */
    function getPooledEthByShares(uint256 _sharesAmount) public view returns (uint256) {
        return _getPooledEthByShares(_sharesAmount);
    }
    
    /**
     * @notice โอน shares โดยตรง (สำหรับ protocol integrations)
     * @dev ใช้เพื่อหลีกเลี่ยง rebasing accounting issues
     */
    function transferShares(
        address _recipient,
        uint256 _sharesAmount
    ) public returns (uint256 tokensAmount) {
        _transferShares(msg.sender, _recipient, _sharesAmount);
        tokensAmount = _getPooledEthByShares(_sharesAmount);
        emit Transfer(msg.sender, _recipient, tokensAmount);
        return tokensAmount;
    }
    
    // ============ Internal ============
    
    function _transfer(
        address _sender,
        address _recipient,
        uint256 _amount
    ) internal {
        uint256 _sharesToTransfer = _getSharesByPooledEth(_amount);
        _transferShares(_sender, _recipient, _sharesToTransfer);
        emit Transfer(_sender, _recipient, _amount);
    }
    
    function _transferShares(
        address _sender,
        address _recipient,
        uint256 _sharesAmount
    ) internal {
        require(_sender != address(0), "Zero sender");
        require(_recipient != address(0), "Zero recipient");
        require(_shares[_sender] >= _sharesAmount, "Insufficient shares");
        
        _shares[_sender] -= _sharesAmount;
        _shares[_recipient] += _sharesAmount;
        
        emit TransferShares(_sender, _recipient, _sharesAmount);
    }
    
    function _getSharesByPooledEth(uint256 _ethAmount) internal view returns (uint256) {
        return _totalPooledEther == 0 
            ? _ethAmount 
            : _ethAmount * _totalShares / _totalPooledEther;
    }
    
    function _getPooledEthByShares(uint256 _sharesAmount) internal view returns (uint256) {
        return _totalShares == 0 
            ? _sharesAmount 
            : _sharesAmount * _totalPooledEther / _totalShares;
    }
    
    // ============ Oracle/Admin Functions (simplified) ============
    
    /**
     * @notice Oracle รายงาน rewards ใหม่ → rebase
     */
    function handleOracleReport(
        uint256 _reportTimestamp,
        uint256 _timeElapsed,
        uint256 _clBalanceDiff,      // CL rewards
        uint256 _withdrawalVaultBalance,
        uint256 _elRewardsVaultBalance,
        uint256 _sharesRequestedToBurn,
        uint256 _withdrawalFinalizationBatches,
        uint256 _simulatedShareRate
    ) external {
        // ในการใช้งานจริง: ตรวจสอบว่าเรียกโดย oracle committee
        
        uint256 preTotalShares = _totalShares;
        uint256 preTotalEther = _totalPooledEther;
        
        // เพิ่ม ETH ใน pool (rewards)
        _totalPooledEther += _clBalanceDiff + _elRewardsVaultBalance;
        
        // Fees: 10% ของ rewards ไปยัง treasury + node operators
        uint256 totalFees = (_clBalanceDiff * 10) / 100;
        uint256 sharesMintedAsFees = _getSharesByPooledEth(totalFees);
        
        // Mint shares สำหรับ fees
        _shares[address(this)] += sharesMintedAsFees;
        _totalShares += sharesMintedAsFees;
        
        emit Rebase(
            preTotalShares, 
            preTotalEther,
            _totalShares,
            _totalPooledEther,
            sharesMintedAsFees
        );
    }
}
```

### 85.1.2 Withdrawal Queue

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title WithdrawalQueue
 * @notice จัดการ queue สำหรับถอน stETH → ETH
 * @dev Lido ใช้ withdrawal queue เพื่อจัดการ:
 *      1. Request withdrawal (burn stETH, รับ NFT)
 *      2. รอ ETH จาก beacon chain
 *      3. Finalize batch (เมื่อมี ETH พอ)
 *      4. Claim ETH (แลก NFT → ETH)
 */
contract WithdrawalQueue {
    
    struct WithdrawalRequest {
        uint128 amountOfStETH;      // stETH ที่ request
        uint128 amountOfShares;     // shares ที่ burn
        address owner;              // เจ้าของ
        uint40 timestamp;           // เวลาที่ request
        bool claimed;               // claimed แล้วหรือยัง
        uint256 ethClaimable;       // ETH ที่จะได้รับ (set เมื่อ finalize)
    }
    
    WithdrawalRequest[] public queue;
    uint256 public lastFinalizedRequestId;
    uint256 public lastRequestId;
    
    uint256 public MIN_STETH_WITHDRAWAL_AMOUNT = 100 wei;
    uint256 public MAX_STETH_WITHDRAWAL_AMOUNT = 1000 ether;
    
    StETH public immutable stETH;
    
    // NFT tracking (แทน ERC-721 แบบง่าย)
    mapping(uint256 => address) public nftOwners;
    
    event WithdrawalRequested(
        uint256 indexed requestId,
        address indexed requestor,
        address indexed owner,
        uint256 amountOfStETH,
        uint256 amountOfShares
    );
    
    event WithdrawalClaimed(
        uint256 indexed requestId,
        address indexed owner,
        address indexed receiver,
        uint256 amountOfETH
    );
    
    event WithdrawalsFinalized(
        uint256 indexed from,
        uint256 indexed to,
        uint256 amountOfETHLocked,
        uint256 sharesToBurn,
        uint256 timestamp
    );
    
    constructor(address _stETH) {
        stETH = StETH(_stETH);
        // Request ID เริ่มที่ 1
        queue.push(); // index 0 เป็น placeholder
        lastRequestId = 0;
        lastFinalizedRequestId = 0;
    }
    
    /**
     * @notice ขอถอน stETH หลายๆ จำนวนพร้อมกัน
     * @param _amounts จำนวน stETH ที่ต้องการถอน
     * @param _owner เจ้าของ NFT ที่จะรับ
     * @return requestIds request IDs
     */
    function requestWithdrawals(
        uint256[] calldata _amounts,
        address _owner
    ) external returns (uint256[] memory requestIds) {
        requestIds = new uint256[](_amounts.length);
        
        for (uint256 i = 0; i < _amounts.length; i++) {
            requestIds[i] = _requestWithdrawal(_amounts[i], _owner);
        }
    }
    
    function _requestWithdrawal(
        uint256 _amountOfStETH,
        address _owner
    ) internal returns (uint256 requestId) {
        require(
            _amountOfStETH >= MIN_STETH_WITHDRAWAL_AMOUNT,
            "Amount too small"
        );
        require(
            _amountOfStETH <= MAX_STETH_WITHDRAWAL_AMOUNT,
            "Amount too large"
        );
        
        // Convert stETH amount → shares
        uint256 amountOfShares = stETH.getSharesByPooledEth(_amountOfStETH);
        
        // ดึง stETH จาก user
        stETH.transferFrom(msg.sender, address(this), _amountOfStETH);
        
        requestId = ++lastRequestId;
        
        queue.push(WithdrawalRequest({
            amountOfStETH: uint128(_amountOfStETH),
            amountOfShares: uint128(amountOfShares),
            owner: _owner,
            timestamp: uint40(block.timestamp),
            claimed: false,
            ethClaimable: 0
        }));
        
        // Mint NFT
        nftOwners[requestId] = _owner;
        
        emit WithdrawalRequested(
            requestId,
            msg.sender,
            _owner,
            _amountOfStETH,
            amountOfShares
        );
    }
    
    /**
     * @notice Finalize batch (เรียกโดย oracle หลังได้รับ ETH จาก validator)
     * @param _lastRequestIdToBeFinalized request ID สุดท้ายที่จะ finalize
     * @param _maxShareRate share rate สูงสุดที่จะใช้คำนวณ
     */
    function finalize(
        uint256 _lastRequestIdToBeFinalized,
        uint256 _maxShareRate
    ) external payable {
        // ในการใช้งานจริง: เรียกโดย oracle/admin เท่านั้น
        require(
            _lastRequestIdToBeFinalized > lastFinalizedRequestId,
            "Already finalized"
        );
        require(
            _lastRequestIdToBeFinalized <= lastRequestId,
            "Invalid request id"
        );
        
        uint256 ethToLock = 0;
        uint256 sharesToBurn = 0;
        
        for (uint256 i = lastFinalizedRequestId + 1; 
             i <= _lastRequestIdToBeFinalized; 
             i++) {
            WithdrawalRequest storage req = queue[i];
            
            // คำนวณ ETH ที่จะได้ตาม share rate
            uint256 ethAmount = (uint256(req.amountOfShares) * _maxShareRate) / 1e27;
            
            // ไม่ให้เกิน original stETH amount
            if (ethAmount > uint256(req.amountOfStETH)) {
                ethAmount = uint256(req.amountOfStETH);
            }
            
            req.ethClaimable = ethAmount;
            ethToLock += ethAmount;
            sharesToBurn += uint256(req.amountOfShares);
        }
        
        require(msg.value >= ethToLock, "Insufficient ETH");
        
        lastFinalizedRequestId = _lastRequestIdToBeFinalized;
        
        emit WithdrawalsFinalized(
            lastFinalizedRequestId + 1,
            _lastRequestIdToBeFinalized,
            ethToLock,
            sharesToBurn,
            block.timestamp
        );
    }
    
    /**
     * @notice Claim ETH สำหรับ request ที่ finalize แล้ว
     * @param _requestIds request IDs ที่ต้องการ claim
     * @param _recipient ที่อยู่ที่รับ ETH
     */
    function claimWithdrawals(
        uint256[] calldata _requestIds,
        address payable _recipient
    ) external {
        for (uint256 i = 0; i < _requestIds.length; i++) {
            _claimWithdrawal(_requestIds[i], _recipient);
        }
    }
    
    function _claimWithdrawal(
        uint256 _requestId,
        address payable _recipient
    ) internal {
        require(_requestId <= lastFinalizedRequestId, "Not finalized yet");
        
        WithdrawalRequest storage req = queue[_requestId];
        require(nftOwners[_requestId] == msg.sender, "Not NFT owner");
        require(!req.claimed, "Already claimed");
        
        req.claimed = true;
        
        uint256 ethAmount = req.ethClaimable;
        require(ethAmount > 0, "No ETH to claim");
        
        // Burn NFT
        delete nftOwners[_requestId];
        
        // โอน ETH
        (bool success,) = _recipient.call{value: ethAmount}("");
        require(success, "ETH transfer failed");
        
        emit WithdrawalClaimed(_requestId, msg.sender, _recipient, ethAmount);
    }
    
    // View functions
    
    function getWithdrawalStatus(
        uint256[] calldata _requestIds
    ) external view returns (
        uint256[] memory amountsOfStETH,
        uint256[] memory amountsOfShares,
        bool[] memory isFinalizeds,
        uint256[] memory timestamps,
        bool[] memory isClaimed
    ) {
        amountsOfStETH = new uint256[](_requestIds.length);
        amountsOfShares = new uint256[](_requestIds.length);
        isFinalizeds = new bool[](_requestIds.length);
        timestamps = new uint256[](_requestIds.length);
        isClaimed = new bool[](_requestIds.length);
        
        for (uint256 i = 0; i < _requestIds.length; i++) {
            WithdrawalRequest memory req = queue[_requestIds[i]];
            amountsOfStETH[i] = req.amountOfStETH;
            amountsOfShares[i] = req.amountOfShares;
            isFinalizeds[i] = _requestIds[i] <= lastFinalizedRequestId;
            timestamps[i] = req.timestamp;
            isClaimed[i] = req.claimed;
        }
    }
}
```

---

## 85.2 Rocket Pool: Minipool Concept

### สถาปัตยกรรม Rocket Pool

```
Rocket Pool Architecture:
                                           
Node Operators              Stakers              Protocol
──────────────              ───────              ────────
                                                 
Provide 16 ETH ──────►  ┌──────────────┐
+ RPL collateral         │   Minipool   │
                         │  (32 ETH    │◄──── 16 ETH from stakers
                         │   validator)│
                         └──────┬───────┘
                                │ stake
                         Beacon Chain
                         Validator
                                │ rewards
                         ┌──────▼───────┐
                         │  rETH        │◄──── Stakers get rETH
                         │  (receipt    │      (appreciating, not rebasing)
                         │   token)     │
                         └──────────────┘
                                
Node Operators: ต้องมี RPL ≥ 10% ของ ETH value
                ได้ commission 14% ของ staking rewards
```

### 85.2.1 rETH Token (Non-Rebasing)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title RocketETH (rETH)
 * @notice Rocket Pool's Liquid Staking Token
 * @dev รETH เป็น appreciating token (ไม่ใช่ rebasing)
 *      Exchange rate ของ rETH:ETH เพิ่มขึ้นตามเวลา
 *      ข้อแตกต่างจาก stETH:
 *      - stETH: balance เพิ่ม (rebasing)
 *      - rETH: exchange rate เพิ่ม (appreciating)
 */
contract RocketETH is ERC20, Ownable {
    
    // ============ State ============
    
    // Total ETH ใน Rocket Pool
    uint256 public totalETHBalance;
    
    // Protocol contracts
    address public rocketPoolDeposit;
    address public rocketPoolWithdraw;
    
    // ============ Events ============
    
    event ETHToRETHConversion(uint256 ethAmount, uint256 rethAmount);
    event RETHToETHConversion(uint256 rethAmount, uint256 ethAmount);
    event ExchangeRateUpdated(uint256 newRate);
    
    // ============ Constructor ============
    
    constructor() ERC20("Rocket Pool ETH", "rETH") Ownable(msg.sender) {}
    
    // ============ Exchange Rate ============
    
    /**
     * @notice Exchange rate ปัจจุบัน: ETH ต่อ 1 rETH
     * @dev เพิ่มขึ้นตามเวลาเมื่อ rewards สะสม
     *      เริ่มต้น: 1 rETH = 1 ETH
     *      หลัง 1 ปี: 1 rETH ≈ 1.04 ETH (4% staking APY)
     */
    function getExchangeRate() public view returns (uint256) {
        uint256 supply = totalSupply();
        if (supply == 0) {
            return 1 ether; // 1:1 เมื่อเริ่มต้น
        }
        return (totalETHBalance * 1 ether) / supply;
    }
    
    /**
     * @notice แปลง ETH amount เป็น rETH amount
     */
    function getRethValue(uint256 _ethAmount) public view returns (uint256) {
        uint256 rate = getExchangeRate();
        return (_ethAmount * 1 ether) / rate;
    }
    
    /**
     * @notice แปลง rETH amount เป็น ETH amount
     */
    function getEthValue(uint256 _rethAmount) public view returns (uint256) {
        uint256 rate = getExchangeRate();
        return (_rethAmount * rate) / 1 ether;
    }
    
    // ============ Deposit/Withdraw ============
    
    /**
     * @notice Stake ETH → รับ rETH
     */
    function deposit() external payable returns (uint256 rethMinted) {
        require(msg.value > 0, "No ETH sent");
        
        // คำนวณ rETH ที่จะ mint
        rethMinted = getRethValue(msg.value);
        require(rethMinted > 0, "Zero rETH");
        
        // อัปเดต total ETH
        totalETHBalance += msg.value;
        
        // Mint rETH
        _mint(msg.sender, rethMinted);
        
        emit ETHToRETHConversion(msg.value, rethMinted);
    }
    
    /**
     * @notice Burn rETH → รับ ETH
     * @dev ต้องรอ unbonding period ใน production
     */
    function withdraw(uint256 _rethAmount) external returns (uint256 ethAmount) {
        require(_rethAmount > 0, "Zero rETH");
        require(balanceOf(msg.sender) >= _rethAmount, "Insufficient rETH");
        
        // คำนวณ ETH ที่จะได้
        ethAmount = getEthValue(_rethAmount);
        require(address(this).balance >= ethAmount, "Insufficient ETH liquidity");
        
        // อัปเดต total ETH
        totalETHBalance -= ethAmount;
        
        // Burn rETH
        _burn(msg.sender, _rethAmount);
        
        // โอน ETH
        (bool success,) = msg.sender.call{value: ethAmount}("");
        require(success, "ETH transfer failed");
        
        emit RETHToETHConversion(_rethAmount, ethAmount);
    }
    
    /**
     * @notice อัปเดต total ETH (เรียกโดย oracle หลัง rewards)
     */
    function updateTotalETHBalance(uint256 _newTotalETH) external onlyOwner {
        require(_newTotalETH >= totalETHBalance || totalSupply() == 0, "ETH cannot decrease");
        totalETHBalance = _newTotalETH;
        emit ExchangeRateUpdated(getExchangeRate());
    }
    
    receive() external payable {
        totalETHBalance += msg.value;
    }
}
```

### 85.2.2 Minipool Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title RocketMinipool
 * @notice Minipool contract สำหรับแต่ละ validator
 * @dev แต่ละ minipool:
 *      - Node Operator ฝาก 16 ETH
 *      - Protocol ฝาก 16 ETH (จาก stakers)
 *      - รวม 32 ETH → ส่งไปยัง Beacon Chain
 */
contract RocketMinipool {
    
    enum MinipoolStatus {
        Initialized,        // สร้างแล้ว รอ deposit
        Prelaunch,          // มี ETH ครบ รอ launch
        Staking,            // กำลัง stake บน Beacon Chain
        Withdrawable,       // ถอนได้แล้ว (หลัง exit)
        Dissolved           // ยกเลิก (ไม่ได้ launch)
    }
    
    struct MinipoolDetails {
        address nodeAddress;        // Node operator
        uint256 nodeFee;            // Commission rate (bps)
        uint256 nodeDepositBalance; // ETH ที่ node deposit
        uint256 nodeRefundBalance;  // ETH ที่คืน node
        uint256 userDepositBalance; // ETH ที่ users deposit
        MinipoolStatus status;
        uint256 statusBlock;
        uint256 statusTime;
        bool finalised;
        address rocketPoolAddress;
        bytes validatorPubkey;
    }
    
    MinipoolDetails public details;
    
    // ============ Events ============
    
    event MinipoolStatusUpdated(MinipoolStatus indexed status, uint256 time);
    event EtherDeposited(address indexed from, uint256 amount, uint256 time);
    event EtherWithdrawn(address indexed to, uint256 amount, uint256 time);
    
    // ============ Constants ============
    
    uint256 constant NODE_DEPOSIT_SIZE = 16 ether;
    uint256 constant USER_DEPOSIT_SIZE = 16 ether;
    uint256 constant FULL_DEPOSIT_SIZE = 32 ether;
    
    // ============ Constructor ============
    
    constructor(
        address _nodeAddress,
        uint256 _nodeFee,
        address _rocketPoolAddress
    ) payable {
        details.nodeAddress = _nodeAddress;
        details.nodeFee = _nodeFee;
        details.rocketPoolAddress = _rocketPoolAddress;
        details.status = MinipoolStatus.Initialized;
        details.statusTime = block.timestamp;
    }
    
    // ============ Deposit Functions ============
    
    /**
     * @notice Node operator deposit 16 ETH
     */
    function nodeDeposit(bytes calldata _validatorPubkey) external payable {
        require(msg.sender == details.nodeAddress, "Not node operator");
        require(msg.value == NODE_DEPOSIT_SIZE, "Invalid deposit amount");
        require(details.status == MinipoolStatus.Initialized, "Wrong status");
        
        details.nodeDepositBalance = NODE_DEPOSIT_SIZE;
        details.validatorPubkey = _validatorPubkey;
        
        emit EtherDeposited(msg.sender, msg.value, block.timestamp);
        
        // ถ้ามี user deposit แล้ว → prelaunch
        if (details.userDepositBalance == USER_DEPOSIT_SIZE) {
            _setStatus(MinipoolStatus.Prelaunch);
        }
    }
    
    /**
     * @notice Protocol deposit 16 ETH (จาก user stakers)
     */
    function userDeposit() external payable {
        require(msg.sender == details.rocketPoolAddress, "Not Rocket Pool");
        require(msg.value == USER_DEPOSIT_SIZE, "Invalid deposit amount");
        require(
            details.status == MinipoolStatus.Initialized ||
            details.status == MinipoolStatus.Prelaunch,
            "Wrong status"
        );
        
        details.userDepositBalance = USER_DEPOSIT_SIZE;
        
        emit EtherDeposited(msg.sender, msg.value, block.timestamp);
        
        // ถ้ามี node deposit แล้ว → prelaunch
        if (details.nodeDepositBalance == NODE_DEPOSIT_SIZE) {
            _setStatus(MinipoolStatus.Prelaunch);
        }
    }
    
    /**
     * @notice Launch validator (ส่ง 32 ETH ไปยัง Beacon Chain)
     */
    function launch(
        bytes calldata _validatorSignature,
        bytes32 _depositDataRoot
    ) external {
        require(msg.sender == details.rocketPoolAddress, "Not Rocket Pool");
        require(details.status == MinipoolStatus.Prelaunch, "Wrong status");
        require(address(this).balance >= FULL_DEPOSIT_SIZE, "Insufficient ETH");
        
        // ส่ง 32 ETH ไปยัง Beacon Deposit Contract
        // IBeaconDeposit(BEACON_DEPOSIT_CONTRACT).deposit{value: 32 ether}(
        //     details.validatorPubkey,
        //     withdrawalCredentials,
        //     _validatorSignature,
        //     _depositDataRoot
        // );
        
        _setStatus(MinipoolStatus.Staking);
    }
    
    /**
     * @notice Distribute rewards หลัง validator exit
     * @dev แบ่งตาม commission rate
     */
    function distributeBalance() external {
        require(details.status == MinipoolStatus.Withdrawable, "Wrong status");
        require(!details.finalised, "Already finalised");
        
        uint256 totalBalance = address(this).balance;
        require(totalBalance > 0, "No balance");
        
        // คำนวณ rewards
        uint256 nodeCommission = 0;
        uint256 nodeShare = NODE_DEPOSIT_SIZE;
        uint256 userShare = USER_DEPOSIT_SIZE;
        
        if (totalBalance > FULL_DEPOSIT_SIZE) {
            // มี rewards
            uint256 totalRewards = totalBalance - FULL_DEPOSIT_SIZE;
            
            // Node gets commission ของ user's portion
            uint256 userRewards = (totalRewards * USER_DEPOSIT_SIZE) / FULL_DEPOSIT_SIZE;
            nodeCommission = (userRewards * details.nodeFee) / 10000;
            
            nodeShare = NODE_DEPOSIT_SIZE + 
                       (totalRewards * NODE_DEPOSIT_SIZE) / FULL_DEPOSIT_SIZE +
                       nodeCommission;
            
            userShare = totalBalance - nodeShare;
        } else if (totalBalance < FULL_DEPOSIT_SIZE) {
            // Slashing เกิดขึ้น
            uint256 loss = FULL_DEPOSIT_SIZE - totalBalance;
            // Node operator รับ loss ก่อน (ลด nodeShare)
            nodeShare = NODE_DEPOSIT_SIZE > loss ? NODE_DEPOSIT_SIZE - loss : 0;
            userShare = totalBalance - nodeShare;
        }
        
        details.finalised = true;
        
        // โอน shares
        if (nodeShare > 0) {
            (bool success1,) = details.nodeAddress.call{value: nodeShare}("");
            require(success1, "Node transfer failed");
            emit EtherWithdrawn(details.nodeAddress, nodeShare, block.timestamp);
        }
        
        if (userShare > 0) {
            (bool success2,) = details.rocketPoolAddress.call{value: userShare}("");
            require(success2, "Protocol transfer failed");
            emit EtherWithdrawn(details.rocketPoolAddress, userShare, block.timestamp);
        }
    }
    
    function _setStatus(MinipoolStatus _status) internal {
        details.status = _status;
        details.statusBlock = block.number;
        details.statusTime = block.timestamp;
        emit MinipoolStatusUpdated(_status, block.timestamp);
    }
    
    receive() external payable {
        emit EtherDeposited(msg.sender, msg.value, block.timestamp);
    }
}
```

---

## 85.3 stETH Integration Pitfalls

### ปัญหาหลักของ stETH (Rebasing Token)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title stETH Integration Issues & Solutions
 * @notice แสดงปัญหาและวิธีแก้ไขการใช้ stETH ใน DeFi
 */

// ============ ปัญหาที่ 1: Snapshot Balance ============

/**
 * @dev ปัญหา: protocol บันทึก balance ณ เวลา deposit
 *      เมื่อ rebase เกิด balance เพิ่ม แต่ protocol ยังคิดตามเดิม
 */
contract BrokenStETHVault {
    mapping(address => uint256) public deposits; // ผิด! ใช้ ETH amount
    
    IERC20 public stETH;
    
    function deposit(uint256 _amount) external {
        stETH.transferFrom(msg.sender, address(this), _amount);
        deposits[msg.sender] += _amount; // บันทึก ETH amount
        // ปัญหา: หลัง rebase balance เพิ่ม
        //        แต่ deposits[user] ยังเป็นค่าเดิม
        //        ส่วนต่างนั้นจะสูญหาย!
    }
    
    function withdraw(uint256 _amount) external {
        require(deposits[msg.sender] >= _amount, "Insufficient");
        deposits[msg.sender] -= _amount;
        stETH.transfer(msg.sender, _amount);
        // ปัญหา: ถ้า rebase เกิดขึ้น user ได้ค่าเก่า
        //        rewards ทั้งหมดสูญหายใน contract
    }
}

/**
 * @dev วิธีแก้: ใช้ shares แทน amounts
 */
contract CorrectStETHVault {
    mapping(address => uint256) public shareDeposits; // ถูก! ใช้ shares
    
    IStETH public stETH; // interface ที่มี shares functions
    
    function deposit(uint256 _amount) external {
        // แปลงเป็น shares ก่อนบันทึก
        uint256 shares = stETH.getSharesByPooledEth(_amount);
        stETH.transferFrom(msg.sender, address(this), _amount);
        shareDeposits[msg.sender] += shares;
        // ✓ shares คงที่, rewards จะสะสมใน contract value
    }
    
    function withdraw() external {
        uint256 shares = shareDeposits[msg.sender];
        require(shares > 0, "No deposit");
        
        shareDeposits[msg.sender] = 0;
        
        // โอน shares โดยตรง (รวม rewards)
        stETH.transferShares(msg.sender, shares);
        // ✓ user ได้รับ original deposit + rewards
    }
}

// ============ ปัญหาที่ 2: ERC-20 Approval ============

/**
 * @dev ปัญหา: approve(1000) แล้ว rebase เพิ่มเป็น 1005
 *      allowance ยังเป็น 1000 → protocol ใช้ได้แค่ 1000
 *      
 * วิธีแก้: ใช้ wstETH (wrapped stETH) แทน
 * wstETH เป็น non-rebasing, exchange rate เพิ่มขึ้นแทน
 */

interface IWstETH {
    function wrap(uint256 _stETHAmount) external returns (uint256);
    function unwrap(uint256 _wstETHAmount) external returns (uint256);
    function stEthPerToken() external view returns (uint256);
    function tokensPerStEth() external view returns (uint256);
    function getWstETHByStETH(uint256 _stETHAmount) external view returns (uint256);
    function getStETHByWstETH(uint256 _wstETHAmount) external view returns (uint256);
}

/**
 * @title SafeStETHVault
 * @notice Vault ที่ใช้ wstETH ภายใน เพื่อหลีกเลี่ยงปัญหา rebasing
 */
contract SafeStETHVault {
    using SafeERC20 for IERC20;
    
    IERC20 public immutable stETH;
    IWstETH public immutable wstETH;
    
    // ใช้ wstETH เป็น accounting unit
    mapping(address => uint256) public wstETHBalances;
    uint256 public totalWstETH;
    
    event DepositedSTETH(address indexed user, uint256 stETHAmount, uint256 wstETHReceived);
    event WithdrawnSTETH(address indexed user, uint256 wstETHAmount, uint256 stETHReceived);
    
    constructor(address _stETH, address _wstETH) {
        stETH = IERC20(_stETH);
        wstETH = IWstETH(_wstETH);
    }
    
    /**
     * @notice ฝาก stETH → แปลงเป็น wstETH ทันที
     */
    function depositStETH(uint256 _stETHAmount) external {
        require(_stETHAmount > 0, "Zero amount");
        
        // ดึง stETH จาก user
        stETH.safeTransferFrom(msg.sender, address(this), _stETHAmount);
        
        // Approve wstETH contract
        stETH.safeIncreaseAllowance(address(wstETH), _stETHAmount);
        
        // แปลงเป็น wstETH
        uint256 wstETHReceived = wstETH.wrap(_stETHAmount);
        
        // บันทึกด้วย wstETH
        wstETHBalances[msg.sender] += wstETHReceived;
        totalWstETH += wstETHReceived;
        
        emit DepositedSTETH(msg.sender, _stETHAmount, wstETHReceived);
    }
    
    /**
     * @notice ถอน → รับ stETH พร้อม rewards
     */
    function withdrawAll() external returns (uint256 stETHReceived) {
        uint256 wstETHAmount = wstETHBalances[msg.sender];
        require(wstETHAmount > 0, "No balance");
        
        wstETHBalances[msg.sender] = 0;
        totalWstETH -= wstETHAmount;
        
        // แปลง wstETH → stETH (รวม rewards แล้ว!)
        stETHReceived = wstETH.unwrap(wstETHAmount);
        
        // โอน stETH ให้ user
        stETH.safeTransfer(msg.sender, stETHReceived);
        
        emit WithdrawnSTETH(msg.sender, wstETHAmount, stETHReceived);
    }
    
    /**
     * @notice ดูว่า user มี stETH เท่าไร (รวม rewards)
     */
    function getStETHBalance(address _user) external view returns (uint256) {
        uint256 wstETHBal = wstETHBalances[_user];
        if (wstETHBal == 0) return 0;
        return wstETH.getStETHByWstETH(wstETHBal);
    }
}
```

---

## 85.4 LiquidStakingVault: Complete Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Pausable.sol";

/**
 * @title LiquidStakingVault
 * @notice Vault สำหรับ stake ETH และรับ LST (vETH)
 *         พร้อม reward accumulation และ unbonding queue
 * @dev Features:
 *      - Stake ETH → รับ vETH (appreciating token)
 *      - Request unstake → เข้า unbonding queue
 *      - Claim ETH หลัง unbonding period
 *      - Operator management สำหรับ validator operations
 *      - Emergency pause mechanism
 */
contract LiquidStakingVault is ERC20, ReentrancyGuard, Ownable, Pausable {
    using SafeERC20 for IERC20;
    
    // ============ Constants ============
    
    uint256 public constant UNBONDING_PERIOD = 7 days;
    uint256 public constant MIN_STAKE = 0.01 ether;
    uint256 public constant MAX_STAKE_PER_TX = 1000 ether;
    uint256 public constant PERFORMANCE_FEE_BPS = 1000; // 10%
    uint256 public constant BPS_DENOMINATOR = 10000;
    
    // ============ State ============
    
    // Total ETH ใน protocol (staked + pending rewards)
    uint256 public totalETHStaked;
    uint256 public totalETHPendingRewards;
    
    // Unbonding requests
    struct UnbondingRequest {
        uint256 vETHAmount;
        uint256 ethAmount;       // ETH ที่จะได้รับ (lock in ณ เวลา request)
        uint256 unlocksAt;       // timestamp ที่สามารถ claim ได้
        bool claimed;
    }
    
    mapping(address => UnbondingRequest[]) public unbondingRequests;
    
    // Total ETH ที่อยู่ใน unbonding
    uint256 public totalETHUnbonding;
    
    // Operators (validators)
    mapping(address => bool) public operators;
    uint256 public operatorCount;
    
    // Treasury (รับ performance fees)
    address public treasury;
    
    // Oracle สำหรับ report rewards
    address public oracle;
    
    // Historical exchange rates (สำหรับ audit)
    struct RateSnapshot {
        uint256 rate;
        uint256 timestamp;
    }
    RateSnapshot[] public rateHistory;
    
    // ============ Events ============
    
    event Staked(address indexed user, uint256 ethAmount, uint256 vETHMinted);
    event UnstakeRequested(
        address indexed user, 
        uint256 requestIndex,
        uint256 vETHBurned, 
        uint256 ethToReceive,
        uint256 unlocksAt
    );
    event Claimed(address indexed user, uint256 requestIndex, uint256 ethAmount);
    event RewardsReceived(uint256 amount, uint256 performanceFee);
    event OperatorAdded(address indexed operator);
    event OperatorRemoved(address indexed operator);
    event OracleUpdated(address indexed newOracle);
    event TreasuryUpdated(address indexed newTreasury);
    
    // ============ Errors ============
    
    error BelowMinStake();
    error ExceedsMaxStake();
    error NotYetUnlocked();
    error AlreadyClaimed();
    error NoUnbondingRequests();
    error NotOracle();
    error NotOperator();
    error ZeroAddress();
    error InsufficientLiquidity();
    
    // ============ Constructor ============
    
    constructor(
        address _treasury,
        address _oracle
    ) ERC20("Vault ETH", "vETH") Ownable(msg.sender) {
        if (_treasury == address(0) || _oracle == address(0)) revert ZeroAddress();
        treasury = _treasury;
        oracle = _oracle;
    }
    
    // ============ Core Staking Functions ============
    
    /**
     * @notice Stake ETH และรับ vETH
     * @dev vETH เป็น appreciating token:
     *      - ปริมาณ vETH ที่ได้ = ETH / exchange_rate
     *      - exchange_rate เพิ่มขึ้นตาม rewards
     *      - เมื่อ unstake: รับ ETH ตาม exchange_rate ปัจจุบัน
     */
    function stake() external payable nonReentrant whenNotPaused returns (uint256 vETHMinted) {
        if (msg.value < MIN_STAKE) revert BelowMinStake();
        if (msg.value > MAX_STAKE_PER_TX) revert ExceedsMaxStake();
        
        // คำนวณ vETH จาก current exchange rate
        vETHMinted = _ethToVETH(msg.value);
        require(vETHMinted > 0, "Zero vETH");
        
        // อัปเดต state
        totalETHStaked += msg.value;
        
        // Mint vETH
        _mint(msg.sender, vETHMinted);
        
        emit Staked(msg.sender, msg.value, vETHMinted);
    }
    
    /**
     * @notice ขอ unstake vETH → เริ่ม unbonding
     * @param _vETHAmount จำนวน vETH ที่ต้องการ unstake
     * @return requestIndex index ของ unbonding request
     */
    function requestUnstake(
        uint256 _vETHAmount
    ) external nonReentrant returns (uint256 requestIndex) {
        require(_vETHAmount > 0, "Zero amount");
        require(balanceOf(msg.sender) >= _vETHAmount, "Insufficient vETH");
        
        // คำนวณ ETH ที่จะได้รับ (lock in ณ ตอนนี้)
        uint256 ethToReceive = _vETHToETH(_vETHAmount);
        require(ethToReceive > 0, "Zero ETH");
        
        // Burn vETH ทันที
        _burn(msg.sender, _vETHAmount);
        
        // อัปเดต state
        totalETHStaked -= ethToReceive;
        totalETHUnbonding += ethToReceive;
        
        // เพิ่ม unbonding request
        requestIndex = unbondingRequests[msg.sender].length;
        unbondingRequests[msg.sender].push(UnbondingRequest({
            vETHAmount: _vETHAmount,
            ethAmount: ethToReceive,
            unlocksAt: block.timestamp + UNBONDING_PERIOD,
            claimed: false
        }));
        
        emit UnstakeRequested(
            msg.sender, 
            requestIndex,
            _vETHAmount, 
            ethToReceive,
            block.timestamp + UNBONDING_PERIOD
        );
    }
    
    /**
     * @notice Claim ETH หลัง unbonding period
     * @param _requestIndex index ของ request ที่ต้องการ claim
     */
    function claim(uint256 _requestIndex) external nonReentrant {
        UnbondingRequest[] storage requests = unbondingRequests[msg.sender];
        if (_requestIndex >= requests.length) revert NoUnbondingRequests();
        
        UnbondingRequest storage req = requests[_requestIndex];
        if (req.claimed) revert AlreadyClaimed();
        if (block.timestamp < req.unlocksAt) revert NotYetUnlocked();
        
        uint256 ethAmount = req.ethAmount;
        if (address(this).balance < ethAmount + totalETHUnbonding - ethAmount) {
            revert InsufficientLiquidity();
        }
        
        req.claimed = true;
        totalETHUnbonding -= ethAmount;
        
        // โอน ETH
        (bool success,) = msg.sender.call{value: ethAmount}("");
        require(success, "ETH transfer failed");
        
        emit Claimed(msg.sender, _requestIndex, ethAmount);
    }
    
    /**
     * @notice Claim หลาย requests พร้อมกัน
     */
    function claimBatch(uint256[] calldata _requestIndices) external nonReentrant {
        uint256 totalEth = 0;
        
        for (uint256 i = 0; i < _requestIndices.length; i++) {
            UnbondingRequest storage req = unbondingRequests[msg.sender][_requestIndices[i]];
            if (req.claimed || block.timestamp < req.unlocksAt) continue;
            
            req.claimed = true;
            totalEth += req.ethAmount;
            totalETHUnbonding -= req.ethAmount;
            
            emit Claimed(msg.sender, _requestIndices[i], req.ethAmount);
        }
        
        if (totalEth > 0) {
            (bool success,) = msg.sender.call{value: totalEth}("");
            require(success, "ETH transfer failed");
        }
    }
    
    // ============ Oracle Functions ============
    
    /**
     * @notice Report rewards จาก staking
     * @dev เรียกโดย oracle หลังได้รับ rewards จาก validators
     * @param _rewardAmount จำนวน ETH rewards
     */
    function reportRewards(uint256 _rewardAmount) external {
        if (msg.sender != oracle) revert NotOracle();
        require(_rewardAmount > 0, "Zero rewards");
        require(address(this).balance >= _rewardAmount, "Insufficient ETH for rewards");
        
        // คำนวณ performance fee (10%)
        uint256 performanceFee = (_rewardAmount * PERFORMANCE_FEE_BPS) / BPS_DENOMINATOR;
        uint256 netRewards = _rewardAmount - performanceFee;
        
        // เพิ่ม rewards เข้า pool (ทำให้ exchange rate เพิ่ม)
        totalETHStaked += netRewards;
        
        // จ่าย performance fee ให้ treasury
        if (performanceFee > 0) {
            (bool success,) = treasury.call{value: performanceFee}("");
            require(success, "Treasury transfer failed");
        }
        
        // บันทึก exchange rate snapshot
        rateHistory.push(RateSnapshot({
            rate: getExchangeRate(),
            timestamp: block.timestamp
        }));
        
        emit RewardsReceived(_rewardAmount, performanceFee);
    }
    
    // ============ Exchange Rate Functions ============
    
    /**
     * @notice Exchange rate: ETH ต่อ 1 vETH
     * @dev เพิ่มขึ้นตามเวลา เมื่อ rewards สะสม
     *      เริ่มต้น: 1 vETH = 1 ETH
     *      หลังมี rewards: 1 vETH > 1 ETH
     */
    function getExchangeRate() public view returns (uint256) {
        uint256 supply = totalSupply();
        if (supply == 0) return 1 ether;
        return (totalETHStaked * 1 ether) / supply;
    }
    
    function _ethToVETH(uint256 _ethAmount) internal view returns (uint256) {
        uint256 supply = totalSupply();
        if (supply == 0) return _ethAmount; // 1:1 เมื่อเริ่มต้น
        return (_ethAmount * supply) / totalETHStaked;
    }
    
    function _vETHToETH(uint256 _vETHAmount) internal view returns (uint256) {
        uint256 supply = totalSupply();
        if (supply == 0) return _vETHAmount;
        return (_vETHAmount * totalETHStaked) / supply;
    }
    
    // ============ View Functions ============
    
    function getUnbondingRequests(address _user) 
        external view returns (UnbondingRequest[] memory) 
    {
        return unbondingRequests[_user];
    }
    
    function getETHBalance(address _user) external view returns (uint256) {
        return _vETHToETH(balanceOf(_user));
    }
    
    function getUnclaimedETH(address _user) external view returns (uint256 total) {
        UnbondingRequest[] memory requests = unbondingRequests[_user];
        for (uint256 i = 0; i < requests.length; i++) {
            if (!requests[i].claimed && block.timestamp >= requests[i].unlocksAt) {
                total += requests[i].ethAmount;
            }
        }
    }
    
    function getPendingUnbondingETH(address _user) external view returns (uint256 total) {
        UnbondingRequest[] memory requests = unbondingRequests[_user];
        for (uint256 i = 0; i < requests.length; i++) {
            if (!requests[i].claimed && block.timestamp < requests[i].unlocksAt) {
                total += requests[i].ethAmount;
            }
        }
    }
    
    function getAPY() external view returns (uint256 apy) {
        if (rateHistory.length < 2) return 0;
        
        RateSnapshot memory latest = rateHistory[rateHistory.length - 1];
        RateSnapshot memory oldest = rateHistory[0];
        
        if (latest.timestamp <= oldest.timestamp || oldest.rate == 0) return 0;
        
        uint256 rateIncrease = latest.rate - oldest.rate;
        uint256 timePeriod = latest.timestamp - oldest.timestamp;
        
        // APY = (rateIncrease / oldRate) * (365 days / timePeriod) * 10000
        apy = (rateIncrease * 365 days * 10000) / (oldest.rate * timePeriod);
    }
    
    // ============ Admin Functions ============
    
    function addOperator(address _operator) external onlyOwner {
        if (_operator == address(0)) revert ZeroAddress();
        require(!operators[_operator], "Already operator");
        operators[_operator] = true;
        operatorCount++;
        emit OperatorAdded(_operator);
    }
    
    function removeOperator(address _operator) external onlyOwner {
        require(operators[_operator], "Not operator");
        operators[_operator] = false;
        operatorCount--;
        emit OperatorRemoved(_operator);
    }
    
    function setOracle(address _oracle) external onlyOwner {
        if (_oracle == address(0)) revert ZeroAddress();
        oracle = _oracle;
        emit OracleUpdated(_oracle);
    }
    
    function setTreasury(address _treasury) external onlyOwner {
        if (_treasury == address(0)) revert ZeroAddress();
        treasury = _treasury;
        emit TreasuryUpdated(_treasury);
    }
    
    function pause() external onlyOwner { _pause(); }
    function unpause() external onlyOwner { _unpause(); }
    
    // รับ ETH จาก validator rewards
    receive() external payable {}
}
```

---

## 85.5 LST Composability: ใช้ LST ใน DeFi

### 85.5.1 LST เป็น Collateral ใน Lending Protocol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title LSTPoweredLending
 * @notice Lending protocol ที่รับ stETH/rETH/wstETH เป็น collateral
 * @dev ปัญหาพิเศษของ LST collateral:
 *      1. stETH rebase ทำให้ collateral value เพิ่มขึ้นอัตโนมัติ
 *      2. rETH/wstETH exchange rate เพิ่มขึ้น → ต้องใช้ oracle
 *      3. ในกรณี de-peg → liquidation อาจเกิดขึ้น
 */
contract LSTPoweredLending {
    using SafeERC20 for IERC20;
    
    struct CollateralConfig {
        address priceFeed;       // Chainlink price feed
        uint256 ltv;             // Loan-to-Value ratio (bps)
        uint256 liquidationThreshold; // Threshold สำหรับ liquidation (bps)
        uint256 liquidationBonus;     // Bonus สำหรับ liquidator (bps)
        bool isActive;
        bool isLST;              // เป็น LST หรือไม่
        address underlyingToken; // underlying ETH equivalent
    }
    
    struct UserPosition {
        uint256 collateralShares; // ใช้ shares สำหรับ LST
        uint256 debtAmount;
        uint256 lastInterestAccrual;
    }
    
    mapping(address => CollateralConfig) public collateralConfigs;
    // user => collateral token => position
    mapping(address => mapping(address => UserPosition)) public positions;
    
    address public stableToken; // stablecoin ที่ให้กู้
    address public oracle;
    
    uint256 constant LTV_STETH = 7500;  // 75% LTV สำหรับ stETH
    uint256 constant LTV_WSTETH = 7500; // 75% LTV สำหรับ wstETH
    
    /**
     * @notice วาง stETH เป็น collateral
     * @dev ใช้ shares เพื่อ track rebasing อย่างถูกต้อง
     */
    function depositStETHCollateral(
        address _stETH,
        uint256 _stETHAmount
    ) external {
        CollateralConfig memory config = collateralConfigs[_stETH];
        require(config.isActive, "Collateral not supported");
        
        // ดึง stETH จาก user
        IERC20(_stETH).safeTransferFrom(msg.sender, address(this), _stETHAmount);
        
        // แปลงเป็น shares
        uint256 shares = IStETHShares(_stETH).getSharesByPooledEth(_stETHAmount);
        
        // บันทึกเป็น shares (ไม่ใช่ ETH amount!)
        positions[msg.sender][_stETH].collateralShares += shares;
    }
    
    /**
     * @notice ยืม stablecoin โดยใช้ LST เป็น collateral
     */
    function borrow(
        address _collateralToken,
        uint256 _borrowAmount
    ) external {
        UserPosition storage pos = positions[msg.sender][_collateralToken];
        CollateralConfig memory config = collateralConfigs[_collateralToken];
        
        // คำนวณ collateral value ใน USD
        uint256 collateralValueUSD = _getCollateralValue(
            _collateralToken, 
            pos.collateralShares,
            config
        );
        
        // ตรวจสอบ LTV
        uint256 maxBorrow = (collateralValueUSD * config.ltv) / 10000;
        uint256 currentDebt = pos.debtAmount;
        
        require(
            currentDebt + _borrowAmount <= maxBorrow,
            "Exceeds LTV"
        );
        
        pos.debtAmount += _borrowAmount;
        
        // Mint stablecoin ให้ user
        IERC20(stableToken).safeTransfer(msg.sender, _borrowAmount);
    }
    
    /**
     * @dev คำนวณ collateral value ในปัจจุบัน (รวม staking rewards)
     */
    function _getCollateralValue(
        address _token,
        uint256 _shares,
        CollateralConfig memory _config
    ) internal view returns (uint256 valueUSD) {
        // แปลง shares → ETH amount ปัจจุบัน (รวม rewards)
        uint256 ethAmount;
        
        if (_config.isLST) {
            // สำหรับ stETH: แปลง shares → stETH
            ethAmount = IStETHShares(_token).getPooledEthByShares(_shares);
        } else {
            // สำหรับ wstETH, rETH: ใช้ exchange rate
            ethAmount = IExchangeRateToken(_token).getEthValue(_shares);
        }
        
        // ดึงราคา ETH จาก oracle
        uint256 ethPriceUSD = IChainlinkAggregator(_config.priceFeed).latestAnswer();
        
        // คำนวณ USD value (8 decimals สำหรับ Chainlink)
        valueUSD = (ethAmount * ethPriceUSD) / 1e8;
    }
    
    /**
     * @notice Liquidate position ที่ under-collateralized
     */
    function liquidate(
        address _borrower,
        address _collateralToken,
        uint256 _repayAmount
    ) external {
        UserPosition storage pos = positions[_borrower][_collateralToken];
        CollateralConfig memory config = collateralConfigs[_collateralToken];
        
        uint256 collateralValueUSD = _getCollateralValue(
            _collateralToken,
            pos.collateralShares,
            config
        );
        
        // ตรวจสอบว่า position ต่ำกว่า liquidation threshold
        uint256 liquidationLimit = (collateralValueUSD * config.liquidationThreshold) / 10000;
        require(pos.debtAmount > liquidationLimit, "Position is healthy");
        
        // คืน stablecoin
        IERC20(stableToken).safeTransferFrom(msg.sender, address(this), _repayAmount);
        
        pos.debtAmount -= _repayAmount;
        
        // คำนวณ collateral ที่ liquidator ได้รับ (รวม bonus)
        uint256 collateralToSeize = (_repayAmount * (10000 + config.liquidationBonus)) / 10000;
        
        // คำนวณ shares ที่ต้องโอน
        uint256 sharesToSeize;
        if (config.isLST) {
            sharesToSeize = IStETHShares(_collateralToken).getSharesByPooledEth(
                collateralToSeize
            );
        } else {
            sharesToSeize = collateralToSeize; // Simplified
        }
        
        pos.collateralShares -= sharesToSeize;
        
        // โอน collateral ให้ liquidator
        // (แปลง shares → token แล้วโอน)
        IStETHShares(_collateralToken).transferShares(msg.sender, sharesToSeize);
    }
}

interface IStETHShares {
    function getSharesByPooledEth(uint256 _ethAmount) external view returns (uint256);
    function getPooledEthByShares(uint256 _sharesAmount) external view returns (uint256);
    function transferShares(address _recipient, uint256 _sharesAmount) external returns (uint256);
}

interface IExchangeRateToken {
    function getEthValue(uint256 _tokenAmount) external view returns (uint256);
}

interface IChainlinkAggregator {
    function latestAnswer() external view returns (uint256);
}
```

### 85.5.2 LST ใน AMM (wstETH-ETH Curve Pool)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title LSTAMMPool
 * @notice Simplified AMM สำหรับ wstETH/ETH trading
 * @dev ใช้ stable swap curve (Curve-style) สำหรับ correlated assets
 *      เพราะ wstETH ≈ ETH ในแง่ราคา → ต้องการ low slippage
 */
contract LSTAMMPool {
    using SafeERC20 for IERC20;
    
    // ============ Constants ============
    
    uint256 constant N_COINS = 2;
    uint256 constant PRECISION = 1e18;
    uint256 constant A = 100; // Amplification factor (สูง = stable)
    uint256 constant FEE = 4000000; // 0.04% fee
    uint256 constant FEE_DENOMINATOR = 10_000_000_000;
    
    // ============ State ============
    
    address[N_COINS] public coins;    // [wstETH, WETH]
    uint256[N_COINS] public balances; // pool balances
    
    uint256 public totalSupply;
    mapping(address => uint256) public lpBalances;
    
    IWstETH public wstETH;
    address public WETH;
    
    event TokenExchange(
        address indexed buyer,
        uint256 i,          // sold token index
        uint256 j,          // bought token index
        uint256 dx,         // sold amount
        uint256 dy          // bought amount
    );
    
    event AddLiquidity(
        address indexed provider,
        uint256[N_COINS] amounts,
        uint256 lpMinted
    );
    
    constructor(address _wstETH, address _WETH) {
        wstETH = IWstETH(_wstETH);
        WETH = _WETH;
        coins[0] = _wstETH;
        coins[1] = _WETH;
    }
    
    /**
     * @notice Swap tokens
     * @param i index ของ token ที่ขาย (0=wstETH, 1=WETH)
     * @param j index ของ token ที่ซื้อ
     * @param dx จำนวนที่ขาย
     * @param min_dy จำนวนขั้นต่ำที่ซื้อ
     */
    function exchange(
        uint256 i,
        uint256 j,
        uint256 dx,
        uint256 min_dy
    ) external returns (uint256 dy) {
        require(i != j && i < N_COINS && j < N_COINS, "Invalid indices");
        require(dx > 0, "Zero amount");
        
        // คำนวณ output ตาม stable swap formula
        dy = _getDy(i, j, dx);
        
        // หัก fee
        uint256 fee = (dy * FEE) / FEE_DENOMINATOR;
        dy -= fee;
        
        require(dy >= min_dy, "Slippage too high");
        
        // โอน tokens
        IERC20(coins[i]).safeTransferFrom(msg.sender, address(this), dx);
        IERC20(coins[j]).safeTransfer(msg.sender, dy);
        
        // อัปเดต balances
        balances[i] += dx;
        balances[j] -= dy;
        
        emit TokenExchange(msg.sender, i, j, dx, dy);
    }
    
    /**
     * @dev คำนวณ output จาก stable swap formula
     *      x^A + y^A + xy >= K (StableSwap invariant)
     */
    function _getDy(
        uint256 i,
        uint256 j,
        uint256 dx
    ) internal view returns (uint256 dy) {
        // Convert wstETH → ETH equivalent สำหรับการคำนวณ
        uint256[N_COINS] memory xp = _getNormalizedBalances();
        
        uint256 x = xp[i] + dx;
        uint256 y = _getY(i, j, x, xp);
        
        dy = xp[j] - y - 1;
        
        // Convert กลับ (ถ้า j = wstETH index)
        if (j == 0) {
            dy = (dy * PRECISION) / wstETH.stEthPerToken();
        }
    }
    
    /**
     * @dev คำนวณ y จาก stable swap invariant
     */
    function _getY(
        uint256 i,
        uint256 j,
        uint256 x,
        uint256[N_COINS] memory xp
    ) internal pure returns (uint256 y) {
        uint256 D = _getD(xp);
        uint256 Ann = A * N_COINS;
        
        uint256 c = D;
        uint256 S_ = 0;
        uint256 _x = 0;
        
        for (uint256 k = 0; k < N_COINS; k++) {
            if (k == i) {
                _x = x;
            } else if (k != j) {
                _x = xp[k];
            } else {
                continue;
            }
            S_ += _x;
            c = (c * D) / (_x * N_COINS);
        }
        
        c = (c * D) / (Ann * N_COINS);
        uint256 b = S_ + D / Ann;
        
        y = D;
        for (uint256 k = 0; k < 255; k++) {
            uint256 y_prev = y;
            y = (y * y + c) / (2 * y + b - D);
            if (y > y_prev) {
                if (y - y_prev <= 1) break;
            } else {
                if (y_prev - y <= 1) break;
            }
        }
    }
    
    /**
     * @dev คำนวณ D (invariant)
     */
    function _getD(uint256[N_COINS] memory xp) internal pure returns (uint256 D) {
        uint256 S = 0;
        for (uint256 i = 0; i < N_COINS; i++) {
            S += xp[i];
        }
        if (S == 0) return 0;
        
        uint256 Dprev = 0;
        D = S;
        uint256 Ann = A * N_COINS;
        
        for (uint256 i = 0; i < 255; i++) {
            uint256 D_P = D;
            for (uint256 j = 0; j < N_COINS; j++) {
                D_P = (D_P * D) / (xp[j] * N_COINS);
            }
            Dprev = D;
            D = ((Ann * S + D_P * N_COINS) * D) / ((Ann - 1) * D + (N_COINS + 1) * D_P);
            if (D > Dprev) {
                if (D - Dprev <= 1) break;
            } else {
                if (Dprev - D <= 1) break;
            }
        }
    }
    
    /**
     * @dev Normalize balances เป็น ETH equivalent
     *      wstETH balance แปลงเป็น stETH equivalent
     */
    function _getNormalizedBalances() internal view returns (uint256[N_COINS] memory xp) {
        // wstETH → stETH equivalent (ตาม current exchange rate)
        xp[0] = (balances[0] * wstETH.stEthPerToken()) / PRECISION;
        xp[1] = balances[1]; // WETH ไม่ต้อง normalize
    }
    
    /**
     * @notice Quote ราคาสำหรับ swap
     */
    function getQuote(
        uint256 i,
        uint256 j,
        uint256 dx
    ) external view returns (uint256 dy, uint256 priceImpact) {
        dy = _getDy(i, j, dx);
        uint256 fee = (dy * FEE) / FEE_DENOMINATOR;
        dy -= fee;
        
        // คำนวณ price impact
        uint256 spotPrice = _getDy(i, j, PRECISION);
        uint256 executionPrice = (dy * PRECISION) / dx;
        
        if (spotPrice > executionPrice) {
            priceImpact = ((spotPrice - executionPrice) * 10000) / spotPrice;
        }
    }
}
```

---

## 85.6 Workshop: ทดสอบ LiquidStakingVault

```javascript
// Workshop: ทดสอบ lifecycle ของ LiquidStakingVault

const { ethers } = require("hardhat");
const { time } = require("@nomicfoundation/hardhat-toolbox/network-helpers");

async function testLiquidStakingVault() {
    const [deployer, oracle, treasury, user1, user2] = await ethers.getSigners();
    
    // Deploy
    const Vault = await ethers.getContractFactory("LiquidStakingVault");
    const vault = await Vault.deploy(treasury.address, oracle.address);
    
    console.log("Vault deployed:", vault.target);
    
    // ============ Test 1: Stake ETH ============
    
    const stakeAmount = ethers.parseEther("10");
    
    await vault.connect(user1).stake({ value: stakeAmount });
    
    const vETHBalance = await vault.balanceOf(user1.address);
    console.log("vETH received:", ethers.formatEther(vETHBalance));
    // ควรเป็น 10 vETH (1:1 เริ่มต้น)
    
    // ============ Test 2: More users stake ============
    
    await vault.connect(user2).stake({ value: ethers.parseEther("5") });
    
    console.log("Exchange rate (initial):", ethers.formatEther(await vault.getExchangeRate()));
    // 1 ETH per vETH
    
    // ============ Test 3: Oracle Reports Rewards ============
    
    // ส่ง 0.5 ETH rewards (5% APY simulation)
    await deployer.sendTransaction({ to: vault.target, value: ethers.parseEther("0.5") });
    
    await vault.connect(oracle).reportRewards(ethers.parseEther("0.5"));
    
    const newRate = await vault.getExchangeRate();
    console.log("Exchange rate (after rewards):", ethers.formatEther(newRate));
    // ควรสูงกว่า 1 ETH per vETH
    
    // ============ Test 4: Check User Gains ============
    
    const ethValueUser1 = await vault.getETHBalance(user1.address);
    console.log("User1 ETH value:", ethers.formatEther(ethValueUser1));
    // ควรมากกว่า 10 ETH เล็กน้อย (รับส่วนแบ่ง rewards)
    
    // ============ Test 5: Request Unstake ============
    
    const halfVETH = vETHBalance / 2n;
    await vault.connect(user1).requestUnstake(halfVETH);
    
    const requests = await vault.getUnbondingRequests(user1.address);
    console.log("Unbonding request:", {
        ethAmount: ethers.formatEther(requests[0].ethAmount),
        unlocksAt: new Date(Number(requests[0].unlocksAt) * 1000).toISOString()
    });
    
    // ============ Test 6: รอ 7 วัน แล้ว Claim ============
    
    await time.increase(7 * 24 * 60 * 60 + 1); // 7 days + 1 second
    
    const balanceBefore = await ethers.provider.getBalance(user1.address);
    await vault.connect(user1).claim(0);
    const balanceAfter = await ethers.provider.getBalance(user1.address);
    
    console.log("ETH received from unstake:", 
        ethers.formatEther(balanceAfter - balanceBefore));
    
    // ============ Test 7: APY Calculation ============
    
    // Simulate more rewards over time
    for (let i = 0; i < 3; i++) {
        await time.increase(30 * 24 * 60 * 60); // 1 month
        await deployer.sendTransaction({ to: vault.target, value: ethers.parseEther("0.5") });
        await vault.connect(oracle).reportRewards(ethers.parseEther("0.5"));
    }
    
    const apy = await vault.getAPY();
    console.log("Estimated APY:", Number(apy) / 100, "%");
}
```

---

## 85.7 LST Comparison Table

| Feature | stETH (Lido) | rETH (Rocket Pool) | wstETH | cbETH (Coinbase) |
|---------|-------------|---------------------|--------|-----------------|
| Type | Rebasing | Appreciating | Non-rebasing wrapper | Appreciating |
| TVL (2024) | $35B+ | $4B | $10B+ (wrapped stETH) | $2B |
| Decentralization | Low (DAO) | High (permissionless) | Same as stETH | Centralized |
| Oracle dependency | Yes (committee) | Yes (DAO) | Minimal | Yes |
| AMM compatibility | ต้องระวัง rebase | ดี | ดีที่สุด | ดี |
| Lending collateral | ใช้ wstETH | ดี | ดีที่สุด | ดี |
| slashing risk | Distributed | Operator bears | Same as stETH | Low |

---

## สรุป Part 85

- **Lido stETH** ใช้ shares model เพื่อ handle rebasing อย่างถูกต้อง — protocol ต้อง track shares ไม่ใช่ token amounts
- **Rocket Pool rETH** เป็น appreciating token ที่ exchange rate เพิ่มขึ้นตามเวลา — ง่ายกว่า สำหรับ protocol integration
- **wstETH** คือ wrapped stETH แบบ non-rebasing — เป็น standard ที่แนะนำสำหรับ DeFi integration
- **LiquidStakingVault** แสดง architecture สำหรับสร้าง liquid staking protocol ตั้งแต่ deposit, rewards, unbonding queue จนถึง claim
- **LST Composability**: ใช้ wstETH เป็น collateral ใน lending (ต้องใช้ shares), ใช้ใน stable swap AMM (ต้อง normalize ตาม exchange rate)
- ข้อผิดพลาดที่พบบ่อย: บันทึก stETH amount แทน shares → สูญเสีย staking rewards ของ user ทั้งหมด

## Next: Part 86 - Cross-Chain Bridges & Messaging (LayerZero, Axelar, Wormhole, IBC)
