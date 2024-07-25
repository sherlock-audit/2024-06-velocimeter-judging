Careful Wooden Caribou

Medium

# Owner is not initialized in GaugePlugin contract

## Summary
`_owner` variable in `Ownable` contract is initialized in either the constructor or through `transferOwnership()` function. The `GaugePlugin` contract does not pass the address in the constructor of `Ownable` or call `transferOwnership()` in the contructor.
Hence, the owner is not set.

## Vulnerability Detail
The state level variable is not initialized and hence owner for `GaugePlugin` contract is not set. As such, when the owner wants to set a new governor address, the functionality to set will not be available as the `_owner`  was left uninitialized.

## Impact
`setGovernor` function will not work, when the owner wants to set a new governor address.
## Code Snippet

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/GaugePlugin.sol#L8-L16

The `setGovernor` function will not work as the ownership was not initialized.

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/GaugePlugin.sol#L28-L31

## Tool used

Manual Review

## Recommendation

```solidity
 constructor(address flow, address weth, address _governor, address initialOwner)  Ownable(initialOwner)  {
        governor = _governor;
        isWhitelistedForGaugeCreation[flow] = true;
        isWhitelistedForGaugeCreation[weth] = true;
    }
```
