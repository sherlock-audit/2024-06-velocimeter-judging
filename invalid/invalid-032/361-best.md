Dandy Shamrock Sheep

Medium

# Unreachable Code in Reset Function

## Summary
There's an unreachable else branch in the _reset function due to the use of unsigned integers.

## Vulnerability Detail
In the _reset function:
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L220-L225

## Impact
* Unnecessary code that may confuse developers
*  misunderstanding the function's behavior

## Code Snippet


## Tool used

Manual Review


## Recommendation
Remove the unreachable else branch:
```solidity
if (_votes > 0) {
    IBribe(external_bribes[gauges[_pool]])._withdraw(_votes, _tokenId);
    _totalWeight += _votes;
}
```
