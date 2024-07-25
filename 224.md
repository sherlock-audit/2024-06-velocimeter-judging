Slow Steel Seahorse

Medium

# Donating rewards to a gauge might actually lead to less rewards being distributed overall

## Summary
Donating rewards to a gauge might actually lead to less rewards being distributed overall

## Vulnerability Detail
When the `Voter` attempts to distribute funds to a gauge, there's a check that the `claimable` amount is more than the remaining (to-be-distributed) rewards in the Gauge.

```solidity
       uint _claimable = claimable[_gauge];
        if (_claimable > IGauge(_gauge).left(base) && _claimable / DURATION > 0) {
            claimable[_gauge] = 0;
            if((_claimable * 1e18) / currentEpochRewardAmount > minShareForActiveGauge) {
                activeGaugeNumber += 1;
            }

            IGauge(_gauge).notifyRewardAmount(base, _claimable);
```

However, the problem is that not only the rewards will not be distributed, but the gauge will not be counted as active. If the gauge is not counted as active, less tokens will be minted for next week.

This means that if a user donates enough rewards to a gauge, it might lead to the gauge being unable to be counted as active and hence less amount of tokens getting overall minted.

## Impact
Less `FLOW` tokens minted than supposed to. 

## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L553
## Tool used

Manual Review

## Recommendation
Remove the check