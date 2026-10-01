# Part 95: Capstone - Frontend & Integration

## บทนำ

ใน Part 95 เราจะสร้าง **Frontend** สำหรับ OmniYield ด้วย:
- **React 18** + **TypeScript**
- **Wagmi V2** สำหรับ Ethereum interactions
- **Viem** สำหรับ type-safe contract calls
- **The Graph** สำหรับ indexed blockchain data
- **TailwindCSS** สำหรับ styling

---

## 1. Project Setup

### 1.1 Create Vite Project

```bash
# สร้าง project
npm create vite@latest omniyield-app -- --template react-ts

cd omniyield-app

# ติดตั้ง dependencies
npm install wagmi viem @tanstack/react-query

# UI dependencies
npm install @headlessui/react @heroicons/react

# Graph dependencies
npm install @apollo/client graphql

# Utilities
npm install dayjs numeral

# Dev
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### 1.2 Project Structure

```
omniyield-app/
├── src/
│   ├── main.tsx                    # entry point
│   ├── App.tsx                     # root component
│   ├── config/
│   │   ├── wagmi.ts                # wagmi config
│   │   ├── contracts.ts            # contract addresses + ABIs
│   │   └── graph.ts                # GraphQL client
│   ├── components/
│   │   ├── vault/
│   │   │   ├── VaultCard.tsx       # แสดง APY, TVL, position
│   │   │   ├── DepositModal.tsx    # deposit ด้วย permit
│   │   │   ├── WithdrawModal.tsx   # withdraw
│   │   │   └── StrategyList.tsx    # แสดง strategies
│   │   ├── governance/
│   │   │   ├── ProposalList.tsx    # list proposals
│   │   │   ├── ProposalCard.tsx    # proposal detail
│   │   │   ├── VoteModal.tsx       # cast vote
│   │   │   └── DelegatePanel.tsx   # delegate votes
│   │   └── shared/
│   │       ├── ConnectButton.tsx
│   │       ├── TokenAmount.tsx
│   │       └── TransactionButton.tsx
│   ├── hooks/
│   │   ├── useVault.ts             # vault hooks
│   │   ├── useGovernance.ts        # governance hooks
│   │   └── useTokens.ts            # token hooks
│   ├── queries/
│   │   ├── vault.graphql           # vault queries
│   │   └── governance.graphql      # governance queries
│   └── utils/
│       ├── formatters.ts
│       └── constants.ts
├── package.json
└── index.html
```

---

## 2. Wagmi Configuration

```typescript
// src/config/wagmi.ts
import { http, createConfig } from 'wagmi'
import { mainnet, sepolia, hardhat } from 'wagmi/chains'
import { injected, walletConnect, metaMask, coinbaseWallet } from 'wagmi/connectors'

// WalletConnect Project ID (จาก cloud.walletconnect.com)
const WC_PROJECT_ID = import.meta.env.VITE_WC_PROJECT_ID as string

export const wagmiConfig = createConfig({
  chains: [mainnet, sepolia, hardhat],
  connectors: [
    injected(),
    metaMask(),
    walletConnect({
      projectId: WC_PROJECT_ID,
    }),
    coinbaseWallet({
      appName: 'OmniYield',
    }),
  ],
  transports: {
    [mainnet.id]: http(import.meta.env.VITE_MAINNET_RPC),
    [sepolia.id]: http(import.meta.env.VITE_SEPOLIA_RPC),
    [hardhat.id]: http('http://localhost:8545'),
  },
})

// ประกาศ type เพื่อ type inference
declare module 'wagmi' {
  interface Register {
    config: typeof wagmiConfig
  }
}
```

### 2.2 Contract Addresses & ABIs

```typescript
// src/config/contracts.ts
import { Address } from 'viem'

// ─── Addresses ────────────────────────────────────────────────

export const CONTRACTS = {
  sepolia: {
    VAULT: '0x...' as Address,
    OYT_TOKEN: '0x...' as Address,
    VE_OYT: '0x...' as Address,
    GOVERNOR: '0x...' as Address,
    TIMELOCK: '0x...' as Address,
    REGISTRY: '0x...' as Address,
    USDC: '0x...' as Address, // test USDC
  },
  mainnet: {
    VAULT: '0x...' as Address,
    OYT_TOKEN: '0x...' as Address,
    VE_OYT: '0x...' as Address,
    GOVERNOR: '0x...' as Address,
    TIMELOCK: '0x...' as Address,
    REGISTRY: '0x...' as Address,
    USDC: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48' as Address,
  },
} as const

// ─── ABIs ─────────────────────────────────────────────────────

export const VAULT_ABI = [
  // ERC-4626
  {
    name: 'deposit',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'assets', type: 'uint256' },
      { name: 'receiver', type: 'address' },
    ],
    outputs: [{ name: 'shares', type: 'uint256' }],
  },
  {
    name: 'withdraw',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'assets', type: 'uint256' },
      { name: 'receiver', type: 'address' },
      { name: 'owner', type: 'address' },
    ],
    outputs: [{ name: 'shares', type: 'uint256' }],
  },
  {
    name: 'redeem',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'shares', type: 'uint256' },
      { name: 'receiver', type: 'address' },
      { name: 'owner', type: 'address' },
    ],
    outputs: [{ name: 'assets', type: 'uint256' }],
  },
  {
    name: 'totalAssets',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ name: '', type: 'uint256' }],
  },
  {
    name: 'totalSupply',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ name: '', type: 'uint256' }],
  },
  {
    name: 'balanceOf',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'account', type: 'address' }],
    outputs: [{ name: '', type: 'uint256' }],
  },
  {
    name: 'convertToAssets',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'shares', type: 'uint256' }],
    outputs: [{ name: 'assets', type: 'uint256' }],
  },
  {
    name: 'maxWithdraw',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'owner', type: 'address' }],
    outputs: [{ name: '', type: 'uint256' }],
  },
  {
    name: 'asset',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ name: '', type: 'address' }],
  },
  {
    name: 'getStrategies',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ name: '', type: 'address[]' }],
  },
] as const

