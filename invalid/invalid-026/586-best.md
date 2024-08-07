Dazzling Mossy Vulture

High

# If a max locked nft expires, it will be stuck forever

## Summary
The ```votingEscrow``` contract has a ```max_lock``` feature by which an veNFT owner can turn max lock on for their nfts which will re-lock the nft for the max duration on any interaction ( increasing amount, increasing unlock time etc.), and this can also be done by anyone since max_lock is a public function.

The problem is that if a max locked NFT's lock expires, then the NFT will be stuck forever.

## Vulnerability Detail
Any interaction with the NFT calls isApprovedOrOwner() for auth check. The max_lock() function is embedded into isApprovedorOwner() to facilitate re-locking on any modifications of the NFT position as well as whenever it uses vote/poke/claim in voter contract. 

This is the max lock logic :

```solidity

    function max_lock(uint _tokenId) public {  
        if(maxLockIdToIndex[_tokenId] !=0 && max_lock_enabled) {
            LockedBalance memory _locked = locked[_tokenId];
            uint unlock_time = (block.timestamp + MAXTIME) / WEEK * WEEK; 

            if(unlock_time > _locked.end) { 
            
                require(_locked.end > block.timestamp, 'Lock expired');
                require(_locked.amount > 0, 'Nothing is locked');
                require(unlock_time <= block.timestamp + MAXTIME, 'Voting lock can be 52 weeks max'); 

                _deposit_for(_tokenId, 0, unlock_time, _locked, DepositType.INCREASE_UNLOCK_TIME);
            }
        } 
    }
```

The relevant part here is this : ```require(_locked.end > block.timestamp, 'Lock expired');```
This means that if the max_lock() function is reached after the lock expires, it will always revert.

Now since the ```isApprovedOrOwner() => max_lock``` is looped in for all max locked nfts in all functions, this logic will always revert after the lock had expired.

- All position modification functions like increase_unlock_time will revert.
- The position can not be withdrawn now because withdraw also first calls into isApprovedOrOwner => max_lock which will revert, thus all locked lp tokens will be stuck forever
- Likewise all voting/poke/claim functions will revert in voter.sol => preveting the user from claiming any pending rewards 
- The position cannot be merged/split => because they too first call isApprovedOrOwner() => max_lock
- If the user tries to solve this problem by disabling the max_lock, Even disable_max_lock will revert because it also calls into isApprovedOrOwner before any other logic. 

Now the lock can expire naturally if the user did not interact with it for some time, and after that the lp tokens associated with the lock will be stuck forever in the votingescrow contract. 

This can happen for many users and tokenIDs. 

## Impact
Users locked tokens will be permanently stuck in the contract. 

High severity because this can happen under normal operations and easily brick funds of a lot of users. 

## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/VotingEscrow.sol#L918

## Tool used

Manual Review

## Recommendation
Remove the lock expired check from max_lock logic or refactor the max locking logic by calling it separately at required places and removing it from isApprovedOrOwner() flow. 