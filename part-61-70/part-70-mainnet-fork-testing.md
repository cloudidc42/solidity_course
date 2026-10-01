# Part 70: Mainnet Fork Testing

## บทนำ

**Mainnet Fork Testing** คือเทคนิคที่ทรงพลังที่สุดสำหรับทดสอบ smart contracts ในสภาพแวดล้อมที่ใกล้เคียง production มากที่สุด โดยการ "fork" state ของ mainnet มาทดสอบ locally ทำให้คุณสามารถ:

- ทดสอบกับ liquidity จริงของ Uniswap, Aave, Curve
- จำลองพฤติกรรมของ whale addresses
- ทดสอบ contract upgrades กับ state จริง
- Debug transactions ที่ fail บน mainnet

ในบทนี้จะครอบคลุม:
1. **Hardhat** mainnet fork configuration
2. **Foundry** fork cheatcodes
3. ทดสอบกับ **Uniswap V3** จริง
4. จำลอง **Whale Attacks**
5. ทดสอบ **Contract Upgrades** กับ live state

---

## 1. Hardhat Mainnet Fork

### 1.1 Configuration

```javascript
// hardhat.config.ts
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
import "@nomicfoundation/hardhat-foundry";

const config: HardhatUserConfig = {
  solidity: {
    version: "0.8.24",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200,
      },
      viaIR: true,
    },
  },

  networks: {
    hardhat: {
      // ============================================================
      // Mainnet Fork Configuration
      // ============================================================
      forking: {
        // ใช้ Alchemy, Infura, หรือ Ankr
        url: process.env.MAINNET_RPC_URL || "",

        // Pin block number เพื่อให้ tests reproducible
        // ไม่ pin = ใช้ latest (tests อาจ flaky)
        blockNumber: 19_500_000,  // เลือก block ที่ต้องการ

        // เปิดหรือปิด forking (ปิดเมื่อ run local tests ที่ไม่ต้องการ fork)
        enabled: process.env.FORK_ENABLED === "true",
      },

      // Gas settings
      gasPrice: 20_000_000_000,  // 20 gwei
      gas: 30_000_000,

      // เพิ่ม accounts สำหรับ test
      accounts: {
        count: 20,
        accountsBalance: "10000000000000000000000",  // 10,000 ETH
      },

      // Mining settings
      mining: {
        auto: true,
        interval: 0,  // instant mining
      },
    },

    // Forked network แยกต่างหาก (ไม่ pin block)
    mainnet_fork_latest: {
      url: "http://localhost:8545",
      chainId: 1,
    },
  },
};

export default config;
```

### 1.2 Hardhat Fork Utilities

```typescript
// test/helpers/fork-helpers.ts
import { ethers, network } from "hardhat";
import type { Signer } from "ethers";

/**
 * ============================================================
 * Fork Helper Functions
 * ============================================================
 */

/**
 * Impersonate account บน forked network
 * ทำให้สามารถ sign transactions ในนามของ address นั้นได้
 *
 * @param address ที่อยู่ที่ต้องการ impersonate
 * @returns Signer ของ address นั้น
 */
export async function impersonateAccount(address: string): Promise<Signer> {
  await network.provider.request({
    method: "hardhat_impersonateAccount",
    params: [address],
  });

  return ethers.getSigner(address);
}

/**
 * หยุด impersonate account
 * @param address ที่อยู่ที่ต้องการ stop impersonate
 */
export async function stopImpersonating(address: string): Promise<void> {
  await network.provider.request({
    method: "hardhat_stopImpersonatingAccount",
    params: [address],
  });
}

/**
 * ตั้ง ETH balance ของ address ใดๆ
 * @param address ที่อยู่
 * @param ethAmount จำนวน ETH
 */
export async function setEthBalance(
  address: string,
  ethAmount: bigint
): Promise<void> {
  await network.provider.send("hardhat_setBalance", [
    address,
    "0x" + ethAmount.toString(16),
  ]);
}

/**
 * ตั้ง ERC-20 token balance โดย manipulate storage โดยตรง
 * @param tokenAddress ที่อยู่ token
 * @param userAddress ที่อยู่ user
 * @param amount จำนวน token
 * @param balanceSlot slot number ของ balances mapping (หาได้จาก storage layout)
 */
export async function setTokenBalance(
  tokenAddress: string,
  userAddress: string,
  amount: bigint,
  balanceSlot: number = 0
): Promise<void> {
  // คำนวณ storage slot สำหรับ mapping
  // keccak256(abi.encodePacked(key, slot))
  const storageSlot = ethers.solidityPackedKeccak256(
    ["uint256", "uint256"],
    [userAddress, balanceSlot]
  );

  await network.provider.send("hardhat_setStorageAt", [
    tokenAddress,
    storageSlot,
    ethers.AbiCoder.defaultAbiCoder().encode(["uint256"], [amount]),
  ]);
}

/**
 * Reset network state กลับไปเป็น fork point
 * ใช้ใน afterEach เพื่อ isolate tests
 */
export async function resetFork(blockNumber?: number): Promise<void> {
  await network.provider.request({
    method: "hardhat_reset",
    params: [
      {
        forking: {
          jsonRpcUrl: process.env.MAINNET_RPC_URL!,
          blockNumber: blockNumber,
        },
      },
    ],
  });
}

/**
 * Snapshot state ปัจจุบันเพื่อ restore ทีหลัง
 * @returns snapshot ID
 */
export async function takeSnapshot(): Promise<string> {
  return await network.provider.send("evm_snapshot", []);
}

/**
 * Restore state จาก snapshot
 * @param snapshotId ID จาก takeSnapshot
 */
export async function restoreSnapshot(snapshotId: string): Promise<void> {
  await network.provider.send("evm_revert", [snapshotId]);
}

/**
 * เลื่อนเวลาไปข้างหน้า
 * @param seconds จำนวนวินาที
 */
export async function advanceTime(seconds: number): Promise<void> {
  await network.provider.send("evm_increaseTime", [seconds]);
  await network.provider.send("evm_mine", []);
}

/**
 * Mine blocks
 * @param count จำนวน blocks
 */
export async function mineBlocks(count: number): Promise<void> {
  for (let i = 0; i < count; i++) {
    await network.provider.send("evm_mine", []);
  }
}
```

### 1.3 ทดสอบกับ Uniswap V3 (Hardhat)

