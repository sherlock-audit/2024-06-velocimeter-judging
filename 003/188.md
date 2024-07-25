Vast Vermilion Aardvark

Medium

# Users will have to pay more than max payment amount when exercising ve (or lp) in some certain cases

## Summary
Max payment amount is one of input parameters of exercising functions, stating the max amount that a user is willing to pay. If the total amount exceeds this max amount, transactions will revert.

The problem is that the check for maximum payment amount is incorrect for exercisingVe and exercisingLp, making users pay more than maximum amount in some certain cases.
 
## Vulnerability Detail
The total amount users have to pay/transfer is `paymentAmountToAddLiquidity + paymentAmount`:

```solidity
uint256 paymentGaugeRewardAmount = _discount == 0 ? 0 : _takeFees(
            paymentToken,
            paymentAmount
        );
        _safeTransferFrom(
            paymentToken,
            msg.sender,
            address(this),
            paymentGaugeRewardAmount + paymentAmountToAddLiquidity
        );

```

when calculating actual payment amount for exercising ve/lp, the check does not take `paymentAmountToAddLiquidity`, which also contributes to the total amount users will have to pay besides `paymentAmount` , into account. Thus, sometimes users will have to pay more than the maximum payment amount that they agree to pay.

```solidity
if (paymentAmount > _maxPaymentAmount)
            revert OptionToken_SlippageTooHigh();
```



## Impact
Users have to pay more than intended, meaning users lose some extra amount of their funds
## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L612-L621

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L608-L609

## Tool used

Manual Review

## Recommendation
Consider including `paymentAmountToAddLiquidity` into the check
```solidity
if (paymentAmoun + paymentAmountToAddLiquidity > _maxPaymentAmount)
            revert OptionToken_SlippageTooHigh();
```