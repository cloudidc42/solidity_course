# Part 41: Full-Stack DApp Development

## สารบัญ
1. DApp Architecture
2. ethers.js / viem Integration
3. React + Wagmi Hooks
4. Contract ABI Management
5. Workshop: Complete DApp

---

## 1. DApp Architecture

```
Full-Stack DApp:

Frontend (React/Next.js)
    ↓ ethers.js / viem / wagmi
Wallet (MetaMask/WalletConnect)
    ↓ JSON-RPC
Node (Infura/Alchemy/Self-hosted)
    ↓
Smart Contracts (Ethereum/L2)

Backend (Optional):
- Indexer (The Graph / custom)
- API (Express/FastAPI)
- Database (PostgreSQL/MongoDB)
- IPFS (Pinata/web3.storage)

Key Tools:
- wagmi: React hooks for Ethereum
- viem: TypeScript Ethereum library
- RainbowKit: wallet connection UI
- The Graph: on-chain data indexing
- Hardhat: development framework
```

---

## 2. Contract + TypeScript Integration

```typescript
// hardhat.config.ts
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
import "@typechain/hardhat";

const config: HardhatUserConfig = {
  solidity: {
    version: "0.8.24",
    settings: {
      optimizer: { enabled: true, runs: 200 },
    },
  },
  networks: {
    hardhat: {},
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL!,
      accounts: [process.env.PRIVATE_KEY!],
    },
    mainnet: {
      url: process.env.MAINNET_RPC_URL!,
      accounts: [process.env.PRIVATE_KEY!],
    },
  },
  typechain: {
    outDir: "typechain-types",
    target: "ethers-v6",
  },
  etherscan: {
    apiKey: process.env.ETHERSCAN_API_KEY!,
  },
};

export default config;
```

```typescript
// scripts/deploy.ts
import { ethers } from "hardhat";
import { MyToken__factory } from "../typechain-types";

async function main() {
  const [deployer] = await ethers.getSigners();
  
  console.log(`Deploying from: ${deployer.address}`);
  console.log(`Balance: ${ethers.formatEther(await deployer.provider.getBalance(deployer.address))} ETH`);
  
  // Deploy with TypeChain factory
  const factory = new MyToken__factory(deployer);
  const token = await factory.deploy(
    "My Token",
    "MTK",
    ethers.parseEther("1000000") // 1M tokens
  );
  
  await token.waitForDeployment();
  const address = await token.getAddress();
  
  console.log(`Token deployed to: ${address}`);
  
  // Verify on Etherscan
  if (process.env.ETHERSCAN_API_KEY) {
    console.log("Waiting 5 confirmations before verify...");
    await token.deploymentTransaction()?.wait(5);
    
    await run("verify:verify", {
      address,
      constructorArguments: ["My Token", "MTK", ethers.parseEther("1000000")],
    });
  }
}

main().catch(console.error);
```

```typescript
// lib/contract.ts
import { ethers } from "ethers";
import { MyToken, MyToken__factory } from "../typechain-types";

const TOKEN_ADDRESS = "0x..."; // deployed address

export async function getToken(signerOrProvider: ethers.Signer | ethers.Provider): Promise<MyToken> {
  return MyToken__factory.connect(TOKEN_ADDRESS, signerOrProvider);
}

export async function getBalance(userAddress: string): Promise<string> {
  const provider = new ethers.JsonRpcProvider(process.env.NEXT_PUBLIC_RPC_URL);
  const token = await getToken(provider);
  
  const balance = await token.balanceOf(userAddress);
  return ethers.formatEther(balance);
}

export async function transferTokens(
  signer: ethers.Signer,
  to: string,
  amount: string
): Promise<ethers.TransactionResponse> {
  const token = await getToken(signer);
  const amountWei = ethers.parseEther(amount);
  
  // Estimate gas first
  const gasEstimate = await token.transfer.estimateGas(to, amountWei);
  
  return token.transfer(to, amountWei, {
    gasLimit: gasEstimate * 120n / 100n, // 20% buffer
  });
}
```

---

## 3. React + Wagmi Hooks

```typescript
// app/providers.tsx
"use client";

import { WagmiProvider, createConfig, http } from "wagmi";
import { mainnet, sepolia } from "wagmi/chains";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { RainbowKitProvider, getDefaultConfig } from "@rainbow-me/rainbowkit";
import "@rainbow-me/rainbowkit/styles.css";

const config = getDefaultConfig({
  appName: "My DApp",
  projectId: process.env.NEXT_PUBLIC_WALLETCONNECT_ID!,
  chains: [mainnet, sepolia],
  transports: {
    [mainnet.id]: http(process.env.NEXT_PUBLIC_MAINNET_RPC!),
    [sepolia.id]: http(process.env.NEXT_PUBLIC_SEPOLIA_RPC!),
  },
});

const queryClient = new QueryClient();

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <WagmiProvider config={config}>
      <QueryClientProvider client={queryClient}>
        <RainbowKitProvider>
          {children}
        </RainbowKitProvider>
      </QueryClientProvider>
    </WagmiProvider>
  );
}
```

