Slow Steel Seahorse

High

# User will lose all their unclaimed rewards once their lock expires

## Summary
User will lose all their unclaimed rewards once their lock expires

## Vulnerability Detail
Let's look at the code of `_claim` within `RewardsDistributorV2`

```solidity
        for (uint i = 0; i < 50; i++) {
            if (week_cursor >= _last_token_time) break;

            if (week_cursor >= user_point.ts && user_epoch <= max_user_epoch) {
                user_epoch += 1;
                old_user_point = user_point;
                if (user_epoch > max_user_epoch) {
                    user_point = IVotingEscrow.Point(0,0,0,0);
                } else {
                    user_point = IVotingEscrow(ve).user_point_history(_tokenId, user_epoch);
                }
            } else {
                int128 dt = int128(int256(week_cursor - old_user_point.ts));
                uint balance_of = Math.max(uint(int256(old_user_point.bias - dt * old_user_point.slope)), 0);
                if (balance_of == 0 && user_epoch > max_user_epoch) break;
                if (balance_of != 0) {
                    to_distribute += balance_of * tokens_per_week[week_cursor] / ve_supply[week_cursor];
                }
                week_cursor += WEEK;
            }
        }
```

As we can see, it gets the latest point prior to the week to check against and based on the slope and bias calculates the reward. However, let's look at the case where the user's lock would have expired at the checkpoint we're looking against.
```solidity
                int128 dt = int128(int256(week_cursor - old_user_point.ts));
                uint balance_of = Math.max(uint(int256(old_user_point.bias - dt * old_user_point.slope)), 0);
```

As the lock would have expired, `old_user_point.bias - dt * old_user_point.slope` would return value less than 0. Keep in mind that the value is first wrapped in uint256 and then `Math.max` is performed.

So in the situation where the calculated bias would be `-1e18`, it would first be turned into `uint256.max - 1e18` and then the higher of it and 0 would be assigned to `balance_of`, which would basically mean the user's balance will be just a little under `uint256.max`

Which would then on the next line overflow and revert.

```solidity
                if (balance_of != 0) {
                    to_distribute += balance_of * tokens_per_week[week_cursor] / ve_supply[week_cursor];
                }
```

Meaning that if a user has had unclaimed rewards up to the point where their lock has expired, they'll remain permanently locked.

Note: same issue appears on 4 instances within the contract, this is simply the highest impact that could be done. 

## Impact
Loss of funds

## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/RewardsDistributorV2.sol#L265

## Tool used

Manual Review

## Recommendation
First check which is the higher number and then turn it into `uint256`