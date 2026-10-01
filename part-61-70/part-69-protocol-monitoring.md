# Part 69: Protocol Monitoring & Alerting

## บทนำ

การ Deploy smart contract ไปยัง mainnet ไม่ใช่จุดสิ้นสุด แต่เป็นจุดเริ่มต้นของความรับผิดชอบที่ใหญ่กว่า การ Monitor protocol อย่างต่อเนื่องเป็นสิ่งจำเป็นสำหรับ:
- ตรวจจับการโจมตีหรือพฤติกรรมผิดปกติ
- ตอบสนองต่อเหตุการณ์ฉุกเฉินก่อนที่ funds จะหมด
- รักษา uptime และ health ของ protocol

ในบทนี้จะครอบคลุม:
- **Forta**: Decentralized threat detection network
- **OpenZeppelin Defender**: Suite เครื่องมือ operations สำหรับ smart contracts
- **Custom Ethers.js Watcher**: สำหรับ monitoring ที่ customize ได้เต็มที่
- **Alert Severity Tiers**: การจัดการ alerts ตาม criticality
- **Incident Response**: On-call rotation และ runbooks

---

## 1. Forta: Decentralized Threat Detection

### 1.1 สถาปัตยกรรม Forta

```
Blockchain Node
    ↓ blocks/transactions
Forta Node (scan node)
    ↓ runs detection bots
Bot Container (Docker)
    ├── handleTransaction(txEvent)
    ├── handleBlock(blockEvent)
    └── handleAlert(alertEvent)
    ↓ findings
Forta Network
    ↓ aggregates
Alert Consumers (you)
```

### 1.2 โครงสร้าง Forta Bot ใน TypeScript

```typescript
// src/agent.ts
import {
  Finding,
  HandleTransaction,
  HandleBlock,
  TransactionEvent,
  BlockEvent,
  FindingSeverity,
  FindingType,
  Label,
  EntityType,
  getEthersProvider,
} from "forta-agent";
import { ethers } from "ethers";

// ============================================================
//                     CONSTANTS
// ============================================================

// Contract ที่ต้องการ monitor (ตัวอย่าง: Uniswap V3 Pool)
const MONITORED_CONTRACT = "0x8ad599c3A0ff1De082011EFDDc58f1908eb6e6D5"; // USDC/ETH 0.3%

// Flash loan threshold: ถ้ายืมเกิน 10M USDC ถือว่า suspicious
const FLASH_LOAN_THRESHOLD = ethers.parseUnits("10000000", 6); // 10M USDC

// Large swap threshold: swap เกิน 1M USDC
const LARGE_SWAP_THRESHOLD = ethers.parseUnits("1000000", 6); // 1M USDC

// ABI สำหรับ events ที่สนใจ
const UNISWAP_V3_POOL_ABI = [
  "event Swap(address indexed sender, address indexed recipient, int256 amount0, int256 amount1, uint160 sqrtPriceX96, uint128 liquidity, int24 tick)",
  "event Flash(address indexed sender, address indexed recipient, uint256 amount0, uint256 amount1, uint256 paid0, uint256 paid1)",
  "event Mint(address sender, address indexed owner, int24 indexed tickLower, int24 indexed tickUpper, uint128 amount, uint256 amount0, uint256 amount1)",
  "event Burn(address indexed owner, int24 indexed tickLower, int24 indexed tickUpper, uint128 amount, uint256 amount0, uint256 amount1)",
];

const poolInterface = new ethers.Interface(UNISWAP_V3_POOL_ABI);

// ============================================================
//                  HANDLER: TRANSACTION
// ============================================================

/**
 * handleTransaction - เรียกทุก transaction ที่ confirmed
 * @param txEvent ข้อมูล transaction
 * @returns findings array ของ alerts ที่ตรวจพบ
 */
export const handleTransaction: HandleTransaction = async (
  txEvent: TransactionEvent
): Promise<Finding[]> => {
  const findings: Finding[] = [];

  // ตรวจสอบเฉพาะ transactions ที่เกี่ยวข้องกับ contract ที่ monitor
  if (
    txEvent.to?.toLowerCase() !== MONITORED_CONTRACT.toLowerCase() &&
    !txEvent.logs.some(
      (log) => log.address.toLowerCase() === MONITORED_CONTRACT.toLowerCase()
    )
  ) {
    return findings;
  }

  // Parse logs จาก monitored contract
  const relevantLogs = txEvent.logs.filter(
    (log) => log.address.toLowerCase() === MONITORED_CONTRACT.toLowerCase()
  );

  for (const log of relevantLogs) {
    try {
      const parsedLog = poolInterface.parseLog(log);
      if (!parsedLog) continue;

      // ============================================================
      // ตรวจจับ: Flash Loan ขนาดใหญ่
      // ============================================================
      if (parsedLog.name === "Flash") {
        const { sender, recipient, amount0, amount1 } = parsedLog.args;
        const flashAmount = amount0 > amount1 ? amount0 : amount1;

        if (flashAmount >= FLASH_LOAN_THRESHOLD) {
          findings.push(
            Finding.fromObject({
              name: "Large Flash Loan Detected",
              description: `Flash loan of ${ethers.formatUnits(flashAmount, 6)} USDC detected from ${sender}`,
              alertId: "UNISWAP-LARGE-FLASH-LOAN",
              severity: FindingSeverity.High,
              type: FindingType.Suspicious,
              metadata: {
                sender,
                recipient,
                amount0: amount0.toString(),
                amount1: amount1.toString(),
                txHash: txEvent.hash,
                blockNumber: txEvent.blockNumber.toString(),
              },
              labels: [
                Label.fromObject({
                  entity: sender,
                  entityType: EntityType.Address,
                  label: "flash-loan-user",
                  confidence: 0.8,
                }),
              ],
            })
          );
        }
      }

      // ============================================================
      // ตรวจจับ: Swap ขนาดใหญ่ (อาจเป็น whale manipulation)
      // ============================================================
      if (parsedLog.name === "Swap") {
        const { sender, recipient, amount0, amount1 } = parsedLog.args;

        // amount0 และ amount1 เป็น int256 (ลบได้)
        const absAmount0 = amount0 < 0n ? -amount0 : amount0;
        const absAmount1 = amount1 < 0n ? -amount1 : amount1;
        const swapAmount = absAmount0 > absAmount1 ? absAmount0 : absAmount1;

        if (swapAmount >= LARGE_SWAP_THRESHOLD) {
          findings.push(
            Finding.fromObject({
              name: "Large Swap Detected",
              description: `Large swap of ${ethers.formatUnits(swapAmount, 6)} detected`,
              alertId: "UNISWAP-LARGE-SWAP",
              severity: FindingSeverity.Medium,
              type: FindingType.Info,
              metadata: {
                sender,
                recipient,
                amount0: amount0.toString(),
                amount1: amount1.toString(),
                txHash: txEvent.hash,
              },
            })
          );
        }
      }
    } catch {
      // Log parsing อาจ fail ถ้า event ไม่ match ABI
      continue;
    }
  }

  // ============================================================
  // ตรวจจับ: Reentrancy pattern (transaction ที่ call เยอะผิดปกติ)
  // ============================================================
  if (txEvent.traces) {
    const callsToContract = txEvent.traces.filter(
      (trace) =>
        trace.to?.toLowerCase() === MONITORED_CONTRACT.toLowerCase()
    ).length;

    if (callsToContract > 5) {
      findings.push(
        Finding.fromObject({
          name: "Possible Reentrancy Attack",
          description: `Contract called ${callsToContract} times in single transaction`,
          alertId: "POSSIBLE-REENTRANCY",
          severity: FindingSeverity.Critical,
          type: FindingType.Exploit,
          metadata: {
            callCount: callsToContract.toString(),
            txHash: txEvent.hash,
            attacker: txEvent.from,
          },
        })
      );
    }
  }

  return findings;
};

// ============================================================
//                  HANDLER: BLOCK
// ============================================================

/**
 * handleBlock - เรียกทุก block ใหม่
 * @param blockEvent ข้อมูล block
 * @returns findings
 */
export const handleBlock: HandleBlock = async (
  blockEvent: BlockEvent
): Promise<Finding[]> => {
  const findings: Finding[] = [];
  const provider = getEthersProvider();

  // ============================================================
  // ตรวจสอบ Liquidity Level ทุก block
  // ============================================================
  try {
    const pool = new ethers.Contract(MONITORED_CONTRACT, [
      "function liquidity() external view returns (uint128)",
    ], provider);

    const currentLiquidity: bigint = await pool.liquidity();

    // ถ้า liquidity ลดลงเกิน 50% จาก expected ให้ alert
    const MINIMUM_LIQUIDITY = ethers.parseUnits("1000000", 18); // arbitrary threshold

    if (currentLiquidity < MINIMUM_LIQUIDITY) {
      findings.push(
        Finding.fromObject({
          name: "Critical: Low Pool Liquidity",
          description: `Pool liquidity dropped to ${currentLiquidity.toString()}`,
          alertId: "LOW-POOL-LIQUIDITY",
          severity: FindingSeverity.Critical,
          type: FindingType.Suspicious,
          metadata: {
            currentLiquidity: currentLiquidity.toString(),
            blockNumber: blockEvent.blockNumber.toString(),
          },
        })
      );
    }
  } catch (e) {
    // RPC call อาจ fail ได้ ไม่ throw
  }

  return findings;
};

// Export default
export default {
  handleTransaction,
  handleBlock,
};
```

