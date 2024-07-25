Able Gingham Manatee

Medium

# Invalid Check in `OptionTokenV4._exerciseLp()`

## Summary
Invalid check will always cause DOS in `OptionTokenV4._exerciseLp()` 

## Vulnerability Detail
In `OptionTokenV4._exerciseLp()` there will always be reverts due to this check
```solidity
        if (_discount > minLPDiscount || _discount < maxLPDiscount)
            revert OptionToken_InvalidDiscount();
```

Now lets say 
- `minLPDiscount` is 5%

- `maxLPDiscount` is 90%

- and `_discount` is 20% 

There will be a revert BUT this shouldn't be and here's why, `minLPDiscount` is supposed to be least rate  and `maxLPDiscount` is supposed to be maximum rate.. so given the above values, `_discount` is > least rate and < maximum rate so its not supposed to revert BUT IT DOES HENCE IT IS AN ISSUE.

## Impact
Invalid Check in `OptionTokenV4._exerciseLp()`  will always cause reverts.
## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L661-L662
## Tool used

Manual Review

## Recommendation
change this:
```solidity
-        if (_discount > minLPDiscount || _discount < maxLPDiscount)
             revert OptionToken_InvalidDiscount();
```

to this:
```solidity
+        if (_discount < minLPDiscount || _discount > maxLPDiscount)
             revert OptionToken_InvalidDiscount();
```