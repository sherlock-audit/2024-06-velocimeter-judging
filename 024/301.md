Gorgeous Snowy Skunk

Medium

# `Voter.replaceFactory()` and `Voter.addFactory()` functions are broken.

## Summary

The `Voter.replaceFactory()` and `Voter.addFactory()` functions are broken due to invalid validation.

## Vulnerability Detail

1. In the `addFactory()` function, the line `require(!isFactory[_pairFactory], 'factory true');` is missing.
2. In the `replaceFactory()` function, the `isFactory` and `isGaugeFactory` checks are incorrect:

```solidity
        require(isFactory[_pairFactory], 'factory false'); // <=== should be !isFactory
        require(isGaugeFactory[_gaugeFactory], 'g.fact false'); // <=== should be !isGaugeFactory
```

These issues lead to the invariant being broken, allowing multiple instances of a factory or gauge to be pushed to the `factories` and `gaugeFactories` arrays.

## Impact

Broken code. DoS when calling `Voter.createGauge()`.

## Code Snippet

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L155-L185

## Tool used

Manual Review

## Recommendation

1. Add the `require(!isFactory[_pairFactory], 'factory true');` validation to the `addFactory()` function.
2. Fix the checks in the `replaceFactory()` function:

```diff
-        require(isFactory[_pairFactory], 'factory false');
+        require(!isFactory[_pairFactory], 'factory true');
-        require(isGaugeFactory[_gaugeFactory], 'g.fact false');
+        require(!isGaugeFactory[_gaugeFactory], 'g.fact true');
```
