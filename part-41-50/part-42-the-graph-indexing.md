# Part 42: The Graph Protocol - On-Chain Indexing

## สารบัญ
1. The Graph Overview
2. Subgraph Schema
3. Mapping Handlers
4. GraphQL Queries
5. Workshop: DEX Subgraph

---

## 1. The Graph Overview

```
The Graph คืออะไร:
- Decentralized indexing protocol สำหรับ blockchain data
- Query on-chain data ด้วย GraphQL (แทนที่จะ scan blocks)
- Subgraph: ชุด rules สำหรับ index events

ทำไมต้องใช้:
- Ethereum node ไม่มี query layer
- การ scan events เองช้าและแพง
- The Graph ทำ indexing ให้ → query เร็วมาก

Flow:
1. เขียน Subgraph (schema + mappings)
2. Deploy ไปที่ The Graph
3. Indexer nodes ประมวลผล
4. Frontend query ผ่าน GraphQL endpoint

Subgraph Components:
- subgraph.yaml: manifest (contract, events)
- schema.graphql: data model
- src/mappings.ts: event handlers (AssemblyScript)
```

---

## 2. Subgraph Manifest

```yaml
# subgraph.yaml
specVersion: 0.0.5
schema:
  file: ./schema.graphql
dataSources:
  - kind: ethereum
    name: MyToken
    network: mainnet
    source:
      address: "0x1234567890123456789012345678901234567890"
      abi: MyToken
      startBlock: 18000000
    mapping:
      kind: ethereum/events
      apiVersion: 0.0.7
      language: wasm/assemblyscript
      entities:
        - Token
        - Transfer
        - Account
      abis:
        - name: MyToken
          file: ./abis/MyToken.json
      eventHandlers:
        - event: Transfer(indexed address,indexed address,uint256)
          handler: handleTransfer
        - event: Approval(indexed address,indexed address,uint256)
          handler: handleApproval
      callHandlers:
        - function: mint(address,uint256)
          handler: handleMint
      file: ./src/token.ts
```

---

## 3. Schema Design

```graphql
# schema.graphql

type Token @entity {
  id: ID!                    # contract address
  name: String!
  symbol: String!
  decimals: Int!
  totalSupply: BigDecimal!
  
  # Derived fields
  transfers: [Transfer!]! @derivedFrom(field: "token")
  holders: [Account!]! @derivedFrom(field: "tokens")
  
  # Aggregates
  transferCount: BigInt!
  holderCount: BigInt!
  volumeTotal: BigDecimal!
}

type Account @entity {
  id: ID!                    # user address
  
  # Token balances
  tokens: [AccountBalance!]! @derivedFrom(field: "account")
  
  # Activity
  transfersFrom: [Transfer!]! @derivedFrom(field: "from")
  transfersTo: [Transfer!]! @derivedFrom(field: "to")
  
  totalSent: BigDecimal!
  totalReceived: BigDecimal!
}

type AccountBalance @entity {
  id: ID!                    # account-token
  account: Account!
  token: Token!
  balance: BigDecimal!
  
  lastUpdated: BigInt!       # block number
}

type Transfer @entity {
  id: ID!                    # txHash-logIndex
  token: Token!
  from: Account!
  to: Account!
  amount: BigDecimal!
  
  blockNumber: BigInt!
  timestamp: BigInt!
  transactionHash: Bytes!
}

type DailyVolume @entity {
  id: ID!                    # date string YYYY-MM-DD
  token: Token!
  date: Int!                 # Unix timestamp (day start)
  volume: BigDecimal!
  transferCount: BigInt!
}
```

---

## 4. AssemblyScript Mappings

