Atomic Citron Fly

High

# Duplicated tokenIDs within a checkpoint will lead to inflated voting balance

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L1413

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L1362

## Summary
The `_moveTokenDelegates` function contains a vulnerability in checkpoint management that can lead to some issues. A checkpoint can contain duplicated (tokenIDs) under certain circumstances leading to double counting of voting balance.

## Vulnerability Detail
The `_moveTokenDelegates` function contains a vulnerability in checkpoint management that can lead to some issues. A checkpoint can contain duplicated (tokenIDs) under certain circumstances leading to double counting
of voting balance. Malicious users could exploit this vulnerability to inflate the voting balance of their accounts and
participate in governance and gauge weight voting, potentially causing loss of assets or rewards for other users if
the inflated voting balance is used in a malicious manner

consider the following in `_moveTokenDelegates`

1. Assuming moving tokenID=888 from Alice to Bob.
2. Source Code Logic (Moving tokenID=888 out of Alice)
   • Fetch the existing Alice's token IDs and assign them to srcRepOld
   • Create a new empty array = srcRepNew
   • Copy all the token IDs in srcRepOld to srcRepNew except for tokenID=888
3. Destination Code Logic (Moving tokenID=888 into Bob)
   • Fetch the existing Bobs' token IDs and assign them to dstRepOld
   • Create a new empty array = dstRepNew
   • Copy all the token IDs in dstRepOld to dstRepNew
   • Copy tokenID=888 to dstRepNew
   The existing logic works fine as long as a new empty array (srcRepNew or dstRepNew) is created every single
   time. The code relies on the `_findWhatCheckpointToWrite` function to return the index of a new checkpoint

```javascript
function _moveTokenDelegates(
        address srcRep,
        address dstRep,
        uint _tokenId
    ) internal {
        if (srcRep != dstRep && _tokenId > 0) {
            if (srcRep != address(0)) {
                uint32 srcRepNum = numCheckpoints[srcRep];
                uint[] storage srcRepOld = srcRepNum > 0
                    ? checkpoints[srcRep][srcRepNum - 1].tokenIds
                    : checkpoints[srcRep][0].tokenIds;
                uint32 nextSrcRepNum = _findWhatCheckpointToWrite(srcRep);
                uint[] storage srcRepNew = checkpoints[srcRep][
                    nextSrcRepNum
                ].tokenIds;
                ...
```

However, the problem is that the `_findWhatCheckpointToWrite` function does not always return the index of a new
checkpoint. It will return the last checkpoint if it has already been written once in the same block

```javascript
function _findWhatCheckpointToWrite(address account)
        internal
        view
        returns (uint32)
    {
        uint _timestamp = block.timestamp;
        uint32 _nCheckPoints = numCheckpoints[account];

        if (
            _nCheckPoints > 0 &&
            checkpoints[account][_nCheckPoints - 1].timestamp == _timestamp
        ) {
            return _nCheckPoints - 1;
        } else {
            return _nCheckPoints;
        }
    }

```

If a user triggers the `_moveTokenDelegates` more than once in the same block (e.g. perform NFT transfer
twice to the same person), the `_findWhatCheckpointToWrite` function will return a new checkpoint in the first
transfer but will return the last/previous checkpoint in the second transfer. This will cause the move token delegate
logic to be off during the second transfer.

First Transfer at block 1000
Assume the following states:

```javascript
numCheckpoints[Alice] = 1
_checkpoints[Alice][0].tokenIds = [n1, n2] <== Most recent checkpoint
numCheckpoints[Bob] = 1
_checkpoints[Bob][0].tokenIds = [n3] <== Most recent checkpoint
```

To move tokenID=2 from Alice to Bob, the `_moveTokenDelegates(Alice, Bob, n2)` function will be triggered.
The `_findWhatCheckpointToWrite` will return the index of 1 which points to a new array.

The end states of the first transfer will be as follows:

```javascript
numCheckpoints[Alice] = 2
_checkpoints[Alice][0].tokenIds = [n1, n2]
_checkpoints[Alice][1].tokenIds = [n1] <== Most recent checkpoint
numCheckpoints[Bob] = 2
_checkpoints[Bob][0].tokenIds = [n3]
_checkpoints[Bob][1].tokenIds = [n2, n3] <== Most recent checkpoint
```

