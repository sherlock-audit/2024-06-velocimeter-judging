Slow Clay Python

Medium

# Use safeTransfer() instead of transfer()

## Summary

The ERC20.transfer() and ERC20.transferFrom() functions return a boolean value indicating success. This parameter needs to be checked for success. Some tokens do not revert if the transfer fails but return false instead.

## Vulnerability Detail

Some tokens (like USDT) don't correctly implement the EIP20 standard and their transfer/ transferFrom function return void instead of a success boolean. Calling these functions with the correct EIP20 function signatures will always revert.

## Impact

Tokens that don't perform the transfer and return false are still counted as a correct transfer and tokens that don't correctly implement the latest EIP20 spec, like USDT, will be unusable in the protocol as they revert the transaction because of the missing return value.

## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Minter.sol#L127

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/RewardsDistributorV2.sol#L323

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L805

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L972

## Tool used

Manual Review

## Recommendation

Recommend using OpenZeppelin's SafeERC20 versions with the safeTransfer and safeTransferFrom functions that handle the return value check as well as non-standard-compliant tokens.
