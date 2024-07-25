Active Lace Hippo

Medium

# `OptionTokenV4::setTwapPoints` Results In Stepwise Price Jump Backrun

## Summary

Invocations of [`setTwapPoints(uint256)`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/OptionTokenV4.sol#L509C14-L509C48) induce stepwise changes in price that can be exploited for arbitrage.

## Vulnerability Detail

The [`OptionTokenV4`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol) enables [`onlyAdmin`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/OptionTokenV4.sol#L509C58-L509C67) to modify the number of points required to compute the time weighted average price. However when doing so, this can introduce a stepwise change in price which can be exploited by arbitrageurs.

In the proof of concept below, the `owner` increases the `twapPoints` from `4` to `15`, (as per [`testSetTwapPoints()`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/test/OptionTokenV4.t.sol#L252C5-L260C6)). We demonstrate that this stepwise jump invariably results in a price delta that can be exploited for arbitrage:

### OptionTokenV4.t.sol

```solidity
function testFuzzSherlockSetTwapPointsPriceImpact(uint256 amount) public {
    vm.assume(amount <= type(uint128).max);
    vm.startPrank(address(owner));

        /// @audit Create some trading history to produce
        /// @audit a suitable number of observations for the
        /// @audit twap.
        for (uint256 i = 0; i < 6; i++) washTrades();

        assertEq(oFlowV4.twapPoints(), 4);
        uint256 beforeTwap = oFlowV4.getTimeWeightedAveragePrice(amount);
        oFlowV4.setTwapPoints(15);
        uint256 afterTwap = oFlowV4.getTimeWeightedAveragePrice(amount);
    vm.stopPrank();

    assert(afterTwap <= beforeTwap); /// @audit price_is_reduced
}
```

## Impact

Inducing a stepwise change for an [`OptionTokenV4`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol) can easily result in price arbitrage.

Various in-protocol pricing mechanisms are vulnerable to this vector for value extraction, such as [`getDiscountedPrice(uint256)`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/OptionTokenV4.sol#L320C5-L325C6) and [`getLpDiscountedPrice(uint256,uint256)`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/OptionTokenV4.sol#L327C5-L336C6), which are [relied upon during exercising](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/OptionTokenV4.sol#L579C25-L579C43).

## Code Snippet

```solidity
/// @notice Sets the twap points. to control the length of our twap
/// @param _twapPoints The new twap points.
function setTwapPoints(uint256 _twapPoints) external onlyAdmin { /// @custom:hunter stepwise_change_exploit_arbitrage
    if (_twapPoints > MAX_TWAP_POINTS || _twapPoints == 0)
        revert OptionToken_InvalidTwapPoints();
    twapPoints = _twapPoints;
    emit SetTwapPoints(_twapPoints);
}
```

https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/OptionTokenV4.sol#L507C5-L514C6

## Tool used

Manual Review

## Recommendation

Consider linearly interpolating the [`twapPoints`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/OptionTokenV4.sol#L159C20-L159C30) to the target on each new price observation in order to minimize price impact.