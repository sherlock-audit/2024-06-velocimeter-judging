Boxy Plastic Turtle

High

# Incorrect Fee Calculation in `OptionTokenV4::_takeFees()` will lead to inaccurate fee distribution for treasuries

## Summary

## Vulnerability Detail
The [`OptionTokenV4`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol) contract is an ERC20 token contract that allows holders to exercise options to purchase an underlying token at a discounted TWAP price, with various roles, discounts, and lock durations managed by an admin. The current implementation of [`_takeFees()`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L718-L732) contains an issue in its fee calculation logic. Instead of calculating each treasury's fee based on the remaining amount after previous fee deductions, it consistently uses the original `paymentAmount` for all calculations. This approach leads to an overestimation of fees, particularly for treasuries that are processed later in the sequence.

The root cause of this issue lies in the following code snippet from the [`_takeFees()`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L718-L732) function:

```solidity
function _takeFees(
    address token,
    uint256 paymentAmount
) internal returns (uint256 remaining) {
    remaining = paymentAmount;
    for (uint i; i < treasurys.length; i++) {
        uint256 _fee = (paymentAmount * treasurys[i].fee) / 100; // @audit - Incorrect calculation
        _safeTransferFrom(token, msg.sender, treasurys[i].treasury, _fee);
        remaining = remaining - _fee;
        // ...
    }
}
```

The problem arises because the [`_fee`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L724) calculation uses `paymentAmount` instead of `remaining`. As a result, each treasury's fee is calculated based on the full original amount, not accounting for the fees already deducted by previous treasuries in the loop.

This issue affects multiple core functions of the contract, including [`_exercise()`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L569-L591), [`_exerciseVe()`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L593-L650), and [`_exerciseLp()`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L652-L716), all of which rely on [`_takeFees()`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L718-L732) for fee distribution. The highest impact scenario occurs when there are multiple treasuries with significant fee percentages, potentially leading to a situation where the total fees deducted exceed the original `paymentAmount`.

## Impact

The incorrect fee calculation in [`_takeFees()`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L718-L732) results in an unfair and inaccurate distribution of fees among treasuries. Treasuries processed later in the sequence receive a disproportionately larger share of fees than intended, while earlier treasuries may receive less than their fair share. This discrepancy undermines the intended fee distribution mechanism of the protocol and could lead to financial losses for certain treasuries.

Moreover, in extreme cases where the sum of treasury fees is high, the total fees deducted could potentially exceed the original payment amount, leading to unexpected behavior or transaction reverts. 

## Proof of Concept

Consider a scenario with three treasuries, each set to receive a 20% fee:

1. User initiates an operation that triggers a fee payment of 1000 tokens.
2. `_takeFees()` is called with `paymentAmount = 1000`.
3. For the first treasury:
   - Fee calculated: 1000 * 20% = 200
   - Remaining after deduction: 800
4. For the second treasury:
   - Fee calculated (incorrectly): 1000 * 20% = 200 (should be 800 * 20% = 160)
   - Remaining after deduction: 600
5. For the third treasury:
   - Fee calculated (incorrectly): 1000 * 20% = 200 (should be 640 * 20% = 128)
   - Remaining after deduction: 400

Result: Total fees deducted = 600, which is 60% of the original amount instead of the intended 48.8% (1 - 0.8^3).

## Code Snippet
- https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L718-L732
- https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L724
- https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L569-L591
- https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L593-L650
- https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L652-L716

## Tools Used

Manual Review

## Recommendation

To fix this issue, modify the [`_takeFees()`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L718-L732) function to calculate fees based on the remaining amount after each deduction. Here's the recommended change:

```diff
function _takeFees(
    address token,
    uint256 paymentAmount
) internal returns (uint256 remaining) {
    remaining = paymentAmount;
    for (uint i; i < treasurys.length; i++) {
-       uint256 _fee = (paymentAmount * treasurys[i].fee) / 100;
+       uint256 _fee = (remaining * treasurys[i].fee) / 100;
        _safeTransferFrom(token, msg.sender, treasurys[i].treasury, _fee);
        remaining = remaining - _fee;
        // ...
    }
}
```

This change ensures that each treasury's fee is calculated based on the actual remaining amount, resulting in a fair and accurate fee distribution among all treasuries.