```typescript
// test/fork/UniswapV3.fork.test.ts
import { expect } from "chai";
import { ethers, network } from "hardhat";
import type { Signer } from "ethers";
import {
  impersonateAccount,
  setEthBalance,
  takeSnapshot,
  restoreSnapshot,
} from "../helpers/fork-helpers";

// ============================================================
//                    CONTRACT ADDRESSES (Mainnet)
// ============================================================

const ADDRESSES = {
  UNISWAP_V3_ROUTER: "0xE592427A0AEce92De3Edee1F18E0157C05861564",
  UNISWAP_V3_QUOTER: "0xb27308f9F90D607463bb33eA1BeBb41C27CE5AB6",
  UNISWAP_V3_FACTORY: "0x1F98431c8aD98523631AE4a59f267346ea31F984",

  USDC: "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  WETH: "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2",
  DAI: "0x6B175474E89094C44Da98b954EedeAC495271d0F",

  // Whale addresses (large token holders)
  USDC_WHALE: "0x47ac0Fb4F2D84898e4D9E7b4DaB3C24507a6D503",  // Binance
  WETH_WHALE: "0x2F0b23f53734252Bda2277357e97e1517d6B042A",
  DAI_WHALE: "0x60FaAe176336dAb62e284Fe19B885a9142A33B7a",
} as const;

// ABIs (simplified)
const ERC20_ABI = [
  "function balanceOf(address) external view returns (uint256)",
  "function approve(address spender, uint256 amount) external returns (bool)",
  "function transfer(address to, uint256 amount) external returns (bool)",
  "function decimals() external view returns (uint8)",
  "function symbol() external view returns (string)",
];

const SWAP_ROUTER_ABI = [
  `function exactInputSingle(
    (address tokenIn, address tokenOut, uint24 fee, address recipient,
     uint256 deadline, uint256 amountIn, uint256 amountOutMinimum,
     uint160 sqrtPriceLimitX96) params
  ) external payable returns (uint256 amountOut)`,
  `function exactInput(
    (bytes path, address recipient, uint256 deadline,
     uint256 amountIn, uint256 amountOutMinimum) params
  ) external payable returns (uint256 amountOut)`,
];

const QUOTER_ABI = [
  `function quoteExactInputSingle(
    address tokenIn, address tokenOut, uint24 fee,
    uint256 amountIn, uint160 sqrtPriceLimitX96
  ) external returns (uint256 amountOut)`,
];

// ============================================================
//                         TEST SUITE
// ============================================================

describe("Uniswap V3 Fork Tests", function() {
  // เพิ่ม timeout เพราะ fork tests ช้ากว่าปกติ
  this.timeout(120_000);

  let snapshotId: string;
  let trader: Signer;
  let traderAddress: string;
  let usdcWhale: Signer;

  let usdc: ReturnType<typeof ethers.getContractAt> extends Promise<infer T> ? T : never;
  let weth: ReturnType<typeof ethers.getContractAt> extends Promise<infer T> ? T : never;
  let router: ReturnType<typeof ethers.getContractAt> extends Promise<infer T> ? T : never;
  let quoter: ReturnType<typeof ethers.getContractAt> extends Promise<infer T> ? T : never;

  before(async function() {
    // ตรวจว่า fork enabled
    if (!process.env.FORK_ENABLED) {
      this.skip();
      return;
    }

    [trader] = await ethers.getSigners();
    traderAddress = await trader.getAddress();

    // Setup contracts
    usdc = await ethers.getContractAt(ERC20_ABI, ADDRESSES.USDC);
    weth = await ethers.getContractAt(ERC20_ABI, ADDRESSES.WETH);
    router = await ethers.getContractAt(SWAP_ROUTER_ABI, ADDRESSES.UNISWAP_V3_ROUTER);
    quoter = await ethers.getContractAt(QUOTER_ABI, ADDRESSES.UNISWAP_V3_QUOTER);

    // ให้ ETH กับ USDC whale เพื่อจ่าย gas
    usdcWhale = await impersonateAccount(ADDRESSES.USDC_WHALE);
    await setEthBalance(ADDRESSES.USDC_WHALE, ethers.parseEther("10"));

    // ย้าย USDC จาก whale มาให้ trader
    const usdcAmount = ethers.parseUnits("100000", 6);  // 100,000 USDC
    await (usdc.connect(usdcWhale) as any).transfer(traderAddress, usdcAmount);
  });

  beforeEach(async function() {
    snapshotId = await takeSnapshot();
  });

  afterEach(async function() {
    await restoreSnapshot(snapshotId);
  });

  // ============================================================
  // TEST 1: ดึง Quote จาก Uniswap V3
  // ============================================================
  it("should get quote for USDC -> WETH swap", async function() {
    const amountIn = ethers.parseUnits("1000", 6);  // 1,000 USDC

    // ใช้ staticCall เพื่อดู quote (quoter ใช้ gas เพราะมี state changes)
    const amountOut = await (quoter as any).quoteExactInputSingle.staticCall(
      ADDRESSES.USDC,
      ADDRESSES.WETH,
      3000,  // 0.3% fee tier
      amountIn,
      0      // no price limit
    );

    console.log(`Quote: 1000 USDC = ${ethers.formatEther(amountOut)} WETH`);

    // ราคา ETH ควรอยู่ระหว่าง $500-$10000
    const ethPrice = Number(amountIn) / Number(amountOut) * 1e12; // adjust decimals
    expect(ethPrice).to.be.greaterThan(500);
    expect(ethPrice).to.be.lessThan(10000);
  });

  // ============================================================
  // TEST 2: Swap USDC -> WETH
  // ============================================================
  it("should successfully swap USDC for WETH", async function() {
    const amountIn = ethers.parseUnits("10000", 6);  // 10,000 USDC

    // Approve router
    await (usdc.connect(trader) as any).approve(ADDRESSES.UNISWAP_V3_ROUTER, amountIn);

    const wethBefore = await (weth as any).balanceOf(traderAddress);
    const usdcBefore = await (usdc as any).balanceOf(traderAddress);

    // Get quote first
    const expectedOut = await (quoter as any).quoteExactInputSingle.staticCall(
      ADDRESSES.USDC, ADDRESSES.WETH, 3000, amountIn, 0
    );

    // Swap ด้วย 1% slippage tolerance
    const minOut = expectedOut * 99n / 100n;

    const deadline = Math.floor(Date.now() / 1000) + 3600;

    const tx = await (router.connect(trader) as any).exactInputSingle({
      tokenIn: ADDRESSES.USDC,
      tokenOut: ADDRESSES.WETH,
      fee: 3000,
      recipient: traderAddress,
      deadline,
      amountIn,
      amountOutMinimum: minOut,
      sqrtPriceLimitX96: 0,
    });

    await tx.wait();

    const wethAfter = await (weth as any).balanceOf(traderAddress);
    const usdcAfter = await (usdc as any).balanceOf(traderAddress);

    // ตรวจสอบผลลัพธ์
    expect(wethAfter).to.be.greaterThan(wethBefore);
    expect(usdcAfter).to.equal(usdcBefore - amountIn);

    const wethReceived = wethAfter - wethBefore;
    console.log(`Swapped ${ethers.formatUnits(amountIn, 6)} USDC for ${ethers.formatEther(wethReceived)} WETH`);
  });

  // ============================================================
  // TEST 3: ทดสอบ Slippage Protection
  // ============================================================
  it("should revert when slippage is too high", async function() {
    const amountIn = ethers.parseUnits("1000000", 6);  // 1M USDC (huge swap)

    await (usdc.connect(trader) as any).approve(ADDRESSES.UNISWAP_V3_ROUTER, amountIn);

    const deadline = Math.floor(Date.now() / 1000) + 3600;

    // ตั้ง minOut สูงมาก (เป็นไปไม่ได้จะได้)
    const impossibleMinOut = ethers.parseEther("1000");  // ต้องการ 1000 ETH

    await expect(
      (router.connect(trader) as any).exactInputSingle({
        tokenIn: ADDRESSES.USDC,
        tokenOut: ADDRESSES.WETH,
        fee: 3000,
        recipient: traderAddress,
        deadline,
        amountIn,
        amountOutMinimum: impossibleMinOut,
        sqrtPriceLimitX96: 0,
      })
    ).to.be.revertedWith("Too little received");
  });
});
```

---

## 2. Foundry Fork Testing

### 2.1 Foundry Fork Cheatcodes

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test, console} from "forge-std/Test.sol";
import {StdCheats} from "forge-std/StdCheats.sol";