### 1.3 Finding Severity Levels

```typescript
// src/severity-guide.ts
import { FindingSeverity, FindingType } from "forta-agent";

/**
 * คำแนะนำการใช้ Severity Levels
 *
 * CRITICAL (5): โจมตีกำลังเกิดขึ้น funds ใกล้หมด
 *   Examples:
 *   - Reentrancy attack detected
 *   - Funds draining from contract
 *   - Price oracle manipulation
 *   Response: Page on-call immediately, consider pause
 *
 * HIGH (4): สัญญาณน่าเป็นห่วงมาก
 *   Examples:
 *   - Flash loan ขนาดใหญ่ผิดปกติ
 *   - Admin key ถูก use จาก unknown address
 *   - Unusual withdrawal pattern
 *   Response: Alert team, investigate within 15 min
 *
 * MEDIUM (3): ควรตรวจสอบ
 *   Examples:
 *   - Swap ขนาดใหญ่ที่ไม่ปกติ
 *   - Gas ราคาสูงผิดปกติ (อาจมี MEV)
 *   - Parameter ถูกเปลี่ยน
 *   Response: Log and investigate within 1 hour
 *
 * LOW (2): เพื่อ awareness
 *   Examples:
 *   - New deployer address detected
 *   - Contract state change
 *   Response: Log for records
 *
 * INFO (1): สำหรับ debugging
 *   Examples:
 *   - Normal large transaction
 *   - Routine contract interaction
 *   Response: No action needed
 */

export const SeverityGuide = {
  CRITICAL: FindingSeverity.Critical,
  HIGH: FindingSeverity.High,
  MEDIUM: FindingSeverity.Medium,
  LOW: FindingSeverity.Low,
  INFO: FindingSeverity.Info,
} as const;

// Finding Types
export const TypeGuide = {
  EXPLOIT: FindingType.Exploit,          // Attack กำลังเกิดขึ้น
  SUSPICIOUS: FindingType.Suspicious,   // น่าสงสัย อาจเป็น attack
  DEGRADED: FindingType.Degraded,       // Performance ลดลง
  INFO: FindingType.Info,               // ข้อมูลทั่วไป
  UNKNOWN: FindingType.Unknown,         // ไม่แน่ใจ
} as const;
```

### 1.4 package.json สำหรับ Forta Bot

```json
{
  "name": "uniswap-v3-monitor-bot",
  "version": "1.0.0",
  "description": "Forta bot for monitoring Uniswap V3 USDC/ETH pool",
  "main": "dist/agent.js",
  "scripts": {
    "build": "tsc",
    "start": "npm run build && forta-agent run",
    "test": "jest --detectOpenHandles",
    "publish": "forta-agent publish"
  },
  "dependencies": {
    "forta-agent": "^0.1.48",
    "ethers": "^6.0.0"
  },
  "devDependencies": {
    "@types/jest": "^29.0.0",
    "jest": "^29.0.0",
    "ts-jest": "^29.0.0",
    "typescript": "^5.0.0"
  }
}
```

---

## 2. OpenZeppelin Defender

### 2.1 Defender Sentinel: ติดตาม Contract Events

```typescript
// defender/sentinel-config.ts
// Configuration สำหรับ Sentinel ที่ monitor Uniswap Pool

/**
 * Sentinel Configuration
 *
 * Sentinels ใน Defender ทำงานแบบ no-code/low-code
 * สามารถ configure ผ่าน Defender Dashboard หรือ API
 */

export const sentinelConfig = {
  type: "BLOCK",  // หรือ "FORTA" สำหรับ Forta alerts
  name: "Uniswap V3 Pool Monitor",
  network: "mainnet",

  // Contract ที่ monitor
  addresses: [
    "0x8ad599c3A0ff1De082011EFDDc58f1908eb6e6D5"  // USDC/ETH 0.3%
  ],

  // ABI ของ contract
  abi: JSON.stringify([
    {
      "name": "Swap",
      "type": "event",
      "inputs": [
        { "name": "sender", "type": "address", "indexed": true },
        { "name": "recipient", "type": "address", "indexed": true },
        { "name": "amount0", "type": "int256" },
        { "name": "amount1", "type": "int256" },
        { "name": "sqrtPriceX96", "type": "uint160" },
        { "name": "liquidity", "type": "uint128" },
        { "name": "tick", "type": "int24" }
      ]
    }
  ]),

  // Filter conditions
  conditions: {
    event: [
      {
        signature: "Swap(address,address,int256,int256,uint160,uint128,int24)",
        // Alert เฉพาะ swap ที่ amount0 > 1M USDC
        expression: "abs(amount0) > 1000000000000"  // 1M USDC with 6 decimals
      }
    ]
  },

  // Notification channels
  notificationChannels: ["email", "slack", "pagerduty"],

  // Autotask ที่จะ trigger
  autotaskTrigger: {
    autotaskId: "at-xxxx-yyyy"
  }
};
```

### 2.2 Defender Autotask: Emergency Pause