export const ERC20_ABI = [
  {
    name: 'approve',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'spender', type: 'address' },
      { name: 'amount', type: 'uint256' },
    ],
    outputs: [{ name: '', type: 'bool' }],
  },
  {
    name: 'allowance',
    type: 'function',
    stateMutability: 'view',
    inputs: [
      { name: 'owner', type: 'address' },
      { name: 'spender', type: 'address' },
    ],
    outputs: [{ name: '', type: 'uint256' }],
  },
  {
    name: 'balanceOf',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'account', type: 'address' }],
    outputs: [{ name: '', type: 'uint256' }],
  },
  {
    name: 'decimals',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ name: '', type: 'uint8' }],
  },
  // EIP-2612 Permit
  {
    name: 'permit',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'owner', type: 'address' },
      { name: 'spender', type: 'address' },
      { name: 'value', type: 'uint256' },
      { name: 'deadline', type: 'uint256' },
      { name: 'v', type: 'uint8' },
      { name: 'r', type: 'bytes32' },
      { name: 's', type: 'bytes32' },
    ],
    outputs: [],
  },
  {
    name: 'DOMAIN_SEPARATOR',
    type: 'function',
    stateMutability: 'view',
    inputs: [],
    outputs: [{ name: '', type: 'bytes32' }],
  },
  {
    name: 'nonces',
    type: 'function',
    stateMutability: 'view',
    inputs: [{ name: 'owner', type: 'address' }],
    outputs: [{ name: '', type: 'uint256' }],
  },
] as const
```

---

## 3. Custom Hooks

### 3.1 useVault Hook

```typescript
// src/hooks/useVault.ts
import { useReadContract, useReadContracts, useWriteContract,
         useWaitForTransactionReceipt, useChainId, useAccount } from 'wagmi'
import { formatUnits, parseUnits } from 'viem'
import { CONTRACTS, VAULT_ABI, ERC20_ABI } from '../config/contracts'

const USDC_DECIMALS = 6

export function useVaultInfo() {
  const chainId = useChainId()
  const contracts = CONTRACTS[chainId === 1 ? 'mainnet' : 'sepolia']

  const { data, isLoading, refetch } = useReadContracts({
    contracts: [
      {
        address: contracts.VAULT,
        abi: VAULT_ABI,
        functionName: 'totalAssets',
      },
      {
        address: contracts.VAULT,
        abi: VAULT_ABI,
        functionName: 'totalSupply',
      },
    ],
  })

  const totalAssets = data?.[0]?.result
  const totalSupply = data?.[1]?.result

  // คำนวณ share price
  const sharePrice = totalAssets && totalSupply && totalSupply > 0n
    ? (totalAssets * BigInt(1e18)) / totalSupply
    : BigInt(1e18)

  return {
    totalAssets: totalAssets ? formatUnits(totalAssets, USDC_DECIMALS) : '0',
    totalSupply: totalSupply ? formatUnits(totalSupply, 18) : '0',
    sharePriceFormatted: sharePrice
      ? parseFloat(formatUnits(sharePrice, 18)).toFixed(6)
      : '1.000000',
    isLoading,
    refetch,
  }
}

export function useUserPosition() {
  const { address } = useAccount()
  const chainId = useChainId()
  const contracts = CONTRACTS[chainId === 1 ? 'mainnet' : 'sepolia']

  const { data, isLoading } = useReadContracts({
    contracts: address ? [
      {
        address: contracts.VAULT,
        abi: VAULT_ABI,
        functionName: 'balanceOf',
        args: [address],
      },
      {
        address: contracts.VAULT,
        abi: VAULT_ABI,
        functionName: 'maxWithdraw',
        args: [address],
      },
      {
        address: contracts.USDC,
        abi: ERC20_ABI,
        functionName: 'balanceOf',
        args: [address],
      },
    ] : [],
    query: { enabled: !!address },
  })

  const shares = data?.[0]?.result ?? 0n
  const maxWithdraw = data?.[1]?.result ?? 0n
  const usdcBalance = data?.[2]?.result ?? 0n

  return {
    shares: formatUnits(shares, 18),
    maxWithdraw: formatUnits(maxWithdraw, USDC_DECIMALS),
    usdcBalance: formatUnits(usdcBalance, USDC_DECIMALS),
    isLoading,
  }
}

// ─── APY Calculation ──────────────────────────────────────────

export function useVaultAPY() {
  // ใน production จะคำนวณจาก historical data (The Graph)
  // สำหรับตอนนี้ใช้ mock data
  return {
    apy: '12.4',
    breakdown: [
      { strategy: 'Aave V3', apy: '5.2', allocation: '50%' },
      { strategy: 'Uniswap V3', apy: '18.6', allocation: '30%' },
      { strategy: 'Idle', apy: '0', allocation: '20%' },
    ],
    isLoading: false,
  }
}
```

### 3.2 usePermitAndDeposit Hook

```typescript
// src/hooks/usePermitAndDeposit.ts
import { useState } from 'react'
import { useAccount, useChainId, useWriteContract,
         useWaitForTransactionReceipt } from 'wagmi'
import { parseUnits, parseErc6492Signature, hashTypedData, type Address } from 'viem'
import { useWalletClient, usePublicClient } from 'wagmi'
import { CONTRACTS, VAULT_ABI, ERC20_ABI } from '../config/contracts'

export function usePermitAndDeposit() {
  const { address } = useAccount()
  const chainId = useChainId()
  const contracts = CONTRACTS[chainId === 1 ? 'mainnet' : 'sepolia']

  const { data: walletClient } = useWalletClient()
  const publicClient = usePublicClient()

  const { writeContract, data: txHash, isPending } = useWriteContract()
  const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({
    hash: txHash,
  })

  const [status, setStatus] = useState<string>('')

  const permitAndDeposit = async (amountStr: string) => {
    if (!address || !walletClient || !publicClient) {
      throw new Error('Wallet not connected')
    }

    const amount = parseUnits(amountStr, 6) // USDC has 6 decimals
    const deadline = BigInt(Math.floor(Date.now() / 1000) + 3600) // 1 hour

    setStatus('Preparing permit signature...')

    // ─── Step 1: Sign Permit ──────────────────────────────────

    // Get domain separator and nonce
    const [domainSeparator, nonce] = await Promise.all([
      publicClient.readContract({
        address: contracts.USDC,
        abi: ERC20_ABI,
        functionName: 'DOMAIN_SEPARATOR',
      }),
      publicClient.readContract({
        address: contracts.USDC,
        abi: ERC20_ABI,
        functionName: 'nonces',
        args: [address],
      }),
    ])

    // EIP-712 typed data สำหรับ permit
    const typedData = {
      domain: {
        name: 'USD Coin', // ต้องตรงกับ token name
        version: '2',     // ต้องตรงกับ contract version
        chainId: chainId,
        verifyingContract: contracts.USDC,
      },
      types: {
        Permit: [
          { name: 'owner', type: 'address' },
          { name: 'spender', type: 'address' },
          { name: 'value', type: 'uint256' },
          { name: 'nonce', type: 'uint256' },
          { name: 'deadline', type: 'uint256' },
        ],
      },
      primaryType: 'Permit' as const,
      message: {
        owner: address,
        spender: contracts.VAULT,
        value: amount,
        nonce: nonce,
        deadline: deadline,
      },
    }

    setStatus('Please sign the permit in your wallet...')

    // Request signature
    const signature = await walletClient.signTypedData(typedData)

    // Parse signature
    const { v, r, s } = parseErc6492Signature(signature)

    setStatus('Depositing...')

    // ─── Step 2: Permit + Deposit ─────────────────────────────

    // ใน production จะเรียก vault.depositWithPermit() ที่รับ permit params
    // แต่เนื่องจาก standard ERC-4626 ไม่มี ต้องทำ 2 calls หรือใช้ multicall

    // Option 1: แยก 2 transactions (permit ก่อน แล้ว deposit)
    // Option 2: ใช้ multicall3
    // Option 3: เพิ่ม depositWithPermit() function ใน vault

    // ตัวอย่าง: เรียก permit ก่อน
    writeContract({
      address: contracts.USDC,
      abi: ERC20_ABI,
      functionName: 'permit',
      args: [address, contracts.VAULT, amount, deadline, v, r, s],
    })

    setStatus('Transaction submitted, waiting for confirmation...')
  }

  return {
    permitAndDeposit,
    txHash,
    isPending,
    isConfirming,
    isSuccess,
    status,
  }
}
```

---

## 4. React Components

### 4.1 App.tsx (Root Component)

```typescript
// src/App.tsx
import { WagmiProvider } from 'wagmi'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { ApolloProvider } from '@apollo/client'
import { wagmiConfig } from './config/wagmi'
import { apolloClient } from './config/graph'
import { Header } from './components/shared/Header'
import { VaultPage } from './pages/VaultPage'
import { GovernancePage } from './pages/GovernancePage'
import { useState } from 'react'

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30_000, // 30 seconds
      refetchInterval: 60_000, // 1 minute
    },
  },
})

