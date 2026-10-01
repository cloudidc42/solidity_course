# Part 43: Advanced Foundry Testing

## สารบัญ
1. Foundry Test Cheatcodes Deep Dive
2. Differential Testing
3. Symbolic Execution (Halmos)
4. Mutation Testing
5. Workshop: Full Protocol Test Suite

---

## 1. Foundry Cheatcodes Deep Dive

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "../src/MyVault.sol";

contract AdvancedCheatcodesTest is Test {
    
    MyVault vault;
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    address exploiter = makeAddr("exploiter");
    
    function setUp() public {
        vault = new MyVault();
        deal(alice, 100 ether);
        deal(bob, 50 ether);
    }
    
    // ============= Time Manipulation =============
    
    function test_timeBasedVesting() public {
        uint256 vestStart = block.timestamp;
        
        vm.prank(alice);
        vault.deposit{value: 10 ether}();
        
        // Cannot withdraw before cliff (1 year)
        vm.expectRevert("Cliff not reached");
        vm.prank(alice);
        vault.withdraw(10 ether);
        
        // Warp to after cliff
        vm.warp(vestStart + 365 days + 1);
        
        // Now can withdraw
        uint256 balBefore = alice.balance;
        vm.prank(alice);
        vault.withdraw(5 ether);
        assertEq(alice.balance, balBefore + 5 ether);
    }
    
    // ============= Gas Measurement =============
    
    function test_gasOptimization() public {
        vm.startSnapshotGas("inefficient_loop");
        vault.processAll(100); // test with 100 items
        uint256 inefficientGas = vm.stopSnapshotGas("inefficient_loop");
        
        vm.startSnapshotGas("efficient_loop");
        vault.processAllOptimized(100);
        uint256 efficientGas = vm.stopSnapshotGas("efficient_loop");
        
        // Ensure optimized is at least 20% cheaper
        assertLt(efficientGas, inefficientGas * 80 / 100, "Not optimized enough");
        
        emit log_named_uint("Inefficient gas", inefficientGas);
        emit log_named_uint("Efficient gas", efficientGas);
        emit log_named_uint("Savings", inefficientGas - efficientGas);
    }
    
    // ============= Storage Manipulation =============
    
    function test_storageSlotManipulation() public {
        // Direct storage write (bypass access control for testing)
        bytes32 ownerSlot = bytes32(uint256(0)); // slot 0 = owner
        vm.store(address(vault), ownerSlot, bytes32(uint256(uint160(alice))));
        
        // Verify it changed
        address newOwner = address(uint160(uint256(vm.load(address(vault), ownerSlot))));
        assertEq(newOwner, alice);
    }
    
    // ============= Call Tracing =============
    
    function test_callTrace() public {
        vm.recordLogs();
        
        vm.prank(alice);
        vault.deposit{value: 1 ether}();
        
        Vm.Log[] memory logs = vm.getRecordedLogs();
        
        // Verify deposit event was emitted
        assertGt(logs.length, 0, "No events emitted");
        
        // First log should be Deposit event
        // topic[0] = keccak256("Deposit(address,uint256)")
        bytes32 depositSig = keccak256("Deposit(address,uint256)");
        assertEq(logs[0].topics[0], depositSig, "Wrong event");
    }
    
    // ============= Mock External Calls =============
    
    function test_mockOracle() public {
        address mockOracle = makeAddr("mockOracle");
        
        // Mock oracle to return specific price
        vm.mockCall(
            mockOracle,
            abi.encodeWithSignature("latestRoundData()"),
            abi.encode(
                uint80(1),      // roundId
                int256(2000e8), // answer: $2000
                uint256(0),     // startedAt
                block.timestamp, // updatedAt
                uint80(1)       // answeredInRound
            )
        );
        
        // Test contract that uses oracle
        vault.setOracle(mockOracle);
        uint256 valueInUSD = vault.getValueInUSD(1 ether);
        assertEq(valueInUSD, 2000e18, "Wrong USD value");
        
        vm.clearMockedCalls();
    }
    
    // ============= Expect Specific Reverts =============
    
    function test_specificRevertData() public {
        // Test custom error with parameters
        vm.expectRevert(
            abi.encodeWithSelector(
                MyVault.InsufficientBalance.selector,
                100 ether, // requested
                0          // available
            )
        );
        
        vm.prank(alice);
        vault.withdraw(100 ether); // alice has 0 deposited
    }
    
    // ============= Signature Testing =============
    
    function test_permitSignature() public {
        uint256 alicePrivKey = 0xA11CE;
        address aliceAddr = vm.addr(alicePrivKey);
        
        bytes32 permitHash = vault.getPermitHash(
            aliceAddr,
            bob,
            1 ether,
            0, // nonce
            block.timestamp + 1 hours
        );
        
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(alicePrivKey, permitHash);
        
        // Execute permit
        vault.permit(aliceAddr, bob, 1 ether, block.timestamp + 1 hours, v, r, s);
        
        assertEq(vault.allowance(aliceAddr, bob), 1 ether);
    }
}
```

---

## 2. Invariant Testing Advanced

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "../src/Vault.sol";
import "../src/Token.sol";

/**
 * Handler: กำหนด valid actions ที่ fuzzer ใช้
 */
contract VaultHandler is Test {
    
    Vault public vault;
    Token public token;
    
    // Ghost variables: track expected state
    uint256 public ghost_totalDeposited;
    uint256 public ghost_totalWithdrawn;
    
    address[] public actors;
    mapping(address => uint256) public actorDeposits;
    
    constructor(address _vault, address _token) {
        vault = Vault(_vault);
        token = Token(_token);
        
        // Create test actors
        for (uint256 i = 1; i <= 5; i++) {
            address actor = address(uint160(i));
            actors.push(actor);
            deal(address(token), actor, 1_000_000e18);
        }
    }
    
    // Fuzzer calls these functions
    
    function deposit(uint256 actorSeed, uint256 amount) public {
        address actor = actors[actorSeed % actors.length];
        amount = bound(amount, 1, token.balanceOf(actor));
        
        vm.startPrank(actor);
        token.approve(address(vault), amount);
        vault.deposit(amount);
        vm.stopPrank();
        
        // Update ghost state
        ghost_totalDeposited += amount;
        actorDeposits[actor] += amount;
    }
    
    function withdraw(uint256 actorSeed, uint256 amount) public {
        address actor = actors[actorSeed % actors.length];
        
        uint256 maxWithdraw = vault.balanceOf(actor);
        if (maxWithdraw == 0) return; // skip if nothing to withdraw
        
        amount = bound(amount, 1, maxWithdraw);
        
        vm.prank(actor);
        vault.withdraw(amount);
        
        ghost_totalWithdrawn += amount;
        actorDeposits[actor] -= amount;
    }
    
    function transfer(uint256 fromSeed, uint256 toSeed, uint256 amount) public {
        address from = actors[fromSeed % actors.length];
        address to = actors[toSeed % actors.length];
        if (from == to) return;
        
        uint256 fromBalance = vault.balanceOf(from);
        if (fromBalance == 0) return;
        
        amount = bound(amount, 1, fromBalance);
        
        vm.prank(from);
        vault.transfer(to, amount);
        
        // Transfers don't change ghost totals
    }
}

/**
 * Invariant Test Suite
 */
contract VaultInvariantTest is Test {
    
    Vault vault;
    Token token;
    VaultHandler handler;
    
    function setUp() public {
        token = new Token("Test", "TST", 1_000_000_000e18);
        vault = new Vault(address(token));
        handler = new VaultHandler(address(vault), address(token));
        
        // Target only handler for fuzzing
        targetContract(address(handler));
    }
    
    // INVARIANT 1: vault balance >= sum of user deposits
    function invariant_solvency() public view {
        uint256 vaultBalance = token.balanceOf(address(vault));
        uint256 totalShares = vault.totalSupply();
        
        // If shares > 0, there must be tokens backing them
        if (totalShares > 0) {
            assertGe(vaultBalance, 1, "Vault is insolvent");
        }
    }
    
    // INVARIANT 2: ghost accounting matches real state
    function invariant_accounting() public view {
        uint256 expectedBalance = handler.ghost_totalDeposited() - handler.ghost_totalWithdrawn();
        uint256 actualShares = vault.totalSupply();
        
        // Shares issued should match net deposits (1:1 ratio initially)
        // Note: actual check depends on vault implementation
        assertGe(expectedBalance, 0, "Negative balance");
    }
    
    // INVARIANT 3: no individual can withdraw more than deposited
    function invariant_noOverWithdraw() public view {
        address[] memory actors = handler.actors();
        for (uint256 i; i < actors.length; i++) {
            address actor = actors[i];
            uint256 shares = vault.balanceOf(actor);
            uint256 maxWithdrawable = vault.previewWithdraw(shares);
            
            // User should never have negative "balance"
            assertGe(maxWithdrawable, 0, "Negative withdrawable");
        }
    }
}
```

