Handsome Holographic Turtle

Medium

# The `Voter.emitWithdraw()` function lacks permission control

## Summary

The `Voter.emitWithdraw()` function lacks permission control, allowing anyone to call it and emit invalid logs. This can complicate off-chain data analysis and statistics. 

## Vulnerability Detail
The `Voter.emitWithdraw()` function solely serves the purpose of emitting a `withdraw` log. This function lacks any permission control, allowing anyone to call it and subsequently emit the log. Malicious users can exploit this by repeatedly calling the function, thereby generating a large number of invalid logs. This can complicate off-chain data analysis and statistics, making it difficult to accurately interpret the data.

```solidity
  function emitWithdraw(uint tokenId, address account, uint amount) external {
        emit Withdraw(account, msg.sender, tokenId, amount);
    }

```

## Impact

This can complicate off-chain data analysis and statistics, making it difficult to accurately interpret the data.
## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L449-L451
## Tool used

Manual Review

## Recommendation
It is recommended to add access control to this function to prevent unauthorized usage.