```typescript
// defender/autotasks/emergency-pause.ts
/**
 * Autotask: Emergency Pause
 *
 * Autotask นี้จะถูก trigger โดย Sentinel เมื่อตรวจพบ suspicious activity
 * มันจะ:
 * 1. ตรวจสอบว่า threat ยังมีอยู่
 * 2. Pause contract ผ่าน Relayer
 * 3. ส่งการแจ้งเตือนไปยัง team
 *
 * Environment variables ที่ต้องตั้งใน Defender:
 * - CONTRACT_ADDRESS: ที่อยู่ของ pausable contract
 * - SLACK_WEBHOOK_URL: สำหรับ notification
 * - MINIMUM_SWAP_THRESHOLD: threshold ที่ถือว่า suspicious
 */

import {
  DefenderRelayProvider,
  DefenderRelaySigner
} from "@openzeppelin/defender-relay-client/lib/ethers";
import { ethers } from "ethers";

interface SentinelEvent {
  type: "BLOCK" | "FORTA";
  hash?: string;
  blockNumber?: number;
  transaction?: {
    hash: string;
    from: string;
    to: string;
    value: string;
  };
  matchReasons?: Array<{
    type: string;
    signature: string;
    args: Record<string, string>;
  }>;
}

interface AutotaskEvent {
  autotaskId: string;
  autotaskName: string;
  request: {
    body: SentinelEvent;
  };
  credentials: {
    relayerApiKey: string;
    relayerApiSecret: string;
  };
}

// ABI ของ Pausable contract
const PAUSABLE_ABI = [
  "function pause() external",
  "function unpause() external",
  "function paused() external view returns (bool)",
  "function owner() external view returns (address)",
];

/**
 * Main handler - Autotask entry point
 */
exports.handler = async function(event: AutotaskEvent): Promise<void> {
  const { credentials, request } = event;

  // สร้าง provider และ signer ผ่าน Defender Relay
  const provider = new DefenderRelayProvider(credentials);
  const signer = new DefenderRelaySigner(credentials, provider, {
    speed: "fast",
  });

  const contractAddress = process.env.CONTRACT_ADDRESS!;
  const contract = new ethers.Contract(contractAddress, PAUSABLE_ABI, signer);

  try {
    // ============================================================
    // Step 1: ตรวจสอบ event ที่ trigger
    // ============================================================
    const sentinelEvent = request.body;
    console.log("Sentinel triggered:", JSON.stringify(sentinelEvent, null, 2));

    // วิเคราะห์ match reasons
    const swapMatch = sentinelEvent.matchReasons?.find(
      (r) => r.signature?.includes("Swap")
    );

    if (!swapMatch) {
      console.log("No swap match found, skipping");
      return;
    }

    // ============================================================
    // Step 2: ตรวจสอบว่า contract ยัง paused อยู่หรือไม่
    // ============================================================
    const isPaused = await contract.paused();
    if (isPaused) {
      console.log("Contract already paused, skipping");
      await notifySlack("⚠️ Contract already paused - duplicate alert?");
      return;
    }

    // ============================================================
    // Step 3: Verify threat severity
    // ============================================================
    const amount0 = BigInt(swapMatch.args.amount0 || "0");
    const threshold = BigInt(process.env.MINIMUM_SWAP_THRESHOLD || "1000000000000");

    if (Math.abs(Number(amount0)) < Number(threshold)) {
      console.log("Swap below threshold, no action needed");
      return;
    }

    // ============================================================
    // Step 4: Execute Emergency Pause
    // ============================================================
    console.log(`🚨 CRITICAL: Large swap detected. Amount: ${amount0}`);
    console.log("Executing emergency pause...");

    const tx = await contract.pause({
      gasLimit: 100_000,
    });

    console.log(`Pause transaction sent: ${tx.hash}`);
    await tx.wait();
    console.log("Contract paused successfully");

    // ============================================================
    // Step 5: Notify team
    // ============================================================
    await notifySlack(
      `🚨 *EMERGENCY PAUSE EXECUTED*\n` +
      `Contract: \`${contractAddress}\`\n` +
      `Trigger: Large swap detected\n` +
      `Amount: ${(Number(amount0) / 1e6).toFixed(2)} USDC\n` +
      `Pause Tx: \`${tx.hash}\`\n` +
      `*ACTION REQUIRED: Investigate immediately*`
    );

  } catch (error) {
    const errorMessage = error instanceof Error ? error.message : String(error);
    console.error("Autotask failed:", errorMessage);

    await notifySlack(
      `❌ *AUTOTASK FAILED*\n` +
      `Error: ${errorMessage}\n` +
      `Manual intervention required!`
    );

    throw error;  // Re-throw เพื่อให้ Defender บันทึก error
  }
};

/**
 * Send notification to Slack
 */
async function notifySlack(message: string): Promise<void> {
  const webhookUrl = process.env.SLACK_WEBHOOK_URL;
  if (!webhookUrl) return;

  try {
    const response = await fetch(webhookUrl, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ text: message }),
    });

    if (!response.ok) {
      console.error(`Slack notification failed: ${response.status}`);
    }
  } catch (e) {
    console.error("Failed to send Slack notification:", e);
  }
}
```

### 2.3 Defender Relayer: Meta-Transactions

```typescript
// defender/relayer-service.ts
/**
 * Defender Relayer Service
 *
 * Relayer ช่วยให้ส่ง transactions โดยไม่ต้องให้ user มี ETH
 * ใช้ใน:
 * - Meta-transactions (gasless UX)
 * - Automated operations (emergency pause, rebalance)
 * - Scheduled maintenance
 */

import { Relayer } from "@openzeppelin/defender-relay-client";
import { ethers } from "ethers";

interface RelayerConfig {
  apiKey: string;
  apiSecret: string;
}

// Contract ABI
const CONTRACT_ABI = [
  "function executeMetaTx(address user, bytes calldata functionData, bytes calldata signature) external",
  "function nonces(address user) external view returns (uint256)",
];

/**
 * สร้าง meta-transaction สำหรับ gasless UX
 * @param userAddress ที่อยู่ user ที่ต้องการทำธุรกรรม
 * @param functionData encoded function call
 * @param userSig ลายเซ็นของ user
 * @param config Relayer credentials
 */
export async function sendMetaTransaction(
  userAddress: string,
  functionData: string,
  userSig: string,
  config: RelayerConfig,
  contractAddress: string
): Promise<string> {
  const relayer = new Relayer(config);

  // Encode transaction data
  const iface = new ethers.Interface(CONTRACT_ABI);
  const data = iface.encodeFunctionData("executeMetaTx", [
    userAddress,
    functionData,
    userSig,
  ]);

  // ส่งผ่าน Relayer (Relayer จ่าย gas)
  const txResponse = await relayer.sendTransaction({
    to: contractAddress,
    data,
    gasLimit: "200000",
    speed: "average",
  });

  console.log(`Meta-tx sent via Relayer: ${txResponse.transactionId}`);
  return txResponse.transactionId;
}

/**
 * ติดตามสถานะ transaction
 */
