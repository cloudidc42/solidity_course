# Part 49: Smart Contract Incident Response

## สารบัญ
1. Incident Response Playbook
2. Monitoring and Alerting
3. White Hat Rescue
4. Post-Incident Analysis
5. Workshop: Emergency Response System

---

## 1. Incident Response Playbook

```
Smart Contract Incident เกิดขึ้นอย่างไร:
1. Exploit ถูกส่ง transaction
2. Funds ถูก drain
3. Community สังเกตเห็น (price impact, strange txs)
4. Team ตรวจสอบ
5. Pause (ถ้ามีระบบ)
6. ประกาศ community
7. Fix หรือ recovery

Preparation (ก่อนเกิดเหตุ):
- Emergency pause mechanism
- Guardian multisig พร้อมใช้งาน
- On-call engineer ตลอด 24/7
- Runbook สำหรับ common incidents
- Bug bounty program (ช่วย find bugs before exploit)
- Insurance (Nexus Mutual, UnoRe)

Response Timeline (Minutes):
0-5: Detect anomaly
5-15: Confirm it's an attack
15-30: Pause protocol
30-60: Notify community (Twitter, Discord)
1-24h: Root cause analysis
24h+: Fix and recovery plan

Communication:
- Twitter: ประกาศ status อย่างโปร่งใส
- Discord: detailed technical updates
- Blog post: post-mortem (เสมอ!)
- Never blame others or minimize

Postmortem Template:
1. Summary
2. Timeline
3. Root Cause Analysis
4. Impact Assessment
5. Lessons Learned
6. Action Items
```

---

## 2. Monitoring System