Everything is working fine at this point in time.
Second Transfer at block 1000 (same block)
To move tokenID=1 from Alice to Bob, the `_moveTokenDelegates(Alice, Bob, n1)` function will be triggered.
This time round since the last checkpoint timestamp is the same as the current timestamp, the `_findWhatCheckpointToWrite`
function will return the last checkpoint instead of a new checkpoint.
The srcRepNew and dstRepNew will end up referencing the old checkpoint instead of a new checkpoint. As such,
the srcRepNew and dstRepNew array will reference back to the old checkpoint `_checkpoints[Alice][1].tokenIds`
and `_checkpoints[Bob][1].tokenIds` respectively.
The end state of the second transfer will be as follows:

```javascript
numCheckpoints[Alice] = 3
_checkpoints[Alice][0].tokenIds = [n1, n2]
_checkpoints[Alice][1].tokenIds = [n1] <== Most recent checkpoint
numCheckpoints[Bob] = 3
_checkpoints[Bob][0].tokenIds = [n3]
_checkpoints[Bob][1].tokenIds = [n2, n3, n2, n3, n1] <== Most recent checkpoint
```

Four (4) problems could be observed from the end state:

1. The numCheckpoints is incorrect. Should be two (2) instead to three (3)
2. TokenID=1 has been added to Bob's Checkpoint, but it has not been removed from Alice's Checkpoint
3. Bob's Checkpoint contains duplicated tokenIDs (e.g. there are two TokenID=2 and TokenID=3)
4. TokenID is not unique (e.g. TokenID appears more than once)
   Since the token IDs within the checkpoint will be used to determine the voting power, the voting power will be
   inflated in this case as there will be a double count of certain NFTs.

```javascript
function _moveTokenDelegates(
..SNIP..
uint32 nextSrcRepNum = _findWhatCheckpointToWrite(srcRep);
uint256[] storage srcRepNew = _checkpoints[srcRep][nextSrcRepNum].tokenIds;
```

Additionally in `nextSrcRepNum` variable the code wrongly assumes that the `_findWhatCheckpointToWrite` function will always return
the index of the next new checkpoint. The `_findWhatCheckpointToWrite` function could return the index of the latest
checkpoint instead of a new one if `block.timestamp == checkpoint.timestamp`.

Additional Comment about `numCheckpoints`
The function computes the new number of checkpoints by incrementing the `srcRepNum` by one.
However, this is incorrect because if block.timestamp == checkpoint.timestamp, then the number of checkpoints
remains the same and does not increment.

```javascript
numCheckpoints[srcRep] = srcRepNum + 1;
```

This should be updated only if a new checkpoint is created

```javascript
// Update numCheckpoints only if a new checkpoint is created
if (nextSrcRepNum > srcRepNum) {
  numCheckpoints[srcRep] = srcRepNum + 1;
}
```

## Impact
Malicious users could exploit this vulnerability to inflate the voting balance of their accounts and
participate in governance and gauge weight voting, potentially causing loss of assets or rewards for other users if
the inflated voting balance is used in a malicious manner

## Code Snippet
```javascript
function _findWhatCheckpointToWrite(address account)
        internal
        view
        returns (uint32)
    {
        uint _timestamp = block.timestamp;
        uint32 _nCheckPoints = numCheckpoints[account];

        if (
            _nCheckPoints > 0 &&
            checkpoints[account][_nCheckPoints - 1].timestamp == _timestamp
        ) {
            return _nCheckPoints - 1;
        } else {
            return _nCheckPoints;
        }
    }

```

## Tool used

Manual Review

## Recommendation
Update the move token delegate logic within the affected functions `_moveTokenDelegates`
and `_moveAllDelegates` to ensure that the latest checkpoint is overwritten correctly
when the functions are triggered more than once. see similar fix [here](https://github.com/velodrome-finance/contracts/commit/a670bfb62c69fe38ac918b734a03b68a5505f8a2).