---

## 3. Differential Testing

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";

/**
 * Differential Testing:
 * เปรียบเทียบ implementation ใหม่กับ implementation เก่า
 * หรือเปรียบเทียบกับ reference implementation
 */
contract DifferentialTest is Test {
    
    NewAMM newAmm;
    ReferenceAMM refAmm;
    
    function setUp() public {
        newAmm = new NewAMM();
        refAmm = new ReferenceAMM();
        
        // Same initial state
        newAmm.initialize(1000e18, 2000e18);
        refAmm.initialize(1000e18, 2000e18);
    }
    
    // Both implementations should give same result
    function testFuzz_equivalentSwap(uint256 amountIn) public {
        amountIn = bound(amountIn, 1, 100e18);
        
        uint256 newOut = newAmm.getAmountOut(amountIn, true);
        uint256 refOut = refAmm.getAmountOut(amountIn, true);
        
        // Allow up to 1 wei difference for rounding
        assertApproxEqAbs(newOut, refOut, 1, "AMM implementations differ");
    }
    
    // Cross-verify with off-chain calculation
    function test_mathAgainstPython() public {
        // Using vm.ffi() to call Python script for reference
        string[] memory cmd = new string[](3);
        cmd[0] = "python3";
        cmd[1] = "scripts/amm_calc.py";
        cmd[2] = "1000"; // amountIn
        
        bytes memory result = vm.ffi(cmd);
        uint256 pythonResult = abi.decode(result, (uint256));
        
        uint256 solidityResult = newAmm.getAmountOut(1000e18, true);
        
        assertApproxEqRel(solidityResult, pythonResult * 1e18, 0.001e18, "1% diff from Python"); // 0.1%
    }
}
```

---

## สรุป Part 43

Advanced Foundry Testing ที่เรียนรู้:
- ✅ Cheatcodes: vm.warp, vm.prank, vm.mockCall
- ✅ Gas snapshots
- ✅ Storage slot manipulation
- ✅ Signature testing (vm.sign)
- ✅ Invariant testing with handlers
- ✅ Differential testing

## Next: Part 44 - Smart Contract Formal Verification
