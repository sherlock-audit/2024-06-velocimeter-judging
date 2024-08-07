Furry Clear Chinchilla

High

# When a gauge is killed, its weight is not decreased

## Summary

When a gauge is killed, its weight is not decreased. This will lead to portion of the reward stuck in the contract.

## Vulnerability Detail

When a gauge is killed/paused using the `pauseGauge()` or `killGaugeTotally()` functions in the `Voter` contract, the gauge-related data is cleared out. 
However, it is possible that the killed gauge has already been voted for, and its respective vote weight has been added to the period's total vote weight:

```solidity
    function _vote(uint _tokenId, address[] memory _poolVote, uint256[] memory _weights) internal {
        _reset(_tokenId);
        uint _poolCnt = _poolVote.length;
        uint256 _weight = IVotingEscrow(_ve).balanceOfNFT(_tokenId);
        uint256 _totalVoteWeight = 0;
        uint256 _totalWeight = 0;
        uint256 _usedWeight = 0;

        for (uint i = 0; i < _poolCnt; i++) {
            _totalVoteWeight += _weights[i];
        }

        for (uint i = 0; i < _poolCnt; i++) {
            address _pool = _poolVote[i];
            address _gauge = gauges[_pool];

            if (isGauge[_gauge]) {
                require(isAlive[_gauge], "gauge already dead");
                uint256 _poolWeight = _weights[i] * _weight / _totalVoteWeight;
                require(votes[_tokenId][_pool] == 0);
                require(_poolWeight != 0);
                _updateFor(_gauge);

                poolVote[_tokenId].push(_pool);

                weights[_pool] += _poolWeight;
                votes[_tokenId][_pool] += _poolWeight;
               IBribe(external_bribes[_gauge])._deposit(uint256(_poolWeight), _tokenId);
                _usedWeight += _poolWeight;
                _totalWeight += _poolWeight;  //@audit - Killing a Gauge not update _totalWeight
                emit Voted(msg.sender, _tokenId, _poolWeight);
            }
        }
        if (_usedWeight > 0) IVotingEscrow(_ve).voting(_tokenId);
        totalWeight += uint256(_totalWeight); //@audit - Killing a Gauge not update _totalWeight
        usedWeights[_tokenId] = uint256(_usedWeight);
    }
```

Specifically, there are two problematic scenarios regarding the rewards accounting of the current and previous voting periods relative to the time a gauge is killed.

**Effects on the Current Voting Period**:
If a gauge is killed during an active voting period, it may have already received votes from users. These votes contribute to the `_totalWeight`, which is later used to compute the reward index for that period.

When the gauge is killed, its data is cleared, but its share of the rewards (based on the votes it received) remains in the contract. This portion of the reward is not redistributed to other gauges, resulting in unused rewards getting stuck in the contract.

**Effects on the Previous Voting Period**:
A gauge can be killed after a voting period has ended but before rewards have been distributed. The `_totalWeights` used to compute the reward index for that period includes the killed gauge's weight.

Since the killed gauge is no longer eligible for rewards, its share remains undistributed, and these rewards also get stuck in the contract.

## Impact

The period's total weight is not adjusted when a gauge is killed, resulting in the portion of the reward that is committed to that gauge remaining unused and getting stuck in the contract. 
## Code Snippet

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L407-L429
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L265-L285
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L490

## Tool used

Manual Review

## Recommendation

To ensure that rewards are distributed correctly, the `_totalWeights` should be updated when a gauge is killed. This involves removing the killed gauge's weight from the total weight to ensure the remaining gauges receive the correct share of rewards.