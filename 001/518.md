Square Arctic Chicken

Medium

# `Voter._vote(...)` does not apply all the available voting power during voting due to rounding error

## Summary
When voting, the `_vote(...)` function is used to allocate weights to pools, however, due to rounding error all the user's available voting power is not not used up. This can put some voters at a disadvantage leading to skewed voting.

## Vulnerability Detail
When voting, weights for each pool are allocated proportionally to the current voting power of the user's `tokenId` as shown on L252 and L267 below.


```solidity
File: Voter.sol
249:     function _vote(uint _tokenId, address[] memory _poolVote, uint256[] memory _weights) internal {
250:         _reset(_tokenId);
251:         uint _poolCnt = _poolVote.length;
252:         uint256 _weight = IVotingEscrow(_ve).balanceOfNFT(_tokenId); // total weight of _tokenId in ve
......
256: 
257:         for (uint i = 0; i < _poolCnt; i++) {
258:   >>>       _totalVoteWeight += _weights[i];
259:         }
260: 
261:         for (uint i = 0; i < _poolCnt; i++) {
.....
265:             if (isGauge[_gauge]) {
266:                 require(isAlive[_gauge], "gauge already dead");
267:       >>>       uint256 _poolWeight = _weights[i] * _weight / _totalVoteWeight; // @audit can round down to deflate 
.....
274:       >>>        weights[_pool] += _poolWeight;
...
278:                 _usedWeight += _poolWeight;
278:     >>>         _totalWeight += _poolWeight;


```

However, due to solidity rounding direction (down) the calculation of the `_poolWeight` of each gauge's pool can be rounded down leaving some of the users current voting power unused. In addition to the rounding issue, the `vote(...)` function does not check if the `_totalWeight` used is equal to the user's total voting power.

Consider for example
- totalPower = 500 
- weight[1] = 10 weight[2] = 50 weight[3] = 75
- total weight used
 ```solidity
 poolWeight1 = (10 * 500)/135 = 37
 poolWeight2 = (50 * 500)/135 = 185 
 poolWeight3 = (75 * 500)/135 = 277
_usedWeight = 37 + 155 + 277 =  499 
```

## Impact
Voters may not be able to use up their total voting power when they vote on multiple pools due to rounding error

## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L249-L285

## Tool used

Manual Review

## Recommendation
When a user votes on multiple pools, the vote allocated to the last of his chosen pools should be the difference between the used vote and the total initial vote.
Also add a check to ensure that the total used weight ie equal to the available voting power of the tokenId