/**
 * @title ForkCheatcodesDemo
 * @notice การสาธิต Fork cheatcodes ใน Foundry
 *
 * Foundry Fork Cheatcodes:
 * - vm.createFork(url, blockNumber): สร้าง fork ใหม่
 * - vm.selectFork(forkId): เลือก active fork
 * - vm.activeFork(): ดู ID ของ fork ที่ active อยู่
 * - vm.makePersistent(addr): ทำให้ contract persistent ระหว่าง fork switches
 * - vm.revokePersistent(addr): ยกเลิก persistent
 * - vm.isPersistent(addr): ตรวจสอบว่า persistent ไหม
 * - vm.rollFork(blockNumber): เปลี่ยน block number ของ fork ปัจจุบัน
 * - vm.rollFork(forkId, blockNumber): เปลี่ยน block number ของ fork ที่ระบุ
 * - vm.deal(addr, amount): ตั้ง ETH balance
 * - vm.store(contract, slot, value): เขียน storage โดยตรง
 * - vm.load(contract, slot): อ่าน storage
 */
contract ForkCheatcodesDemo is Test {

    // Fork IDs
    uint256 public mainnetFork;
    uint256 public goerliFork;

    // Block numbers (pinned เพื่อ reproducibility)
    uint256 constant MAINNET_BLOCK = 19_500_000;
    uint256 constant GOERLI_BLOCK = 10_000_000;

    // ============================================================
    //                    SETUP
    // ============================================================

    function setUp() public {
        // สร้าง forks (ต้องตั้ง MAINNET_RPC_URL และ GOERLI_RPC_URL ใน .env)
        mainnetFork = vm.createFork(vm.envString("MAINNET_RPC_URL"), MAINNET_BLOCK);
        goerliFork = vm.createFork(vm.envString("GOERLI_RPC_URL"), GOERLI_BLOCK);
    }

    // ============================================================
    //                   BASIC FORK OPERATIONS
    // ============================================================

    /**
     * @notice Demo: สลับระหว่าง fork
     */
    function test_switchBetweenForks() public {
        // เลือก mainnet fork
        vm.selectFork(mainnetFork);
        assertEq(vm.activeFork(), mainnetFork);

        // ตรวจสอบ block number
        assertEq(block.number, MAINNET_BLOCK);
        assertEq(block.chainid, 1);  // mainnet chain ID

        // สลับไป Goerli
        vm.selectFork(goerliFork);
        assertEq(vm.activeFork(), goerliFork);
        assertEq(block.chainid, 5);  // Goerli chain ID
    }

    /**
     * @notice Demo: makePersistent - contract ที่สร้างใน fork นึง
     *         จะยังคงอยู่เมื่อสลับ fork
     */
    function test_persistentContracts() public {
        vm.selectFork(mainnetFork);

        // Deploy contract ใน mainnet fork
        MockContract mc = new MockContract();
        mc.setValue(42);

        // ทำให้ persistent
        vm.makePersistent(address(mc));
        assertTrue(vm.isPersistent(address(mc)));

        // สลับ fork - mc ยังคงอยู่
        vm.selectFork(goerliFork);

        // ยังเข้าถึง mc ได้
        assertEq(mc.getValue(), 42);
    }

    /**
     * @notice Demo: rollFork - เปลี่ยน block number
     */
    function test_rollFork() public {
        vm.selectFork(mainnetFork);
        assertEq(block.number, MAINNET_BLOCK);

        // เลื่อนไปที่ block อื่น
        vm.rollFork(MAINNET_BLOCK + 100);
        assertEq(block.number, MAINNET_BLOCK + 100);
    }
}

/**
 * @notice Helper contract สำหรับ demo
 */
contract MockContract {
    uint256 private _value;

    function setValue(uint256 v) external { _value = v; }
    function getValue() external view returns (uint256) { return _value; }
}
```

### 2.2 Deal และ Store Cheatcodes

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test, console} from "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

/**
 * @title DealAndStoreDemo
 * @notice Demo การใช้ deal() และ store() cheatcodes
 */
contract DealAndStoreDemo is Test {

    // Mainnet token addresses
    address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address constant USDT = 0xdAC17F958D2ee523a2206206994597C13D831ec7;

    uint256 mainnetFork;

    function setUp() public {
        mainnetFork = vm.createFork(
            vm.envOr("MAINNET_RPC_URL", string("https://eth.llamarpc.com")),
            19_500_000
        );
        vm.selectFork(mainnetFork);
    }

    // ============================================================
    //               deal() - ตั้ง Token Balance
    // ============================================================

    /**
     * @notice deal() สำหรับ native ETH
     */
    function test_dealEth() public {
        address richUser = makeAddr("richUser");

        // ให้ ETH 100 ether
        vm.deal(richUser, 100 ether);

        assertEq(richUser.balance, 100 ether);
    }

    /**
     * @notice deal() สำหรับ ERC-20 (Foundry จัดการ storage slot อัตโนมัติ)
     */
    function test_dealERC20_foundry() public {
        address user = makeAddr("user");
        uint256 amount = 1_000_000e6;  // 1M USDC

        // deal() ใน StdCheats จัดการ find balance slot อัตโนมัติ
        deal(USDC, user, amount);

        assertEq(IERC20(USDC).balanceOf(user), amount);
    }

    /**
     * @notice deal() สำหรับ WETH
     */
    function test_dealWETH() public {
        address user = makeAddr("user");
        uint256 amount = 100 ether;

        deal(WETH, user, amount);

        assertEq(IERC20(WETH).balanceOf(user), amount);
        console.log("User WETH balance:", IERC20(WETH).balanceOf(user));
    }

    // ============================================================
    //         vm.store() - เขียน Storage Slot โดยตรง
    // ============================================================

    /**
     * @notice vm.store() สำหรับ manipulate storage
     * ใช้เมื่อ deal() ไม่รู้จัก token หรือต้องการ bypass access control
     */
    function test_storeStorage_manual() public {
        address user = makeAddr("user");
        uint256 amount = 5_000_000e6;  // 5M USDC

        // USDC ใช้ EIP-1967 proxy + implementation
        // balances mapping อยู่ที่ slot 9 (ใน USDC implementation)
        // storage layout: mapping(address => uint256) balances at slot 9

        // คำนวณ slot: keccak256(abi.encode(user, 9))
        bytes32 slot = keccak256(abi.encode(user, uint256(9)));

        // เขียน balance
        vm.store(USDC, slot, bytes32(amount));

        // ตรวจสอบ
        assertEq(IERC20(USDC).balanceOf(user), amount);
    }

    /**
     * @notice อ่าน storage ก่อน manipulate
     */
    function test_readAndModifyStorage() public {
        address usdcProxy = USDC;

        // อ่าน implementation address (EIP-1967 slot)
        bytes32 implementationSlot = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
        bytes32 implAddr = vm.load(usdcProxy, implementationSlot);

        console.log("USDC implementation:", address(uint160(uint256(implAddr))));

        // ตรวจสอบว่าไม่ใช่ zero address
        assertNotEq(address(uint160(uint256(implAddr))), address(0));
    }

    // ============================================================
    //            impersonateAccount - ใช้ใน Foundry
    // ============================================================

    /**
     * @notice vm.prank() และ vm.startPrank() สำหรับ impersonate
     */
    function test_impersonateForOneCall() public {
        address richAccount = 0x47ac0Fb4F2D84898e4D9E7b4DaB3C24507a6D503;

        uint256 balanceBefore = IERC20(USDC).balanceOf(richAccount);
        console.log("Rich account USDC:", balanceBefore / 1e6);

        // ให้ richAccount มี ETH สำหรับ gas
        vm.deal(richAccount, 1 ether);

        address receiver = makeAddr("receiver");

        // Impersonate ชั่วคราว (เฉพาะ 1 call ถัดไป)
        vm.prank(richAccount);
        IERC20(USDC).transfer(receiver, 1_000e6);  // โอน 1,000 USDC

        assertEq(IERC20(USDC).balanceOf(receiver), 1_000e6);
    }

    /**
     * @notice vm.startPrank() สำหรับ impersonate หลาย calls
     */
    function test_impersonateMultipleCalls() public {
        address whale = 0x47ac0Fb4F2D84898e4D9E7b4DaB3C24507a6D503;
        vm.deal(whale, 1 ether);

        address receiver1 = makeAddr("receiver1");
        address receiver2 = makeAddr("receiver2");

        // เริ่ม impersonate
        vm.startPrank(whale);

        IERC20(USDC).transfer(receiver1, 1_000e6);
        IERC20(USDC).transfer(receiver2, 2_000e6);

        // หยุด impersonate
        vm.stopPrank();

        assertEq(IERC20(USDC).balanceOf(receiver1), 1_000e6);
        assertEq(IERC20(USDC).balanceOf(receiver2), 2_000e6);
    }
}
```

