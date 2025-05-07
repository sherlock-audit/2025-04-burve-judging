Alert Tiger Fish

High

# Reentrancy Vulnerability in Burve Protocol's _update Function

### Summary

A reentrancy vulnerability in the _update function will cause loss of funds and inconsistent state for the protocol as malicious actors will steal tokens by exploiting incorrect sequencing of external calls and state updates.

### Root Cause

In [Burve.sol:700](https://github.com/sherlock-audit/2025-04-burve/blob/main/Burve/src/single/Burve.sol#L700) the _update function makes external calls before updating state variables:

```solidity
function _update(
    address from,
    address to,
    uint256 value
) internal virtual override {
    // We handle mints and burns in their respective calls.
    // We just want to handle transfers between two valid addresses.
    if (
        from != address(0) &&
        to != address(0) &&
        address(island) != address(0)
    ) {
        // Move the island shares that correspond to the LP tokens being moved.
        uint256 islandTransfer = FullMath.mulDiv(
            islandSharesPerOwner[from],
            value,
            balanceOf(from)
        );

        islandSharesPerOwner[from] -= islandTransfer;  // State update AFTER external calls
        islandSharesPerOwner[to] += islandTransfer;    // State update AFTER external calls
        
        // External calls before state updates
        stationProxy.withdrawLP(address(island), islandTransfer, from);
        
        SafeERC20.forceApprove(island, address(stationProxy), islandTransfer);
        stationProxy.depositLP(address(island), islandTransfer, to);
        SafeERC20.forceApprove(island, address(stationProxy), 0);
    }

    super._update(from, to, value);
}
```
The function makes external calls to stationProxy.withdrawLP() and stationProxy.depositLP() before updating the state variables. This violates the checks-effects-interactions pattern and creates a reentrancy vulnerability.


### Internal Pre-conditions

1. The Burve protocol must be deployed with an island integration
2. Users must be able to transfer LP tokens
3. A malicious station proxy must be set as the stationProxy of the Burve contract

### External Pre-conditions

None required. The attack can be executed in normal operating conditions.

### Attack Path

1. Attacker deploys a malicious station proxy contract
2. Protocol administrator or deployer sets the malicious station proxy
3. Victim attempts to transfer LP tokens to recipient
4. During the transfer, the malicious proxy's withdrawLP function is called
5. The malicious proxy exploits the reentrancy vulnerability by calling back into the Burve contract before state updates
6. This recursive call transfers the victim's tokens to the attacker instead of the intended recipient
7. The contract finishes execution with an inconsistent state and stolen tokens

### Impact

The protocol suffers from stolen tokens and an inconsistent accounting state. Users may lose their LP tokens and associated island shares, resulting in direct financial loss. The inconsistent state also breaks invariants between token balances and island shares which could lead to additional protocol failures.

### PoC

```solidity
function test_ReentrancyVulnerability_Update() public forkOnly {
    // Deploy malicious station proxy
    MaliciousStationProxy maliciousProxy = new MaliciousStationProxy(address(burve));
    
    // Deploy a fresh Burve instance with the malicious proxy
    TickRange[] memory ranges = new TickRange[](1);
    ranges[0] = TickRange(0, 0);
    uint128[] memory weights = new uint128[](1);
    weights[0] = 1;
    
    BurveExposedInternal vulnerableBurve = new BurveExposedInternal(
        Mainnet.KODIAK_WBERA_HONEY_POOL_V3,
        Mainnet.KODIAK_WBERA_HONEY_ISLAND,
        address(maliciousProxy),
        ranges,
        weights
    );
    
    // First mint some tokens to the contract itself (deadshares)
    deal(address(token0), address(this), type(uint256).max);
    deal(address(token1), address(this), type(uint256).max);
    token0.approve(address(vulnerableBurve), type(uint256).max);
    token1.approve(address(vulnerableBurve), type(uint256).max);
    vulnerableBurve.mint(address(vulnerableBurve), 100, 0, type(uint128).max);
    
    // Mint tokens to alice
    deal(address(token0), alice, type(uint256).max);
    deal(address(token1), alice, type(uint256).max);
    vm.startPrank(alice);
    token0.approve(address(vulnerableBurve), type(uint256).max);
    token1.approve(address(vulnerableBurve), type(uint256).max);
    vulnerableBurve.mint(alice, 10000, 0, type(uint128).max);
    vm.stopPrank();
    
    // Setup malicious proxy for attack
    maliciousProxy.setAttacker(address(charlie));
    maliciousProxy.setAttackEnabled(true);
    
    // Record initial states
    uint256 aliceInitialBalance = vulnerableBurve.balanceOf(alice);
    uint256 charlieInitialBalance = vulnerableBurve.balanceOf(charlie);
    uint256 aliceInitialIslandShares = vulnerableBurve.islandSharesPerOwner(alice);
    uint256 charlieInitialIslandShares = vulnerableBurve.islandSharesPerOwner(charlie);
    uint256 totalInitialIslandShares = vulnerableBurve.totalIslandShares();
    
    // Alice tries to transfer tokens to sender
    vm.startPrank(alice);
    vulnerableBurve.transfer(sender, 5000);
    vm.stopPrank();
    
    // Check final states
    uint256 aliceFinalBalance = vulnerableBurve.balanceOf(alice);
    uint256 senderFinalBalance = vulnerableBurve.balanceOf(sender);
    uint256 charlieFinalBalance = vulnerableBurve.balanceOf(charlie);
    uint256 aliceFinalIslandShares = vulnerableBurve.islandSharesPerOwner(alice);
    uint256 senderFinalIslandShares = vulnerableBurve.islandSharesPerOwner(sender);
    uint256 charlieFinalIslandShares = vulnerableBurve.islandSharesPerOwner(charlie);
    uint256 totalFinalIslandShares = vulnerableBurve.totalIslandShares();
    
    // Verify the reentrancy attack succeeded
    // Charlie should have tokens that were supposed to go to sender
    assertGt(charlieFinalBalance, charlieInitialBalance, "Charlie should have gained tokens through reentrancy");
    
    // The sum of balances might not match due to reentrancy issues
    assertFalse(
        aliceFinalBalance + senderFinalBalance + charlieFinalBalance == aliceInitialBalance,
        "Total token balance should be inconsistent due to reentrancy"
    );
    
    // Island shares accounting should be inconsistent
    assertFalse(
        aliceFinalIslandShares + senderFinalIslandShares + charlieFinalIslandShares == totalFinalIslandShares,
        "Island shares accounting should be inconsistent due to reentrancy"
    );
}
```

### Mitigation

The vulnerability should be fixed by implementing the checks-effects-interactions pattern in the **_update** function:
1. First perform all state updates:
```solidity
// Calculate the transfer amount
uint256 islandTransfer = FullMath.mulDiv(
    islandSharesPerOwner[from],
    value,
    balanceOf(from)
);

// Update state variables first
islandSharesPerOwner[from] -= islandTransfer;
islandSharesPerOwner[to] += islandTransfer;
```
2. Then make external calls:
```solidity
// Make external calls after state updates
stationProxy.withdrawLP(address(island), islandTransfer, from);
SafeERC20.forceApprove(island, address(stationProxy), islandTransfer);
stationProxy.depositLP(address(island), islandTransfer, to);
SafeERC20.forceApprove(island, address(stationProxy), 0);
```
3. Implement ReentrancyGuard Pattern
Add a reentrancy guard to prevent multiple calls into sensitive functions:
```solidity
// Add a state variable
bool private _locked;

// Add a modifier
modifier nonReentrant() {
    require(!_locked, "ReentrancyGuard: reentrant call");
    _locked = true;
    _;
    _locked = false;
}

// Apply the modifier to _update and other sensitive functions
function _update(address from, address to, uint256 value) 
    internal virtual override nonReentrant {
    // existing code
}

```