export default function App() {
  const [activePage, setActivePage] = useState<'vault' | 'governance'>('vault')

  return (
    <WagmiProvider config={wagmiConfig}>
      <QueryClientProvider client={queryClient}>
        <ApolloProvider client={apolloClient}>
          <div className="min-h-screen bg-gray-950 text-white">
            <Header activePage={activePage} onNavigate={setActivePage} />
            <main className="container mx-auto px-4 py-8 max-w-7xl">
              {activePage === 'vault' ? <VaultPage /> : <GovernancePage />}
            </main>
          </div>
        </ApolloProvider>
      </QueryClientProvider>
    </WagmiProvider>
  )
}
```

### 4.2 VaultCard Component

```typescript
// src/components/vault/VaultCard.tsx
import { useState } from 'react'
import { useAccount } from 'wagmi'
import { useVaultInfo, useUserPosition, useVaultAPY } from '../../hooks/useVault'
import { DepositModal } from './DepositModal'
import { WithdrawModal } from './WithdrawModal'
import { formatNumber } from '../../utils/formatters'

interface VaultCardProps {
  vaultAddress: string
  vaultName: string
  assetSymbol: string
}

export function VaultCard({ vaultAddress, vaultName, assetSymbol }: VaultCardProps) {
  const { isConnected } = useAccount()
  const { totalAssets, sharePriceFormatted, isLoading: vaultLoading } = useVaultInfo()
  const { shares, maxWithdraw, usdcBalance } = useUserPosition()
  const { apy, breakdown } = useVaultAPY()

  const [showDepositModal, setShowDepositModal] = useState(false)
  const [showWithdrawModal, setShowWithdrawModal] = useState(false)
  const [showBreakdown, setShowBreakdown] = useState(false)

  return (
    <>
      <div className="bg-gray-900 rounded-2xl border border-gray-800 overflow-hidden">
        {/* ─── Header ─────────────────────────────────────────────── */}
        <div className="p-6 border-b border-gray-800">
          <div className="flex items-center justify-between">
            <div className="flex items-center gap-3">
              {/* Vault Icon */}
              <div className="w-12 h-12 rounded-full bg-gradient-to-br from-blue-500 to-purple-600 flex items-center justify-center">
                <span className="text-white font-bold text-lg">Ω</span>
              </div>
              <div>
                <h3 className="text-lg font-bold text-white">{vaultName}</h3>
                <p className="text-gray-400 text-sm">ERC-4626 Yield Vault</p>
              </div>
            </div>

            {/* APY Badge */}
            <div className="text-right">
              <div className="text-3xl font-bold text-green-400">{apy}%</div>
              <div className="text-gray-400 text-sm">APY</div>
            </div>
          </div>
        </div>

        {/* ─── Stats Grid ─────────────────────────────────────────── */}
        <div className="grid grid-cols-3 divide-x divide-gray-800 border-b border-gray-800">
          <div className="p-4 text-center">
            <div className="text-gray-400 text-xs mb-1">TVL</div>
            <div className="font-semibold text-white">
              {vaultLoading ? (
                <span className="animate-pulse">Loading...</span>
              ) : (
                `$${formatNumber(totalAssets)}`
              )}
            </div>
          </div>

          <div className="p-4 text-center">
            <div className="text-gray-400 text-xs mb-1">Share Price</div>
            <div className="font-semibold text-white">${sharePriceFormatted}</div>
          </div>

          <div className="p-4 text-center">
            <div className="text-gray-400 text-xs mb-1">Strategies</div>
            <div className="font-semibold text-white">2 Active</div>
          </div>
        </div>

        {/* ─── Your Position ──────────────────────────────────────── */}
        {isConnected && (
          <div className="p-6 border-b border-gray-800 bg-gray-900/50">
            <h4 className="text-gray-400 text-sm font-medium mb-3">Your Position</h4>
            <div className="grid grid-cols-2 gap-4">
              <div>
                <div className="text-gray-500 text-xs">Shares</div>
                <div className="text-white font-semibold">
                  {parseFloat(shares).toFixed(4)} omUSDC
                </div>
              </div>
              <div>
                <div className="text-gray-500 text-xs">Value</div>
                <div className="text-white font-semibold">
                  ${formatNumber(maxWithdraw)}
                </div>
              </div>
            </div>
          </div>
        )}

        {/* ─── Strategy Breakdown (Toggle) ────────────────────────── */}
        <button
          className="w-full p-4 text-left hover:bg-gray-800/50 transition-colors"
          onClick={() => setShowBreakdown(!showBreakdown)}
        >
          <div className="flex items-center justify-between">
            <span className="text-gray-400 text-sm">Strategy Breakdown</span>
            <span className="text-gray-500">{showBreakdown ? '▲' : '▼'}</span>
          </div>
        </button>

        {showBreakdown && (
          <div className="px-6 pb-4 space-y-2">
            {breakdown.map((item) => (
              <div key={item.strategy} className="flex items-center justify-between">
                <div className="flex items-center gap-2">
                  <div className="w-2 h-2 rounded-full bg-blue-400"></div>
                  <span className="text-gray-300 text-sm">{item.strategy}</span>
                </div>
                <div className="flex items-center gap-4 text-sm">
                  <span className="text-gray-500">{item.allocation}</span>
                  <span className="text-green-400 font-medium">{item.apy}%</span>
                </div>
              </div>
            ))}
          </div>
        )}

        {/* ─── Actions ─────────────────────────────────────────────── */}
        <div className="p-6 flex gap-3">
          <button
            onClick={() => setShowDepositModal(true)}
            disabled={!isConnected}
            className="flex-1 bg-blue-600 hover:bg-blue-500 disabled:opacity-50 disabled:cursor-not-allowed text-white font-semibold py-3 px-4 rounded-xl transition-colors"
          >
            Deposit
          </button>
          <button
            onClick={() => setShowWithdrawModal(true)}
            disabled={!isConnected || parseFloat(shares) === 0}
            className="flex-1 bg-gray-700 hover:bg-gray-600 disabled:opacity-50 disabled:cursor-not-allowed text-white font-semibold py-3 px-4 rounded-xl transition-colors"
          >
            Withdraw
          </button>
        </div>
      </div>

      {/* ─── Modals ─────────────────────────────────────────────────── */}
      <DepositModal
        isOpen={showDepositModal}
        onClose={() => setShowDepositModal(false)}
        vaultName={vaultName}
        assetSymbol={assetSymbol}
        usdcBalance={usdcBalance}
        apy={apy}
      />

      <WithdrawModal
        isOpen={showWithdrawModal}
        onClose={() => setShowWithdrawModal(false)}
        shares={shares}
        maxWithdraw={maxWithdraw}
        assetSymbol={assetSymbol}
      />
    </>
  )
}
```

### 4.3 DepositModal Component

```typescript
// src/components/vault/DepositModal.tsx
import { Fragment, useState, useEffect } from 'react'
import { Dialog, Transition } from '@headlessui/react'
import { useAccount, useChainId } from 'wagmi'
import { parseUnits, formatUnits } from 'viem'
import { usePermitAndDeposit } from '../../hooks/usePermitAndDeposit'