### 2.3 Foundry Fork: ทดสอบ Uniswap V3

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test, console} from "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

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

    function exactInputSingle(ExactInputSingleParams calldata params)
        external payable returns (uint256 amountOut);
}

interface IUniswapV3Pool {
    function slot0() external view returns (
        uint160 sqrtPriceX96,
        int24 tick,
        uint16 observationIndex,
        uint16 observationCardinality,
        uint16 observationCardinalityNext,
        uint8 feeProtocol,
        bool unlocked
    );

    function liquidity() external view returns (uint128);
    function token0() external view returns (address);
    function token1() external view returns (address);
    function fee() external view returns (uint24);
}

/**
 * @title UniswapV3ForkTest
 * @notice ทดสอบกับ Uniswap V3 บน mainnet fork ด้วย Foundry
 */
contract UniswapV3ForkTest is Test {

    // ============================================================
    //                    CONSTANTS
    // ============================================================

    address constant SWAP_ROUTER = 0xE592427A0AEce92De3Edee1F18E0157C05861564;
    address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;

    // USDC/WETH 0.3% pool
    address constant USDC_WETH_POOL = 0x8ad599c3A0ff1De082011EFDDc58f1908eb6e6D5;

    ISwapRouter public router;
    IUniswapV3Pool public pool;

    address public trader;

    uint256 mainnetFork;

    // ============================================================
    //                    SETUP
    // ============================================================

    function setUp() public {
        mainnetFork = vm.createFork(
            vm.envOr("MAINNET_RPC_URL", string("https://eth.llamarpc.com")),
            19_500_000
        );
        vm.selectFork(mainnetFork);

        router = ISwapRouter(SWAP_ROUTER);
        pool = IUniswapV3Pool(USDC_WETH_POOL);

        trader = makeAddr("trader");

        // ให้ trader มี token สำหรับ test
        deal(USDC, trader, 10_000_000e6);   // 10M USDC
        deal(WETH, trader, 10_000 ether);   // 10,000 WETH
        vm.deal(trader, 100 ether);          // ETH สำหรับ gas
    }

    // ============================================================
    //            TEST 1: ตรวจสอบ Pool State
    // ============================================================

    function test_poolState() public {
        (uint160 sqrtPriceX96, int24 tick, , , , , bool unlocked) = pool.slot0();
        uint128 liquidity = pool.liquidity();

        console.log("Pool liquidity:", liquidity);
        console.log("Current tick:", uint256(int256(tick)));
        console.log("Pool unlocked:", unlocked);

        assertTrue(unlocked, "Pool should be unlocked");
        assertGt(liquidity, 0, "Pool should have liquidity");
        assertGt(sqrtPriceX96, 0, "Price should be positive");
    }

    // ============================================================
    //            TEST 2: Swap USDC -> WETH
    // ============================================================

    function test_swapUSDCToWETH() public {
        uint256 amountIn = 100_000e6;  // 100,000 USDC

        uint256 wethBefore = IERC20(WETH).balanceOf(trader);

        vm.startPrank(trader);
        IERC20(USDC).approve(address(router), amountIn);

        uint256 amountOut = router.exactInputSingle(
            ISwapRouter.ExactInputSingleParams({
                tokenIn: USDC,
                tokenOut: WETH,
                fee: 3000,          // 0.3%
                recipient: trader,
                deadline: block.timestamp + 3600,
                amountIn: amountIn,
                amountOutMinimum: 0, // ไม่ใส่ slippage protection ใน test
                sqrtPriceLimitX96: 0
            })
        );
        vm.stopPrank();

        uint256 wethAfter = IERC20(WETH).balanceOf(trader);

        assertGt(amountOut, 0, "Should receive WETH");
        assertEq(wethAfter - wethBefore, amountOut, "Balance should increase by amountOut");

        // ราคา ETH ควรอยู่ในช่วงที่สมเหตุสมผล ($500-$10000)
        uint256 ethPriceInUsdc = (amountIn * 1e12) / amountOut;  // adjust decimals
        console.log("ETH price (USDC):", ethPriceInUsdc);
        assertGt(ethPriceInUsdc, 500e6, "ETH price too low");
        assertLt(ethPriceInUsdc, 10_000e6, "ETH price too high");
    }

    // ============================================================
    //         TEST 3: Multi-hop Swap USDC -> DAI (ผ่าน WETH)
    // ============================================================

    function test_multiHopSwap() public {
        uint256 amountIn = 10_000e6;  // 10,000 USDC

        // Path: USDC --0.3%--> WETH --0.05%--> DAI
        // encode: tokenIn, fee, tokenMiddle, fee, tokenOut
        bytes memory path = abi.encodePacked(
            USDC,
            uint24(3000),   // USDC/WETH 0.3%
            WETH,
            uint24(500),    // WETH/DAI 0.05%
            DAI
        );

        uint256 daiBefore = IERC20(DAI).balanceOf(trader);

        vm.startPrank(trader);
        IERC20(USDC).approve(address(router), amountIn);

        // ต้องใช้ interface อื่น (exactInput)
        // สำหรับ demo นี้ใช้ interface โดยตรง
        (bool success, bytes memory data) = address(router).call(
            abi.encodeWithSignature(
                "exactInput((bytes,address,uint256,uint256,uint256))",
                path,
                trader,
                block.timestamp + 3600,
                amountIn,
                0
            )
        );
        vm.stopPrank();

        assertTrue(success, "Multi-hop swap should succeed");

        uint256 daiReceived = IERC20(DAI).balanceOf(trader) - daiBefore;
        console.log("DAI received:", daiReceived / 1e18);

        // ควรได้ DAI ใกล้เคียง USDC (stable swap)
        assertGt(daiReceived, 9_000e18, "Should receive close to input amount");
        assertLt(daiReceived, 11_000e18, "Should not gain excessive DAI");
    }
}
```

---

## 3. Simulating Whale Attacks

### 3.1 Price Impact Attack Simulation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test, console} from "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

interface IUniswapV2Router {
    function getAmountsOut(uint256 amountIn, address[] calldata path)
        external view returns (uint256[] memory amounts);

    function swapExactTokensForTokens(
        uint256 amountIn,
        uint256 amountOutMin,
        address[] calldata path,
        address to,
        uint256 deadline
    ) external returns (uint256[] memory amounts);
}

interface IUniswapV2Pair {
    function getReserves() external view returns (
        uint112 reserve0,
        uint112 reserve1,
        uint32 blockTimestampLast
    );
    function token0() external view returns (address);
    function token1() external view returns (address);
}

/**
 * @title WhaleAttackSimulation
 * @notice จำลอง whale attack และทดสอบการป้องกัน
 *
 * Scenarios:
 * 1. Price manipulation ผ่าน large swap
 * 2. Front-running attack
 * 3. Sandwich attack
 */
contract WhaleAttackSimulation is Test {

    // Uniswap V2 (ง่ายกว่า V3 สำหรับ demo)
    address constant UNISWAP_V2_ROUTER = 0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D;
    address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

    // Known large holder (Binance cold wallet)
    address constant WHALE = 0x47ac0Fb4F2D84898e4D9E7b4DaB3C24507a6D503;

    IUniswapV2Router router;

    uint256 mainnetFork;

    function setUp() public {
        mainnetFork = vm.createFork(
            vm.envOr("MAINNET_RPC_URL", string("https://eth.llamarpc.com")),
            19_500_000
        );
        vm.selectFork(mainnetFork);

        router = IUniswapV2Router(UNISWAP_V2_ROUTER);

        // Setup whale
        vm.deal(WHALE, 100 ether);
    }

    // ============================================================
    //           TEST: Price Impact from Whale Swap
    // ============================================================

    /**
     * @notice วัด price impact เมื่อ whale ทำ large swap
     * ทดสอบว่า protocol มี safeguards ป้องกันได้ไหม
     */
    function test_whalePriceImpact() public {
        address[] memory path = new address[](2);
        path[0] = USDC;
        path[1] = WETH;

        // ==========================================
        // Step 1: วัดราคาปกติ
        // ==========================================
        uint256 normalSwapAmount = 1_000e6;  // 1,000 USDC
        uint256[] memory normalAmounts = router.getAmountsOut(normalSwapAmount, path);
        uint256 normalEthOut = normalAmounts[1];
        uint256 normalPrice = (normalSwapAmount * 1e12) / normalEthOut;

        console.log("Normal price (USDC/ETH):", normalPrice / 1e6);

        // ==========================================
        // Step 2: จำลอง whale swap ขนาดใหญ่
        // ==========================================
        uint256 whaleSwapAmount = 50_000_000e6;  // 50M USDC

        // ให้ whale มี USDC เยอะๆ
        deal(USDC, WHALE, whaleSwapAmount * 2);

        vm.startPrank(WHALE);
        IERC20(USDC).approve(address(router), whaleSwapAmount);

        uint256[] memory whaleAmountsPreview = router.getAmountsOut(whaleSwapAmount, path);
        uint256 expectedWhaleEth = whaleAmountsPreview[1];

        // คำนวณ price impact ก่อน execute
        uint256 priceImpact = 10000 - ((expectedWhaleEth * 10000) /
            (normalEthOut * (whaleSwapAmount / normalSwapAmount)));

        console.log("Expected price impact (bps):", priceImpact);

        // Execute whale swap
        router.swapExactTokensForTokens(
            whaleSwapAmount,
            expectedWhaleEth * 95 / 100,  // 5% slippage
            path,
            WHALE,
            block.timestamp + 3600
        );
        vm.stopPrank();

        // ==========================================
        // Step 3: วัดราคาหลังจาก whale swap
        // ==========================================
        uint256[] memory newAmounts = router.getAmountsOut(normalSwapAmount, path);
        uint256 newEthOut = newAmounts[1];
        uint256 newPrice = (normalSwapAmount * 1e12) / newEthOut;

        console.log("New price after whale swap (USDC/ETH):", newPrice / 1e6);
        console.log("Price change (bps):", ((newPrice - normalPrice) * 10000) / normalPrice);

        // ราคาควรเปลี่ยนไปหลังจาก large swap
        assertNotEq(normalPrice, newPrice, "Price should change after whale swap");
    }

    // ============================================================
    //           TEST: Sandwich Attack Simulation
    // ============================================================

    /**
     * @notice จำลอง sandwich attack
     *
     * Sandwich Attack Flow:
     * 1. Attacker เห็น victim's pending tx ใน mempool
     * 2. Attacker front-run: swap ก่อน victim (ราคาขึ้น)
     * 3. Victim's tx execute (ราคาสูงกว่าที่คาด)
     * 4. Attacker back-run: ขายออก (กำไรจาก price movement)
     */
    function test_sandwichAttack() public {
        address attacker = makeAddr("attacker");
        address victim = makeAddr("victim");

        // Setup
        deal(USDC, attacker, 10_000_000e6);  // 10M USDC สำหรับ front-run
        deal(USDC, victim, 100_000e6);       // 100K USDC สำหรับ victim

        vm.deal(attacker, 10 ether);
        vm.deal(victim, 1 ether);

        address[] memory pathBuy = new address[](2);
        pathBuy[0] = USDC;
        pathBuy[1] = WETH;

        address[] memory pathSell = new address[](2);
        pathSell[0] = WETH;
        pathSell[1] = USDC;

        // ==========================================
        // ราคาปกติก่อน attack
        // ==========================================
        uint256 victimAmount = 100_000e6;
        uint256[] memory normalOut = router.getAmountsOut(victimAmount, pathBuy);
        uint256 normalEthForVictim = normalOut[1];
        console.log("Normal ETH victim would get:", normalEthForVictim / 1e18);

        // ==========================================
        // Step 1: Front-run (attacker buys first)
        // ==========================================
        vm.startPrank(attacker);
        IERC20(USDC).approve(address(router), 10_000_000e6);

        uint256 frontRunAmount = 5_000_000e6;  // 5M USDC
        uint256[] memory frontRunOut = router.getAmountsOut(frontRunAmount, pathBuy);

        router.swapExactTokensForTokens(
            frontRunAmount,
            frontRunOut[1] * 95 / 100,
            pathBuy,
            attacker,
            block.timestamp + 3600
        );
        vm.stopPrank();

        uint256 attackerWethAfterFrontRun = IERC20(WETH).balanceOf(attacker);

        // ==========================================
        // Step 2: Victim's transaction (gets worse rate)
        // ==========================================
        vm.startPrank(victim);
        IERC20(USDC).approve(address(router), victimAmount);

        uint256[] memory manipulatedOut = router.getAmountsOut(victimAmount, pathBuy);
        uint256 manipulatedEthForVictim = manipulatedOut[1];

        router.swapExactTokensForTokens(
            victimAmount,
            manipulatedEthForVictim * 95 / 100,
            pathBuy,
            victim,
            block.timestamp + 3600
        );
        vm.stopPrank();

        // ==========================================
        // Step 3: Back-run (attacker sells at higher price)
        // ==========================================
        vm.startPrank(attacker);
        IERC20(WETH).approve(address(router), attackerWethAfterFrontRun);

        uint256[] memory backRunOut = router.getAmountsOut(attackerWethAfterFrontRun, pathSell);

        router.swapExactTokensForTokens(
            attackerWethAfterFrontRun,
            backRunOut[1] * 95 / 100,
            pathSell,
            attacker,
            block.timestamp + 3600
        );
        vm.stopPrank();

        // ==========================================
        // วิเคราะห์ผลลัพธ์
        // ==========================================
        uint256 attackerProfit = IERC20(USDC).balanceOf(attacker) -
            (10_000_000e6 - frontRunAmount);

        uint256 victimLoss = normalEthForVictim - IERC20(WETH).balanceOf(victim);

        console.log("Victim ETH loss (due to sandwich):", victimLoss / 1e18);
        console.log("Attacker USDC profit:", attackerProfit / 1e6);

        // Victim ได้ ETH น้อยกว่าที่ควรได้
        assertLt(
            IERC20(WETH).balanceOf(victim),
            normalEthForVictim,
            "Victim should get less ETH due to front-running"
        );
    }

    // ============================================================
    //     TEST: Defense - Slippage Protection ป้องกัน Sandwich
    // ============================================================

    /**
     * @notice ทดสอบว่า slippage protection ป้องกัน sandwich attack ได้
     */
    function test_slippageProtectionDefense() public {
        address victim = makeAddr("victim");
        address attacker = makeAddr("attacker");

        deal(USDC, victim, 100_000e6);
        deal(USDC, attacker, 10_000_000e6);
        vm.deal(victim, 1 ether);
        vm.deal(attacker, 10 ether);

        address[] memory pathBuy = new address[](2);
        pathBuy[0] = USDC;
        pathBuy[1] = WETH;

        // Victim กำหนด slippage protection เข้มงวด (0.1%)
        uint256 victimAmount = 100_000e6;
        uint256[] memory expectedOut = router.getAmountsOut(victimAmount, pathBuy);
        uint256 minOut = expectedOut[1] * 999 / 1000;  // 0.1% max slippage

        // Attacker front-runs
        vm.startPrank(attacker);
        IERC20(USDC).approve(address(router), 10_000_000e6);
        uint256[] memory frontRunOut = router.getAmountsOut(5_000_000e6, pathBuy);
        router.swapExactTokensForTokens(
            5_000_000e6,
            frontRunOut[1] * 95 / 100,
            pathBuy,
            attacker,
            block.timestamp + 3600
        );
        vm.stopPrank();

        // Victim's tx ควร revert เพราะ slippage เกิน
        vm.startPrank(victim);
        IERC20(USDC).approve(address(router), victimAmount);

        vm.expectRevert("UniswapV2Router: INSUFFICIENT_OUTPUT_AMOUNT");
        router.swapExactTokensForTokens(
            victimAmount,
            minOut,       // เข้มงวดมาก = ป้องกัน sandwich ได้
            pathBuy,
            victim,
            block.timestamp + 3600
        );
        vm.stopPrank();

        console.log("Slippage protection successfully prevented sandwich attack");
    }
}
```

