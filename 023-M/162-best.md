Odd Carmine Cougar

High

# Balance of NFTs and totalSupply of the VotingEscrow can be obtained wrong.

## Summary
`VotingEscrow._checkpoint()` increases the epoch even in the same block number and timestamp.
This causes the problem that balance of NFTs and totalSupply of the VotingEscrow can be obtained wrong.

## Vulnerability Detail
`VotingEscrow._checkpoint()` always increases the epoch of `point_history`. If `_checkpoint()` is called several times in the same block, the epoch of `point_history` is increased by the number of times `_checkpoint()` is called. That is, `point_history` might have several `Point`s with the same block number and timestamp. Since `point_history` is used to get the balance of NFTs and the totalSupply for voting, it will cause serious problems for the protocol.

*** Proof of Concept ***
The following is the test code for `VotingEscrow.t.sol`.
```solidity
    function test_point_history() public 
    {
        flowDaiPair.approve(address(escrow), TOKEN_10);
        uint256 lockDuration = ONE_WEEK;

        uint tokenId = escrow.create_lock(TOKEN_1, lockDuration);

        escrow.deposit_for(tokenId, TOKEN_1);

        vm.warp(block.timestamp + 100);
        vm.roll(block.number + 10);

        escrow.deposit_for(tokenId, TOKEN_1);

        uint epoch = escrow.epoch();
        console.log("epoch = ", epoch);
        for (uint i = 1; i <= epoch; i++) {
            (int128 bias, int128 slope, uint ts, uint blk) = escrow.point_history(i);
            console.log("at:", i);
            console.log("--bias:", uint(uint128(bias)));
            console.log("--slope:", uint(uint128(slope)));
            console.log("--ts:", ts);
            console.log("--blk:", blk);
        }
    }
```
The following is the test log.
```log
  epoch =  3
  at: 1
  --bias: 19230737433314004
  --slope: 31796906796
  --ts: 1
  --blk: 1
  at: 2
  --bias: 38461474867232807
  --slope: 63593813593
  --ts: 1
  --blk: 1
  at: 3
  --bias: 57682673229112610
  --slope: 95390720390
  --ts: 101
  --blk: 11
```
As shown above, the `point_history[1]` and `point_history[2]` are at the same block number `1` and timestamp `1`.

## Impact
Since `VotingEscrow._checkpoint()` increases the epoch even in the same block number and timestamp, the balance of NFTs and totalSupply of the VotingEscrow can be obtained wrong.
It will cause serious problems for the protocol.

