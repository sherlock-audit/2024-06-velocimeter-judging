Keen Black Dalmatian

High

# poke() may be dos

## Summary
Poke() may be dos and this will cause we use the previous voting power to calculate the distribution.

## Vulnerability Detail
When we enter next voting epoch, all voting powers will be expected to revote or poke their voting position with updated voting power.  Considering that one NFT's voting power will decrease over time, the ve NFT owner does not have the incentive to revote if they don't want to change the voted pool. And the pool reward will be distributed with the previous `weights[_pool]`. In order to avoid this case, the governor can trigger poke() function to revote for the NFT owner with the same ratio of last epoch's vote.
The vulnerability is that the poke() function can be dos to prevent the revoting.
In `poke()`, we will calculate each pool's voting weight via `_poolWeight = _weights[i] * _weight / _totalVoteWeight` and add different voting weight to different pool. `poke()` does not allow one pool's voting weight is zero. If we can make one pool's weight to 0 in poke(), we can let poke() reverted to avoid the revote in the new epoch.
This is possible. The attack vector is like as below:
- When we first vote, we vote for several pools with different weight, we need to make sure `_poolWeight` for one pool is 1.
- When it comes to the next epoch, the governor want to poke this NFT, the contract will calculate the pool's weight via `uint256 _poolWeight = _weights[i] * _weight / _totalVoteWeight;`. And the `_weight` equals `IVotingEscrow(_ve).balanceOfNFT(_tokenId)`. The `_weight` in this epoch will be decreased compared with last epoch's voting power. The `_poolWeight` is probably round down to zero. Then the poke() will be reverted.


```javascript
    function _updateFor(address _gauge) internal {
        address _pool = poolForGauge[_gauge];
        uint256 _supplied = weights[_pool];
        if (_supplied > 0) {
            uint _supplyIndex = supplyIndex[_gauge];
            uint _index = index; // get global index0 for accumulated distro
            supplyIndex[_gauge] = _index; // update _gauge current position to global position
            uint _delta = _index - _supplyIndex; // see if there is any difference that need to be accrued
            if (_delta > 0) {
                uint _share = uint(_supplied) * _delta / 1e18; // add accrued difference for each supplied token
                if (isAlive[_gauge]) {
                    claimable[_gauge] += _share;
                }
            }
        } else {
            supplyIndex[_gauge] = index; // new users are set to the default global state
        }
    }
```
```javascript
    function poke(uint _tokenId) external onlyNewEpoch(_tokenId) {
        require(IVotingEscrow(_ve).isApprovedOrOwner(msg.sender, _tokenId) || msg.sender == governor);
        ......
        _vote(_tokenId, _poolVote, _weights);
    }
    function _vote(uint _tokenId, address[] memory _poolVote, uint256[] memory _weights) internal {
       ......
        uint256 _weight = IVotingEscrow(_ve).balanceOfNFT(_tokenId);
        uint256 _totalVoteWeight = 0;
        uint256 _totalWeight = 0;
        uint256 _usedWeight = 0;
        console.log("Current NFT Balance is :", _weight);
        for (uint i = 0; i < _poolCnt; i++) {
            _totalVoteWeight += _weights[i];
        }
        // pool cannot repeated.
        for (uint i = 0; i < _poolCnt; i++) {
            // One pool, one gauge
            address _pool = _poolVote[i];
            address _gauge = gauges[_pool];

            if (isGauge[_gauge])
            {
                // Cannot vote for one paused or killed gauge
                require(isAlive[_gauge], "gauge already dead");
                uint256 _poolWeight = _weights[i] * _weight / _totalVoteWeight;
                require(votes[_tokenId][_pool] == 0);
 @=>       require(_poolWeight != 0, 'Pool weight is zero');
            ......
```

### Poc
Add this test case into VeloVoting.t.sol, change two pool's weight ratio to make sure one pool's actual voting weight is 1. When we comes to the next epoch, the test case will be reverted.
```javascript
    function testCannotChangeVoteAndPokeAndResetInSameEpoch() public {
        address pair = router.pairFor(address(FRAX), address(FLOW), false);
        address pair1 = router.pairFor(address(FRAX), address(DAI), true);
        // vote
        vm.warp(block.timestamp + 1 weeks);
        address[] memory pools = new address[](2);
        pools[0] = address(pair);
        pools[1] = address(pair1);
        uint256[] memory weights = new uint256[](2);
        weights[0] = 1;
        weights[1] = 900000000000000000;
        voter.vote(1, pools, weights);

        // fwd half epoch
        vm.warp(block.timestamp + 1 weeks);

        // try voting again and fail
        //pools[0] = address(pair2);
        //vm.expectRevert(abi.encodePacked("TOKEN_ALREADY_VOTED_THIS_EPOCH"));
        //voter.vote(1, pools, weights);

        // try poking and fail
        //vm.expectRevert(abi.encodePacked("TOKEN_ALREADY_VOTED_THIS_EPOCH"));
        console.log("Try to poke");
        voter.poke(1);
    }

```

## Impact
In normal case, one veNFT's voting power will decrease over time. Hackers can make use of this vulnerability to hold his veNFT's voting power and gain more rewards.

## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L249-L285

## Tool used

Manual Review

## Recommendation
We should make sure poke() can always succeed.