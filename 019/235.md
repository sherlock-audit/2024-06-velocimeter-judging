Slow Steel Seahorse

High

# If user merges their `veNFT`, they'll lose part of their rewards

## Summary
If user merges their `veNFT`, they'll lose part of their rewards

## Vulnerability Detail
When users claim rewards, they can at most claim up to the week before `last_token_time`.
```solidity
        for (uint i = 0; i < 50; i++) {
            if (week_cursor >= _last_token_time) break;
```
And given that `last_token_time` can at most be this week, this means that rewards in the `RewardsDistributor` are lagging at least a week at a time.

Then, if we look at the code of `merge` we'll see that the `from` token is actually burned.

```solidity
    function merge(uint _from, uint _to) external {
        require(attachments[_from] == 0 && !voted[_from], "attached");
        require(_from != _to);
        require(_isApprovedOrOwner(msg.sender, _from));
        require(_isApprovedOrOwner(msg.sender, _to));

        LockedBalance memory _locked0 = locked[_from];
        LockedBalance memory _locked1 = locked[_to];
        uint value0 = uint(int256(_locked0.amount));
        uint end = _locked0.end >= _locked1.end ? _locked0.end : _locked1.end;

        locked[_from] = LockedBalance(0, 0);
        _checkpoint(_from, _locked0, LockedBalance(0, 0));
        _burn(_from);
        _deposit_for(_to, value0, end, _locked1, DepositType.MERGE_TYPE);
    }
```

Since `claim` requires `msg.sender` to be approved or owner, because the token is burned, they won't be able to claim the rewards.
Any time a user merges their `veNFT`, they'll lose at least 1 week of rewards.

## Impact
Loss of funds

## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L1208

## Tool used

Manual Review

## Recommendation
Do not burn the token