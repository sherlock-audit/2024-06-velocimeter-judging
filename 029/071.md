Melted Quartz Rattlesnake

High

# h-01 nonReentrant plus non-trusted procedure 0xaliyah

## Summary

1. missing correctness for the check, effect, and interaction, pattern

## Vulnerability Detail

1. L972 hands control to non-trusted token transfer procedure where the effect kind procedure on L975 should execute first
2. token burn state change on L975 should take priority to execute as an effect and prior to L972 token transfer was visited

## Impact

1. may be labeled for high vulnerability
2. might have to follow recommended pattern of check, effect, and interaction

## Code Snippet

[POC](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L972)

## Tool used

Manual Review

## Recommendation

[Checks Effects Interactions](https://fravoll.github.io/solidity-patterns/checks_effects_interactions.html)
[Will Shahda](https://medium.com/coinmonks/protect-your-solidity-smart-contracts-from-reentrancy-attacks-9972c3af7c21)