interface DepositModalProps {
  isOpen: boolean
  onClose: () => void
  vaultName: string
  assetSymbol: string
  usdcBalance: string
  apy: string
}

export function DepositModal({
  isOpen,
  onClose,
  vaultName,
  assetSymbol,
  usdcBalance,
  apy,
}: DepositModalProps) {
  const [amount, setAmount] = useState('')
  const [usePermit, setUsePermit] = useState(true) // ค่าเริ่มต้น: ใช้ permit

  const { permitAndDeposit, isPending, isConfirming, isSuccess, status } = usePermitAndDeposit()

  // Reset เมื่อ modal ปิด
  useEffect(() => {
    if (!isOpen) {
      setAmount('')
    }
  }, [isOpen])

  // ปิด modal เมื่อ tx สำเร็จ
  useEffect(() => {
    if (isSuccess) {
      setTimeout(onClose, 2000)
    }
  }, [isSuccess, onClose])

  const handleDeposit = async () => {
    if (!amount || parseFloat(amount) <= 0) return

    try {
      await permitAndDeposit(amount)
    } catch (error) {
      console.error('Deposit failed:', error)
    }
  }

  const setMaxAmount = () => {
    setAmount(usdcBalance)
  }

  const estimatedShares = amount
    ? parseFloat(amount).toFixed(4) // simplified: 1:1 initially
    : '0'

  return (
    <Transition appear show={isOpen} as={Fragment}>
      <Dialog as="div" className="relative z-50" onClose={onClose}>
        {/* Backdrop */}
        <Transition.Child
          as={Fragment}
          enter="ease-out duration-300"
          enterFrom="opacity-0"
          enterTo="opacity-100"
          leave="ease-in duration-200"
          leaveFrom="opacity-100"
          leaveTo="opacity-0"
        >
          <div className="fixed inset-0 bg-black/70" />
        </Transition.Child>

        {/* Modal */}
        <div className="fixed inset-0 overflow-y-auto">
          <div className="flex min-h-full items-center justify-center p-4">
            <Transition.Child
              as={Fragment}
              enter="ease-out duration-300"
              enterFrom="opacity-0 scale-95"
              enterTo="opacity-100 scale-100"
              leave="ease-in duration-200"
              leaveFrom="opacity-100 scale-100"
              leaveTo="opacity-0 scale-95"
            >
              <Dialog.Panel className="w-full max-w-md bg-gray-900 rounded-2xl border border-gray-700 shadow-xl">
                {/* Header */}
                <div className="flex items-center justify-between p-6 border-b border-gray-800">
                  <Dialog.Title className="text-lg font-bold text-white">
                    Deposit to {vaultName}
                  </Dialog.Title>
                  <button
                    onClick={onClose}
                    className="text-gray-400 hover:text-white transition-colors"
                  >
                    ✕
                  </button>
                </div>

                {/* Content */}
                <div className="p-6 space-y-6">
                  {/* Amount Input */}
                  <div>
                    <div className="flex items-center justify-between mb-2">
                      <label className="text-gray-400 text-sm">Amount</label>
                      <span className="text-gray-500 text-sm">
                        Balance: {parseFloat(usdcBalance).toFixed(2)} {assetSymbol}
                      </span>
                    </div>

                    <div className="relative flex items-center bg-gray-800 rounded-xl border border-gray-700 focus-within:border-blue-500 transition-colors">
                      <input
                        type="number"
                        value={amount}
                        onChange={(e) => setAmount(e.target.value)}
                        placeholder="0.00"
                        className="flex-1 bg-transparent text-white text-xl font-semibold p-4 outline-none placeholder-gray-600"
                      />
                      <div className="flex items-center gap-2 pr-4">
                        <button
                          onClick={setMaxAmount}
                          className="text-blue-400 hover:text-blue-300 text-sm font-medium px-2 py-1 bg-blue-500/10 rounded-lg"
                        >
                          MAX
                        </button>
                        <span className="text-gray-300 font-medium">{assetSymbol}</span>
                      </div>
                    </div>
                  </div>

                  {/* Permit Toggle */}
                  <div className="flex items-center justify-between bg-gray-800 rounded-xl p-4">
                    <div>
                      <div className="text-white text-sm font-medium">Use Gasless Permit</div>
                      <div className="text-gray-500 text-xs mt-0.5">
                        Sign instead of approve (saves 1 transaction)
                      </div>
                    </div>
                    <button
                      onClick={() => setUsePermit(!usePermit)}
                      className={`relative w-12 h-6 rounded-full transition-colors ${
                        usePermit ? 'bg-blue-600' : 'bg-gray-600'
                      }`}
                    >
                      <div
                        className={`absolute top-1 w-4 h-4 bg-white rounded-full transition-transform ${
                          usePermit ? 'translate-x-7' : 'translate-x-1'
                        }`}
                      />
                    </button>
                  </div>

                  {/* Summary */}
                  {amount && parseFloat(amount) > 0 && (
                    <div className="bg-gray-800/50 rounded-xl p-4 space-y-2">
                      <div className="flex justify-between text-sm">
                        <span className="text-gray-400">You deposit</span>
                        <span className="text-white">{amount} {assetSymbol}</span>
                      </div>
                      <div className="flex justify-between text-sm">
                        <span className="text-gray-400">You receive</span>
                        <span className="text-white">~{estimatedShares} omUSDC</span>
                      </div>
                      <div className="flex justify-between text-sm">
                        <span className="text-gray-400">Expected APY</span>
                        <span className="text-green-400 font-medium">{apy}%</span>
                      </div>
                      <div className="border-t border-gray-700 pt-2 flex justify-between text-xs">
                        <span className="text-gray-500">Transactions needed</span>
                        <span className="text-gray-300">{usePermit ? '1 (permit + deposit)' : '2 (approve + deposit)'}</span>
                      </div>
                    </div>
                  )}

                  {/* Status Message */}
                  {status && (
                    <div className="text-blue-400 text-sm text-center animate-pulse">
                      {status}
                    </div>
                  )}

                  {isSuccess && (
                    <div className="text-green-400 text-sm text-center">
                      ✓ Deposit successful! Your shares are ready.
                    </div>
                  )}

                  {/* Action Button */}
                  <button
                    onClick={handleDeposit}
                    disabled={
                      !amount ||
                      parseFloat(amount) <= 0 ||
                      parseFloat(amount) > parseFloat(usdcBalance) ||
                      isPending ||
                      isConfirming
                    }
                    className="w-full bg-blue-600 hover:bg-blue-500 disabled:opacity-50 disabled:cursor-not-allowed text-white font-semibold py-4 rounded-xl transition-colors text-lg"
                  >
                    {isPending
                      ? 'Check Wallet...'
                      : isConfirming
                      ? 'Confirming...'
                      : isSuccess
                      ? 'Deposited!'
                      : `Deposit ${assetSymbol}`}
                  </button>
                </div>
              </Dialog.Panel>
            </Transition.Child>
          </div>
        </div>
      </Dialog>
    </Transition>
  )
}
```

---

## 5. Governance UI Components

### 5.1 ProposalList Component

```typescript
// src/components/governance/ProposalList.tsx
import { useQuery } from '@apollo/client'
import { GET_PROPOSALS } from '../../queries/governance.graphql'
import { ProposalCard } from './ProposalCard'