export async function waitForTransaction(
  txId: string,
  config: RelayerConfig
): Promise<void> {
  const relayer = new Relayer(config);

  let attempts = 0;
  while (attempts < 30) {  // timeout 5 นาที
    const tx = await relayer.getTransaction(txId);

    if (tx.status === "mined") {
      console.log(`Transaction mined: ${tx.hash}`);
      return;
    }

    if (tx.status === "failed") {
      throw new Error(`Transaction failed: ${txId}`);
    }

    await new Promise(resolve => setTimeout(resolve, 10_000));  // รอ 10 วินาที
    attempts++;
  }

  throw new Error(`Transaction timeout: ${txId}`);
}
```

---

## 3. Custom Ethers.js Watcher

### 3.1 Protocol Health Monitor

```typescript
// monitoring/protocol-monitor.ts
import { ethers } from "ethers";
import EventEmitter from "events";

// ============================================================
//                     TYPES & INTERFACES
// ============================================================

export enum AlertSeverity {
  INFO = "INFO",
  MEDIUM = "MEDIUM",
  HIGH = "HIGH",
  CRITICAL = "CRITICAL",
}

export interface Alert {
  id: string;
  severity: AlertSeverity;
  title: string;
  description: string;
  timestamp: number;
  metadata: Record<string, unknown>;
  autoResolved: boolean;
}

export interface ProtocolHealth {
  isHealthy: boolean;
  timestamp: number;
  metrics: {
    blockNumber: number;
    tvl: bigint;
    pendingTransactions: number;
    lastActivityBlock: number;
    poolLiquidity: bigint;
  };
}

// ============================================================
//                   PROTOCOL MONITOR CLASS
// ============================================================

/**
 * ProtocolMonitor: Custom monitoring solution ด้วย ethers.js
 *
 * Features:
 * - Real-time event monitoring
 * - Health checks ทุก N seconds
 * - Anomaly detection (ด้วย statistical methods)
 * - Auto-pause เมื่อ CRITICAL alert
 * - Alert deduplication
 */
export class ProtocolMonitor extends EventEmitter {
  private provider: ethers.WebSocketProvider;
  private contracts: Map<string, ethers.Contract>;
  private healthHistory: ProtocolHealth[];
  private activeAlerts: Map<string, Alert>;
  private healthCheckInterval: NodeJS.Timer | null;
  private isRunning: boolean;

  // Baseline metrics สำหรับ anomaly detection
  private baselineMetrics: {
    avgTvl: bigint;
    avgSwapVolume: bigint;
    avgBlockTime: number;
    sampleCount: number;
  };

  constructor(
    private wsRpcUrl: string,
    private contractAddresses: {
      pool: string;
      pausable?: string;
    },
    private config: {
      healthCheckIntervalMs: number;
      anomalyThresholdPercent: number;  // % deviation ที่ถือว่า anomaly
      autoPauseOnCritical: boolean;
      maxHistoryLength: number;
    }
  ) {
    super();
    this.contracts = new Map();
    this.healthHistory = [];
    this.activeAlerts = new Map();
    this.healthCheckInterval = null;
    this.isRunning = false;
    this.baselineMetrics = {
      avgTvl: 0n,
      avgSwapVolume: 0n,
      avgBlockTime: 12,  // Ethereum ~12s block time
      sampleCount: 0,
    };

    // เริ่ม provider
    this.provider = new ethers.WebSocketProvider(wsRpcUrl);
  }

  // ============================================================
  //                      PUBLIC METHODS
  // ============================================================

  /**
   * เริ่ม monitoring
   */
  async start(): Promise<void> {
    if (this.isRunning) {
      throw new Error("Monitor is already running");
    }

    console.log("🚀 Starting Protocol Monitor...");
    this.isRunning = true;

    // Initialize contracts
    await this._initContracts();

    // เริ่ม event listeners
    await this._startEventListeners();

    // เริ่ม health check loop
    this._startHealthCheck();

    console.log("✅ Protocol Monitor started");
    this.emit("started");
  }

  /**
   * หยุด monitoring
   */
  async stop(): Promise<void> {
    if (!this.isRunning) return;

    this.isRunning = false;

    if (this.healthCheckInterval) {
      clearInterval(this.healthCheckInterval as NodeJS.Timeout);
      this.healthCheckInterval = null;
    }

    // Remove all listeners
    this.contracts.forEach((contract) => {
      contract.removeAllListeners();
    });

    // Close WebSocket
    await this.provider.destroy();

    console.log("🛑 Protocol Monitor stopped");
    this.emit("stopped");
  }

  /**
   * ดู active alerts
   */
  getActiveAlerts(): Alert[] {
    return Array.from(this.activeAlerts.values());
  }

  /**
   * ดู health history
   */
  getHealthHistory(): ProtocolHealth[] {
    return [...this.healthHistory];
  }

  // ============================================================
  //                     PRIVATE METHODS
  // ============================================================

  private async _initContracts(): Promise<void> {
    const poolAbi = [
      "event Swap(address indexed sender, address indexed recipient, int256 amount0, int256 amount1, uint160 sqrtPriceX96, uint128 liquidity, int24 tick)",
      "event Flash(address indexed sender, address indexed recipient, uint256 amount0, uint256 amount1, uint256 paid0, uint256 paid1)",
      "function liquidity() external view returns (uint128)",
      "function slot0() external view returns (uint160 sqrtPriceX96, int24 tick, uint16 observationIndex, uint16 observationCardinality, uint16 observationCardinalityNext, uint8 feeProtocol, bool unlocked)",
    ];

    const pool = new ethers.Contract(
      this.contractAddresses.pool,
      poolAbi,
      this.provider
    );
    this.contracts.set("pool", pool);

    if (this.contractAddresses.pausable) {
      const pausableAbi = [
        "function pause() external",
        "function paused() external view returns (bool)",
      ];

      // ต้องการ signer สำหรับ pause - ใช้ private key จาก env
      const privateKey = process.env.EMERGENCY_SIGNER_PRIVATE_KEY;
      if (privateKey) {
        const signer = new ethers.Wallet(privateKey, this.provider);
        const pausable = new ethers.Contract(
          this.contractAddresses.pausable,
          pausableAbi,
          signer
        );
        this.contracts.set("pausable", pausable);
      }
    }
  }

  private async _startEventListeners(): Promise<void> {
    const pool = this.contracts.get("pool")!;

    // ============================================================
    // Listen: Swap events
    // ============================================================
    pool.on("Swap", async (sender, recipient, amount0, amount1, sqrtPriceX96, liquidity, tick, event) => {
      const absAmount0 = amount0 < 0n ? -amount0 : amount0;

      // Anomaly detection: เทียบกับ baseline
      if (this.baselineMetrics.sampleCount > 10) {
        const deviationPercent =
          Number((absAmount0 * 100n) / this.baselineMetrics.avgSwapVolume) - 100;

        if (deviationPercent > this.config.anomalyThresholdPercent) {
          await this._raiseAlert({
            severity: AlertSeverity.HIGH,
            title: "Anomalous Swap Volume Detected",
            description: `Swap volume ${deviationPercent.toFixed(0)}% above baseline`,
            metadata: {
              sender,
              recipient,
              amount0: amount0.toString(),
              txHash: event.transactionHash,
              deviation: deviationPercent,
            },
          });
        }
      }

      // อัพเดท baseline
      this._updateBaseline("swapVolume", absAmount0);
    });

    // ============================================================
    // Listen: Flash Loan events
    // ============================================================
    pool.on("Flash", async (sender, recipient, amount0, amount1, paid0, paid1, event) => {
      const FLASH_THRESHOLD = ethers.parseUnits("5000000", 6); // 5M USDC

      if (amount0 > FLASH_THRESHOLD || amount1 > FLASH_THRESHOLD) {
        await this._raiseAlert({
          severity: AlertSeverity.HIGH,
          title: "Large Flash Loan",
          description: `Flash loan: ${ethers.formatUnits(amount0, 6)} USDC`,
          metadata: {
            sender,
            recipient,
            amount0: amount0.toString(),
            amount1: amount1.toString(),
            txHash: event.transactionHash,
          },
        });
      }
    });

    // ============================================================
    // Listen: Provider errors
    // ============================================================
    this.provider.on("error", async (error: Error) => {
      await this._raiseAlert({
        severity: AlertSeverity.MEDIUM,
        title: "WebSocket Provider Error",
        description: error.message,
        metadata: { error: error.stack },
      });

      // พยายาม reconnect
      await this._reconnect();
    });

    console.log("✅ Event listeners started");
  }