## Code Snippet
- [v4-contracts/contracts/VotingEscrow.sol#L598-L733](https://github.com/sherlock-audit/2024-06-velocimeter/tree/main/v4-contracts/contracts/VotingEscrow.sol#L598-L733)

## Tool used
Manual Review

## Recommendation
To fix the issue of the `VotingEscrow._checkpoint()` incrementing the epoch even when called multiple times within the same block, you can check if the last point in the `point_history` has the same block number and timestamp as the current block and timestamp. If they match, avoid incrementing the epoch and instead update the existing point.

Here’s a proposed solution:
```diff
    function _checkpoint(
        uint _tokenId,
        LockedBalance memory old_locked,
        LockedBalance memory new_locked
    ) internal {
        Point memory u_old;
        Point memory u_new;
        int128 old_dslope = 0;
        int128 new_dslope = 0;
        uint _epoch = epoch;

        if (_tokenId != 0) {
            // Calculate slopes and biases
            // Kept at zero when they have to
            if (old_locked.end > block.timestamp && old_locked.amount > 0) {
                u_old.slope = old_locked.amount / iMAXTIME;
                u_old.bias = u_old.slope * int128(int256(old_locked.end - block.timestamp));
            }
            if (new_locked.end > block.timestamp && new_locked.amount > 0) {
                u_new.slope = new_locked.amount / iMAXTIME;
                u_new.bias = u_new.slope * int128(int256(new_locked.end - block.timestamp));
            }

            // Read values of scheduled changes in the slope
            // old_locked.end can be in the past and in the future
            // new_locked.end can ONLY by in the FUTURE unless everything expired: than zeros
            old_dslope = slope_changes[old_locked.end];
            if (new_locked.end != 0) {
                if (new_locked.end == old_locked.end) {
                    new_dslope = old_dslope;
                } else {
                    new_dslope = slope_changes[new_locked.end];
                }
            }
        }

        Point memory last_point = Point({bias: 0, slope: 0, ts: block.timestamp, blk: block.number});
        if (_epoch > 0) {
            last_point = point_history[_epoch];
        }
-       uint last_checkpoint = last_point.ts;
+       uint last_ts = last_point.ts;
        // initial_last_point is used for extrapolation to calculate block number
        // (approximately, for *At methods) and save them
        // as we cannot figure that out exactly from inside the contract
        Point memory initial_last_point = last_point;
        uint block_slope = 0; // dblock/dt
        if (block.timestamp > last_point.ts) {
            block_slope = (MULTIPLIER * (block.number - last_point.blk)) / (block.timestamp - last_point.ts);
        }
        // If last point is already recorded in this block, slope=0
        // But that's ok b/c we know the block in such case

        // Go over weeks to fill history and calculate what the current point is
        {
            uint t_i = (last_checkpoint / WEEK) * WEEK;
            for (uint i = 0; i < 255; ++i) {
                // Hopefully it won't happen that this won't get used in 5 years!
                // If it does, users will be able to withdraw but vote weight will be broken
                t_i += WEEK;
                int128 d_slope = 0;
                if (t_i > block.timestamp) {
                    t_i = block.timestamp;
                } else {
                    d_slope = slope_changes[t_i];
                }
                last_point.bias -= last_point.slope * int128(int256(t_i - last_checkpoint));
                last_point.slope += d_slope;
                if (last_point.bias < 0) {
                    // This can happen
                    last_point.bias = 0;
                }
                if (last_point.slope < 0) {
                    // This cannot happen - just in case
                    last_point.slope = 0;
                }
                last_checkpoint = t_i;
                last_point.ts = t_i;
                last_point.blk = initial_last_point.blk + (block_slope * (t_i - initial_last_point.ts)) / MULTIPLIER;
                _epoch += 1;
                if (t_i == block.timestamp) {
                    last_point.blk = block.number;
                    break;
                } else {
                    point_history[_epoch] = last_point;
                }
            }
        }

+       if (epoch > 0 && block.timestamp == last_ts) _epoch -= 1;
        epoch = _epoch;
        // Now point_history is filled until t=now

        if (_tokenId != 0) {
            // If last point was in this block, the slope change has been applied already
            // But in such case we have 0 slope(s)
            last_point.slope += (u_new.slope - u_old.slope);
            last_point.bias += (u_new.bias - u_old.bias);
            if (last_point.slope < 0) {
                last_point.slope = 0;
            }
            if (last_point.bias < 0) {
                last_point.bias = 0;
            }
        }

        // Record the changed point into history
        point_history[_epoch] = last_point;

        if (_tokenId != 0) {
            // Schedule the slope changes (slope is going down)
            // We subtract new_user_slope from [new_locked.end]
            // and add old_user_slope to [old_locked.end]
            if (old_locked.end > block.timestamp) {
                // old_dslope was <something> - u_old.slope, so we cancel that
                old_dslope += u_old.slope;
                if (new_locked.end == old_locked.end) {
                    old_dslope -= u_new.slope; // It was a new deposit, not extension
                }
                slope_changes[old_locked.end] = old_dslope;
            }

            if (new_locked.end > block.timestamp) {
                if (new_locked.end > old_locked.end) {
                    new_dslope -= u_new.slope; // old slope disappeared at this point
                    slope_changes[new_locked.end] = new_dslope;
                }
                // else: we recorded it already in old_dslope
            }
            // Now handle user history
            uint user_epoch = user_point_epoch[_tokenId] + 1;

            user_point_epoch[_tokenId] = user_epoch;
            u_new.ts = block.timestamp;
            u_new.blk = block.number;
            user_point_history[_tokenId][user_epoch] = u_new;
        }
    }
```