---

## 4. Testing Contract Upgrades Against Live State

### 4.1 Upgradeable Contract Setup

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {UUPSUpgradeable} from "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import {OwnableUpgradeable} from "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
import {Initializable} from "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

/**
 * @title VaultV1
 * @notice Version 1 ของ Vault (deployed บน mainnet แล้ว)
 * @dev เก็บ USDC ให้ผู้ใช้ deposit/withdraw
 */
contract VaultV1 is Initializable, UUPSUpgradeable, OwnableUpgradeable {
    using SafeERC20 for IERC20;

    // Storage layout - ต้องไม่เปลี่ยนลำดับเมื่อ upgrade
    IERC20 public token;
    mapping(address => uint256) public balances;
    uint256 public totalDeposited;

    // Events
    event Deposited(address indexed user, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);

    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }

    function initialize(address _token, address _owner) external initializer {
        __Ownable_init(_owner);
        __UUPSUpgradeable_init();
        token = IERC20(_token);
    }

    function deposit(uint256 amount) external {
        token.safeTransferFrom(msg.sender, address(this), amount);
        balances[msg.sender] += amount;
        totalDeposited += amount;
        emit Deposited(msg.sender, amount);
    }

    function withdraw(uint256 amount) external {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        balances[msg.sender] -= amount;
        totalDeposited -= amount;
        token.safeTransfer(msg.sender, amount);
        emit Withdrawn(msg.sender, amount);
    }

    function _authorizeUpgrade(address) internal override onlyOwner {}
}