  private _startHealthCheck(): void {
    this.healthCheckInterval = setInterval(
      () => this._performHealthCheck(),
      this.config.healthCheckIntervalMs
    );
  }

  private async _performHealthCheck(): Promise<void> {
    try {
      const pool = this.contracts.get("pool")!;

      // ดึง metrics
      const [blockNumber, liquidity, slot0] = await Promise.all([
        this.provider.getBlockNumber(),
        pool.liquidity(),
        pool.slot0(),
      ]);

      const health: ProtocolHealth = {
        isHealthy: true,
        timestamp: Date.now(),
        metrics: {
          blockNumber,
          tvl: liquidity,
          pendingTransactions: 0,
          lastActivityBlock: blockNumber,
          poolLiquidity: liquidity,
        },
      };

      // ตรวจสอบ liquidity drop
      if (this.baselineMetrics.sampleCount > 5) {
        const liquidityDrop =
          100 -
          Number((liquidity * 100n) / this.baselineMetrics.avgTvl);

        if (liquidityDrop > 30) {
          health.isHealthy = false;

          await this._raiseAlert({
            severity: liquidityDrop > 70 ? AlertSeverity.CRITICAL : AlertSeverity.HIGH,
            title: "Significant Liquidity Drop",
            description: `Pool liquidity dropped ${liquidityDrop.toFixed(1)}%`,
            metadata: {
              currentLiquidity: liquidity.toString(),
              baseline: this.baselineMetrics.avgTvl.toString(),
              dropPercent: liquidityDrop,
            },
          });
        }
      }

      // ตรวจสอบ slot0.unlocked (ถ้า false = reentrancy guard active)
      if (!slot0.unlocked) {
        await this._raiseAlert({
          severity: AlertSeverity.CRITICAL,
          title: "Pool Locked - Possible Reentrancy",
          description: "Pool slot0.unlocked is false",
          metadata: { blockNumber },
        });
      }

      // บันทึก health
      this.healthHistory.push(health);
      if (this.healthHistory.length > this.config.maxHistoryLength) {
        this.healthHistory.shift();
      }

      // อัพเดท baseline
      this._updateBaseline("tvl", liquidity);

      this.emit("healthCheck", health);
    } catch (error) {
      const errMsg = error instanceof Error ? error.message : String(error);
      console.error("Health check failed:", errMsg);

      await this._raiseAlert({
        severity: AlertSeverity.MEDIUM,
        title: "Health Check Failed",
        description: `Cannot fetch metrics: ${errMsg}`,
        metadata: { error: errMsg },
      });
    }
  }

  private async _raiseAlert(params: {
    severity: AlertSeverity;
    title: string;
    description: string;
    metadata: Record<string, unknown>;
  }): Promise<void> {
    // Deduplication: ใช้ title + severity เป็น key
    const alertKey = `${params.severity}:${params.title}`;

    // ถ้า alert เดิมยัง active ภายใน 5 นาที ไม่ raise ซ้ำ
    const existing = this.activeAlerts.get(alertKey);
    if (existing && Date.now() - existing.timestamp < 5 * 60 * 1000) {
      return;
    }

    const alert: Alert = {
      id: `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
      ...params,
      timestamp: Date.now(),
      autoResolved: false,
    };

    this.activeAlerts.set(alertKey, alert);

    console.log(`🚨 [${alert.severity}] ${alert.title}: ${alert.description}`);

    // Emit event ให้ consumer รับ
    this.emit("alert", alert);

    // Auto-pause ถ้า CRITICAL
    if (
      params.severity === AlertSeverity.CRITICAL &&
      this.config.autoPauseOnCritical
    ) {
      await this._autoPause(alert);
    }

    // ส่งไปยัง notification channels
    await this._sendNotifications(alert);
  }

  private async _autoPause(alert: Alert): Promise<void> {
    const pausable = this.contracts.get("pausable");
    if (!pausable) {
      console.warn("No pausable contract configured");
      return;
    }

    try {
      const isPaused = await pausable.paused();
      if (isPaused) {
        console.log("Contract already paused");
        return;
      }

      console.log("⏸️ AUTO-PAUSE: Pausing contract due to CRITICAL alert...");
      const tx = await pausable.pause({ gasLimit: 100_000 });
      await tx.wait();

      console.log(`Contract paused. Tx: ${tx.hash}`);
      this.emit("autoPaused", { alert, txHash: tx.hash });
    } catch (e) {
      console.error("Auto-pause failed:", e);
      this.emit("autoPauseFailed", { alert, error: e });
    }
  }

  private async _sendNotifications(alert: Alert): Promise<void> {
    // PagerDuty สำหรับ HIGH/CRITICAL
    if (
      alert.severity === AlertSeverity.HIGH ||
      alert.severity === AlertSeverity.CRITICAL
    ) {
      await this._notifyPagerDuty(alert);
    }

    // Slack สำหรับทุก severity
    await this._notifySlack(alert);

    // Telegram สำหรับ CRITICAL
    if (alert.severity === AlertSeverity.CRITICAL) {
      await this._notifyTelegram(alert);
    }
  }

  private async _notifySlack(alert: Alert): Promise<void> {
    const webhookUrl = process.env.SLACK_WEBHOOK_URL;
    if (!webhookUrl) return;

    const severityEmoji = {
      INFO: "ℹ️",
      MEDIUM: "⚠️",
      HIGH: "🔴",
      CRITICAL: "🚨",
    }[alert.severity];

    const payload = {
      blocks: [
        {
          type: "header",
          text: {
            type: "plain_text",
            text: `${severityEmoji} [${alert.severity}] ${alert.title}`,
          },
        },
        {
          type: "section",
          text: {
            type: "mrkdwn",
            text: alert.description,
          },
        },
        {
          type: "section",
          fields: [
            {
              type: "mrkdwn",
              text: `*Alert ID:*\n\`${alert.id}\``,
            },
            {
              type: "mrkdwn",
              text: `*Timestamp:*\n${new Date(alert.timestamp).toISOString()}`,
            },
          ],
        },
      ],
    };

    try {
      await fetch(webhookUrl, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(payload),
      });
    } catch (e) {
      console.error("Slack notification failed:", e);
    }
  }

  private async _notifyPagerDuty(alert: Alert): Promise<void> {
    const routingKey = process.env.PAGERDUTY_ROUTING_KEY;
    if (!routingKey) return;

    const severity = alert.severity === AlertSeverity.CRITICAL ? "critical" : "error";

    try {
      await fetch("https://events.pagerduty.com/v2/enqueue", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          routing_key: routingKey,
          event_action: "trigger",
          dedup_key: `protocol-monitor-${alert.id}`,
          payload: {
            summary: `[${alert.severity}] ${alert.title}`,
            severity,
            source: "protocol-monitor",
            custom_details: alert.metadata,
          },
        }),
      });
    } catch (e) {
      console.error("PagerDuty notification failed:", e);
    }
  }

  private async _notifyTelegram(alert: Alert): Promise<void> {
    const botToken = process.env.TELEGRAM_BOT_TOKEN;
    const chatId = process.env.TELEGRAM_CHAT_ID;
    if (!botToken || !chatId) return;

    const message =
      `🚨 *CRITICAL ALERT*\n\n` +
      `*Title:* ${alert.title}\n` +
      `*Description:* ${alert.description}\n` +
      `*Time:* ${new Date(alert.timestamp).toISOString()}\n` +
      `*ID:* \`${alert.id}\``;

    try {
      await fetch(`https://api.telegram.org/bot${botToken}/sendMessage`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          chat_id: chatId,
          text: message,
          parse_mode: "Markdown",
        }),
      });
    } catch (e) {
      console.error("Telegram notification failed:", e);
    }
  }

  private _updateBaseline(
    metric: "tvl" | "swapVolume",
    value: bigint
  ): void {
    const n = BigInt(this.baselineMetrics.sampleCount + 1);

    if (metric === "tvl") {
      this.baselineMetrics.avgTvl =
        (this.baselineMetrics.avgTvl * BigInt(this.baselineMetrics.sampleCount) + value) / n;
    } else if (metric === "swapVolume") {
      this.baselineMetrics.avgSwapVolume =
        (this.baselineMetrics.avgSwapVolume * BigInt(this.baselineMetrics.sampleCount) + value) / n;
    }

    this.baselineMetrics.sampleCount++;
  }

  private async _reconnect(): Promise<void> {
    console.log("🔄 Reconnecting WebSocket provider...");

    try {
      await this.provider.destroy();
      this.provider = new ethers.WebSocketProvider(this.wsRpcUrl);
      await this._initContracts();
      await this._startEventListeners();
      console.log("✅ Reconnected successfully");
    } catch (e) {
      console.error("Reconnection failed:", e);
      // ลอง reconnect อีกครั้งหลังจาก 30 วินาที
      setTimeout(() => this._reconnect(), 30_000);
    }
  }
}
```

### 3.2 Monitor Entry Point

```typescript
// monitoring/index.ts
import { ProtocolMonitor, AlertSeverity, Alert } from "./protocol-monitor";