export type ProposalStatus = 'Active' | 'Succeeded' | 'Defeated' | 'Queued' | 'Executed' | 'Canceled'

export interface Proposal {
  id: string
  proposalId: string
  proposer: string
  description: string
  status: ProposalStatus
  forVotes: string
  againstVotes: string
  abstainVotes: string
  startBlock: string
  endBlock: string
  createdAt: string
}

export function ProposalList() {
  const { data, loading, error } = useQuery(GET_PROPOSALS, {
    variables: { first: 20 },
    pollInterval: 30000, // refresh ทุก 30 วินาที
  })

  if (loading) {
    return (
      <div className="space-y-4">
        {[...Array(3)].map((_, i) => (
          <div key={i} className="bg-gray-900 rounded-2xl h-40 animate-pulse border border-gray-800" />
        ))}
      </div>
    )
  }

  if (error) {
    return (
      <div className="bg-red-900/20 border border-red-800 rounded-2xl p-6 text-red-400">
        Failed to load proposals: {error.message}
      </div>
    )
  }

  const proposals: Proposal[] = data?.proposals ?? []

  if (proposals.length === 0) {
    return (
      <div className="text-center py-20 text-gray-500">
        <div className="text-6xl mb-4">🗳️</div>
        <p className="text-lg">No proposals yet</p>
        <p className="text-sm mt-2">Be the first to propose a change!</p>
      </div>
    )
  }

  return (
    <div className="space-y-4">
      {proposals.map((proposal) => (
        <ProposalCard key={proposal.id} proposal={proposal} />
      ))}
    </div>
  )
}
```

### 5.2 ProposalCard Component

```typescript
// src/components/governance/ProposalCard.tsx
import { useState } from 'react'
import { useAccount } from 'wagmi'
import { formatUnits } from 'viem'
import type { Proposal } from './ProposalList'
import { VoteModal } from './VoteModal'

interface ProposalCardProps {
  proposal: Proposal
}

const STATUS_COLORS = {
  Active: 'bg-green-500/20 text-green-400 border-green-800',
  Succeeded: 'bg-blue-500/20 text-blue-400 border-blue-800',
  Defeated: 'bg-red-500/20 text-red-400 border-red-800',
  Queued: 'bg-yellow-500/20 text-yellow-400 border-yellow-800',
  Executed: 'bg-gray-500/20 text-gray-400 border-gray-700',
  Canceled: 'bg-gray-500/20 text-gray-500 border-gray-700',
} as const

export function ProposalCard({ proposal }: ProposalCardProps) {
  const { isConnected } = useAccount()
  const [showVoteModal, setShowVoteModal] = useState(false)

  const forVotes = parseFloat(formatUnits(BigInt(proposal.forVotes), 18))
  const againstVotes = parseFloat(formatUnits(BigInt(proposal.againstVotes), 18))
  const totalVotes = forVotes + againstVotes

  const forPercentage = totalVotes > 0 ? (forVotes / totalVotes) * 100 : 0
  const againstPercentage = totalVotes > 0 ? (againstVotes / totalVotes) * 100 : 0

  const shortId = `#${proposal.proposalId.slice(0, 6)}`

  // Parse description: format "Title: body"
  const [title, ...bodyParts] = proposal.description.split(':')
  const body = bodyParts.join(':').trim()

  return (
    <>
      <div className="bg-gray-900 rounded-2xl border border-gray-800 overflow-hidden hover:border-gray-700 transition-colors">
        <div className="p-6">
          {/* ─── Header ──────────────────────────────────────────────── */}
          <div className="flex items-start justify-between gap-4 mb-4">
            <div className="flex-1">
              <div className="flex items-center gap-2 mb-1">
                <span className="text-gray-500 text-sm font-mono">{shortId}</span>
                <span
                  className={`text-xs font-medium px-2 py-0.5 rounded-full border ${STATUS_COLORS[proposal.status]}`}
                >
                  {proposal.status}
                </span>
              </div>
              <h3 className="text-white font-bold text-lg">{title}</h3>
              {body && (
                <p className="text-gray-400 text-sm mt-1 line-clamp-2">{body}</p>
              )}
            </div>
          </div>

          {/* ─── Vote Bar ─────────────────────────────────────────────── */}
          <div className="mb-4">
            <div className="flex justify-between text-xs text-gray-500 mb-1">
              <span>For: {forVotes.toLocaleString(undefined, { maximumFractionDigits: 0 })} votes</span>
              <span>Against: {againstVotes.toLocaleString(undefined, { maximumFractionDigits: 0 })} votes</span>
            </div>

            <div className="h-2 bg-gray-800 rounded-full overflow-hidden flex">
              <div
                className="bg-green-500 h-full transition-all duration-300"
                style={{ width: `${forPercentage}%` }}
              />
              <div
                className="bg-red-500 h-full transition-all duration-300"
                style={{ width: `${againstPercentage}%` }}
              />
            </div>

            <div className="flex justify-between text-xs mt-1">
              <span className="text-green-400">{forPercentage.toFixed(1)}%</span>
              <span className="text-red-400">{againstPercentage.toFixed(1)}%</span>
            </div>
          </div>

          {/* ─── Footer ─────────────────────────────────────────────── */}
          <div className="flex items-center justify-between">
            <div className="text-gray-500 text-xs">
              Proposed by {proposal.proposer.slice(0, 6)}...{proposal.proposer.slice(-4)}
            </div>

            {proposal.status === 'Active' && isConnected && (
              <button
                onClick={() => setShowVoteModal(true)}
                className="bg-blue-600 hover:bg-blue-500 text-white text-sm font-medium px-4 py-2 rounded-lg transition-colors"
              >
                Cast Vote
              </button>
            )}
          </div>
        </div>
      </div>

      <VoteModal
        isOpen={showVoteModal}
        onClose={() => setShowVoteModal(false)}
        proposal={proposal}
      />
    </>
  )
}
```

### 5.3 VoteModal Component

```typescript
// src/components/governance/VoteModal.tsx
import { Fragment, useState } from 'react'
import { Dialog, Transition } from '@headlessui/react'
import { useWriteContract, useWaitForTransactionReceipt } from 'wagmi'
import { CONTRACTS, GOVERNOR_ABI } from '../../config/contracts'
import { useChainId } from 'wagmi'
import type { Proposal } from './ProposalList'