/**
 * @title VaultV2
 * @notice Version 2 - เพิ่ม fee mechanism และ emergency pause
 * @dev Storage ต้องต่อจาก V1 ห้ามเปลี่ยนลำดับ storage เดิม
 */
contract VaultV2 is VaultV1 {
    using SafeERC20 for IERC20;

    // ============================================================
    // NEW STORAGE (ต้องเพิ่มต่อจาก V1 เท่านั้น!)
    // ============================================================

    bool public paused;
    uint256 public withdrawalFee;  // basis points (100 = 1%)
    address public feeRecipient;
    uint256 public totalFeesCollected;

    // ============================================================
    // NEW EVENTS
    // ============================================================

    event FeeCollected(address indexed from, uint256 amount, address indexed recipient);
    event EmergencyPaused(address indexed by);
    event EmergencyUnpaused(address indexed by);
    event FeeUpdated(uint256 oldFee, uint256 newFee);

    // ============================================================
    // NEW ERRORS
    // ============================================================

    error ContractPaused();
    error FeeExceedsAmount(uint256 fee, uint256 amount);
    error InvalidFee(uint256 fee);

    // ============================================================
    // MODIFIERS
    // ============================================================

    modifier whenNotPaused() {
        if (paused) revert ContractPaused();
        _;
    }

    // ============================================================
    // INITIALIZER (สำหรับ V2 upgrade)
    // ============================================================

    /**
     * @notice Initialize V2 state (เรียกครั้งเดียวตอน upgrade)
     * @dev ใช้ reinitializer(2) เพื่อให้ call ได้หลังจาก V1 initialize แล้ว
     */
    function initializeV2(
        address _feeRecipient,
        uint256 _withdrawalFee
    ) external reinitializer(2) {
        if (_withdrawalFee > 1000) revert InvalidFee(_withdrawalFee);  // max 10%
        feeRecipient = _feeRecipient;
        withdrawalFee = _withdrawalFee;
        paused = false;
    }

    // ============================================================
    // UPGRADED FUNCTIONS
    // ============================================================

    /**
     * @notice Deposit (เหมือน V1 แต่เพิ่ม whenNotPaused)
     */
    function deposit(uint256 amount) external override whenNotPaused {
        super.deposit(amount);  // เรียก V1 logic
    }

    /**
     * @notice Withdraw พร้อมหัก fee
     */
    function withdraw(uint256 amount) external override whenNotPaused {
        require(balances[msg.sender] >= amount, "Insufficient balance");

        // คำนวณ fee
        uint256 fee = (amount * withdrawalFee) / 10000;
        uint256 netAmount = amount - fee;

        if (fee > amount) revert FeeExceedsAmount(fee, amount);

        // อัพเดท state
        balances[msg.sender] -= amount;
        totalDeposited -= amount;

        // Transfer
        if (fee > 0) {
            token.safeTransfer(feeRecipient, fee);
            totalFeesCollected += fee;
            emit FeeCollected(msg.sender, fee, feeRecipient);
        }

        token.safeTransfer(msg.sender, netAmount);
        emit Withdrawn(msg.sender, netAmount);
    }

    // ============================================================
    // ADMIN FUNCTIONS
    // ============================================================

    function pause() external onlyOwner {
        paused = true;
        emit EmergencyPaused(msg.sender);
    }

    function unpause() external onlyOwner {
        paused = false;
        emit EmergencyUnpaused(msg.sender);
    }

    function updateFee(uint256 newFee) external onlyOwner {
        if (newFee > 1000) revert InvalidFee(newFee);
        uint256 oldFee = withdrawalFee;
        withdrawalFee = newFee;
        emit FeeUpdated(oldFee, newFee);
    }
}
```

### 4.2 Testing Upgrade Against Live State (Foundry)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test, console} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {VaultV1} from "../src/VaultV1.sol";
import {VaultV2} from "../src/VaultV2.sol";

/**
 * @title VaultUpgradeTest
 * @notice ทดสอบ upgrade VaultV1 -> VaultV2 บน mainnet fork
 *
 * Test Strategy:
 * 1. Deploy VaultV1 ด้วย state ที่จำลองจาก mainnet
 * 2. สร้าง existing deposits
 * 3. Upgrade ไป VaultV2
 * 4. Verify state ยังคงอยู่ครบ
 * 5. Test V2 functionality ใหม่
 */
contract VaultUpgradeTest is Test {

    address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

    VaultV1 public proxy;
    VaultV2 public proxyV2;
    VaultV1 public implementationV1;
    VaultV2 public implementationV2;

    address public owner;
    address public user1;
    address public user2;
    address public feeRecipient;

    uint256 mainnetFork;

    // State สำหรับ regression testing
    uint256 user1DepositedAmount;
    uint256 user2DepositedAmount;
    uint256 totalDepositedBeforeUpgrade;

    function setUp() public {
        mainnetFork = vm.createFork(
            vm.envOr("MAINNET_RPC_URL", string("https://eth.llamarpc.com")),
            19_500_000
        );
        vm.selectFork(mainnetFork);

        owner = makeAddr("owner");
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");
        feeRecipient = makeAddr("feeRecipient");

        // Setup token
        deal(USDC, user1, 1_000_000e6);
        deal(USDC, user2, 500_000e6);

        // ============================================================
        // Deploy VaultV1 (จำลองว่า deploy มาก่อนแล้ว)
        // ============================================================
        implementationV1 = new VaultV1();

        bytes memory initData = abi.encodeWithSelector(
            VaultV1.initialize.selector,
            USDC,
            owner
        );

        ERC1967Proxy proxyContract = new ERC1967Proxy(
            address(implementationV1),
            initData
        );

        proxy = VaultV1(address(proxyContract));

        // ============================================================
        // สร้าง state บน V1 (จำลอง existing deposits)
        // ============================================================

        // User1 deposit
        vm.startPrank(user1);
        IERC20(USDC).approve(address(proxy), 500_000e6);
        proxy.deposit(500_000e6);
        vm.stopPrank();

        user1DepositedAmount = 500_000e6;

        // User2 deposit
        vm.startPrank(user2);
        IERC20(USDC).approve(address(proxy), 200_000e6);
        proxy.deposit(200_000e6);
        vm.stopPrank();

        user2DepositedAmount = 200_000e6;

        totalDepositedBeforeUpgrade = proxy.totalDeposited();

        console.log("V1 deployed and funded");
        console.log("Total deposited:", totalDepositedBeforeUpgrade / 1e6, "USDC");
    }

    // ============================================================
    //         TEST 1: Upgrade และ Verify State Preservation
    // ============================================================

    function test_upgradePreservesState() public {
        // ============================================================
        // UPGRADE: Deploy V2 และ upgrade proxy
        // ============================================================
        implementationV2 = new VaultV2();

        vm.startPrank(owner);

        // Upgrade ไป V2
        proxy.upgradeToAndCall(
            address(implementationV2),
            abi.encodeWithSelector(
                VaultV2.initializeV2.selector,
                feeRecipient,
                50  // 0.5% fee
            )
        );

        vm.stopPrank();

        // ============================================================
        // VERIFY: State ยังคงอยู่ครบ
        // ============================================================
        proxyV2 = VaultV2(address(proxy));

        // Balances ต้องยังคงอยู่
        assertEq(
            proxyV2.balances(user1),
            user1DepositedAmount,
            "User1 balance should be preserved"
        );
        assertEq(
            proxyV2.balances(user2),
            user2DepositedAmount,
            "User2 balance should be preserved"
        );

        // Total deposited ต้องยังคงอยู่
        assertEq(
            proxyV2.totalDeposited(),
            totalDepositedBeforeUpgrade,
            "Total deposited should be preserved"
        );

        // Token address ต้องยังคงอยู่
        assertEq(address(proxyV2.token()), USDC, "Token should be preserved");

        // New V2 state ถูกตั้งค่าแล้ว
        assertEq(proxyV2.withdrawalFee(), 50, "Withdrawal fee should be set");
        assertEq(proxyV2.feeRecipient(), feeRecipient, "Fee recipient should be set");
        assertEq(proxyV2.paused(), false, "Should not be paused initially");

        console.log("✅ State preserved after upgrade");
    }

    // ============================================================
    //         TEST 2: V2 Functionality หลัง Upgrade
    // ============================================================

    function test_v2FunctionalityAfterUpgrade() public {
        // Upgrade ก่อน
        implementationV2 = new VaultV2();
        vm.prank(owner);
        proxy.upgradeToAndCall(
            address(implementationV2),
            abi.encodeWithSelector(
                VaultV2.initializeV2.selector,
                feeRecipient,
                100  // 1% fee
            )
        );
        proxyV2 = VaultV2(address(proxy));

        // ============================================================
        // Test: Withdrawal ด้วย fee
        // ============================================================
        uint256 withdrawAmount = 100_000e6;  // 100,000 USDC
        uint256 expectedFee = (withdrawAmount * 100) / 10000;  // 1%
        uint256 expectedNet = withdrawAmount - expectedFee;

        uint256 user1BalanceBefore = IERC20(USDC).balanceOf(user1);
        uint256 feeRecipientBefore = IERC20(USDC).balanceOf(feeRecipient);

        vm.prank(user1);
        proxyV2.withdraw(withdrawAmount);

        uint256 user1BalanceAfter = IERC20(USDC).balanceOf(user1);
        uint256 feeRecipientAfter = IERC20(USDC).balanceOf(feeRecipient);

        assertEq(
            user1BalanceAfter - user1BalanceBefore,
            expectedNet,
            "User should receive amount minus fee"
        );
        assertEq(
            feeRecipientAfter - feeRecipientBefore,
            expectedFee,
            "Fee recipient should receive fee"
        );

        console.log("✅ V2 fee mechanism works correctly");
    }

    // ============================================================
    //         TEST 3: Emergency Pause ป้องกัน Operations
    // ============================================================

    function test_emergencyPauseAfterUpgrade() public {
        // Upgrade
        implementationV2 = new VaultV2();
        vm.prank(owner);
        proxy.upgradeToAndCall(
            address(implementationV2),
            abi.encodeWithSelector(
                VaultV2.initializeV2.selector,
                feeRecipient,
                50
            )
        );
        proxyV2 = VaultV2(address(proxy));

        // Pause contract
        vm.prank(owner);
        proxyV2.pause();
        assertTrue(proxyV2.paused(), "Should be paused");

        // Deposit ควร revert
        vm.startPrank(user1);
        IERC20(USDC).approve(address(proxyV2), 1000e6);
        vm.expectRevert(VaultV2.ContractPaused.selector);
        proxyV2.deposit(1000e6);
        vm.stopPrank();

        // Withdraw ควร revert
        vm.prank(user1);
        vm.expectRevert(VaultV2.ContractPaused.selector);
        proxyV2.withdraw(1000e6);

        // Unpause
        vm.prank(owner);
        proxyV2.unpause();
        assertFalse(proxyV2.paused(), "Should be unpaused");

        // ตอนนี้ withdraw ได้แล้ว
        vm.prank(user1);
        proxyV2.withdraw(1000e6);

        console.log("✅ Emergency pause works correctly");
    }

    // ============================================================
    //         TEST 4: Regression Test - V1 Users ยังคง withdraw ได้
    // ============================================================

    function test_regression_v1UsersCanWithdraw() public {
        // Upgrade
        implementationV2 = new VaultV2();
        vm.prank(owner);
        proxy.upgradeToAndCall(
            address(implementationV2),
            abi.encodeWithSelector(
                VaultV2.initializeV2.selector,
                feeRecipient,
                0  // ไม่มี fee สำหรับ regression test
            )
        );
        proxyV2 = VaultV2(address(proxy));

        // User1 ถอนทั้งหมด
        uint256 user1Balance = proxyV2.balances(user1);

        vm.prank(user1);
        proxyV2.withdraw(user1Balance);

        assertEq(proxyV2.balances(user1), 0, "User1 balance should be 0 after full withdrawal");
        assertEq(
            IERC20(USDC).balanceOf(user1),
            1_000_000e6,  // ได้คืนทั้งหมดบวกกับที่เหลืออยู่
            "User1 should have original USDC back"
        );

        // User2 ถอนบางส่วน
        uint256 user2Balance = proxyV2.balances(user2);
        uint256 partialWithdraw = user2Balance / 2;

        vm.prank(user2);
        proxyV2.withdraw(partialWithdraw);

        assertEq(
            proxyV2.balances(user2),
            user2Balance - partialWithdraw,
            "User2 remaining balance should be correct"
        );

        console.log("✅ V1 users can still withdraw after upgrade");
    }

    // ============================================================
    //         TEST 5: Storage Collision Check
    // ============================================================

    /**
     * @notice ตรวจสอบว่า V2 storage ไม่ collide กับ V1
     * @dev ใช้ vm.load เพื่ออ่าน raw storage
     */
    function test_noStorageCollision() public {
        // บันทึก V1 storage values ก่อน upgrade
        bytes32 tokenSlot = vm.load(address(proxy), bytes32(uint256(0)));  // slot 0 = _initialized, etc.
        bytes32 totalDepositedSlot_v1 = vm.load(address(proxy), bytes32(uint256(2)));  // slot 2

        // Upgrade
        implementationV2 = new VaultV2();
        vm.prank(owner);
        proxy.upgradeToAndCall(
            address(implementationV2),
            abi.encodeWithSelector(
                VaultV2.initializeV2.selector,
                feeRecipient,
                50
            )
        );

        // ตรวจสอบว่า V1 storage ไม่เปลี่ยน
        bytes32 totalDepositedSlot_v2 = vm.load(address(proxy), bytes32(uint256(2)));

        assertEq(
            totalDepositedSlot_v1,
            totalDepositedSlot_v2,
            "totalDeposited slot should not change after upgrade"
        );

        console.log("✅ No storage collision detected");
    }
}
```

