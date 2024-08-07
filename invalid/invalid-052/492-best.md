Joyful Cinnabar Shark

High

# Possible Loss of funds due to non ERC721 std compliance

## Summary
Incorrect `function ownerOf()` implementation makes VotingEscrow non-ERC721 compliant, breaking composability leading to possible loss of funds by user and protocol.

Target: https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L190

## Vulnerability Detail
The ERC721 standard requires that ownership queries for NFTs assigned to the zero address must revert, indicating invalid/non-existent NFT. However, this VotingEscrow#ownerOf() implementation  violates this standard. This deviation risks causing functional and security issues in external systems that integrate with VotingEscrow optimistically relying on standard ERC721 behavior.
see impl below :point_below:
```solidity
    function ownerOf(uint _tokenId) public view returns (address) {
        return idToOwner[_tokenId];
    }
```

Illustrating possible loss of fund by protocol:
ExternalBribe.sol::getRewardForOwner()
```solidity
    // used by Voter to allow batched reward claims
    function getRewardForOwner(uint tokenId, address[] memory tokens) external lock  {
        require(msg.sender == voter);
  @>    address _owner = IVotingEscrow(_ve).ownerOf(tokenId); //@audit not checked for address(0)
        for (uint i = 0; i < tokens.length; i++) {
            uint _reward = earned(tokens[i], tokenId);
            lastEarn[tokens[i]][tokenId] = block.timestamp;
    @>      if (_reward > 0) _safeTransfer(tokens[i], _owner, _reward); // @audit send funds to zero address

            emit ClaimRewards(_owner, tokens[i], _reward); //@audit _owner (address(0))
        }
    }

```

## Note
1. This submission does not refer to the standard itself (for the sake of compliance only), rather non-compliance in this instance resulted  in the following impact [see Impact section](#impact)
2. ExternalBribe.sol is out of scope here. In this audit, its used as an accurate illustration how protocol and other potential users(integrators using smart contracts) will possibly lose funds, I did not recomend checking `address _owner` for address(0) in ExternalBribe.sol (as its OOS). Rather I recomend ERC721 standard compliance in VotingEscrow (Which is in scope).
3. The fact that Protocol itself dies not integrate well with this non-compliance is an undeniable evidence the possibility of misintegration by users. Hence very high chance of fund loss (already illustrated).
4. This is a High vilnerability.
## Impact
- Possible Loss of fund 
- Smart contract misbehaviour due to optimistic implementations



## Tool used

Manual Review

## Recommendation

Follow ERC721 standard and refactor all affected code parts.