async function main() {
  const monitor = new ProtocolMonitor(
    process.env.WS_RPC_URL || "wss://mainnet.infura.io/ws/v3/YOUR_KEY",
    {
      pool: "0x8ad599c3A0ff1De082011EFDDc58f1908eb6e6D5",
      pausable: process.env.PAUSABLE_CONTRACT,
    },
    {
      healthCheckIntervalMs: 30_000,        // ทุก 30 วินาที
      anomalyThresholdPercent: 500,         // 5x จาก baseline
      autoPauseOnCritical: process.env.NODE_ENV === "production",
      maxHistoryLength: 1000,
    }
  );

  // ============================================================
  // Event Handlers
  // ============================================================
  monitor.on("alert", (alert: Alert) => {
    console.log(`📢 Alert received: [${alert.severity}] ${alert.title}`);

    // บันทึกลง database (ตัวอย่าง)
    saveAlertToDatabase(alert);
  });

  monitor.on("autoPaused", ({ alert, txHash }) => {
    console.log(`⏸️ Contract auto-paused due to: ${alert.title}`);
    console.log(`Pause tx: ${txHash}`);
  });

  monitor.on("healthCheck", (health) => {
    if (!health.isHealthy) {
      console.warn("⚠️ Protocol health check FAILED");
    }
  });

  // ============================================================
  // Graceful shutdown
  // ============================================================
  process.on("SIGINT", async () => {
    console.log("\nShutting down monitor...");
    await monitor.stop();
    process.exit(0);
  });

  process.on("SIGTERM", async () => {
    await monitor.stop();
    process.exit(0);
  });

  // เริ่ม monitor
  await monitor.start();
}

async function saveAlertToDatabase(alert: Alert): Promise<void> {
  // TODO: implement database saving
  console.log(`💾 Saving alert ${alert.id} to database...`);
}

main().catch(console.error);
```

---

## 4. Alert Severity Tiers และ Response Procedures

### 4.1 Severity Matrix

```typescript
// monitoring/severity-matrix.ts

/**
 * Alert Severity Matrix
 *
 * ตาราง severity, trigger conditions, response times, และ actions
 */

