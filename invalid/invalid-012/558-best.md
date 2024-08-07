Furry Clear Chinchilla

Medium

# The `deposit_for` function lacks restrictions on the types of NFT, and this is a problem

## Summary

The `deposit_for` function lacks restrictions on the types of NFT, and this is a problem.

## Vulnerability Detail

The `deposit_for()` function in the `VotingEscrow.sol` contract allows users to deposit additional tokens for a given NFT. The function is even designed to be called by any user.
However, it currently lacks restrictions on the types of NFTs it accepts:

```solidity
    function deposit_for(uint _tokenId, uint _value) external nonreentrant { 
        LockedBalance memory _locked = locked[_tokenId];

        require(_value > 0); // dev: need non-zero value
        require(_locked.amount > 0, 'No existing lock found');
        require(_locked.end > block.timestamp, 'Cannot add to expired lock. Withdraw'); 
        _deposit_for(_tokenId, _value, 0, _locked, DepositType.DEPOSIT_FOR_TYPE);
    }
```

**Anyone can deposit locked NFT**: Allowing deposits to locked NFTs can lead to increased voting power without affecting the gauge weight. If a locked NFT is later withdrawn, the increased balance can be overwritten, resulting in a loss of funds.

**Anyone can deposit managed NFT**: anyone could also increase the voting power of a managed NFT directly by calling `depositFor()` with a `_tokenId` of a managed NFT, which breaks the main invariant.
## Impact

Currentl, lacks restrictions on the types of NFTs it accepts, which poses two main risks:

- **Locked NFTs**: Locked NFTs should not have their voting power increased through this function.
- **Managed NFTs**: Only specific functions, should be able to deposit for managed NFTs to maintain protocol invariants.

## Code Snippet

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L826-L833

## Tool used

Manual Review

## Recommendation

Add a check to prevent deposits for locked and managed NFTs.