interface VoteModalProps {
  isOpen: boolean
  onClose: () => void
  proposal: Proposal
}

const VOTE_OPTIONS = [
  { value: 1, label: 'For', emoji: '✅', color: 'border-green-600 bg-green-500/10 text-green-400' },
  { value: 0, label: 'Against', emoji: '❌', color: 'border-red-600 bg-red-500/10 text-red-400' },
  { value: 2, label: 'Abstain', emoji: '⚪', color: 'border-gray-600 bg-gray-500/10 text-gray-400' },
] as const

export function VoteModal({ isOpen, onClose, proposal }: VoteModalProps) {
  const chainId = useChainId()
  const contracts = CONTRACTS[chainId === 1 ? 'mainnet' : 'sepolia']

  const [selectedVote, setSelectedVote] = useState<number | null>(null)
  const [reason, setReason] = useState('')

  const { writeContract, data: txHash, isPending } = useWriteContract()
  const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({ hash: txHash })

  const handleVote = () => {
    if (selectedVote === null) return

    if (reason) {
      writeContract({
        address: contracts.GOVERNOR,
        abi: GOVERNOR_ABI,
        functionName: 'castVoteWithReason',
        args: [BigInt(proposal.proposalId), selectedVote, reason],
      })
    } else {
      writeContract({
        address: contracts.GOVERNOR,
        abi: GOVERNOR_ABI,
        functionName: 'castVote',
        args: [BigInt(proposal.proposalId), selectedVote],
      })
    }
  }

  return (
    <Transition appear show={isOpen} as={Fragment}>
      <Dialog as="div" className="relative z-50" onClose={onClose}>
        <Transition.Child
          as={Fragment}
          enter="ease-out duration-300"
          enterFrom="opacity-0"
          enterTo="opacity-100"
          leave="ease-in duration-200"
          leaveFrom="opacity-100"
          leaveTo="opacity-0"
        >
          <div className="fixed inset-0 bg-black/70" />
        </Transition.Child>

        <div className="fixed inset-0 overflow-y-auto">
          <div className="flex min-h-full items-center justify-center p-4">
            <Transition.Child
              as={Fragment}
              enter="ease-out duration-300"
              enterFrom="opacity-0 scale-95"
              enterTo="opacity-100 scale-100"
              leave="ease-in duration-200"
              leaveFrom="opacity-100 scale-100"
              leaveTo="opacity-0 scale-95"
            >
              <Dialog.Panel className="w-full max-w-md bg-gray-900 rounded-2xl border border-gray-700 shadow-xl">
                <div className="flex items-center justify-between p-6 border-b border-gray-800">
                  <Dialog.Title className="text-lg font-bold text-white">Cast Vote</Dialog.Title>
                  <button onClick={onClose} className="text-gray-400 hover:text-white">✕</button>
                </div>

                <div className="p-6 space-y-6">
                  {/* Proposal title */}
                  <div className="bg-gray-800 rounded-xl p-4">
                    <div className="text-gray-400 text-xs mb-1">Voting on</div>
                    <div className="text-white font-medium">{proposal.description.split(':')[0]}</div>
                  </div>

                  {/* Vote options */}
                  <div className="space-y-3">
                    {VOTE_OPTIONS.map((option) => (
                      <button
                        key={option.value}
                        onClick={() => setSelectedVote(option.value)}
                        className={`w-full flex items-center gap-3 p-4 rounded-xl border-2 transition-all ${
                          selectedVote === option.value
                            ? option.color
                            : 'border-gray-700 hover:border-gray-600 text-gray-400'
                        }`}
                      >
                        <span className="text-2xl">{option.emoji}</span>
                        <span className="font-semibold text-lg">{option.label}</span>
                        {selectedVote === option.value && (
                          <span className="ml-auto text-sm">✓ Selected</span>
                        )}
                      </button>
                    ))}
                  </div>

                  {/* Optional reason */}
                  <div>
                    <label className="text-gray-400 text-sm mb-2 block">
                      Reason (optional)
                    </label>
                    <textarea
                      value={reason}
                      onChange={(e) => setReason(e.target.value)}
                      placeholder="Explain your vote..."
                      className="w-full bg-gray-800 border border-gray-700 rounded-xl p-3 text-white placeholder-gray-600 text-sm resize-none outline-none focus:border-blue-500"
                      rows={3}
                    />
                  </div>

                  {isSuccess && (
                    <div className="text-green-400 text-center">✓ Vote submitted successfully!</div>
                  )}

                  <button
                    onClick={handleVote}
                    disabled={selectedVote === null || isPending || isConfirming}
                    className="w-full bg-blue-600 hover:bg-blue-500 disabled:opacity-50 disabled:cursor-not-allowed text-white font-semibold py-4 rounded-xl transition-colors"
                  >
                    {isPending
                      ? 'Confirm in Wallet...'
                      : isConfirming
                      ? 'Submitting...'
                      : isSuccess
                      ? 'Voted!'
                      : 'Submit Vote'}
                  </button>
                </div>
              </Dialog.Panel>
            </Transition.Child>
          </div>
        </div>
      </Dialog>
    </Transition>
  )
}
```

---

## 6. The Graph Subgraph

### 6.1 schema.graphql

```graphql
# subgraph/schema.graphql

