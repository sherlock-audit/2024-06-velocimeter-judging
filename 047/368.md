Sweet Lemonade Lynx

Medium

# Mismatch Between Comment and Code in _transferFrom Function Allows Transfers to Zero Address

## Summary

The `_transferFrom` function in the `VotingEscrow.sol` contract does not properly handle transfers to the zero address despite a comment indicating it should throw an error in such cases.

## Vulnerability Detail

The `_transferFrom` function is expected to prevent transfers to the zero address, as indicated by the function [comment](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L346). However, the implementation lacks the necessary check, allowing transfers to the zero address to pass successfully.

## Code Snippet

```solidity
  /// @dev Throws unless `msg.sender` is the current owner, an authorized operator, or the approved address for this NFT.
    ///      Throws if `_from` is not the current owner.
    ///      Throws if `_to` is the zero address.
    ///      Throws if `_tokenId` is not a valid NFT.
    /// @notice The caller is responsible to confirm that `_to` is capable of receiving NFTs or else
    ///        they maybe be permanently lost.
    /// @param _from The current owner of the NFT.
    /// @param _to The new owner.
    /// @param _tokenId The NFT to transfer.
    function transferFrom(
        address _from,
        address _to,
        uint _tokenId
    ) external {
        _transferFrom(_from, _to, _tokenId, msg.sender);
    }


/// @dev Exeute transfer of a NFT.
///      Throws unless `msg.sender` is the current owner, an authorized operator, or the approved
///      address for this NFT. (NOTE: `msg.sender` not allowed in internal function so pass `_sender`.)
///      Throws if `_to` is the zero address.
///      Throws if `_from` is not the current owner.
///      Throws if `_tokenId` is not a valid NFT.
//  @audit transfer to zero address passes successfully.
function _transferFrom(
    address _from,
    address _to,
    uint _tokenId,
    address _sender
) internal {
    require(attachments[_tokenId] == 0 && !voted[_tokenId], "attached");
    // Check requirements
    require(_isApprovedOrOwner(_sender, _tokenId));
    require(_from != _to,"from == to");

    // Clear approval. Throws if `_from` is not the current owner
    _clearApproval(_from, _tokenId);
    // Remove NFT. Throws if `_tokenId` is not a valid NFT
    _removeTokenFrom(_from, _tokenId);
    // auto re-delegate
    _moveTokenDelegates(delegates(_from), delegates(_to), _tokenId);
    // Add NFT
    _addTokenTo(_to, _tokenId);
    // Set the block of ownership transfer (for Flash NFT protection)
    ownership_change[_tokenId] = block.number;

    //unlock split
    blockedSplit[_tokenId] = false;

    // Log the transfer
    emit Transfer(_from, _to, _tokenId);
}
```

## Impact

This vulnerability allows for unintended transfers to the zero address, potentially leading to the loss of NFTs and associated voting power within the system.

## Tool used

Manual Review

## Recommendation

Add a check to ensure that `_to` is not the zero address within the `_transferFrom` function:

```solidity
require(_to != address(0), "Transfer to zero address");
```
