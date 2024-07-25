High Hotpink Dove

High

# voters cannot disable max lock

## Summary
Voters can enable maxLock and this causes their voting power wouldn't decrease but they cannot disable maxLock

## Vulnerability Detail
**Textual PoC:**
Let's assume three voters lock their assets in ve,hence three nfts will be minted[1,2,3] and after that they [enable maxLock](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L883)

**Initial values**
max_locked_nfts corresponding values:



| index 0 | index 1 | index 2 |
| -------- | -------- | -------- |
| 1     | 2     | 3     |

maxLockIdToIndex corresponding values:

| index 1 | index 2 | index 3 |
| -------- | -------- | -------- |
| 1     | 2     | 3     |
 
 when owner of nft 3 want to disable maxLock he has to call `VotingEscrow::disable_max_lock`  in result :
**variable's values from line 897 til 901:**
* index = 2
* maxLockIdToIndex[3] = 0
* max_locked_nfts[2] = 3

max_locked_nfts corresponding values:



| index 0 | index 1 | index 2 |
| -------- | -------- | -------- |
| 1     | 2     | 3     |

maxLockIdToIndex corresponding values:

| index 1 | index 2 | index 3 |
| -------- | -------- | -------- |
| 1     | 2     | 0     |

finally
* maxLockIdToIndex[max_locked_nfts[2]] => maxLockIdToIndex[3] = 2 + 1
* last element of max_locked_nfts will be deleted

**Coded PoC:**

```solidity
    function testEnableAndDisableMaxLock() external {
        flowDaiPair.approve(address(escrow), TOKEN_1);
        uint256 lockDuration = 7 * 24 * 3600; // 1 week
        escrow.create_lock(400, lockDuration);
        escrow.create_lock(400, lockDuration);
        escrow.create_lock(400, lockDuration);

        assertEq(escrow.currentTokenId(), 3);
        escrow.enable_max_lock(1);
        escrow.enable_max_lock(2);
        escrow.enable_max_lock(3);


        assertEq(escrow.maxLockIdToIndex(1), 1);
        assertEq(escrow.maxLockIdToIndex(2), 2);
        assertEq(escrow.maxLockIdToIndex(3), 3);

        assertEq(escrow.max_locked_nfts(0), 1);
        assertEq(escrow.max_locked_nfts(1), 2);
        assertEq(escrow.max_locked_nfts(2), 3);

        escrow.disable_max_lock(3);

        assertEq(escrow.maxLockIdToIndex(1), 1);
        assertEq(escrow.maxLockIdToIndex(2), 2);
        assertEq(escrow.maxLockIdToIndex(3), 3);//mockLockIdToIndex has to be zero 

        assertEq(escrow.max_locked_nfts(0), 1);
        assertEq(escrow.max_locked_nfts(1), 2);
    }
```



## Impact
Voters cannot withdraw their assets from ve because every time they call `VotingEscrow::withdraw` their lockEnd will be decrease

## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L904

## Tool used

Manual Review

## Recommendation
```diff
    function disable_max_lock(uint _tokenId) external {
        assert(_isApprovedOrOwner(msg.sender, _tokenId));
        require(maxLockIdToIndex[_tokenId] != 0,"disabled");

        uint index =  maxLockIdToIndex[_tokenId] - 1;
        maxLockIdToIndex[_tokenId] = 0;

         // Move the last element into the place to delete
        max_locked_nfts[index] = max_locked_nfts[max_locked_nfts.length - 1];

+         if (index != max_locked_nfts.length - 1) {
+             uint lastTokenId = max_locked_nfts[max_locked_nfts.length - 1];
+             max_locked_nfts[index] = lastTokenId;
+             maxLockIdToIndex[lastTokenId] = index + 1;
+         }
        
+         maxLockIdToIndex[max_locked_nfts[index]] = 0;
        

-       maxLockIdToIndex[max_locked_nfts[index]] = index + 1;//@audit maxLockIdToIndex computes wrongly when lps want to disable last element in array
        
        // Remove the last element
        max_locked_nfts.pop();
    }
```

