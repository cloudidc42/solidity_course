# Part 28: Advanced Testing Patterns

## สารบัญ
1. Foundry Advanced Testing
2. Fuzz Testing
3. Invariant Testing
4. Fork Testing
5. Workshop: Comprehensive Test Suite

---

## 1. Foundry Advanced Testing

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "forge-std/console.sol";

/**
 * Advanced Foundry Techniques:
 * - vm.prank: เปลี่ยน msg.sender
 * - vm.deal: กำหนด ETH balance
 * - vm.warp: เปลี่ยน timestamp
 * - vm.roll: เปลี่ยน block number
 * - vm.expectRevert: expect revert
 * - vm.expectEmit: expect event
 * - vm.snapshot/revertTo: state snapshot
 */
contract AdvancedTest is Test {
    
    MyToken token;
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    address charlie = makeAddr("charlie");
    
    function setUp() public {
        token = new MyToken("Test", "TST", 1_000_000e18);
        
        // Give users tokens
        token.transfer(alice, 10_000e18);
        token.transfer(bob, 5_000e18);
    }
    
    // Test with specific msg.sender
    function test_transferAsAlice() public {
        vm.prank(alice);
        token.transfer(bob, 1_000e18);
        
        assertEq(token.balanceOf(alice), 9_000e18);
        assertEq(token.balanceOf(bob), 6_000e18);
    }
    
    // Expect specific revert message
    function test_revertOnInsufficientBalance() public {
        vm.prank(charlie); // charlie has 0 tokens
        vm.expectRevert(
            abi.encodeWithSelector(MyToken.InsufficientBalance.selector, charlie, 0, 100e18)
        );
        token.transfer(alice, 100e18);
    }
    
    // Test with time manipulation
    function test_vestingAfterCliff() public {
        VestingWallet vesting = new VestingWallet(
            address(token),
            alice,
            block.timestamp,     // start now
            30 days,             // 30 day cliff
            365 days,            // 1 year total
            address(this)
        );
        
        token.transfer(address(vesting), 1_000e18);
        
        // At cliff: nothing available
        assertEq(vesting.releasableAmount(), 0);
        
        // After cliff: partial amount
        vm.warp(block.timestamp + 30 days + 1);
        
        uint256 releasable = vesting.releasableAmount();
        assertGt(releasable, 0);
        
        // At end: all available
        vm.warp(block.timestamp + 365 days);
        assertEq(vesting.releasableAmount(), 1_000e18);
    }
    
    // Expect specific event emission
    function test_transferEmitsEvent() public {
        vm.expectEmit(true, true, false, true);
        emit Transfer(alice, bob, 100e18); // expected event
        
        vm.prank(alice);
        token.transfer(bob, 100e18); // must emit this
    }
    
    // Snapshot and revert
    function test_snapshotRestore() public {
        uint256 snapshot = vm.snapshot();
        
        vm.prank(alice);
        token.transfer(bob, 1_000e18);
        
        assertEq(token.balanceOf(alice), 9_000e18);
        
        vm.revertTo(snapshot);
        
        // State restored
        assertEq(token.balanceOf(alice), 10_000e18);
    }
    
    // Multiple pranks with startPrank/stopPrank
    function test_complexScenario() public {
        vm.startPrank(alice);
        
        token.approve(address(this), 5_000e18);
        token.transfer(bob, 1_000e18);
        
        vm.stopPrank();
        
        // Back to test contract as caller
        token.transferFrom(alice, charlie, 2_000e18);
        
        assertEq(token.balanceOf(alice), 7_000e18);
        assertEq(token.balanceOf(bob), 6_000e18);
        assertEq(token.balanceOf(charlie), 2_000e18);
    }
    
    event Transfer(address indexed from, address indexed to, uint256 value);
}
```

---

## 2. Fuzz Testing

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";

/**
 * Fuzz Testing:
 * Foundry generates random inputs automatically
 * ค้นหา edge cases ที่เราคิดไม่ถึง
 * 
 * Config ใน foundry.toml:
 * [fuzz]
 * runs = 1000  # จำนวนครั้งที่ fuzz
 * seed = 0x1234
 */
contract FuzzTest is Test {
    
    SafeMath math;
    MyToken token;
    
    function setUp() public {
        math = new SafeMath();
        token = new MyToken("Test", "TST", type(uint256).max / 2);
    }
    
    // Foundry จะ generate random a, b values
    function testFuzz_addNoOverflow(uint256 a, uint256 b) public {
        // Constrain inputs
        vm.assume(a <= type(uint128).max);
        vm.assume(b <= type(uint128).max);
        
        uint256 result = math.add(a, b);
        
        assertEq(result, a + b);
        assertGe(result, a); // result >= a (no overflow)
    }
    
    // Fuzz transfer amount
    function testFuzz_transferPreservesTotal(
        address from,
        address to,
        uint256 amount
    ) public {
        // Filter invalid addresses
        vm.assume(from != address(0));
        vm.assume(to != address(0));
        vm.assume(from != to);
        
        // Setup: give 'from' some tokens
        uint256 initialBalance = 1_000e18;
        deal(address(token), from, initialBalance);
        
        // Constrain amount
        amount = bound(amount, 1, initialBalance);
        
        uint256 totalBefore = token.balanceOf(from) + token.balanceOf(to);
        
        vm.prank(from);
        token.transfer(to, amount);
        
        uint256 totalAfter = token.balanceOf(from) + token.balanceOf(to);
        
        // Total tokens should be preserved
        assertEq(totalBefore, totalAfter);
    }
    
    // Fuzz interest calculation
    function testFuzz_interestRateNeverExceedsMax(
        uint256 utilization,
        uint256 kink
    ) public {
        uint256 MAX_RATE = 3e18; // 300% max APR
        
        utilization = bound(utilization, 0, 1e18);
        kink = bound(kink, 0.3e18, 0.9e18);
        
        JumpRateModel model = new JumpRateModel(
            0,           // base rate
            0.2e18,      // multiplier
            2e18,        // jump multiplier
            kink
        );
        
        uint256 borrows = utilization * 1_000_000e18 / 1e18;
        uint256 cash = 1_000_000e18 - borrows;
        
        uint256 rate = model.getBorrowRate(cash, borrows, 0);
        
        assertLe(rate * 2_628_000, MAX_RATE, "Rate exceeds max");
    }
}

contract SafeMath {
    function add(uint256 a, uint256 b) external pure returns (uint256) {
        return a + b;
    }
}
```

