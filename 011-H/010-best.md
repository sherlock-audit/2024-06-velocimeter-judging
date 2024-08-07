Lone Frost Porcupine

Medium

# Calling `killGaugeTotally` will break `vote`, `poke`, and `reset` functions for users who had previously voted on that gauge

## Summary
Calling `killGaugeTotally` will break `vote`, `poke`, and `reset` functions for some users.
## Vulnerability Detail

The [`_reset`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L207-L232) function loops through all pools where votes where cast with the given `tokenId` and removes those votes by adjusting the `weights` and `votes` mappings. Additionally, it invokes the `_withdraw` function within `Bribe`. 
[Here](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L221) is where the issue arises: it retrieves the `Bribe` contract address using the `external_bribes` mapping , regardless of the state of the gauge. This mapping will return `address(0)` if the gauge has been killed via [`killGaugeTotally`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L407-L429) since it [deletes](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L423) the `gauges` mapping. The transaction ends up reverting because it uses `address(0)` to instantiate the `IBribe` interface.

Bellow is a PoC of the bug. Paste it [here](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/test/KillGauges.t.sol):
```solidity

  function test_ElCid_KillGaugesTotally_DOS_reset() public {
      address gaugeAddress = address(gauge);
      address Bob = makeAddr("Bob");

      //@elcid Bob locks some tokens and receives NFT
      deal(address(flowDaiPair), Bob, 100 * TOKEN_1);
      vm.startPrank(Bob);
      flowDaiPair.approve(address(escrow), 100 * TOKEN_1);
      escrow.create_lock(100 * TOKEN_1, FIFTY_TWO_WEEKS);
      vm.roll(block.number + 1); 
      assertEq(Bob, escrow.ownerOf(2)); //Bob is the owner of tokenId 2

      //@elcid Bob votes in some gauge
      vm.warp(block.timestamp + 1 weeks); // fwd one epoch
      address[] memory pools = new address[](1);
      pools[0] = address(pair);
      uint256[] memory weights = new uint256[](1);
      weights[0] = 10000;      
      voter.vote(2, pools, weights);
      vm.stopPrank();

      //@elcid gauge is killed
      voter.killGaugeTotally(gaugeAddress);

      //@elcid Bob tries to vote, reset or poke
      vm.warp(block.timestamp + 1 weeks); // fwd another epoch
      vm.startPrank(Bob);
      vm.expectRevert();
      voter.reset(2);
      address[] memory pools2 = new address[](1);
      pools2[0] = address(pair2);
      uint256[] memory weights2 = new uint256[](1);
      weights2[0] = 10000;      
      vm.expectRevert();
      voter.vote(2, pools2, weights2);
      vm.expectRevert();
      voter.poke(2);
      vm.stopPrank();
  }      
  ```
  


## Impact

If the `emergencyCouncil` or an approved user calls `killGaugeTotally`, the `_reset` function will revert for all users whose last vote was for the killed gauge. i.e. for all users who have the corresponding `pool` present in the array returned by `poolVote[tokenId]` mapping. This effectively makes `vote`, `poke` and `reset` functions unusable since they all interact with `_reset`. 
The affected users won't be able to vote until the gauge goes live again through `createGauge`.

## Code Snippet
```solidity 
    function killGaugeTotally(address _gauge) external {
        if (msg.sender != emergencyCouncil) {
            require(
                IGaugePlugin(gaugePlugin).checkGaugeKillAllowance(msg.sender, _gauge)
            , "Restart gauge not allowed");
        }
        require(isAlive[_gauge], "gauge already dead");

        address _pool = poolForGauge[_gauge];

        delete poolForGauge[_gauge];
        delete isGauge[_gauge];
        delete claimable[_gauge];
        delete supplyIndex[_gauge];
        delete gauges[_pool]; //@audit

        (…)

    }

```
```solidity
    function _reset(uint _tokenId) internal {
        address[] storage _poolVote = poolVote[_tokenId];
        uint _poolVoteCnt = _poolVote.length;
        uint256 _totalWeight = 0;

        for (uint i = 0; i < _poolVoteCnt; i++) {
            address _pool = _poolVote[i];
            uint256 _votes = votes[_tokenId][_pool];

            if (_votes != 0) { 
                _updateFor(gauges[_pool]);
                weights[_pool] -= _votes;
                votes[_tokenId][_pool] -= _votes;
                if (_votes > 0) {
                    IBribe(external_bribes[gauges[_pool]])._withdraw(uint256(_votes), _tokenId); //@audit
                    
        (…)

    }
```

## Tool used

Manual Review
Foundry

## Recommendation
Consider checking if `isGauge` is `true` inside the loop in `_reset` before calling `IBribe._withdraw`. This way the contract can ensure that the `external_bribes` value exists for that gauge.
```solidity
          if (_votes > 0 && isGauge[gauges[pool]] {
              IBribe(external_bribes[gauges[_pool]])._withdraw(uint256(_votes), _tokenId);
              _totalWeight += _votes;
          }
```