Melted Quartz Rattlesnake

Medium

# m-01 2 step ownership transfer not found in the given code

## Summary

1. there is not any 2 step ownership transfer pattern found at the given code

## Vulnerability Detail

1. `setMinter` procedure skipped recommended ownable2step pattern or did not follow recommended ownable2step pattern
2. ownable2step pattern requires to refactor toward creating the pending minter

## Impact

1. medium impact
4. the issue usually can reduced to medium since purely typo(s) argument during function calling

## Code Snippet

[POC](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Flow.sol#L29)

## Tool used

Manual Review

## Recommendation

[AuditBase](https://detectors.auditbase.com/use-ownable2step-solidity)
[rareskills](https://www.rareskills.io/post/openzeppelin-ownable2step)