---

## 3. Invariant Testing

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "forge-std/StdInvariant.sol";

/**
 * Invariant Testing:
 * กำหนด properties ที่ต้องเป็นจริงเสมอ
 * ไม่ว่าจะเกิด sequence of actions อะไร
 * 
 * Config:
 * [invariant]
 * runs = 256
 * depth = 15  # จำนวน actions per run
 */
contract InvariantTest is Test {
    
    MyToken token;
    Handler handler;
    
    function setUp() public {
        token = new MyToken("Test", "TST", 1_000_000e18);
        handler = new Handler(token);
        
        // ให้ handler tokens ไปจัดการ
        token.transfer(address(handler), 1_000_000e18);
        
        // Target: Foundry จะเรียก handler functions แบบ random
        targetContract(address(handler));
    }
    
    // INVARIANT: totalSupply ต้องเท่ากับ sum of all balances เสมอ
    function invariant_totalSupplyEqualsBalances() public {
        assertEq(
            token.totalSupply(),
            token.balanceOf(address(handler)) +
            token.balanceOf(address(this)) +
            handler.ghostTotalDistributed()
        );
    }
    
    // INVARIANT: totalSupply ไม่เพิ่มขึ้น (no minting)
    function invariant_supplyNeverIncreases() public {
        assertLe(token.totalSupply(), 1_000_000e18);
    }
    
    // INVARIANT: no address ควร balance เกิน totalSupply
    function invariant_noBalanceExceedsSupply() public {
        address[] memory actors = handler.actors();
        for (uint256 i = 0; i < actors.length; i++) {
            assertLe(token.balanceOf(actors[i]), token.totalSupply());
        }
    }
}

/**
 * Handler: Proxy สำหรับ invariant testing
 * เก็บ ghost variables เพื่อ track state
 */
