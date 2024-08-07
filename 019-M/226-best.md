Delightful Lavender Goose

Medium

# Incorrect usage of `initialLastPoint` in `VotingEscrow._checkpoint` affects epoch accuracy and weakens snapshot governance.

## Summary
The `VotingEscrow` contract's `initialLastPoint` incorrectly references `lastPoint`, causing miscalculations in block number references for epochs due to shared memory semantics in Solidity.

## Vulnerability Detail
The `Voting Escrow` contract calculates voting power for governance based on users' locked tokens. It determines voting power at specific block heights, which is essential for snapshot-based voting.

```solidity
function balanceOfAtNFT(uint256 _tokenId, uint256 _block) external view returns (uint256) {
    return _balanceOfAtNFT(_tokenId, _block);
}
```

The contract keeps a history of checkpoints, using the `Point` struct to record voting power (`bias` and `slope`), time (`ts`), and block number (`blk`).

In the `_checkpoint` function, the contract uses `initialLastPoint` to store the last checkpoint's state and calculate the block number for new epochs.

```solidity
function _checkpoint(uint256 _tokenId, LockedBalance memory oldLocked, LockedBalance memory newLocked) internal {
    // prev code ...
    Point memory lastPoint = Point({ bias: 0, slope: 0, ts: block.timestamp, blk: block.number });
    if (_epoch > 0) {
        lastPoint = _pointHistory[_epoch];
    }
    uint256 lastCheckpoint = lastPoint.ts;
    // initialLastPoint is used for extrapolation to calculate block number
    // (approximately, for *At methods) and save them
    // as we cannot figure that out exactly from inside the contract
@>> Point memory initialLastPoint = lastPoint;
    uint256 blockSlope = 0; // dblock/dt
    if (block.timestamp > lastPoint.ts) {
        blockSlope = (MULTIPLIER * (block.number - lastPoint.blk)) / (block.timestamp - lastPoint.ts);
    }
    // more code ...
}
```

However, due to Solidity's reference semantics for struct types in memory, the line `Point memory initialLastPoint = lastPoint;` does not create an independent copy of `lastPoint`. Instead, `initialLastPoint` and `lastPoint` point to the same data in memory. Consequently, any modifications to `lastPoint` also affect `initialLastPoint`.

This behavior contradicts the intended use of `initialLastPoint`, which should remain unchanged to accurately calculate the block number for each epoch. Since `initialLastPoint` changes whenever `lastPoint` changes, it leads to miscalculations:

```solidity
lastPoint.ts = tI;
lastPoint.blk = initialLastPoint.blk + (blockSlope * (tI - initialLastPoint.ts)) / MULTIPLIER;
```

Here, `(tI - initialLastPoint.ts)` is supposed to be the time difference since the last checkpoint. However, due to the shared reference, it is always zero, making `tI` equal to `initialLastPoint.ts`. Consequently, `lastPoint.blk` remains the same as `initialLastPoint.blk`, causing all epochs to incorrectly reference the block number of the first checkpoint.


## Impact
The miscalculation of block numbers for epochs due to shared memory references leads to inaccurate voting power calculations. This can disrupt the governance process, as the voting power snapshot is incorrect. It undermines the integrity of the voting system and can result in governance decisions being based on faulty data.

## POC
- Chisel
<img width="726" alt="Screenshot 2024-07-20 at 11 54 35 AM" src="https://github.com/user-attachments/assets/49e46bbd-f74f-40f6-ab6c-d1b890568fe5">

- Foundry
```solidity
     function testSolidityMemoryStruct() public {
      VotingEscrow.Point memory original = VotingEscrow.Point({
        slope: 100,
        bias : 200,
        ts   : block.timestamp,
        blk: 1337
      });

      VotingEscrow.Point memory copy = original;

      assertEq(original.slope, copy.slope);
      assertEq(original.bias, copy.bias);
      assertEq(original.ts, copy.ts);
      assertEq(original.blk, copy.blk);

      original.slope = 7331;
      assertEq(original.slope, 7331);
      assertEq(original.slope, copy.slope);
     }
```
## Code Snippet
```solidity
        Point memory last_point = Point({bias: 0, slope: 0, ts: block.timestamp, blk: block.number});
        if (_epoch > 0) {
            last_point = point_history[_epoch];
        }
        uint last_checkpoint = last_point.ts;
        // initial_last_point is used for extrapolation to calculate block number
        // (approximately, for *At methods) and save them
        // as we cannot figure that out exactly from inside the contract
@>>>    Point memory initial_last_point = last_point;
        ...
@>>>    last_point.ts = t_i;
@>>>    last_point.blk = initial_last_point.blk + (block_slope * (t_i - initial_last_point.ts)) / MULTIPLIER;
```
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L642
## Tool used

Manual Review

## Recommendation
<img width="829" alt="Screenshot 2024-07-20 at 12 00 16 PM" src="https://github.com/user-attachments/assets/7753e017-f226-4d27-be83-ada202306acd">

Recommended to create a new copy with `lastPoint` values

```diff
function _checkpoint(
        uint _tokenId,
        LockedBalance memory old_locked,
        LockedBalance memory new_locked
    ) internal {
       ...
        uint last_checkpoint = last_point.ts;
        // initial_last_point is used for extrapolation to calculate block number
        // (approximately, for *At methods) and save them
        // as we cannot figure that out exactly from inside the contract
-       Point memory initial_last_point = last_point;
+       Point memory initial_last_point = Point({bias: last_point.bias, slope: last_point.slope, ts: last_point.ts, blk: last_point.blk});
      ...
    }
```
