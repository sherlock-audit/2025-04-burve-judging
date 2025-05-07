Prehistoric Jade Albatross

Medium

# Protocol/User suffer Denial of Service in closures where 100% of staked value is BGT-directed due to division by zero

### Summary

A division by zero in internal balance trimming functions will cause a Denial of Service for the Protocol and Users interacting with a closure as any operation requiring a balance trim will revert immediately when attempting to update earning checkpoints if 100% of the currently staked value in that closure is directed towards BGT rewards (`bgtValueStaked == valueStaked`).

### Root Cause

In `Burve/src/multi/closure/Closure.sol`, the function `_trimBalance` is called by `trimBalance` and `trimAllBalances` at the beginning of most state-changing operations. Inside `_trimBalance`, the calculation to update the non-BGT earnings checkpoint (`earningsPerValueX128`) divides by `nonBgtValueStaked` (which equals `self.valueStaked - self.bgtValueStaked`). If all staked value is BGT-directed, this denominator is zero, causing an immediate revert.
- [Burve/src/multi/closure/Closure.sol:742](https://github.com/sherlock-audit/2025-04-burve/blob/main/Burve/src/multi/closure/Closure.sol#L742): `self.earningsPerValueX128[idx] += (earnings << 128) / nonBgtValueStaked;`
- (Note: The same division also occurs in `addEarnings` on [Burve/src/multi/closure/Closure.sol:699](https://github.com/sherlock-audit/2025-04-burve/blob/main/Burve/src/multi/closure/Closure.sol#L699), but the revert in `_trimBalance` will typically happen first).

### Internal Pre-conditions

1. A closure `cid` exists and has active liquidity (`valueStaked > 0`).
2. All LPs currently providing value to closure `cid` need to stake their value directing rewards to BGT, such that `bgtValueStaked` becomes equal to `valueStaked`.

### External Pre-conditions

None.

### Attack Path

1. A closure `cid` reaches the state where `valueStaked > 0` and `valueStaked == bgtValueStaked` through legitimate LP staking/unstaking actions.
2. User calls `SwapFacet.swap(...)` within closure `cid` (or attempts `ValueFacet.addSingleForValue`, `removeSingleForValue`, `stakeValue`, `unstakeValue`, etc.).
3. The called function executes `trimBalance` or `trimAllBalances` near the beginning of its logic.
4. These call `_trimBalance`.
5. Inside `_trimBalance`, the code reaches [line 742](https://github.com/sherlock-audit/2025-04-burve/blob/main/Burve/src/multi/closure/Closure.sol#L742), attempting to calculate `(earnings << 128) / nonBgtValueStaked`.
6. Since `nonBgtValueStaked` (calculated as `self.valueStaked - self.bgtValueStaked`) is zero, the division causes the transaction to revert immediately.

### Impact

The Protocol and Users cannot execute swaps or most other operations that interact with balances or staked value (like single-sided LPing or stake adjustments) within any closure where 100% of the staked value is BGT-directed. This breaks core functionality for affected closures, leading to a Denial of Service until the staking state changes (requiring someone to potentially deploy capital specifically as non-BGT).

### PoC

_No response_

### Mitigation

Add a check before the division on [line 742](https://github.com/sherlock-audit/2025-04-burve/blob/main/Burve/src/multi/closure/Closure.sol#L742) (and also [line 699](https://github.com/sherlock-audit/2025-04-burve/blob/main/Burve/src/multi/closure/Closure.sol#L699)). If the denominator `nonBgtValueStaked` (or `self.valueStaked - self.bgtValueStaked`) is zero, skip the update to `earningsPerValueX128[idx]`.