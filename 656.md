Main Golden Griffin

Medium

# Individual user can attach only one veNFT token to a gauge

## Summary

Users can attach their owned veNFT tokens to the gauges and they can owned several veNFT tokens.
But an user is allowed only to attach one onwed veNFT to a gauge.

## Vulnerability Detail

In `GaugeV4._deposit` function, user can attach the owned veNFT token to gauge from L481 and sets `tokenIds[account]` as `tokenId`.

https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/GaugeV4.sol#L469

```solidity
    function _deposit(address account, uint amount, uint tokenId) private {
        require(amount > 0);
        _updateRewardForAllTokens();

        _safeTransferFrom(stake, msg.sender, address(this), amount);
        totalSupply += amount;
        balanceOf[account] += amount;
        if (tokenId > 0) {
            require(IVotingEscrow(_ve).ownerOf(tokenId) == account);
L479:       if (tokenIds[account] == 0) {
                tokenIds[account] = tokenId;
L481:           IVoter(voter).attachTokenToGauge(tokenId, account);
            }
L483:       require(tokenIds[account] == tokenId);
        } else {
            tokenId = tokenIds[account];
        }
        [...]
    }
```

Users can owned several veNFT tokens.
If an user tries to attach another owned veNFT to gauge, it will be reverted from L483.

## Impact

Individual user can attach only one veNFT token to a gauge, not several tokens

## Code Snippet

https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/GaugeV4.sol#L469

## Tool used

Manual Review

## Recommendation

It is recommended to change the data type of `tokenIds[account]` to list type and push the attached `tokenId`s into `tokenIds[account]` variable.
