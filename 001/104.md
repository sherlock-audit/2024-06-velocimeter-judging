Sweet Lemonade Lynx

Medium

# Division by Zero Vulnerability in notifyRewardAmount Function Due to Zero totalWeight


## Summary

The `notifyRewardAmount` function in `Voter.sol` is vulnerable to a division by zero error when `totalWeight` is zero. This condition can occur as `totalWeight` is dynamically updated in other functions. The lack of a check for this condition can cause the function to fail, preventing proper reward distribution.

## Vulnerability Detail

The `notifyRewardAmount` function calculates a reward ratio by dividing the reward `amount` by `totalWeight`. If `totalWeight` is zero, this division will cause an error, resulting in the function failing to execute correctly. Here is the relevant code [snippet](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L490):

The division by zero occurs at the line `uint256 _ratio = amount * 1e18 / totalWeight;` if `totalWeight` is zero. This condition is not checked, which can lead to a runtime error.

## Impact

If `totalWeight` is zero, the division will cause the `notifyRewardAmount` function to revert, leading to a failure in the reward notification process. This can disrupt the reward distribution mechanism and potentially affect the overall functionality of the contract.

## Code Snippet

```solidity
function notifyRewardAmount(uint amount) external {
    require(msg.sender == minter, "not a minter");
    activeGaugeNumber = 0;
    currentEpochRewardAmount = amount;
    _safeTransferFrom(base, msg.sender, address(this), amount); // transfer the distro in
    uint256 _ratio = amount * 1e18 / totalWeight; // 1e18 adjustment is removed during claim // @audit division by zero not prevented.
    if (_ratio > 0) {
        index += _ratio;
    }
    emit NotifyReward(msg.sender, base, amount);
}
```

## Tool Used

Manual Review

## Recommendation

Add a check to ensure that `totalWeight` is greater than zero before performing the division. If `totalWeight` is zero, handle the situation appropriately to prevent the function from failing.

```solidity
function notifyRewardAmount(uint amount) external {
    require(msg.sender == minter, "not a minter");
    require(totalWeight > 0, "totalWeight is zero");
    activeGaugeNumber = 0;
    currentEpochRewardAmount = amount;
    _safeTransferFrom(base, msg.sender, address(this), amount); // transfer the distro in
    uint256 _ratio = amount * 1e18 / totalWeight; // 1e18 adjustment is removed during claim
    if (_ratio > 0) {
        index += _ratio;
    }
    emit NotifyReward(msg.sender, base, amount);
}
```