type Vault @entity {
  id: ID!
  address: Bytes!
  asset: Bytes!
  totalAssets: BigInt!
  totalSupply: BigInt!
  strategies: [Strategy!]! @derivedFrom(field: "vault")
  deposits: [Deposit!]! @derivedFrom(field: "vault")
  withdrawals: [Withdrawal!]! @derivedFrom(field: "vault")
  harvests: [Harvest!]! @derivedFrom(field: "vault")
  createdAt: BigInt!
  updatedAt: BigInt!
}

type Strategy @entity {
  id: ID!
  address: Bytes!
  vault: Vault!
  name: String!
  allocation: BigInt!
  totalDebt: BigInt!
  totalGain: BigInt!
  totalLoss: BigInt!
  active: Boolean!
  addedAt: BigInt!
}

type Deposit @entity {
  id: ID!
  vault: Vault!
  sender: Bytes!
  owner: Bytes!
  assets: BigInt!
  shares: BigInt!
  timestamp: BigInt!
  txHash: Bytes!
}

type Withdrawal @entity {
  id: ID!
  vault: Vault!
  sender: Bytes!
  receiver: Bytes!
  owner: Bytes!
  assets: BigInt!
  shares: BigInt!
  timestamp: BigInt!
  txHash: Bytes!
}

type Harvest @entity {
  id: ID!
  vault: Vault!
  strategy: Strategy!
  gain: BigInt!
  loss: BigInt!
  timestamp: BigInt!
  txHash: Bytes!
}

type Proposal @entity {
  id: ID!
  proposalId: BigInt!
  proposer: Bytes!
  description: String!
  status: String!
  forVotes: BigInt!
  againstVotes: BigInt!
  abstainVotes: BigInt!
  startBlock: BigInt!
  endBlock: BigInt!
  executedAt: BigInt
  createdAt: BigInt!
}

type Vote @entity {
  id: ID!
  proposal: Proposal!
  voter: Bytes!
  support: Int!
  weight: BigInt!
  reason: String
  timestamp: BigInt!
}

type VaultDailySnapshot @entity {
  id: ID! # vault-address-date
  vault: Vault!
  totalAssets: BigInt!
  totalSupply: BigInt!
  apy: BigDecimal!
  timestamp: BigInt!
}
```

### 6.2 subgraph.yaml

```yaml
specVersion: 0.0.5
schema:
  file: ./schema.graphql

dataSources:
  - kind: ethereum
    name: OmniYieldVault
    network: mainnet
    source:
      address: "0x..." # Vault address
      abi: OmniYieldVault
      startBlock: 19000000
    mapping:
      kind: ethereum/events
      apiVersion: 0.0.7
      language: wasm/assemblyscript
      entities:
        - Vault
        - Strategy
        - Deposit
        - Withdrawal
        - Harvest
      abis:
        - name: OmniYieldVault
          file: ./abis/OmniYieldVault.json
      eventHandlers:
        - event: Deposit(indexed address,indexed address,uint256,uint256)
          handler: handleDeposit
        - event: Withdraw(indexed address,indexed address,indexed address,uint256,uint256)
          handler: handleWithdraw
        - event: StrategyAdded(indexed address,uint256)
          handler: handleStrategyAdded
        - event: StrategyRemoved(indexed address,uint256)
          handler: handleStrategyRemoved
        - event: Harvested(indexed address,uint256,uint256)
          handler: handleHarvested
      file: ./src/vault.ts

  - kind: ethereum
    name: OmniGovernor
    network: mainnet
    source:
      address: "0x..." # Governor address
      abi: OmniGovernor
      startBlock: 19000000
    mapping:
      kind: ethereum/events
      apiVersion: 0.0.7
      language: wasm/assemblyscript
      entities:
        - Proposal
        - Vote
      abis:
        - name: OmniGovernor
          file: ./abis/OmniGovernor.json
      eventHandlers:
        - event: ProposalCreated(uint256,address,address[],uint256[],string[],bytes[],uint256,uint256,string)
          handler: handleProposalCreated
        - event: VoteCast(indexed address,uint256,uint8,uint256,string)
          handler: handleVoteCast
        - event: ProposalExecuted(uint256)
          handler: handleProposalExecuted
        - event: ProposalCanceled(uint256)
          handler: handleProposalCanceled
      file: ./src/governance.ts
```

### 6.3 Mapping Functions

```typescript
// subgraph/src/vault.ts
import {
  Deposit as DepositEvent,
  Withdraw as WithdrawEvent,
  StrategyAdded,
  Harvested,
} from '../generated/OmniYieldVault/OmniYieldVault'
import { Vault, Deposit, Withdrawal, Strategy, Harvest } from '../generated/schema'
import { BigInt, Address } from '@graphprotocol/graph-ts'

// Helper: load or create Vault entity
function getOrCreateVault(address: Address): Vault {
  let vault = Vault.load(address.toHexString())
  if (!vault) {
    vault = new Vault(address.toHexString())
    vault.address = address
    vault.asset = Address.zero()
    vault.totalAssets = BigInt.zero()
    vault.totalSupply = BigInt.zero()
    vault.createdAt = BigInt.zero()
    vault.updatedAt = BigInt.zero()
  }
  return vault
}

export function handleDeposit(event: DepositEvent): void {
  let vault = getOrCreateVault(event.address)

  // สร้าง Deposit entity
  let depositId = event.transaction.hash.toHexString() + '-' + event.logIndex.toString()
  let deposit = new Deposit(depositId)
  deposit.vault = vault.id
  deposit.sender = event.params.sender
  deposit.owner = event.params.owner
  deposit.assets = event.params.assets
  deposit.shares = event.params.shares
  deposit.timestamp = event.block.timestamp
  deposit.txHash = event.transaction.hash
  deposit.save()

  // Update vault totals
  vault.totalAssets = vault.totalAssets.plus(event.params.assets)
  vault.totalSupply = vault.totalSupply.plus(event.params.shares)
  vault.updatedAt = event.block.timestamp
  vault.save()
}

export function handleWithdraw(event: WithdrawEvent): void {
  let vault = getOrCreateVault(event.address)

  let withdrawalId = event.transaction.hash.toHexString() + '-' + event.logIndex.toString()
  let withdrawal = new Withdrawal(withdrawalId)
  withdrawal.vault = vault.id
  withdrawal.sender = event.params.sender
  withdrawal.receiver = event.params.receiver
  withdrawal.owner = event.params.owner
  withdrawal.assets = event.params.assets
  withdrawal.shares = event.params.shares
  withdrawal.timestamp = event.block.timestamp
  withdrawal.txHash = event.transaction.hash
  withdrawal.save()

  // Update vault totals
  vault.totalAssets = vault.totalAssets.minus(event.params.assets)
  vault.totalSupply = vault.totalSupply.minus(event.params.shares)
  vault.updatedAt = event.block.timestamp
  vault.save()
}

