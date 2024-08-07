Polite Butter Gazelle

Medium

# Exercising LP/veNFT may DoS if `payment token` is a revert on 0 transfer token

## Summary

`OptionTokenV4` allows users to exercise their `oTokens` in 3 different ways: `exercise()`, `exerciseVe()`, `exerciseLp()`. In each function call, a specific amount of `payment token` is taken from the caller, at a discounted price. This amount is sent to `treasuries` and to the `gauge`. The `Velocimeter` sponsors confirmed on discord that users may receive a full discount when `exerciseVe()` or `exerciseLp()` is called: `"for exercise LP/VE we are allowing full discount as user still needs to proviade the lp token"`.

The contest README states the following: `"OptionTokenV4 contract by design supports only standard ERC20 - 18 decimal tokens as underlyingToken\paymentToken ( rebase, fee on transfer and not 18 decimals tokens are not supported)"`. Therefore, the `payment token` can be a `revert on 0 transfer` token.

In this case, any payment sent via transferring reward to `gauge` may revert due to 0 transfer, causing DoS of exercising LP/VE.

## Vulnerability Detail

Let's look at what happens when we exercise LP or VE when discount is 100% and payment token is revert on 0 transfer.

[OptionTokenV4.sol#L607](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L607)
```javascript
    (uint256 paymentAmount,uint256 paymentAmountToAddLiquidity) =  getPaymentTokenAmountForExerciseLp(_amount,_discount); // TODO decide if we want to have the curve or just always maxlock
```

`getPaymentTokenAmountForExerciseLp` will return `paymentAmount = 0` if `discount = 0`. Note that if `discount = 0`, that means the user gets a FULL discount (i.e., discount = 20 means user pays 20% (80% discount)).

[OptionTokenV4.sol#L350-L356](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L350-L356)
```javascript
    function getPaymentTokenAmountForExerciseLp(uint256 _amount,uint256 _discount) public view returns (uint256 paymentAmount, uint256 paymentAmountToAddLiquidity)
    {
       
@>      paymentAmount = _discount == 0 ? 0 : getLpDiscountedPrice(_amount, _discount);
        (uint256 underlyingReserve, uint256 paymentReserve) = IRouter(router).getReserves(underlyingToken, paymentToken, false);
        paymentAmountToAddLiquidity = (_amount * paymentReserve) / underlyingReserve;
    }
```

Next, fees are sent to treasuries if `discount` is non-zero. Since in our case `discount = 0`, we only calculate `paymentGaugeRewardAmount = 0`.

[OptionTokenV4.sol#L612-L615](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L612-L615)
```javascript
    uint256 paymentGaugeRewardAmount = _discount == 0 ? 0 : _takeFees(
        paymentToken,
        paymentAmount
    );
```

`paymentGaugeRewardAmount + paymentAmountToAddLiquidity` is pulled from the user.

[OptionTokenV4.sol#L675-L680)](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L675-L680)
```javascript
    _safeTransferFrom(
        paymentToken,
        msg.sender,
        address(this),
        paymentGaugeRewardAmount + paymentAmountToAddLiquidity //@audit paymentGaugeRewardAmount will be 0
    );
```

The entire amount of `paymentAmountToAddLiquidity` is used to add liquidity to the respective underlying token/payment token pool. This means that after the `addLiquidity` call, the contract will be left with 0 `paymentToken`.

At the end of the exercise call, a call to ` _transferRewardToGauge()` is made to send any leftover `payment token` to the gauge of the pool.

```javascript
    function _transferRewardToGauge() internal {
@>      uint256 paymentTokenCollectedAmount = IERC20(paymentToken).balanceOf(address(this)); //@audit this will be 0

        if(rewardsAddress != address(0)) {
@>          _safeTransfer(paymentToken,rewardsAddress,paymentTokenCollectedAmount); //@audit this will revert due to 0 transfer
        } else {
            uint256 leftRewards = IGaugeV4(gauge).left(paymentToken);

            if(paymentTokenCollectedAmount > leftRewards) { // we are sending rewards only if we have more then the current rewards in the gauge
                _safeApprove(paymentToken, gauge, paymentTokenCollectedAmount);
                IGaugeV4(gauge).notifyRewardAmount(paymentToken, paymentTokenCollectedAmount);
            }
        }
    }
```

Any payment sent to the `rewardsAddress` will revert due to 0 transfer, causing DoS of exercising LP/VE. 

I'll note that there are two potential solutions with downsides:

1. Admin sets rewards address to 0, but this would also mean that the rewards address can never be set for any calls to `OptionTokenV4::exercise` (where users don't receive 100% discount), causing the designated rewards address to lose rewards.

2. Setting max discount to a low enough value (i.e, 80%) so that the amount of fees sent to treasuries and gauge/rewards address will not revert on 0 transfer, but this will cause users to lose funds as they intention was they don't have to pay.

## Impact

DoS of exercising LP/VE, inability to have a rewards address set to collect rewards for any call to `OptionTokenV4::exercise`, causing loss of rewards. Reseting the discount to a low enough value will cause users to lose funds.

## Code Snippet

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L607

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L350-L356

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L612-L615

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L675-L680

## Tool used

Manual Review

## Recommendation

Make the following changes to `_exerciseVe()` and `_exerciseLp()`:

```diff
- _transferRewardToGauge();
+ if (paymentGaugeRewardAmount > 0) _transferRewardToGauge();
```