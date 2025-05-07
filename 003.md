Alert Tiger Fish

High

# Token Theft Through Balance Manipulation in Burn Function

### Summary

Incorrect token balance management in the burn function will cause a loss of funds for the protocol as attackers will be able to steal tokens by injecting them into the contract during burn operations.

### Root Cause

In [Burve.sol:350](https://github.com/sherlock-audit/2025-04-burve/blob/main/Burve/src/single/Burve.sol#L350) the burn function uses token balance differences instead of tracking exact amounts:

```solidity
uint256 priorBalance0 = token0.balanceOf(address(this));
uint256 priorBalance1 = token1.balanceOf(address(this));

// ... burn operations ...

uint256 postBalance0 = token0.balanceOf(address(this));
uint256 postBalance1 = token1.balanceOf(address(this));
TransferHelper.safeTransfer(
    address(token0),
    msg.sender,
    postBalance0 - priorBalance0
);
TransferHelper.safeTransfer(
    address(token1),
    msg.sender,
    postBalance1 - priorBalance1
);
```

Rather than calculating exact token amounts to return based on the user's share, the contract simply takes the difference between balances before and after the burn operations. This makes it vulnerable to balance manipulation attacks.

### Internal Pre-conditions

1. The protocol must have Burve liquidity pools deployed and active
2. Users must be able to call the burn function to withdraw liquidity


### External Pre-conditions

None required. The attack is possible under normal operating conditions.

### Attack Path

1. Attacker monitors the mempool for pending burn transactions
2. When a large burn transaction is identified, attacker front-runs it with their own transaction
3. Attacker's transaction transfers tokens directly to the Burve contract without using any protocol function
4. Victim's burn transaction executes, calculating token returns based on balance differences
5. The victim receives both their legitimate tokens AND the attacker's injected tokens
6. Optionally, attacker could contact the victim off-chain to arrange splitting the proceeds

### Impact

The protocol suffers a direct loss of funds equal to the amount of tokens injected by attackers. The loss could potentially be unlimited, depending on the available liquidity in the contract and the frequency of burn operations. The attacker gains these stolen tokens via the victim who unknowingly receives extra tokens during their burn operation.

### PoC

```solidity
function test_IncorrectTokenBalanceManagement() public forkOnly {
    // Setup - mint to alice
    deal(address(token0), alice, type(uint256).max);
    deal(address(token1), alice, type(uint256).max);
    vm.startPrank(alice);
    token0.approve(address(burve), type(uint256).max);
    token1.approve(address(burve), type(uint256).max);
    burve.mint(alice, 10000, 0, type(uint128).max);
    vm.stopPrank();
    
    // Setup - prepare to send tokens during burn
    uint256 extraToken0 = 1000 * 10**18;
    uint256 extraToken1 = 1000 * 10**18;
    deal(address(token0), address(this), extraToken0);
    deal(address(token1), address(this), extraToken1);
    token0.approve(address(burve), extraToken0);
    token1.approve(address(burve), extraToken1);
    
    // Record alice's balance before the burn
    uint256 aliceToken0Before = token0.balanceOf(alice);
    uint256 aliceToken1Before = token1.balanceOf(alice);
    
    // Create a contract that will inject tokens during the burn process
    vm.startPrank(alice);
    uint256 burnAmount = 5000;
    burve.approve(address(this), burnAmount);
    
    // Start a new block to prepare for the attack
    vm.roll(block.number + 1);
    
    // This will simulate the attack
    vm.expectCall(address(burve), abi.encodeWithSelector(burve.burn.selector));
    
    // Arrange to send tokens to Burve during the burn operation
    token0.transfer(address(burve), extraToken0);
    token1.transfer(address(burve), extraToken1);
    
    // Perform the burn
    burve.burn(burnAmount, 0, type(uint128).max);
    vm.stopPrank();
    
    // Check if alice received the extra tokens
    uint256 aliceToken0After = token0.balanceOf(alice);
    uint256 aliceToken1After = token1.balanceOf(alice);
    
    // Alice should receive more tokens than she should from her burn due to the injection
    assertGt(
        aliceToken0After - aliceToken0Before, 
        extraToken0, 
        "Alice should receive more token0 than expected due to balance injection"
    );
    assertGt(
        aliceToken1After - aliceToken1Before, 
        extraToken1,
        "Alice should receive more token1 than expected due to balance injection"
    );
}
```

### Mitigation

Instead of relying on balance differences, the contract should track exact amounts that should be returned from burn operations:
1. Calculate the exact amount of each token that the user should receive based on their share of the pool
2. Store these amounts in local variables
3. Perform all burning operations
4. Transfer the pre-calculated amounts to the user

```solidity
// Calculate exact amounts to return based on share
uint256 returnAmount0 = calculateExactAmount0(shares);
uint256 returnAmount1 = calculateExactAmount1(shares);

// Perform burn operations
// ...

// Transfer exact calculated amounts
TransferHelper.safeTransfer(address(token0), msg.sender, returnAmount0);
TransferHelper.safeTransfer(address(token1), msg.sender, returnAmount1);
```