### 4.3 foundry.toml Configuration

```toml
# foundry.toml

[profile.default]
src = "src"
out = "out"
libs = ["lib"]
solc = "0.8.24"
optimizer = true
optimizer-runs = 200
via-ir = true

# Fork testing
[profile.fork]
# Inherit from default
fuzz = { runs = 256 }

[rpc_endpoints]
mainnet = "${MAINNET_RPC_URL}"
goerli = "${GOERLI_RPC_URL}"
arbitrum = "${ARBITRUM_RPC_URL}"

# Cache forks สำหรับ speed
[fork_block_cache]
enabled = true
cache_dir = "~/.foundry/cache/rpc"

[fuzz]
runs = 1000
max_test_rejects = 65536
seed = "0x3e8"
dictionary_weight = 40
include_storage = true
include_push_bytes = true

[invariant]
runs = 256
depth = 15
fail_on_revert = false
```

---

## Workshop: ทดสอบ Protocol ของคุณบน Mainnet Fork

### Workshop Exercise: Flash Loan Defense Test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test, console} from "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

/**
 * @title FlashLoanDefenseTest
 * @notice Workshop: ทดสอบ defense ต่อ flash loan attacks
 * นักเรียนต้องทำ:
 * 1. Implement PriceOracle contract ที่ต้าน flash loan manipulation
 * 2. ทดสอบว่า oracle ไม่ถูก manipulate ด้วย large flash loan
 * 3. เปรียบเทียบ TWAPOracle กับ SpotPriceOracle
 */
