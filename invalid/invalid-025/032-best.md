Lone Oily Rooster

High

# No Allowance Check in _safeTransferFrom function.

krkbaa
## Summary

## Vulnerability Detail
in OptionTokenV4.sol contract the _safeTransferFrom function does not verify if the from address has granted sufficient allowance to this contract to transfer tokens on its behalf.
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L762-L778
## Impact
it can lead to unauthorized token transfers.
## Code Snippet

## Tool used

Manual Review

## Recommendation
ensure that _from has approved this contract to transfer tokens on its behalf. rs.