export const SEVERITY_MATRIX = {
  INFO: {
    level: 1,
    color: "🔵",
    description: "ข้อมูลทั่วไป ไม่ต้องดำเนินการ",
    triggers: [
      "Normal large transaction",
      "New address interaction",
      "Parameter query",
    ],
    responseTime: "No response required",
    notificationChannels: ["slack-info-channel"],
    actions: ["Log to database"],
    escalateAfter: null,
  },

  MEDIUM: {
    level: 2,
    color: "🟡",
    description: "ควรตรวจสอบภายใน 1 ชั่วโมง",
    triggers: [
      "Swap volume 2x จาก baseline",
      "Gas price spike > 300 gwei",
      "New admin action",
      "Configuration change",
    ],
    responseTime: "1 hour",
    notificationChannels: ["slack-alerts-channel"],
    actions: [
      "Log to database",
      "Assign to on-call engineer",
      "Investigate root cause",
    ],
    escalateAfter: "2 hours",
  },

  HIGH: {
    level: 3,
    color: "🔴",
    description: "ต้องตอบสนองภายใน 15 นาที",
    triggers: [
      "Flash loan > 5M USD",
      "Swap volume 5x จาก baseline",
      "Liquidity drop > 30%",
      "Admin key used from new address",
      "Multiple failed transactions in row",
    ],
    responseTime: "15 minutes",
    notificationChannels: ["slack-critical", "pagerduty"],
    actions: [
      "Page on-call engineer immediately",
      "Start incident channel in Slack",
      "Investigate and assess threat",
      "Consider emergency pause",
    ],
    escalateAfter: "30 minutes",
  },

  CRITICAL: {
    level: 4,
    color: "🚨",
    description: "ต้องดำเนินการทันที funds อาจตกอยู่ในอันตราย",
    triggers: [
      "Reentrancy attack detected",
      "Funds draining from contract",
      "Price manipulation confirmed",
      "Liquidity drop > 70%",
      "Emergency function called",
    ],
    responseTime: "Immediate",
    notificationChannels: ["slack-critical", "pagerduty", "telegram", "phone"],
    actions: [
      "Auto-pause contract (ถ้าเปิดใช้)",
      "Page entire team",
      "Start war room",
      "Contact protocol partners",
      "Prepare incident report",
      "Consider White Hat negotiation",
    ],
    escalateAfter: "5 minutes",
  },
} as const;
```

### 4.2 Smart Contract สำหรับ Emergency Response

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {AccessControl} from "@openzeppelin/contracts/access/AccessControl.sol";
import {Pausable} from "@openzeppelin/contracts/utils/Pausable.sol";

/**
 * @title EmergencyController
 * @notice Contract สำหรับ emergency response automation
 *
 * Roles:
 * - DEFAULT_ADMIN_ROLE: ทำได้ทุกอย่าง
 * - GUARDIAN_ROLE: pause/unpause ได้ (monitoring bots ใช้ role นี้)
 * - INCIDENT_RESPONDER_ROLE: execute emergency actions
 */
contract EmergencyController is AccessControl, Pausable {

    // ============================================================
    //                         ROLES
    // ============================================================

    bytes32 public constant GUARDIAN_ROLE = keccak256("GUARDIAN_ROLE");
    bytes32 public constant INCIDENT_RESPONDER_ROLE = keccak256("INCIDENT_RESPONDER_ROLE");

    // ============================================================
    //                         STORAGE
    // ============================================================

    struct Incident {
        uint256 id;
        string description;
        uint256 severity;        // 1=INFO, 2=MEDIUM, 3=HIGH, 4=CRITICAL
        uint256 timestamp;
        address reportedBy;
        bool resolved;
        string resolution;
    }

    uint256 public incidentCounter;
    mapping(uint256 => Incident) public incidents;

    /// @notice ที่อยู่ protocol contracts ที่ controller ดูแล
    address[] public protectedContracts;

    /// @notice threshold สำหรับ auto-pause (จำนวน HIGH alerts ใน 10 นาที)
    uint256 public autoPauseThreshold = 3;

    /// @notice นับ HIGH alerts ใน window
    mapping(uint256 => uint256) public alertCountInWindow;  // timestamp bucket => count

    // ============================================================
    //                         EVENTS
    // ============================================================

    event IncidentCreated(
        uint256 indexed incidentId,
        string description,
        uint256 severity,
        address reportedBy
    );

    event IncidentResolved(
        uint256 indexed incidentId,
        string resolution,
        address resolvedBy
    );

    event EmergencyPause(
        address indexed triggeredBy,
        string reason,
        uint256 incidentId
    );

    event EmergencyUnpause(
        address indexed triggeredBy,
        string reason
    );

    // ============================================================
    //                         ERRORS
    // ============================================================

    error IncidentNotFound(uint256 incidentId);
    error IncidentAlreadyResolved(uint256 incidentId);
    error InvalidSeverity(uint256 severity);

    // ============================================================
    //                       CONSTRUCTOR
    // ============================================================

    constructor(address admin, address[] memory guardians) {
        _grantRole(DEFAULT_ADMIN_ROLE, admin);

        for (uint i = 0; i < guardians.length; i++) {
            _grantRole(GUARDIAN_ROLE, guardians[i]);
        }
    }

    // ============================================================
    //                  EMERGENCY FUNCTIONS
    // ============================================================

    /**
     * @notice Pause protocol ฉุกเฉิน
     * @param reason สาเหตุของการ pause
     */
    function emergencyPause(string calldata reason)
        external
        onlyRole(GUARDIAN_ROLE)
    {
        _pause();

        // สร้าง incident อัตโนมัติ
        uint256 incidentId = _createIncident(reason, 4, msg.sender);

        emit EmergencyPause(msg.sender, reason, incidentId);
    }

    /**
     * @notice Unpause protocol
     * @param reason สาเหตุของการ unpause
     */
    function emergencyUnpause(string calldata reason)
        external
        onlyRole(INCIDENT_RESPONDER_ROLE)
    {
        _unpause();
        emit EmergencyUnpause(msg.sender, reason);
    }

    /**
     * @notice รายงาน incident ใหม่
     * @param description คำอธิบาย incident
     * @param severity ระดับความรุนแรง (1-4)
     */
    function reportIncident(
        string calldata description,
        uint256 severity
    ) external onlyRole(GUARDIAN_ROLE) returns (uint256 incidentId) {
        if (severity == 0 || severity > 4) revert InvalidSeverity(severity);
        return _createIncident(description, severity, msg.sender);
    }

    /**
     * @notice แก้ไข incident
     * @param incidentId ID ของ incident
     * @param resolution คำอธิบายการแก้ไข
     */
    function resolveIncident(
        uint256 incidentId,
        string calldata resolution
    ) external onlyRole(INCIDENT_RESPONDER_ROLE) {
        if (incidentId >= incidentCounter) revert IncidentNotFound(incidentId);
        Incident storage incident = incidents[incidentId];
        if (incident.resolved) revert IncidentAlreadyResolved(incidentId);

        incident.resolved = true;
        incident.resolution = resolution;

        emit IncidentResolved(incidentId, resolution, msg.sender);
    }

    // ============================================================
    //                    INTERNAL FUNCTIONS
    // ============================================================

    function _createIncident(
        string memory description,
        uint256 severity,
        address reporter
    ) internal returns (uint256 incidentId) {
        incidentId = incidentCounter++;

        incidents[incidentId] = Incident({
            id: incidentId,
            description: description,
            severity: severity,
            timestamp: block.timestamp,
            reportedBy: reporter,
            resolved: false,
            resolution: ""
        });

        emit IncidentCreated(incidentId, description, severity, reporter);
    }
}
```

---

## 5. On-Call Rotation และ Incident Runbooks

### 5.1 On-Call Rotation Design

```typescript
// ops/on-call-rotation.ts

/**
 * On-Call Rotation Configuration
 *
 * ใช้ PagerDuty หรือ OpsGenie สำหรับ rotation จริง
 * ตัวอย่างนี้เป็น minimal implementation
 */

export interface OnCallEngineer {
  name: string;
  email: string;
  phone: string;
  telegram?: string;
  timezone: string;
  skills: ("solidity" | "infra" | "defi" | "incident-response")[];
}

export const ON_CALL_TEAM: OnCallEngineer[] = [
  {
    name: "Alice Chen",
    email: "alice@protocol.io",
    phone: "+1-555-0001",
    telegram: "@alice_devsec",
    timezone: "America/New_York",
    skills: ["solidity", "incident-response"],
  },
  {
    name: "Bob Tanaka",
    email: "bob@protocol.io",
    phone: "+1-555-0002",
    telegram: "@bob_infra",
    timezone: "Asia/Tokyo",
    skills: ["infra", "incident-response"],
  },
  {
    name: "Carol Smith",
    email: "carol@protocol.io",
    phone: "+1-555-0003",
    timezone: "Europe/London",
    skills: ["defi", "solidity", "incident-response"],
  },
];

/**
 * คำนวณว่าใครเป็น on-call ตอนนี้
 * Rotation: ทุก 1 สัปดาห์
 */
export function getCurrentOnCall(): OnCallEngineer {
  const weekNumber = Math.floor(Date.now() / (7 * 24 * 60 * 60 * 1000));
  const index = weekNumber % ON_CALL_TEAM.length;
  return ON_CALL_TEAM[index];
}

/**
 * Escalation Policy
 *
 * Level 1: Primary on-call (0-15 min)
 * Level 2: Secondary on-call (15-30 min)
 * Level 3: Tech Lead (30-45 min)
 * Level 4: All hands (45+ min)
 */
export function getEscalationPolicy(): OnCallEngineer[] {
  const weekNumber = Math.floor(Date.now() / (7 * 24 * 60 * 60 * 1000));
  return [
    ON_CALL_TEAM[weekNumber % ON_CALL_TEAM.length],
    ON_CALL_TEAM[(weekNumber + 1) % ON_CALL_TEAM.length],
    ON_CALL_TEAM[(weekNumber + 2) % ON_CALL_TEAM.length],
  ];
}
```

