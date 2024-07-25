Macho Pine Lemur

Medium

# The `timestamp` variable of a checkpoint is not initialized which leads to wrong checkpointing inside the VotingEscrow contract

## Summary

The timestamp variable of a checkpoint is not initialized properly. This will lead to wrong checkpointing.

## Vulnerability Detail

A checkpoint contains a timestamp variable which stores the timestamp which the checkpoint is created.

```Solidity
    /// @notice A checkpoint for marking delegated tokenIds from a given timestamp
    struct Checkpoint {
        uint timestamp;
        uint[] tokenIds;
    }
```

However, it is seen that the timestamp variable of a checkpoint was not initialized anywhere in the codebase.
Therefore, any function that relies on the timestamp of a checkpoint will break.
The functions `_findWhatCheckpointToWrite()` and `getPastVotesIndex()` inside the VotingEscrow contract rely on the timestamp variable of a checkpoint for computation. 
The following is instances and effects of this uninitialization:

Instance 1 - `_findWhatCheckpointToWrite()` function:

The `_findWhatCheckpointToWrite()` function verifies if the timestamp of the latest checkpoint of an account is equal to the current timestamp.  If true, the function will return the index number of the last checkpoint.

```Solidity
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
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L1413

As such, this function does not work as intended and **will always return the index of a new checkpoint**.

Instance 2 - `getPastVotesIndex()` function:

The `getPastVotesIndex()` function relies on the timestamp of the latest checkpoint for optimization purposes. 
If the request timestamp is the most recently updated checkpoint, it will return the latest index immediately and **skip the binary search**. 
Since the timestamp variable is not populated, the optimization will not work.

```Solidity
    function getPastVotesIndex(address account, uint timestamp) public view returns (uint32) {
        uint32 nCheckpoints = numCheckpoints[account];
        if (nCheckpoints == 0) {
            return 0;
        }
        // First check most recent balance
        if (checkpoints[account][nCheckpoints - 1].timestamp <= timestamp) {
            return (nCheckpoints - 1);
        }

        // Next check implicit zero balance
        if (checkpoints[account][0].timestamp > timestamp) {
            return 0;
        }

        uint32 lower = 0;
        uint32 upper = nCheckpoints - 1;
        while (upper > lower) {
            uint32 center = upper - (upper - lower) / 2; // ceil, avoiding overflow
            Checkpoint storage cp = checkpoints[account][center];
            if (cp.timestamp == timestamp) {
                return center;
            } else if (cp.timestamp < timestamp) {
                lower = center;
            } else {
                upper = center - 1;
            }
        }
        return lower;
    }
```

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L1306

## Proof of Concept

This test aims to do multiple activities, in different timestamps and show that the `getPastVotes()` fails to get the past votes of a user:

```Solidity
    function testWrongIndexInPastVotes() public {

        address user1 = address(0x101);
        address user2 = address(0x102);

        deal(address(flowDaiPair), user1, 100 ether);
        deal(address(FLOW), user1, 100 ether);
    
        vm.startPrank(user1);
        flowDaiPair.approve(address(escrow), type(uint256).max);
        uint256 tokenId = escrow.create_lock(TOKEN_1, 1 weeks);
        uint256 pastVotes = escrow.getVotes(user1);
        uint256 pastVotesAtT1 = pastVotes;
        console2.log("Current Timestamp is: ", block.timestamp);
        console2.log("");
        console2.log("First index Votes is: ", pastVotes);
        console2.log("---------------------------------------");
        vm.stopPrank();

        vm.warp(block.timestamp + 1);
        console2.log("Current Timestamp is: ", block.timestamp);
        console2.log("");
        vm.startPrank(user1);        
        tokenId = escrow.create_lock(TOKEN_1, 1 weeks);
        pastVotes = escrow.getVotes(user1);
        console2.log("Second index Votes is: ", pastVotes);
        escrow.delegate(user2);
        pastVotes = escrow.getVotes(user1);
        console2.log("Third index Votes is:  ", pastVotes);             
        escrow.delegate(user1);
        pastVotes = escrow.getVotes(user1);
        console2.log("Fourth index Votes is: ", pastVotes);  
        console2.log("---------------------------------------");   
        vm.stopPrank();

        vm.warp(block.timestamp + 1);
        console2.log("Current Timestamp is: ", block.timestamp);
        console2.log("");
        vm.startPrank(user1);
        tokenId = escrow.create_lock(TOKEN_1, 1 weeks);
        pastVotes = escrow.getVotes(user1);
        console2.log("Fifth index Votes is: ", pastVotes);
        console2.log("---------------------------------------");
        vm.stopPrank();

        vm.warp(block.timestamp + 1);
        console2.log("Current Timestamp is: ", block.timestamp);
        console2.log("");
        pastVotes = escrow.getPastVotes(user1, 1);
        console2.log("Total Votes at t = 1 was:    ", pastVotesAtT1);
        console2.log("getPastVotes at t = 1 shows: ", pastVotes);

        assertNotEq(pastVotesAtT1, pastVotes);
    }
```

Output of the test:

```Markdown
Ran 1 test for test/VotingEscrow.t.sol:VotingEscrowTest
[PASS] testWrongIndexInPastVotes() (gas: 1707275)
Logs:
  Current Timestamp is:  1

  First index Votes is:  19230737433314004
  ---------------------------------------
  Current Timestamp is:  2

  Second index Votes is:  38461411272814416
  Third index Votes is:   0
  Fourth index Votes is:  38461411272814416
  ---------------------------------------
  Current Timestamp is:  3

  Fifth index Votes is:  57692021518501236
  ---------------------------------------
  Current Timestamp is:  4

  Total Votes at t = 1 was:     19230737433314004
  getPastVotes at t = 1 shows:  57692212299942012

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.37ms (1.69ms CPU time)
```


## Impact

External function and services rely on the checkpointing system of the VotingEscrow contract, e.g. `getPastVotes()`, would fail and show wrong past votes due to uninitialized timestamp variable of a checkpoint.

## Code Snippet

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L1306
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L1413

## Tool used

Manual Review

## Recommendation

Consider initializing the `timestamp` variable of the struct checkpoint inside the VotingEscrow contract