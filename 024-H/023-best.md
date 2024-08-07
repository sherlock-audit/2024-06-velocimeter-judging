Helpful Oily Corgi

High

# Permissioneless LP Pair Creation allows whales to efficiently farm emissions via single sided deposits paired with malicious tokens

## Summary

The decision to allow Gauges on LP pairs in which just one of the two tokens is benign allows farming emissions with malicious tokens

## Vulnerability Detail

Gauges can be created on any pair as long as one of the two tokens are whitelisted:

https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/GaugePlugin.sol#L55-L63

```solidity
    function checkGaugeCreationAllowance(
        address caller,
        address tokenA,
        address tokenB
    ) external view returns (bool) {
        return
            isWhitelistedForGaugeCreation[tokenA] ||
            isWhitelistedForGaugeCreation[tokenB];
    }
```

As long as someone owns (or can get to 1%) of the total votes, they can simply LP in a single sided fashion by pairing a whitelisted token with a malicious tokens that only allows them to use it.

This will guarantee them that they will farm all emissions for themselves

### Optimal attack

This can be done optimally in the following way:

For each unit of approved token and 1% of voting power:

Chunk into multiple 1% of votes and into unit of approved token, pair it with a token that will revert on transfer if the sender is not the owner and if the recipient is not either the pool or the owner

## Impact

The pair will not be usable by anybody except the owner

They will efficiently farm 100% of the emissions for that pair

## Code Snippet

Note that this attack has already happened on the original solidly:
https://hackmd.io/@c30/BJ2PNCwgs#5-Perform-51-attack-on-Solidly-v1

## Tool used

Manual Review

## Mitigation

In my opinion:
- Both tokens need to be approved
- There should be a delay that allows the owner a sufficient amount of time to blacklist the guage
- This delay needs to be in place as to prevent this from being done at the last second, making the attack unavoidable