export function handleStrategyAdded(event: StrategyAdded): void {
  let vault = getOrCreateVault(event.address)

  let strategyId = event.params.strategy.toHexString()
  let strategy = Strategy.load(strategyId)
  if (!strategy) {
    strategy = new Strategy(strategyId)
    strategy.vault = vault.id
    strategy.address = event.params.strategy
    strategy.name = 'Unknown Strategy'
    strategy.totalGain = BigInt.zero()
    strategy.totalLoss = BigInt.zero()
    strategy.addedAt = event.block.timestamp
  }

  strategy.allocation = event.params.allocation
  strategy.totalDebt = BigInt.zero()
  strategy.active = true
  strategy.save()
}

export function handleHarvested(event: Harvested): void {
  let vault = getOrCreateVault(event.address)

  let harvestId = event.transaction.hash.toHexString() + '-' + event.logIndex.toString()
  let harvest = new Harvest(harvestId)
  harvest.vault = vault.id
  harvest.strategy = event.params.strategy.toHexString()
  harvest.gain = event.params.gain
  harvest.loss = event.params.loss
  harvest.timestamp = event.block.timestamp
  harvest.txHash = event.transaction.hash
  harvest.save()
}
```

### 6.4 GraphQL Queries

```typescript
// src/queries/governance.graphql

import { gql } from '@apollo/client'

export const GET_PROPOSALS = gql`
  query GetProposals($first: Int!, $skip: Int) {
    proposals(
      first: $first
      skip: $skip
      orderBy: createdAt
      orderDirection: desc
    ) {
      id
      proposalId
      proposer
      description
      status
      forVotes
      againstVotes
      abstainVotes
      startBlock
      endBlock
      createdAt
    }
  }
`

export const GET_PROPOSAL_DETAIL = gql`
  query GetProposalDetail($proposalId: String!) {
    proposal(id: $proposalId) {
      id
      proposalId
      proposer
      description
      status
      forVotes
      againstVotes
      abstainVotes
      startBlock
      endBlock
      executedAt
      createdAt
      votes(first: 100, orderBy: weight, orderDirection: desc) {
        voter
        support
        weight
        reason
        timestamp
      }
    }
  }
`

export const GET_VAULT_STATS = gql`
  query GetVaultStats($vaultAddress: String!) {
    vault(id: $vaultAddress) {
      totalAssets
      totalSupply
      deposits(first: 10, orderBy: timestamp, orderDirection: desc) {
        assets
        shares
        owner
        timestamp
      }
      harvests(first: 10, orderBy: timestamp, orderDirection: desc) {
        gain
        loss
        strategy {
          name
        }
        timestamp
      }
    }
  }
`

export const GET_USER_DEPOSITS = gql`
  query GetUserDeposits($userAddress: Bytes!) {
    deposits(
      where: { owner: $userAddress }
      orderBy: timestamp
      orderDirection: desc
    ) {
      assets
      shares
      timestamp
      txHash
      vault {
        id
      }
    }
    withdrawals(
      where: { owner: $userAddress }
      orderBy: timestamp
      orderDirection: desc
    ) {
      assets
      shares
      timestamp
      txHash
    }
  }
`
```

---

## 7. Formatter Utilities

```typescript
// src/utils/formatters.ts

export function formatNumber(value: string | number, decimals = 2): string {
  const num = typeof value === 'string' ? parseFloat(value) : value
  if (isNaN(num)) return '0'

  if (num >= 1_000_000_000) {
    return (num / 1_000_000_000).toFixed(decimals) + 'B'
  }
  if (num >= 1_000_000) {
    return (num / 1_000_000).toFixed(decimals) + 'M'
  }
  if (num >= 1_000) {
    return (num / 1_000).toFixed(decimals) + 'K'
  }
  return num.toFixed(decimals)
}

export function formatAddress(address: string): string {
  if (!address || address.length < 10) return address
  return `${address.slice(0, 6)}...${address.slice(-4)}`
}

export function formatDate(timestamp: number | string): string {
  const ts = typeof timestamp === 'string' ? parseInt(timestamp) : timestamp
  return new Intl.DateTimeFormat('th-TH', {
    year: 'numeric',
    month: 'short',
    day: 'numeric',
    hour: '2-digit',
    minute: '2-digit',
  }).format(new Date(ts * 1000))
}

export function formatPercent(value: number, decimals = 2): string {
  return `${value.toFixed(decimals)}%`
}

export function formatTokenAmount(
  bigintValue: bigint,
  decimals: number,
  displayDecimals = 4
): string {
  const value = parseFloat(
    (bigintValue / BigInt(10 ** decimals)).toString()
  )
  return formatNumber(value, displayDecimals)
}
```

---

## Workshop

### Workshop 10.1: เพิ่ม DelegatePanel Component

**โจทย์**: Implement `DelegatePanel` component ที่:
1. แสดง current delegate ของ user
2. ให้ input เพื่อ set delegate ใหม่
3. แสดง voting power ที่ได้รับ delegate มา

```typescript
// TODO: Implement DelegatePanel
export function DelegatePanel() {
  // 1. useReadContract: veOYT.delegates(userAddress)
  // 2. useWriteContract: veOYT.delegate(newDelegate)
  // 3. แสดง current delegate + voting power
  // 4. Input สำหรับ delegate ใหม่

  return (
    <div>
      {/* Implement here */}
    </div>
  )
}
```

### Workshop 10.2: The Graph APY Calculation

**โจทย์**: คำนวณ APY จาก historical harvest data ใน subgraph

```typescript
// เพิ่มใน schema.graphql และ mapping เพื่อ track:
// - daily yields
// - 7-day rolling APY
// - 30-day rolling APY

// Query:
const GET_APY_DATA = gql`
  query GetAPYData($vault: String!, $since: BigInt!) {
    harvests(
      where: { vault: $vault, timestamp_gte: $since }
      orderBy: timestamp
    ) {
      gain
      timestamp
      strategy { name }
    }
  }
`

// TODO: implement calculateAPY(harvests, totalAssets) function
```

---

## สรุป Part 95

- **React + Wagmi V2** ให้ type-safe Ethereum interactions พร้อม auto-reconnect
- **VaultCard** แสดง APY, TVL, user position พร้อม strategy breakdown
- **DepositModal** รองรับ EIP-2612 permit สำหรับ gasless approval (1 transaction แทน 2)
- **Governance UI** รองรับ list proposals, cast vote พร้อม reason, และ delegate
- **The Graph subgraph** index events ทั้งหมดเพื่อ query historical data อย่างมีประสิทธิภาพ
- **Custom hooks** (useVaultInfo, useUserPosition, usePermitAndDeposit) แยก business logic ออกจาก UI

## Next: Part 96 - Advanced Patterns & MEV Protection

ใน Part 96 เราจะเรียนรู้:
- Flashbots / private mempool integration
- Commit-reveal schemes
- TWAP oracles
- Advanced MEV protection patterns
