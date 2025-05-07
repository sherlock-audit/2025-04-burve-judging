Alert Tiger Fish

Medium

# Missing Slippage Protection in Burve Protocol Mint and Burn Functions

### Summary

Insufficient slippage protection in mint and burn functions will cause financial losses for users as front-running attackers will sandwich user transactions through price manipulation.

### Root Cause

In Burve.sol, the [mint](https://github.com/sherlock-audit/2025-04-burve/blob/main/Burve/src/single/Burve.sol#L226) and [burn](https://github.com/sherlock-audit/2025-04-burve/blob/main/Burve/src/single/Burve.sol#L350) functions only check if the current price is within a specified range, but do not enforce minimum output or maximum input constraints:

```solidity 
function mint(
    address recipient,
    uint128 mintNominalLiq,
    uint160 lowerSqrtPriceLimitX96,
    uint160 upperSqrtPriceLimitX96
)
    public
    withinSqrtPX96Limits(lowerSqrtPriceLimitX96, upperSqrtPriceLimitX96)
    returns (uint256 shares)
{
    // ...
}

function burn(
    uint256 shares,
    uint160 lowerSqrtPriceLimitX96,
    uint160 upperSqrtPriceLimitX96
)
    external
    withinSqrtPX96Limits(lowerSqrtPriceLimitX96, upperSqrtPriceLimitX96)
{
    // ...
}
```
The withinSqrtPX96Limits modifier only checks that the current price is within the specified bounds, but does not prevent users from receiving significantly fewer tokens or paying significantly more due to price manipulation.

### Internal Pre-conditions

1. Users must interact with the protocol's mint or burn functions
2. The protocol must be active with sufficient liquidity to enable price manipulation

### External Pre-conditions

1. There must be enough liquidity in the underlying pools to enable price manipulation

### Attack Path

1. Attacker monitors the mempool for pending mint or burn transactions
2. When a large transaction is identified, attacker front-runs it with a swap that moves the price unfavorably
3. The victim's transaction executes at the manipulated price, resulting in significantly worse terms
4. Attacker back-runs with a swap that restores the price, profiting from the manipulation

### Impact

Users consistently suffer financial losses due to sandwich attacks. Based on the test, users can pay at least 20% more tokens for the same amount of LP tokens, or receive 20% fewer tokens when burning, resulting in potentially significant financial damage to users over time. The protocol itself does not directly lose funds, but users' trust in the system will erode as they experience these losses.

### PoC

```solidity
function test_MissingSlippageProtection() public forkOnly {
    // Setup - mint to alice
    deal(address(token0), alice, type(uint256).max);
    deal(address(token1), alice, type(uint256).max);
    vm.startPrank(alice);
    token0.approve(address(burve), type(uint256).max);
    token1.approve(address(burve), type(uint256).max);
    
    // First mint at normal price
    uint256 aliceToken0Before = token0.balanceOf(alice);
    uint256 aliceToken1Before = token1.balanceOf(alice);
    uint128 mintAmount = 10000;
    burve.mint(alice, mintAmount, 0, type(uint128).max);
    uint256 aliceToken0After = token0.balanceOf(alice);
    uint256 aliceToken1After = token1.balanceOf(alice);
    
    // Record how much alice paid for the first mint
    uint256 token0PaidNormal = aliceToken0Before - aliceToken0After;
    uint256 token1PaidNormal = aliceToken1Before - aliceToken1After;
    
    // Now manipulate the price to be unfavorable (as if by a sandwich attack)
    (uint160 sqrtRatioX96, , , , , , ) = pool.slot0();
    deal(address(token0), address(this), 10000000 * 10**18);
    token0.approve(address(pool), 10000000 * 10**18);
    pool.swap(address(this), true, 10000000 * 10**18, type(uint160).max, "");
    
    // Alice mints again with the same parameters, but now at manipulated price
    aliceToken0Before = token0.balanceOf(alice);
    aliceToken1Before = token1.balanceOf(alice);
    burve.mint(alice, mintAmount, 0, type(uint128).max); // Same parameters, no slippage protection
    aliceToken0After = token0.balanceOf(alice);
    aliceToken1After = token1.balanceOf(alice);
    
    // Record how much alice paid at manipulated price
    uint256 token0PaidManipulated = aliceToken0Before - aliceToken0After;
    uint256 token1PaidManipulated = aliceToken1Before - aliceToken1After;
    
    // Restore original price (attacker would do this)
    deal(address(token1), address(this), 10000000 * 10**18);
    token1.approve(address(pool), 10000000 * 10**18);
    pool.swap(address(this), false, 10000000 * 10**18, sqrtRatioX96, "");
    
    vm.stopPrank();
    
    // Check if alice paid significantly more due to price manipulation
    assertTrue(token0PaidManipulated > token0PaidNormal * 12 / 10 || 
              token1PaidManipulated > token1PaidNormal * 12 / 10, 
              "Alice should pay more due to price manipulation without slippage protection");
    
    // The test passes if alice paid at least 20% more for the same amount of LP tokens
    // due to the lack of proper slippage protection
}
```

### Mitigation

To mitigate this vulnerability, the Burve protocol should implement proper slippage protection:

1. For the mint function, add parameters to specify the maximum amount of tokens the user is willing to pay:
```solidity
function mint(
    address recipient,
    uint128 mintNominalLiq,
    uint160 lowerSqrtPriceLimitX96,
    uint160 upperSqrtPriceLimitX96,
    uint256 maxToken0Amount,
    uint256 maxToken1Amount
) public returns (uint256 shares) {
    // ... calculate token0Amount and token1Amount required for mint
    require(token0Amount <= maxToken0Amount, "Slippage: token0 amount exceeds maximum");
    require(token1Amount <= maxToken1Amount, "Slippage: token1 amount exceeds maximum");
    // ... rest of the function
}
```
2. For the burn function, add parameters to specify the minimum amount of tokens the user expects to receive:
```solidity
function burn(
    uint256 shares,
    uint160 lowerSqrtPriceLimitX96,
    uint160 upperSqrtPriceLimitX96,
    uint256 minToken0Amount,
    uint256 minToken1Amount
) external {
    // ... calculate token0Amount and token1Amount to be returned
    require(token0Amount >= minToken0Amount, "Slippage: token0 amount below minimum");
    require(token1Amount >= minToken1Amount, "Slippage: token1 amount below minimum");
    // ... rest of the function
}
```
This ensures users can set acceptable bounds for their transactions and avoid being exploited by sandwich attacks.

