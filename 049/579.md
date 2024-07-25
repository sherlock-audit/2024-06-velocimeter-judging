Amateur Chili Tarantula

Medium

# `Voter.createGauge` will always revert for non-Velocimeter pairs

## Summary

The `Voter.createGauge` function in the Velocimeter protocol is intended to allow the governor or emergency council to create gauges for any pool, including non-Velocimeter pairs. However, the current implementation will almost always revert for non-Velocimeter pools because it assumes the presence of a `externalBribe()` getter, which may not exist for non-Velocimeter pairs.

## Vulnerability Detail

The function `Voter.createGauge` allows any user to create a gauge for a pool if certain conditions are met. However, it allows the governor and emergency council to create gauges for any pool, including non-Velocimeter pools. The problem arises because the function tries to access the `externalBribe()` property of the pool, which may not exist for non-Velocimeter pairs. This will cause the transaction to revert.

```solidity
File: v4-contracts/contracts/Voter.sol
349:         if (msg.sender != governor && msg.sender != emergencyCouncil) { // gov can create for any pool, even non-Velocimeter pairs
350:             require(isPair, "!_pool");
351:             require(IGaugePlugin(gaugePlugin).checkGaugeCreationAllowance(msg.sender, tokenA, tokenB), "!whitelistedForGaugeCreation");
352:         }
353: 
354:         address _external_bribe = IPair(_pool).externalBribe(); //audit
355:         
356:         if(_external_bribe == address(0)) {
357:           _external_bribe = IBribeFactory(bribefactory).createExternalBribe(allowedRewards); 
358:         }
```

In the above code, the `externalBribe()` method is called on `_pool`. For non-Velocimeter pools, this method might not exist, causing the transaction to revert.

## Impact

The `createGauge` function is a core functionality in the Velocimeter v4 update, allowing the creation of gauges for pools. The issue renders this function unusable for non-Velocimeter pools by the governor or emergency council, contrary to its intended purpose. 

## Code Snippet

[Voter.sol](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/Voter.sol#L354-L354)

## Tool used

Manual Review

## Recommendation

To fix this issue, the access to the `externalBribe()` method should be wrapped in a try-catch block, similar to other uncertain function calls in the system. This way, if the method does not exist, it will not cause the transaction to revert, and the function can handle the error appropriately.

Replace the current direct call to `externalBribe()` with a try-catch block:

```solidity
File: v4-contracts/contracts/Voter.sol
349:         if (msg.sender != governor && msg.sender != emergencyCouncil) { // gov can create for any pool, even non-Velocimeter pairs
350:             require(isPair, "!_pool");
351:             require(IGaugePlugin(gaugePlugin).checkGaugeCreationAllowance(msg.sender, tokenA, tokenB), "!whitelistedForGaugeCreation");
352:         }
353: 
354:         address _external_bribe;
355:         try IPair(_pool).externalBribe() returns (address bribeAddress) {
356:             _external_bribe = bribeAddress;
357:         } catch {
358:             _external_bribe = address(0);
359:         }
360:         
361:         if(_external_bribe == address(0)) {
362:           _external_bribe = IBribeFactory(bribefactory).createExternalBribe(allowedRewards); 
363:         }
```

This change ensures that the `externalBribe()` method is safely accessed, and in the case of a non-existent method, it gracefully handles the error without causing the transaction to revert.

#### Existing implementation example:

```solidity
File: v4-contracts/contracts/Voter.sol
  390:         try IPair(_pair).setHasGauge(false) {} catch {}
```