contract FlashLoanDefenseTest is Test {

    address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address constant UNISWAP_V3_POOL = 0x8ad599c3A0ff1De082011EFDDc58f1908eb6e6D5;

    uint256 mainnetFork;

    function setUp() public {
        mainnetFork = vm.createFork(
            vm.envOr("MAINNET_RPC_URL", string("https://eth.llamarpc.com")),
            19_500_000
        );
        vm.selectFork(mainnetFork);
    }

    /**
     * @notice TODO: Implement และ test TWAP oracle ที่ต้าน manipulation
     *
     * Hints:
     * - Uniswap V3 มี observe() function สำหรับ TWAP
     * - TWAP ที่ยาวพอ (เช่น 30 นาที) จะต้าน flash loan ได้
     * - Flash loan เกิดขึ้นภายใน 1 block ซึ่งสั้นมาก
     *
     * ลอง:
     * 1. อ่านราคาปัจจุบัน (spot price)
     * 2. จำลอง large flash loan ที่ manipulate ราคา
     * 3. อ่าน TWAP price หลัง manipulation
     * 4. ตรวจสอบว่า TWAP ยังเสถียร
     */
    function test_twapResistToFlashLoan() public {
        // TODO: Implement this test
        // Your implementation here
    }
}
```

---

## สรุป Part 70

- **Hardhat Fork**: ใช้ `forking.blockNumber` เพื่อ pin block สำหรับ reproducibility, `hardhat_impersonateAccount` เพื่อใช้ address ใดๆ
- **Foundry Fork**: `vm.createFork()` สร้าง fork, `vm.selectFork()` เลือก active fork, `vm.makePersistent()` ทำให้ contract อยู่ข้ามการสลับ fork
- **deal/store**: `deal(token, user, amount)` ตั้ง token balance, `vm.store()` manipulate storage โดยตรง
- **Uniswap V3 Testing**: test swap จริง, ตรวจสอบ price impact, verify slippage protection
- **Whale Attack Simulation**: จำลอง front-running, sandwich attack, ทดสอบ defense mechanisms
- **Upgrade Testing**: verify state preservation, regression test, storage collision check
- **Best Practices**: pin block number, use snapshots ใน beforeEach/afterEach, set long timeout สำหรับ fork tests

## Next: Part 71 - Protocol Architecture at Scale
