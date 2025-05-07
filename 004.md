Prehistoric Jade Albatross

High

# User can drain value from Protocol/LPs by receiving excess shares during swaps into appreciating ERC4626 vaults

### Summary

Incorrect unit handling in `SwapFacet` when transferring output tokens using `E4626ViewAdjustor` will cause a loss of funds for the Protocol/LPs as a User will swap underlying assets into an appreciating ERC4626 vault, receiving more shares than calculated because the function transfers shares based on the underlying asset value instead of the nominal share amount.

### Root Cause

In `Burve/src/multi/facets/SwapFacet.sol`, the calculated real underlying value (`outAmount`) derived from `AdjustorLib.toReal` is incorrectly used as the *quantity* for internal withdrawal and external transfer of the nominal `outToken` (e.g., vault shares), instead of using the calculated nominal share amount (`nominalOut`). This occurs in the exact input swap logic (`amountSpecified > 0`):
- [Burve/src/multi/facets/SwapFacet.sol:132](https://github.com/sherlock-audit/2025-04-burve/blob/main/Burve/src/multi/facets/SwapFacet.sol#L132): `Store.vertex(outVid).withdraw(cid, outAmount, true);` - Uses `outAmount` (real value) to withdraw nominal shares.
- [Burve/src/multi/facets/SwapFacet.sol:134](https://github.com/sherlock-audit/2025-04-burve/blob/main/Burve/src/multi/facets/SwapFacet.sol#L134): `TransferHelper.safeTransfer(outToken, recipient, outAmount);` - Uses `outAmount` (real value) as the quantity of shares to transfer.

### Internal Pre-conditions

1.  Admin needs to register an ERC4626 vault token (`vaultToken`) and its underlying `assetToken`.
2.  Admin needs to configure Burve's adjustor system to set `E4626ViewAdjustor` (linked to `assetToken`) as the adjustor for `vaultToken`.
3.  Admin needs to create a closure (`cid`) containing vertices for both `assetToken` and `vaultToken`.
4.  Protocol/LPs need to deposit liquidity for both `assetToken` and `vaultToken` into the closure `cid`.
5.  The internal state of the vaultToken contract needs to reflect appreciation such that `vaultToken.convertToAssets(1e18 shares)` returns a value greater than `1e18` of `assetToken` (assuming 18 decimals for both).

### External Pre-conditions

1. The specific ERC4626 `vaultToken` outside contract must increase its underlying asset holdings relative to its total share supply. This happens through normal vault operations like accumulating yield or receiving direct asset transfers, causing its share price (1 share = X assetToken) to become greater than 1.

### Attack Path

1.  Attacker calls `assetToken.approve(address(burveDiamond), swapInAmountReal)` to allow Burve to spend their `assetToken`.
2.  Attacker calls `SwapFacet.swap(recipient=attacker, inToken=assetToken, outToken=vaultToken, amountSpecified=swapInAmountReal, amountLimit=0, _cid=cid)`, where `swapInAmountReal` is a positive integer representing the amount of `assetToken` to swap.
3.  SwapFacet internally calls the closure's swap logic which calculates the correct `nominalOut` amount of `vaultToken` shares the attacker should receive based on pool state and fees.
4. SwapFacet then calls `outAmount = AdjustorLib.toReal(vaultToken_idx, nominalOut, false)`. This invokes `E4626ViewAdjustor`, which returns the *value* of `nominalOut` shares in terms of `assetToken`. Because `vaultToken` has appreciated, this `outAmount` value will be numerically larger than `nominalOut`.
5.  SwapFacet incorrectly calls `Store.vertex(outVid).withdraw(cid, outAmount, true)`, attempting to withdraw the `outAmount` quantity (real value) from the vertex which stores shares.
6.  SwapFacet incorrectly calls `TransferHelper.safeTransfer(vaultToken, attacker, outAmount)`, transferring `outAmount` quantity of `vaultToken` shares to the attacker.
7.  Attacker successfully receives `outAmount` shares, which is more than the `nominalOut` shares they were entitled to based on the swap calculation.
8.  Attacker repeats step 2 onwards to drain more excess shares from the pool.

### Impact

The Protocol/Liquidity Providers in the affected closure suffer a direct loss of `vaultToken` shares each time an attacker performs this swap into the appreciated vault. The loss per swap is `outAmount - nominalOut` shares. Since the attack is repeatable, this can lead to a significant (>1%) drain of the pooled `vaultToken` shares over time, transferring value from LPs to the attacker. The attacker gains the excess `vaultToken` shares.

#### Example calculation

1. Assume 1 `vaultToken` share = 1.1 `assetToken`. 
2. Attacker swaps 1 `assetToken` (`swapInAmountReal = 1e18`).
3. Ignoring fees/slippage for clarity, the swap should yield `nominalOut = 1 / 1.1 = 0.90909...e18` shares.
4. The code calculates `outAmount = AdjustorLib.toReal(vaultToken_idx, nominalOut, false)`, which results in `vaultToken.convertToAssets(0.90909...e18 shares) = 1e18` assets.
5. The code then transfers `outAmount` (1e18) quantity of `vaultToken` shares to the attacker.
6. **Loss per swap:** Attacker receives `1e18` shares but should have received `0.90909...e18` shares. The loss for LPs is `1e18 - 0.90909...e18 = 0.09090...e18` shares per 1 `assetToken` swapped.
7. This represents `0.09090...e18 shares * 1.1 assetToken/share = 0.1 assetToken` of value drained from the LPs per 1 `assetToken` swapped in this example.

### PoC

_No response_

### Mitigation

Modify `SwapFacet.sol` in the exact input (`amountSpecified > 0`) flow. After calculating `nominalOut`, use `nominalOut` (potentially denormalized if `vaultToken` decimals != 18) as the amount for the `Store.vertex(...).withdraw(...)` call and the `TransferHelper.safeTransfer(...)` call. The `outAmount` calculated via `AdjustorLib.toReal` should only be used for comparison against the `amountLimit` slippage parameter.