```typescript
// components/TokenCard.tsx
"use client";

import { useAccount, useReadContract, useWriteContract, useWaitForTransactionReceipt } from "wagmi";
import { parseEther, formatEther } from "viem";
import { useState } from "react";

const TOKEN_ABI = [
  {
    name: "balanceOf",
    type: "function",
    inputs: [{ name: "account", type: "address" }],
    outputs: [{ name: "", type: "uint256" }],
    stateMutability: "view",
  },
  {
    name: "transfer",
    type: "function",
    inputs: [
      { name: "to", type: "address" },
      { name: "amount", type: "uint256" },
    ],
    outputs: [{ name: "", type: "bool" }],
    stateMutability: "nonpayable",
  },
  {
    name: "Transfer",
    type: "event",
    inputs: [
      { name: "from", type: "address", indexed: true },
      { name: "to", type: "address", indexed: true },
      { name: "value", type: "uint256", indexed: false },
    ],
  },
] as const;

const TOKEN_ADDRESS = "0x..." as `0x${string}`;

export function TokenCard() {
  const { address, isConnected } = useAccount();
  const [recipient, setRecipient] = useState("");
  const [amount, setAmount] = useState("");
  
  // Read balance
  const { data: balance, refetch } = useReadContract({
    address: TOKEN_ADDRESS,
    abi: TOKEN_ABI,
    functionName: "balanceOf",
    args: [address!],
    query: { enabled: !!address },
  });
  
  // Write transfer
  const { writeContract, data: hash, isPending } = useWriteContract();
  
  // Wait for confirmation
  const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({
    hash,
    onSuccess: () => refetch(), // Refresh balance after success
  });
  
  const handleTransfer = () => {
    if (!recipient || !amount) return;
    
    writeContract({
      address: TOKEN_ADDRESS,
      abi: TOKEN_ABI,
      functionName: "transfer",
      args: [recipient as `0x${string}`, parseEther(amount)],
    });
  };
  
  if (!isConnected) return <p>Please connect wallet</p>;
  
  return (
    <div className="p-6 border rounded-lg">
      <h2 className="text-xl font-bold mb-4">Token Balance</h2>
      
      <p className="text-3xl font-mono mb-6">
        {balance ? formatEther(balance) : "0"} MTK
      </p>
      
      <div className="space-y-3">
        <input
          type="text"
          placeholder="Recipient address"
          value={recipient}
          onChange={(e) => setRecipient(e.target.value)}
          className="w-full border p-2 rounded"
        />
        <input
          type="number"
          placeholder="Amount"
          value={amount}
          onChange={(e) => setAmount(e.target.value)}
          className="w-full border p-2 rounded"
        />
        <button
          onClick={handleTransfer}
          disabled={isPending || isConfirming}
          className="w-full bg-blue-600 text-white p-2 rounded disabled:opacity-50"
        >
          {isPending ? "Signing..." : isConfirming ? "Confirming..." : "Transfer"}
        </button>
        
        {isSuccess && (
          <p className="text-green-600">Transfer successful!</p>
        )}
        {hash && (
          <a 
            href={`https://etherscan.io/tx/${hash}`}
            target="_blank"
            className="text-blue-500 text-sm underline"
          >
            View on Etherscan
          </a>
        )}
      </div>
    </div>
  );
}
```

---

## 4. Event Listening & Real-time Updates

```typescript
// hooks/useTokenEvents.ts
import { useWatchContractEvent } from "wagmi";
import { formatEther } from "viem";
import { useState } from "react";

interface TransferEvent {
  from: string;
  to: string;
  value: bigint;
  txHash: string;
}

export function useTokenEvents(tokenAddress: `0x${string}`) {
  const [events, setEvents] = useState<TransferEvent[]>([]);
  
  useWatchContractEvent({
    address: tokenAddress,
    abi: [{
      name: "Transfer",
      type: "event",
      inputs: [
        { name: "from", type: "address", indexed: true },
        { name: "to", type: "address", indexed: true },
        { name: "value", type: "uint256", indexed: false },
      ],
    }] as const,
    eventName: "Transfer",
    onLogs: (logs) => {
      const newEvents = logs.map((log) => ({
        from: log.args.from!,
        to: log.args.to!,
        value: log.args.value!,
        txHash: log.transactionHash,
      }));
      setEvents((prev) => [...newEvents, ...prev].slice(0, 50)); // Keep last 50
    },
  });
  
  return events;
}

// Usage in component:
// const events = useTokenEvents("0x...");
// events.map(e => <div>{formatEther(e.value)} MTK transferred</div>)
```

---

## สรุป Part 41

Full-Stack DApp ที่เรียนรู้:
- ✅ DApp architecture overview
- ✅ Hardhat + TypeChain setup
- ✅ Deploy scripts with verification
- ✅ wagmi hooks (useReadContract, useWriteContract)
- ✅ RainbowKit wallet connection
- ✅ Event listening real-time

## Next: Part 42 - The Graph Protocol (Indexing)