```typescript
// src/token.ts
import { BigDecimal, BigInt, Bytes } from "@graphprotocol/graph-ts";
import {
  Transfer as TransferEvent,
  Approval as ApprovalEvent,
  MyToken,
} from "../generated/MyToken/MyToken";
import {
  Token,
  Account,
  AccountBalance,
  Transfer,
  DailyVolume,
} from "../generated/schema";

const ZERO = BigDecimal.fromString("0");
const ONE = BigInt.fromI32(1);

function getOrCreateToken(address: string): Token {
  let token = Token.load(address);
  
  if (!token) {
    token = new Token(address);
    
    // Load from contract
    let contract = MyToken.bind(Bytes.fromHexString(address) as any);
    token.name = contract.name();
    token.symbol = contract.symbol();
    token.decimals = contract.decimals();
    token.totalSupply = ZERO;
    token.transferCount = BigInt.fromI32(0);
    token.holderCount = BigInt.fromI32(0);
    token.volumeTotal = ZERO;
    
    token.save();
  }
  
  return token;
}

function getOrCreateAccount(address: string): Account {
  let account = Account.load(address);
  
  if (!account) {
    account = new Account(address);
    account.totalSent = ZERO;
    account.totalReceived = ZERO;
    account.save();
  }
  
  return account;
}

function getOrCreateBalance(accountId: string, tokenId: string): AccountBalance {
  const id = accountId + "-" + tokenId;
  let balance = AccountBalance.load(id);
  
  if (!balance) {
    balance = new AccountBalance(id);
    balance.account = accountId;
    balance.token = tokenId;
    balance.balance = ZERO;
    balance.lastUpdated = BigInt.fromI32(0);
    balance.save();
  }
  
  return balance;
}

function exponentToBigDecimal(decimals: i32): BigDecimal {
  let bd = BigDecimal.fromString("1");
  for (let i = 0; i < decimals; i++) {
    bd = bd.times(BigDecimal.fromString("10"));
  }
  return bd;
}

export function handleTransfer(event: TransferEvent): void {
  const tokenId = event.address.toHexString();
  const fromId = event.params.from.toHexString();
  const toId = event.params.to.toHexString();
  
  const token = getOrCreateToken(tokenId);
  const from = getOrCreateAccount(fromId);
  const to = getOrCreateAccount(toId);
  
  const decimals = token.decimals;
  const amount = event.params.value
    .toBigDecimal()
    .div(exponentToBigDecimal(decimals));
  
  // Create Transfer entity
  const transferId = event.transaction.hash.toHexString() + "-" + event.logIndex.toString();
  const transfer = new Transfer(transferId);
  transfer.token = tokenId;
  transfer.from = fromId;
  transfer.to = toId;
  transfer.amount = amount;
  transfer.blockNumber = event.block.number;
  transfer.timestamp = event.block.timestamp;
  transfer.transactionHash = event.transaction.hash;
  transfer.save();
  
  // Update balances
  if (fromId != "0x0000000000000000000000000000000000000000") {
    const fromBalance = getOrCreateBalance(fromId, tokenId);
    fromBalance.balance = fromBalance.balance.minus(amount);
    fromBalance.lastUpdated = event.block.number;
    fromBalance.save();
    
    from.totalSent = from.totalSent.plus(amount);
    from.save();
  }
  
  const toBalance = getOrCreateBalance(toId, tokenId);
  const wasZero = toBalance.balance.equals(ZERO);
  toBalance.balance = toBalance.balance.plus(amount);
  toBalance.lastUpdated = event.block.number;
  toBalance.save();
  
  to.totalReceived = to.totalReceived.plus(amount);
  to.save();
  
  // Update token stats
  token.transferCount = token.transferCount.plus(ONE);
  token.volumeTotal = token.volumeTotal.plus(amount);
  
  // Update holder count
  if (wasZero && !toBalance.balance.equals(ZERO)) {
    token.holderCount = token.holderCount.plus(ONE);
  }
  
  // Track mint
  if (fromId == "0x0000000000000000000000000000000000000000") {
    token.totalSupply = token.totalSupply.plus(amount);
  }
  
  token.save();
  
  // Update daily volume
  const dayTimestamp = event.block.timestamp.toI32() / 86400 * 86400;
  const dailyId = tokenId + "-" + dayTimestamp.toString();
  
  let daily = DailyVolume.load(dailyId);
  if (!daily) {
    daily = new DailyVolume(dailyId);
    daily.token = tokenId;
    daily.date = dayTimestamp;
    daily.volume = ZERO;
    daily.transferCount = BigInt.fromI32(0);
  }
  
  daily.volume = daily.volume.plus(amount);
  daily.transferCount = daily.transferCount.plus(ONE);
  daily.save();
}
```

---

## 5. GraphQL Queries

```graphql
# Get token info
query GetToken($address: ID!) {
  token(id: $address) {
    name
    symbol
    totalSupply
    holderCount
    transferCount
    volumeTotal
  }
}

# Get top holders
query TopHolders($tokenId: String!, $first: Int!) {
  accountBalances(
    where: { token: $tokenId }
    orderBy: balance
    orderDirection: desc
    first: $first
  ) {
    account {
      id
    }
    balance
  }
}

# Get recent transfers
query RecentTransfers($tokenId: String!, $first: Int!) {
  transfers(
    where: { token: $tokenId }
    orderBy: timestamp
    orderDirection: desc
    first: $first
  ) {
    from { id }
    to { id }
    amount
    timestamp
    transactionHash
  }
}

# Daily volume chart data
query DailyVolume($tokenId: String!, $startDate: Int!) {
  dailyVolumes(
    where: { 
      token: $tokenId
      date_gte: $startDate
    }
    orderBy: date
    orderDirection: asc
  ) {
    date
    volume
    transferCount
  }
}
```

```typescript
// Frontend query with urql/graphql-request
import { createClient, gql } from "urql";

const client = createClient({
  url: `https://api.thegraph.com/subgraphs/name/myproject/mytoken`,
});

async function getTopHolders(tokenAddress: string) {
  const query = gql`
    query TopHolders($tokenId: String!) {
      accountBalances(
        where: { token: $tokenId }
        orderBy: balance
        orderDirection: desc
        first: 10
      ) {
        account { id }
        balance
      }
    }
  `;
  
  const result = await client.query(query, { tokenId: tokenAddress.toLowerCase() }).toPromise();
  return result.data?.accountBalances;
}
```

---

## สรุป Part 42

The Graph ที่เรียนรู้:
- ✅ Subgraph architecture
- ✅ GraphQL schema design
- ✅ AssemblyScript event handlers
- ✅ Entity relationships (@derivedFrom)
- ✅ GraphQL query patterns

## Next: Part 43 - Smart Contract Security Testing with Foundry
