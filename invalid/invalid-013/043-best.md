Mammoth Powder Moose

Medium

# Overflow in `uint256 _poolWeight = _weights[i] * _weight / _totalVoteWeight;` if `_weights` and `_weight` is given a maximum size number

## Summary
Overflow in `uint256 _poolWeight = _weights[i] * _weight / _totalVoteWeight;` if `_weights` and `_weight` is given a maximum size number

## Vulnerability Detail
If _weights receives an array with a very large number, in a uint256.max scenario, _totalVoteWeight would also receive that value.

```javascript
function _vote(uint256 _tokenId, address[] memory _poolVote, uint256[] memory _weights) internal {
        _reset(_tokenId);
        uint256 _poolCnt = _poolVote.length;
        uint256 _weight = IVotingEscrow(_ve).balanceOfNFT(_tokenId);
        uint256 _totalVoteWeight = 0;
        uint256 _totalWeight = 0;
        uint256 _usedWeight = 0;

        for (uint256 i = 0; i < _poolCnt; i++) {
            _totalVoteWeight += _weights[i];
        }

        for (uint256 i = 0; i < _poolCnt; i++) {
            address _pool = _poolVote[i];
            address _gauge = gauges[_pool];

            if (isGauge[_gauge]) {
                require(isAlive[_gauge], "gauge already dead");
@>              uint256 _poolWeight = _weights[i] * _weight / _totalVoteWeight;
                require(votes[_tokenId][_pool] == 0);
                require(_poolWeight != 0);
                _updateFor(_gauge);

                poolVote[_tokenId].push(_pool);

                weights[_pool] += _poolWeight;
                votes[_tokenId][_pool] += _poolWeight;
                IBribe(external_bribes[_gauge])._deposit(uint256(_poolWeight), _tokenId);
                _usedWeight += _poolWeight;
                _totalWeight += _poolWeight;
                emit Voted(msg.sender, _tokenId, _poolWeight);
            }
        }
        if (_usedWeight > 0) IVotingEscrow(_ve).voting(_tokenId);
        totalWeight += uint256(_totalWeight);
        usedWeights[_tokenId] = uint256(_usedWeight);
    }
```

## Impact
In the `Voter::vote` function there is no check to verify a weight size in the `_weight` array, and it can support a maximum size uint. Causing overflow in the `_poolWeight` variable.

## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L267

## Tool used
Manual Review, Foundry

## Recommendation
Perhaps it would be better to scale to try to make these calculations