```typescript
// monitoring/watcher.ts
import { ethers } from "ethers";
import axios from "axios";

interface Alert {
  severity: "low" | "medium" | "high" | "critical";
  title: string;
  message: string;
  txHash?: string;
}

class ProtocolMonitor {
  private provider: ethers.Provider;
  private contractAddress: string;
  private contract: ethers.Contract;
  
  // Thresholds
  private MAX_SINGLE_WITHDRAW = ethers.parseEther("1000");
  private MAX_HOURLY_WITHDRAW = ethers.parseEther("10000");
  private hourlyWithdrawals = 0n;
  
  constructor(rpcUrl: string, contractAddress: string, abi: ethers.InterfaceAbi) {
    this.provider = new ethers.JsonRpcProvider(rpcUrl);
    this.contractAddress = contractAddress;
    this.contract = new ethers.Contract(contractAddress, abi, this.provider);
    
    // Reset hourly counter
    setInterval(() => { this.hourlyWithdrawals = 0n; }, 3600_000);
  }
  
  async start() {
    console.log("Starting protocol monitor...");
    
    // Monitor Withdrawal events
    this.contract.on("Withdrawal", async (user, amount, event) => {
      await this.handleWithdrawal(user, amount, event);
    });
    
    // Monitor all transactions to contract
    this.provider.on({ address: this.contractAddress }, async (log) => {
      await this.analyzeTransaction(log.transactionHash);
    });
    
    // Periodic health checks
    setInterval(() => this.healthCheck(), 60_000);
    
    console.log("Monitor active. Watching", this.contractAddress);
  }
  
  private async handleWithdrawal(user: string, amount: bigint, event: ethers.EventLog) {
    this.hourlyWithdrawals += amount;
    
    // Check single withdrawal
    if (amount > this.MAX_SINGLE_WITHDRAW) {
      await this.sendAlert({
        severity: "high",
        title: "Large Withdrawal Detected",
        message: `${user} withdrew ${ethers.formatEther(amount)} ETH`,
        txHash: event.transactionHash,
      });
    }
    
    // Check hourly total
    if (this.hourlyWithdrawals > this.MAX_HOURLY_WITHDRAW) {
      await this.sendAlert({
        severity: "critical",
        title: "HIGH VOLUME WITHDRAWALS",
        message: `Total hourly withdrawals: ${ethers.formatEther(this.hourlyWithdrawals)} ETH`,
        txHash: event.transactionHash,
      });
    }
  }
  
  private async analyzeTransaction(txHash: string) {
    try {
      const tx = await this.provider.getTransaction(txHash);
      if (!tx) return;
      
      const receipt = await this.provider.getTransactionReceipt(txHash);
      if (!receipt) return;
      
      // Check for reentrant calls (multiple logs from same address)
      const contractLogs = receipt.logs.filter(
        log => log.address.toLowerCase() === this.contractAddress.toLowerCase()
      );
      
      if (contractLogs.length > 5) {
        await this.sendAlert({
          severity: "high",
          title: "Suspicious Transaction Pattern",
          message: `TX ${txHash} had ${contractLogs.length} logs from contract`,
          txHash,
        });
      }
      
      // Check for failed transaction that cost a lot of gas (failed exploit attempt)
      if (!receipt.status && receipt.gasUsed > 200000n) {
        await this.sendAlert({
          severity: "medium",
          title: "Failed High-Gas Transaction",
          message: `May be an exploit attempt: ${txHash}`,
          txHash,
        });
      }
    } catch (err) {
      console.error("Error analyzing tx:", err);
    }
  }
  
  private async healthCheck() {
    try {
      // Check TVL hasn't dropped significantly
      const tvl = await this.contract.totalValueLocked();
      const tvlEth = parseFloat(ethers.formatEther(tvl));
      
      // TODO: compare with previous reading
      // Alert if >10% drop in 1 hour
      
    } catch (err) {
      await this.sendAlert({
        severity: "critical",
        title: "Contract Unresponsive",
        message: `Health check failed: ${err}`,
      });
    }
  }
  
  private async sendAlert(alert: Alert) {
    console.log(`[${alert.severity.toUpperCase()}] ${alert.title}: ${alert.message}`);
    
    // Send to Telegram
    if (process.env.TELEGRAM_BOT_TOKEN && process.env.TELEGRAM_CHAT_ID) {
      const emoji = { low: "ℹ️", medium: "⚠️", high: "🚨", critical: "🔴" }[alert.severity];
      
      await axios.post(
        `https://api.telegram.org/bot${process.env.TELEGRAM_BOT_TOKEN}/sendMessage`,
        {
          chat_id: process.env.TELEGRAM_CHAT_ID,
          text: `${emoji} *${alert.title}*\n${alert.message}${alert.txHash ? `\n\nhttps://etherscan.io/tx/${alert.txHash}` : ""}`,
          parse_mode: "Markdown",
        }
      );
    }
    
    // Send to PagerDuty for critical
    if (alert.severity === "critical" && process.env.PAGERDUTY_KEY) {
      await axios.post("https://events.pagerduty.com/v2/enqueue", {
        routing_key: process.env.PAGERDUTY_KEY,
        event_action: "trigger",
        payload: {
          summary: alert.title,
          severity: "critical",
          source: "protocol-monitor",
          custom_details: alert,
        },
      });
    }
  }
}
```

---

## 3. White Hat Rescue Pattern

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * White Hat Rescue:
 * เมื่อพบ vulnerability ใน protocol ที่ live
 * White hat สามารถ "exploit" ก่อน black hat
 * เพื่อ rescue funds แล้วคืนให้ protocol
 * 
 * ต้องทำ ก่อน black hat exploit!
 */
contract WhiteHatRescue {
    
    address public immutable rescuer; // white hat address
    address public immutable targetProtocol;
    address public immutable safeAddress; // protocol's safe
    
    constructor(address _target, address _safe) {
        rescuer = msg.sender;
        targetProtocol = _target;
        safeAddress = _safe;
    }
    
    /**
     * Rescue vulnerable funds using Flash Loan
     * 
     * Flow:
     * 1. Get flash loan
     * 2. Exploit vulnerability (drain funds)
     * 3. Transfer to safe address
     * 4. Repay flash loan
     */
    function rescueFunds(
        address flashLender,
        address rescueToken,
        uint256 flashAmount,
        bytes calldata rescueCalldata
    ) external {
        require(msg.sender == rescuer, "Not rescuer");
        
        // Get flash loan
        IFlashLoan(flashLender).flashLoan(
            address(this),
            rescueToken,
            flashAmount,
            abi.encode(rescueCalldata)
        );
    }
    
    function onFlashLoan(
        address, address token,
        uint256 amount, uint256 fee,
        bytes calldata data
    ) external returns (bytes32) {
        bytes memory rescueCalldata = abi.decode(data, (bytes));
        
        // Execute rescue (protocol-specific)
        (bool success,) = targetProtocol.call(rescueCalldata);
        require(success, "Rescue failed");
        
        // Transfer rescued funds to safe address
        uint256 rescuedBalance = IERC20(token).balanceOf(address(this));
        uint256 repayAmount = amount + fee;
        
        // Keep only what we need to repay, rest goes to safe
        if (rescuedBalance > repayAmount) {
            IERC20(token).transfer(safeAddress, rescuedBalance - repayAmount);
        }
        
        // Approve flash loan repayment
        IERC20(token).approve(msg.sender, repayAmount);
        
        return keccak256("ERC3156FlashBorrower.onFlashLoan");
    }
}

interface IFlashLoan {
    function flashLoan(address, address, uint256, bytes calldata) external returns (bool);
}

interface IERC20 {
    function transfer(address, uint256) external returns (bool);
    function approve(address, uint256) external returns (bool);
    function balanceOf(address) external view returns (uint256);
}
```

