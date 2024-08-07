Dandy Shamrock Sheep

Medium

# Zero Amount Withdrawal

## Summary
The withdrawToken function in the GaugeV4 contract allows users to withdraw zero amounts, which can lead to unnecessary state changes and potential system manipulation.

## Vulnerability Detail
The withdrawToken function doesn't check if the amount parameter is greater than zero. This allows users to call the function with amount = 0, which still executes all the internal logic, including updating rewards, modifying state variables, and emitting events.
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/GaugeV4.sol#L513-L556

## Impact
* Unnecessary Gas Consumption: Executing the function with zero amount wastes gas on unnecessary computations and state changes.
* Manipulation of Last Action Timestamp: This could be used to update the user's last action timestamp without actually withdrawing anything, potentially interfering with time-based mechanics in the system.
* Emission of Misleading Events: The function emits Withdraw events even for zero-amount withdrawals, which could confuse off-chain systems monitoring these events.
* Detachment of veNFT: If tokenId is provided, the function will detach the veNFT from the gauge even if no actual withdrawal occurs.

## Code Snippet
Add the following test to the GaugeV4Test contract:
```solidity
function testZeroAmountWithdrawal() public {
    // Setup: Deposit some tokens first
    vm.startPrank(address(owner));
    flowDaiPair.approve(address(gauge), 1000);
    gauge.deposit(1000, 0);
    
    // Perform zero amount withdrawal
    uint256 balanceBefore = flowDaiPair.balanceOf(address(owner));
    gauge.withdrawToken(0, 0);
    uint256 balanceAfter = flowDaiPair.balanceOf(address(owner));
    
    // Check that balance hasn't changed
    assertEq(balanceBefore, balanceAfter, "Balance should not change on zero withdrawal");
    
    // But the withdraw event is still emitted
    vm.expectEmit(true, true, true, true);
    emit Withdraw(address(owner), 0, 0);
    gauge.withdrawToken(0, 0);
    
    vm.stopPrank();
}
```

## Tool used

Manual Review

## Recommendation
Implement a check at the beginning of the withdrawToken function to ensure the amount is greater than zero:
```solidity
function withdrawToken(uint amount, uint tokenId) public lock {
    require(amount > 0, "Cannot withdraw zero amount");
    // ... rest of the function
}
```
