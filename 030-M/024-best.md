Helpful Oily Corgi

Medium

# Emission Math can be abused by using Wrappers to farm the same LP Pair

## Summary

Permissioneless Gauge creation with whitelisted pairs incentivizes wrapping of one of the two tokens to duplicate the same pool while receiving an outsized, unintended, proportion of emissions


## Vulnerability Detail

Emissions are distributed as a percentage of the `total emissions` over the `voting power`

For very popular pairs, voting will help gain a higher percentage, but it will not increase the absolute amount of tokens emitted.

In order for bigger pairs to increase emissions and receive a high amount of tokens, wrappers could be used

An LP position for said pool would look as follows:
- The trusted token
- A Wrapper token

By doing this, LPrs will on average receive more emissions than if they simply raise their votes and their stake in the same LP Pool

This creates a negative incentive that:
- Will cause liquidity to be fragmented
- Causes wrapping of tokens as a necessary aspect to have certain pool receive a higher amount of emissions

## Impact

By using wrappers the most popular pairs could monopolize the system by being responsible for creating and receiving the vast majority of all emissions

## Code Snippet

Imagine having USDC and WETH, both being whitelisted, and this pair would be expected to be one of the most voted pairs

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

By wrapping USDC into wrappers we can farm more emissions

Openzeppelin offers a convenient [Wrapepr Contract](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/ERC20Wrapper.sol)

By creating a factory for wrappers we could quickly find all pairs that are adopting this strategy

Similarly, the factory could be setup to delegate voting fairly to all wrappers as a means to ensure that everyone benefits from the emissions in the same way

This would end up giving a vast majority of emissions LPs from the main pair, and breaks the emissions logic which was meant to incentivize different pairs

## Tool used

Manual Review

## Recommendation

I don't believe a single sided whitelist to be safe
I think it fundamentally can always be gamed as shown above