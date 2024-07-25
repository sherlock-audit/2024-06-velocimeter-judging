Main Blonde Beaver

Medium

# `Pair` contract can't come back to old bribe if needed

## Summary
`Pair` contract can't come back to old bribe if needed due to approvement issues with some kind of weird ERC20 tokens
## Vulnerability Detail
Some tokens from the weird ERC20 repo have the following functionality:
>Some tokens (e.g. USDT, KNC) do not allow approving an amount M > 0 when an existing amount N > 0 is already approved. 

This by itself means that if a bribe is once changed, it can never be used again by the same `Pair`. As you can imagine this is bad because if the old bribe is needed by the for the `Pair` contract it can't be changed back ever again 
## Impact
Once a bribe is changed in the context of the `Pair` contract, the old bribe can't be used back again with some kind of weird ERC20 tokens (KNC)

## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Pair.sol#L120-L126
## Tool used

Manual Review

## Recommendation
You should approve the old bribe to 0, before changing the addresses like this:
```diff
    function setExternalBribe(address _externalBribe) external {
        require(msg.sender == voter, "Only voter can set external bribe");
+        _safeApprove(token0, externalBribe, 0);
+       _safeApprove(token1, externalBribe, 0);
        externalBribe = _externalBribe;
        _safeApprove(token0, externalBribe, type(uint).max);
        _safeApprove(token1, externalBribe, type(uint).max);
        emit ExternalBribeSet(_externalBribe);
    }
```
