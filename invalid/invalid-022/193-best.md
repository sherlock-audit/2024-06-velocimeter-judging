Odd Carmine Cougar

High

# `VotingEscrow.getPastVotes()` returns the wrong value.

## Summary
`VotingEscrow.getPastVotes()` calculates the past votes based on the last epoch and so returns the wrong value for the time before last epoch time.

## Vulnerability Detail
The following is the relavant code of `VotingEscrow.getPastVotes()`.
```solidity
    function getPastVotes(address account, uint timestamp)
        public
        view
        returns (uint)
    {
        uint32 _checkIndex = getPastVotesIndex(account, timestamp);
        // Sum votes
        uint[] storage _tokenIds = checkpoints[account][_checkIndex].tokenIds;
        uint votes = 0;
        for (uint i = 0; i < _tokenIds.length; i++) {
            uint tId = _tokenIds[i];
            // Use the provided input timestamp here to get the right decay
1349:       votes = votes + _balanceOfNFT(tId, timestamp);
        }
        return votes;
    }
```
The function calls `_balanceOfNFT()` on `L1349` to calculates the past votes for a tokenId.
```solidity
    function _balanceOfNFT(uint _tokenId, uint _t) internal view returns (uint) {
        uint _epoch = user_point_epoch[_tokenId];
        if (_epoch == 0) {
            return 0;
        } else {
            Point memory last_point = user_point_history[_tokenId][_epoch];
1023:       last_point.bias -= last_point.slope * int128(int256(_t) - int256(last_point.ts));
            if (last_point.bias < 0) {
                last_point.bias = 0;
            }
            return uint(int256(last_point.bias));
        }
    }
```
Since the function calculates the `bias` based on the last epoch for any timestamp `_t`, `int256(_t) - int256(last_point.ts)` may be negative, and thus the `bias` may be inflated.
As a result, the function may return the wrong value for the time before last epoch time.

*** Proof of Concept ***
The following is the test code for `VotingEscrow.t.sol`.
```solidity
    function test_get_past_votes() public {
        flowDaiPair.approve(address(escrow), TOKEN_10);
        uint256 lockDuration = ONE_WEEK;

        uint tokenId = escrow.create_lock(TOKEN_1, lockDuration);
        uint oldPastVotes = escrow.getPastVotes(address(owner), block.timestamp);

        uint oldTimestamp = block.timestamp;
        uint newTimestamp = block.timestamp + 100_000;

        vm.warp(newTimestamp);
        vm.roll(block.number + 10_000);

        escrow.deposit_for(tokenId, TOKEN_1);
        uint newPastVotes = escrow.getPastVotes(address(owner), newTimestamp);

        uint wrongPastVotes = escrow.getPastVotes(address(owner), oldTimestamp);

        assertTrue(newPastVotes > oldPastVotes);
        assertTrue(wrongPastVotes > oldPastVotes);
        assertTrue(wrongPastVotes > newPastVotes);

        console.log("oldPastVotes:%d at:%d", oldPastVotes, oldTimestamp);
        console.log("newPastVotes:%d at:%d", newPastVotes, newTimestamp);
        console.log("wrongPastVotes:%d at:%d", wrongPastVotes, oldTimestamp);
    }
```
The following is the test log.
```log
Ran 1 test for test/VotingEscrow.t.sol:VotingEscrowTest
[PASS] test_get_past_votes() (gas: 693489)
Logs:
  oldPastVotes:19230737433314004 at:1
  newPastVotes:32102093507932807 at:100001
  wrongPastVotes:38461474867232807 at:1

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.76ms (585.40µs CPU time)

Ran 1 test suite in 17.79ms (7.76ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
As shown above, the `wrongPastVotes` is greater than the `oldPastVotes` and even `newPastVotes`.

## Impact
`VotingEscrow.getPastVotes()` returns the wrong value for the past time.
This issue causes serous problems to the voting and governance system.

## Code Snippet
- [v4-contracts/contracts/VotingEscrow.sol#L1017-L1029](https://github.com/sherlock-audit/2024-06-velocimeter/tree/main/v4-contracts/contracts/VotingEscrow.sol#L1017-L1029)
- [v4-contracts/contracts/VotingEscrow.sol#L1337-L1352](https://github.com/sherlock-audit/2024-06-velocimeter/tree/main/v4-contracts/contracts/VotingEscrow.sol#L1337-L1352)

## Tool used

Manual Review

## Recommendation
I recommend that the `VotingEscrow._balanceOfNFT()` calculates the past votes based on appropriate epoch for any timestamp `_t`.
To do that, it may be required to newly implement the `_find_timestamp_epoch()` function which is similar to the `_find_block_epoch()`.