---

## 4. Emergency Response Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * Emergency Response System:
 * ระบบ automated response เมื่อตรวจพบ attack
 */
contract EmergencyResponse {
    
    address public protocol;
    address[] public guardians;
    uint256 public guardianThreshold;
    
    // Off-chain monitor triggers these
    mapping(address => bool) public authorizedMonitors;
    
    // Emergency actions queued
    struct EmergencyAction {
        bytes callData;
        uint256 proposedAt;
        uint256 approvals;
        mapping(address => bool) approved;
        bool executed;
    }
    
    mapping(bytes32 => EmergencyAction) public actions;
    
    uint256 public constant EMERGENCY_DELAY = 15 minutes; // min before exec
    
    event EmergencyProposed(bytes32 indexed actionId, bytes callData);
    event EmergencyApproved(bytes32 indexed actionId, address guardian);
    event EmergencyExecuted(bytes32 indexed actionId);
    event MonitorAlert(address indexed monitor, string reason);
    
    modifier onlyGuardian() {
        bool isGuardian;
        for (uint256 i; i < guardians.length; i++) {
            if (guardians[i] == msg.sender) { isGuardian = true; break; }
        }
        require(isGuardian, "Not guardian");
        _;
    }
    
    constructor(
        address _protocol,
        address[] memory _guardians,
        uint256 _threshold
    ) {
        protocol = _protocol;
        guardians = _guardians;
        guardianThreshold = _threshold;
    }
    
    // Monitor reports anomaly → auto-propose pause
    function reportAnomaly(string calldata reason) external {
        require(authorizedMonitors[msg.sender], "Not monitor");
        
        emit MonitorAlert(msg.sender, reason);
        
        // Auto-propose pause
        bytes memory pauseCall = abi.encodeWithSignature("pause(string)", reason);
        _proposeAction(pauseCall);
    }
    
    function _proposeAction(bytes memory callData) internal returns (bytes32 actionId) {
        actionId = keccak256(abi.encodePacked(callData, block.timestamp));
        
        EmergencyAction storage action = actions[actionId];
        action.callData = callData;
        action.proposedAt = block.timestamp;
        
        emit EmergencyProposed(actionId, callData);
    }
    
    function approveAction(bytes32 actionId) external onlyGuardian {
        EmergencyAction storage action = actions[actionId];
        require(!action.executed, "Already executed");
        require(!action.approved[msg.sender], "Already approved");
        
        action.approved[msg.sender] = true;
        action.approvals++;
        
        emit EmergencyApproved(actionId, msg.sender);
        
        // Auto-execute if threshold met and delay passed
        if (action.approvals >= guardianThreshold &&
            block.timestamp >= action.proposedAt + EMERGENCY_DELAY) {
            _executeAction(actionId);
        }
    }
    
    function executeAction(bytes32 actionId) external onlyGuardian {
        EmergencyAction storage action = actions[actionId];
        require(action.approvals >= guardianThreshold, "Not enough approvals");
        require(block.timestamp >= action.proposedAt + EMERGENCY_DELAY, "Too early");
        
        _executeAction(actionId);
    }
    
    function _executeAction(bytes32 actionId) internal {
        EmergencyAction storage action = actions[actionId];
        require(!action.executed, "Already executed");
        
        action.executed = true;
        
        (bool success,) = protocol.call(action.callData);
        require(success, "Action failed");
        
        emit EmergencyExecuted(actionId);
    }
}
```

---

## สรุป Part 49

Incident Response ที่เรียนรู้:
- ✅ Incident response playbook
- ✅ Real-time monitoring (TypeScript)
- ✅ Alert system (Telegram, PagerDuty)
- ✅ White hat rescue pattern
- ✅ Emergency response contract

## Next: Part 50 - Production Deployment Checklist