### 5.2 Incident Runbook Template

```markdown
# Incident Runbook: Large Flash Loan Detected

**Alert ID**: UNISWAP-LARGE-FLASH-LOAN
**Severity**: HIGH
**Last Updated**: 2024-01-01

## Immediate Steps (0-5 minutes)

1. **Acknowledge alert** ใน PagerDuty
2. **Join war room**: https://protocol.slack.com/archives/incident-XXXXX
3. **ตรวจสอบ transaction**:
   - เปิด Etherscan: `https://etherscan.io/tx/{txHash}`
   - ดูว่า flash loan มาจาก contract ใด
   - ตรวจสอบว่ามี profit หรือไม่ (ถ้ามี = exploit สำเร็จ)

## Assessment (5-15 minutes)

1. **ตรวจสอบ pool balance**:
   ```bash
   cast call 0x8ad599c3... "liquidity()(uint128)" --rpc-url $RPC_URL
   ```

2. **ตรวจสอบ price impact**:
   ```bash
   cast call 0x8ad599c3... "slot0()(uint160,int24,...)" --rpc-url $RPC_URL
   ```

3. **สอบถาม on-chain data**:
   - มี approve ที่น่าสงสัยก่อนหน้าไหม?
   - Attacker มี history ที่ไหนบ้าง?

## Decision Tree

```
Flash loan detected
    ↓
pool balance normal?
    ├── YES → Monitor อย่างใกล้ชิด, ไม่ต้อง pause
    └── NO →
        ↓
        funds stolen?
            ├── YES → PAUSE IMMEDIATELY + Contact security experts
            └── NO → Investigate further, consider pause
```

## Escalation

- **15 min**: ถ้า investigate ไม่เสร็จ → escalate to Level 2
- **30 min**: ถ้ายังไม่ resolve → escalate to Tech Lead
- **45 min**: ถ้า funds at risk → All-hands war room

## Post-Incident

1. Write incident report ภายใน 24 ชั่วโมง
2. Update runbook ถ้าพบ gap
3. Review monitoring rules
4. Debrief meeting ภายใน 72 ชั่วโมง
```

---

## Workshop: สร้าง Monitoring System

### Workshop Exercise 1: เพิ่ม Rate Limiting Detection

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/**
 * @title RateLimitedProtocol
 * @notice ตัวอย่าง protocol ที่มี rate limiting
 * @dev Workshop: เพิ่ม monitoring สำหรับตรวจจับ rate limit violations
 */
contract RateLimitedProtocol {

    mapping(address => uint256) public lastOperationTime;
    mapping(address => uint256) public operationCount;
    uint256 public constant COOLDOWN_PERIOD = 1 minutes;
    uint256 public constant MAX_OPS_PER_HOUR = 10;

    event OperationExecuted(address indexed user, uint256 opCount);
    event RateLimitHit(address indexed user, uint256 attemptedAt);
    event SuspiciousActivity(address indexed user, string reason);

    error CooldownNotExpired(uint256 nextAllowed);
    error TooManyOperations(uint256 current, uint256 max);

    /**
     * @notice ทำ operation (จำลอง)
     * Workshop: เพิ่ม event สำหรับ monitoring anomaly
     */
    function executeOperation() external {
        uint256 currentTime = block.timestamp;

        // Check cooldown
        if (currentTime < lastOperationTime[msg.sender] + COOLDOWN_PERIOD) {
            emit RateLimitHit(msg.sender, currentTime);
            revert CooldownNotExpired(lastOperationTime[msg.sender] + COOLDOWN_PERIOD);
        }

        // Reset hourly count ถ้าผ่าน 1 ชั่วโมงแล้ว
        uint256 hourBucket = currentTime / 1 hours;
        uint256 lastHourBucket = lastOperationTime[msg.sender] / 1 hours;

        if (hourBucket > lastHourBucket) {
            operationCount[msg.sender] = 0;
        }

        if (operationCount[msg.sender] >= MAX_OPS_PER_HOUR) {
            emit SuspiciousActivity(msg.sender, "Hourly limit exceeded");
            revert TooManyOperations(operationCount[msg.sender], MAX_OPS_PER_HOUR);
        }

        lastOperationTime[msg.sender] = currentTime;
        operationCount[msg.sender]++;

        emit OperationExecuted(msg.sender, operationCount[msg.sender]);
    }
}
```

### Workshop Exercise 2: Test Monitoring System

```typescript
// test/monitor.test.ts
import { ProtocolMonitor, AlertSeverity } from "../monitoring/protocol-monitor";
import { describe, it, expect, beforeEach, afterEach } from "@jest/globals";

describe("ProtocolMonitor", () => {
  let monitor: ProtocolMonitor;

  beforeEach(() => {
    monitor = new ProtocolMonitor(
      "wss://fake-rpc.example.com",
      { pool: "0x1234...5678" },
      {
        healthCheckIntervalMs: 1000,
        anomalyThresholdPercent: 200,
        autoPauseOnCritical: false,  // ปิด auto-pause ใน test
        maxHistoryLength: 100,
      }
    );
  });

  it("should emit alert on high severity event", (done) => {
    monitor.on("alert", (alert) => {
      expect(alert.severity).toBe(AlertSeverity.HIGH);
      done();
    });

    // Simulate reentrancy detection
    (monitor as any)._raiseAlert({
      severity: AlertSeverity.HIGH,
      title: "Test Alert",
      description: "Test",
      metadata: {},
    });
  });

  it("should deduplicate alerts within 5 minutes", async () => {
    const alerts: unknown[] = [];
    monitor.on("alert", (alert) => alerts.push(alert));

    // Raise same alert twice
    await (monitor as any)._raiseAlert({
      severity: AlertSeverity.HIGH,
      title: "Duplicate Alert",
      description: "Test",
      metadata: {},
    });

    await (monitor as any)._raiseAlert({
      severity: AlertSeverity.HIGH,
      title: "Duplicate Alert",  // same title + severity
      description: "Test",
      metadata: {},
    });

    // ควรได้รับแค่ 1 alert
    expect(alerts.length).toBe(1);
  });
});
```

---

## สรุป Part 69

- **Forta Bot** ตรวจจับ threats แบบ decentralized ด้วย `handleTransaction` และ `handleBlock`
- **OpenZeppelin Defender** มี 3 components หลัก: Sentinel (event monitoring), Autotask (serverless response), Relayer (meta-transactions)
- **Custom Ethers.js Watcher** ให้ flexibility สูงสุด รองรับ anomaly detection, auto-pause, และ multi-channel notifications
- **Alert Severity Tiers**: INFO (no action) → MEDIUM (1 hour) → HIGH (15 min) → CRITICAL (immediate)
- **On-Call Rotation** ควรมี escalation policy และ runbooks ที่ชัดเจน
- **Auto-pause** ต้องใช้ด้วยความระมัดระวัง - pause เองอาจทำให้ users เสียหายได้

## Next: Part 70 - Mainnet Fork Testing
