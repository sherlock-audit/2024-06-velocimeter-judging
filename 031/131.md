Boxy Plastic Turtle

High

# Non-EOA recipients will lock veNFTs forever, affecting users (`VotingEscrow::_mint`)

## Summary

## Vulnerability Detail

The VotingEscrow contract implements a voting escrow system where users can lock their tokens to receive voting power in the form of non-fungible tokens (veNFTs). The contract includes functions for minting, transferring, and managing these veNFTs. However, there is a critical issue in the `_mint()` function that could lead to permanent loss of veNFTs.

The [`_mint()` function](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L483-L492) is responsible for creating new veNFTs and assigning them to a recipient address. It is called internally by other functions such as `_create_lock()` when users lock their tokens. The function performs basic checks and operations:

```solidity
function _mint(address _to, uint _tokenId) internal returns (bool) {
    assert(_to != address(0));
    _moveTokenDelegates(address(0), delegates(_to), _tokenId);
    _addTokenTo(_to, _tokenId);
    emit Transfer(address(0), _to, _tokenId);
    return true;
}
```

However, this function lacks a crucial check required by the ERC721 standard. When minting to a contract address, it should verify if the recipient contract implements the `onERC721Received()` function. This check is essential to ensure that the recipient contract is capable of handling ERC721 tokens.

The absence of this check means that veNFTs can be minted to contract addresses that are not equipped to handle them. In such cases, the veNFTs would be permanently locked in these contracts, effectively lost to the users who initiated the minting process.

This issue is particularly concerning because the VotingEscrow system is designed to give users voting power proportional to their locked tokens. If users lose access to their veNFTs due to this vulnerability, they would also lose their voting power and any associated rewards, potentially disrupting the governance and incentive mechanisms of the entire system.

The exact similar issue you can find here: https://solodit.xyz/issues/m-06-voting-tokens-may-be-lost-when-given-to-non-eoa-accounts-code4rena-velodrome-finance-velodrome-finance-git

## Impact
Users may permanently lose access to their veNFTs, voting power, and associated rewards if they mint to a contract address that doesn't properly implement ERC721 token handling. This could lead to significant financial losses and undermine the integrity of the voting system. The impact is severe as it affects core functionalities of the protocol and can result in irreversible loss of assets.

## Proof of Concept
1. Alice deploys a simple contract `NonERC721Receiver` that does not implement the `onERC721Received()` function:
   ```solidity
   contract NonERC721Receiver {
       // This contract does not implement onERC721Received
   }
   ```

2. Alice calls the `create_lock()` function of the VotingEscrow contract, providing the address of `NonERC721Receiver` as the recipient:
   ```solidity
   votingEscrow.create_lock_for(1000, 52 weeks, address(nonERC721Receiver));
   ```

3. The `create_lock_for()` function internally calls `_create_lock()`, which in turn calls `_mint()`.

4. The `_mint()` function successfully mints the veNFT to the `NonERC721Receiver` contract address.

5. The veNFT is now locked in the `NonERC721Receiver` contract, and Alice has no way to access or transfer it.


## Code Snippet
- https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L483-L492



## Tools Used
Manual Review

## Recommendation
To fix this vulnerability, the `_mint()` function should be modified to include a check for contract recipients. Here's the recommended change:

```diff
function _mint(address _to, uint _tokenId) internal returns (bool) {
    assert(_to != address(0));
    _moveTokenDelegates(address(0), delegates(_to), _tokenId);
    _addTokenTo(_to, _tokenId);
    emit Transfer(address(0), _to, _tokenId);

+   if (_isContract(_to)) {
+       require(
+           IERC721Receiver(_to).onERC721Received(msg.sender, address(0), _tokenId, "") ==
+           IERC721Receiver(_to).onERC721Received.selector,
+           "ERC721: transfer to non ERC721Receiver implementer"
+       );
+   }

    return true;
}
```

This change ensures that when minting to a contract address, the contract must implement `onERC721Received()` correctly, preventing veNFTs from being locked in contracts that cannot handle them.