contract Handler is Test {
    
    MyToken token;
    
    address[] public actors;
    mapping(address => bool) public isActor;
    
    uint256 public ghostTotalDistributed;
    
    constructor(MyToken _token) {
        token = _token;
        // Create initial actors
        actors.push(makeAddr("actor1"));
        actors.push(makeAddr("actor2"));
        actors.push(makeAddr("actor3"));
        for (uint256 i = 0; i < actors.length; i++) {
            isActor[actors[i]] = true;
        }
    }
    
    // Random transfers between actors
    function transfer(uint256 fromSeed, uint256 toSeed, uint256 amount) external {
        address from = actors[fromSeed % actors.length];
        address to = actors[toSeed % actors.length];
        
        amount = bound(amount, 0, token.balanceOf(from));
        
        if (amount > 0 && from != to) {
            vm.prank(from);
            token.transfer(to, amount);
        }
    }
    
    // Distribute from handler
    function distribute(uint256 seed, uint256 amount) external {
        address to = actors[seed % actors.length];
        amount = bound(amount, 0, token.balanceOf(address(this)));
        
        if (amount > 0) {
            token.transfer(to, amount);
            ghostTotalDistributed += amount;
        }
    }
}
```

---

## 4. Fork Testing

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";

/**
 * Fork Testing:
 * ทดสอบบน fork ของ mainnet/testnet
 * ใช้ state จริงของ blockchain
 * 
 * Command:
 * forge test --fork-url https://eth-mainnet.g.alchemy.com/v2/... -vvv
 * 
 * หรือ ใน foundry.toml:
 * [rpc_endpoints]
 * mainnet = "${MAINNET_RPC_URL}"
 * 
 * ใน test: vm.createFork("mainnet")
 */
contract ForkTest is Test {
    
    // Real Mainnet addresses
    address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address constant UNISWAP_ROUTER = 0xE592427A0AEce92De3Edee1F18E0157C05861564;
    address constant WHALE = 0x28C6c06298d514Db089934071355E5743bf21d60; // Binance
    
    uint256 mainnetFork;
    
    function setUp() public {
        // Create fork at specific block
        mainnetFork = vm.createFork(vm.envString("MAINNET_RPC_URL"), 19_000_000);
    }
    
    function test_swapOnFork() public {
        vm.selectFork(mainnetFork);
        
        // Impersonate whale
        vm.startPrank(WHALE);
        
        IERC20 weth = IERC20(WETH);
        IERC20 usdc = IERC20(USDC);
        
        uint256 wethAmount = 1e18; // 1 WETH
        uint256 usdcBefore = usdc.balanceOf(WHALE);
        
        // Approve router
        weth.approve(UNISWAP_ROUTER, wethAmount);
        
        // Swap WETH → USDC
        ISwapRouter(UNISWAP_ROUTER).exactInputSingle(
            ISwapRouter.ExactInputSingleParams({
                tokenIn: WETH,
                tokenOut: USDC,
                fee: 3000,
                recipient: WHALE,
                deadline: block.timestamp + 1,
                amountIn: wethAmount,
                amountOutMinimum: 0,
                sqrtPriceLimitX96: 0
            })
        );
        
        uint256 usdcAfter = usdc.balanceOf(WHALE);
        uint256 usdcReceived = usdcAfter - usdcBefore;
        
        console.log("USDC received:", usdcReceived / 1e6);
        
        assertGt(usdcReceived, 0);
        
        vm.stopPrank();
    }
    
    // Test our contract with real state
    function test_ourContractWithRealTokens() public {
        vm.selectFork(mainnetFork);
        
        MyProtocol protocol = new MyProtocol(WETH, USDC);
        
        vm.startPrank(WHALE);
        IERC20(WETH).transfer(address(protocol), 10e18);
        vm.stopPrank();
        
        // Test protocol with real WETH
        // ...
    }
}

interface IERC20 {
    function balanceOf(address) external view returns (uint256);
    function transfer(address, uint256) external returns (bool);
    function approve(address, uint256) external returns (bool);
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
    
    function exactInputSingle(ExactInputSingleParams calldata params) external payable returns (uint256);
}

contract MyProtocol {
    IERC20 public tokenA;
    IERC20 public tokenB;
    
    constructor(address _tokenA, address _tokenB) {
        tokenA = IERC20(_tokenA);
        tokenB = IERC20(_tokenB);
    }
}
```

---

## 5. Gas Testing

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";

contract GasTest is Test {
    
    MyToken naive;
    MyToken optimized;
    
    function setUp() public {
        naive = new MyToken("Naive", "NAV", 1_000_000e18);
        optimized = new MyToken("Opt", "OPT", 1_000_000e18);
    }
    
    function test_transferGas() public {
        address alice = makeAddr("alice");
        address bob = makeAddr("bob");
        
        naive.transfer(alice, 1_000e18);
        
        vm.prank(alice);
        uint256 gasBefore = gasleft();
        naive.transfer(bob, 100e18);
        uint256 gasNaive = gasBefore - gasleft();
        
        console.log("Naive transfer gas:", gasNaive);
        
        // Compare with optimized
        optimized.transfer(alice, 1_000e18);
        
        vm.prank(alice);
        gasBefore = gasleft();
        optimized.transfer(bob, 100e18);
        uint256 gasOptimized = gasBefore - gasleft();
        
        console.log("Optimized transfer gas:", gasOptimized);
        console.log("Savings:", gasNaive - gasOptimized);
    }
}
```

---

## สรุป Part 28

Advanced Testing ที่เรียนรู้:
- ✅ Foundry cheatcodes (prank, warp, roll, snapshot)
- ✅ Fuzz testing with bounds
- ✅ Invariant testing + Handler pattern
- ✅ Fork testing บน mainnet state
- ✅ Gas benchmarking

## Quiz

1. Fuzz testing vs Invariant testing ต่างกันอย่างไร?
2. vm.assume vs bound ใช้เมื่อไหร่?
3. Ghost variables ในเทสคืออะไร?
4. Fork testing มีข้อดีอะไรเหนือ unit tests?

---

## Next: Part 29 - DeFi Protocol Design
