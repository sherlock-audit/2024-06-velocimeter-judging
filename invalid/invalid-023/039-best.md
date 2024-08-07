Sweet Pecan Wasp

Medium

# OptionTokenV4::updateGauge() allows anyone to overwrite Admin set Gauge when Voter has no gauge

## Summary

The admin of `OptionTokenV4` can manually set the `gauge` address by calling `setGauge()`, which should be utilised when `Voter::gauges()` does not contain the current pair gauge. However `OptionTokenV4::updateGauge()` allows anyone to overwrite Admin set gauge when Voter has no gauge, setting it to `address(0)`.

## Vulnerability Detail
[OptionTokenV4::L412-L423](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L412-L423)
```solidity
    /// @notice Update gauge address to match with Voter contract
    function updateGauge() external {
        address newGauge = IVoter(voter).gauges(address(pair));
        gauge = newGauge;
        emit SetGauge(newGauge);
    }

    /// @notice Sets the gauge address when the gauge is not listed in Voter. Only callable by the admin.
    /// @param _gauge The new treasury address
    function setGauge(address _gauge) external onlyAdmin {
        gauge = _gauge;
        emit SetGauge(_gauge);
    }
```
`_setGauge()` is meant to be utilised by the admin to manually set a gauge when `Voter` does not have a gauge listed for the current pair utilised in `OptionTokenV4`. However after the admin sets the correct gauge, any user can call `updateGauge` to set `gauge` to `address(0)`, which undoes the admins call.

## Impact

The admin is unable to set a gauge manually when there is no listed gauge in `Voter`, as the `updateGauge()` function has no access control and does not check if the returned gauge from `Voter` is `addresss(0)`, allowing anyone to set `gauge` to `address(0)`.

This will lead to any functions that utilise `_transferRewardToGauge()` to revert as they will attempt to call `IGaugeV4(gauge).left(paymentToken)` which will not be possible on `address(0)`. This will cause a DOS scenario, breaking functionality such as `_exerciseVe()`, `_exerciseLp()` and `exercise()`.

## Code Snippet

[OptionTokenV4::L412-L423](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L412-L423)

## Tool used

Manual Review

## Recommendation

Ensure that `updateGauge()` does not set a new gauge if `newGauge` is `address(0)`:

[OptionTokenV4::updateGauge()](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L412-L416)
```diff
    function updateGauge() external {
        address newGauge = IVoter(voter).gauges(address(pair));
+      require(newGauge != address(0), "newGauge == address(0)");
        gauge = newGauge;
        emit SetGauge(newGauge);
    }
```