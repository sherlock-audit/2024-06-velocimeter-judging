Pet Stone Pelican

Medium

# `Pair.sol#constructor()` - Some tokens have non string metadata fields

## Summary
`Pair.sol#constructor()` - Some tokens have non string metadata fields
## Vulnerability Detail
The `Pair.sol#constructor()` function is as follows.
```solidity
    constructor() {
        factory = msg.sender;
        setVoter();
        (address _token0, address _token1, bool _stable) = IPairFactory(msg.sender).getInitializable();
        (token0, token1, stable) = (_token0, _token1, _stable);
        if (_stable) {
87:         name = string(abi.encodePacked("StableV1 AMM - ", IERC20(_token0).symbol(), "/", IERC20(_token1).symbol()));
88:         symbol = string(abi.encodePacked("sAMM-", IERC20(_token0).symbol(), "/", IERC20(_token1).symbol()));
        } else {
90:         name = string(abi.encodePacked("VolatileV1 AMM - ", IERC20(_token0).symbol(), "/", IERC20(_token1).symbol()));
91:         symbol = string(abi.encodePacked("vAMM-", IERC20(_token0).symbol(), "/", IERC20(_token1).symbol()));
        }

        decimals0 = 10**IERC20(_token0).decimals();
        decimals1 = 10**IERC20(_token1).decimals();

        observations.push(Observation(block.timestamp, 0, 0));
    }
```
As you can see above this code snippet, this function assumes that the `symbol()` function returns a string. 
The issue is that some tokens like MKR have bytes32 metadata fields (name and symbol).
If the symbol() function returns bytes32 and you try to directly concatenate it as a string, it will not work as expected because abi.encodePacked expects string types when concatenating.
As a result, the `constructor()` may be reverted, and therefore the protocol cannot deploy the pair.

Sherlock ReadMe clearly mentions that pools can be created for all ERC20 tokens.
    > Users can create the liquidity pools for any of the ERC20 tokens in permissionless way.

Additionally, Considering MKR has a TVL of $4.5B, we consider this a Medium severity issue.
## Impact
Tokens that have non-string metadata fields cannot be used by the protocol.
## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Pair.sol#L81-L98
## Tool used

Manual Review

## Recommendation
Make a low-level call to retrieve the symbol and then convert it to string.