Breezy Chrome Baboon

Medium

# Delegating votes to address(0) causes permanent loss of votes and NFT transfer issues

## Summary
Delegating votes to address(0) can cause the user to permanently lose their votes and prevent them from transferring their NFT.

## Vulnerability Detail

The `delegateBySig` function is missing a check to avoid delegating votes to the `address(0)`. This is correctly handled when the user uses the `delegate` function, as if `delegatee == address(0)`, the delegatee is set to `msg.sender`. However, this check is missing when a signature is used.

```solidity
    function delegateBySig(
        address delegatee,
        uint nonce,
        uint expiry,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) public {
        //@audit-issue this issue is not fixed: https://code4rena.com/reports/2022-08-nounsdao#h-01-erc721checkpointable-delegatebysig-allows-the-user-to-vote-to-address-0-which-causes-the-user-to-permanently-lose-his-vote-and-cannot-transfer-his-nft

        bytes32 domainSeparator = keccak256(
            abi.encode(
                DOMAIN_TYPEHASH,
                keccak256(bytes(name)),
                keccak256(bytes(version)),
                block.chainid,
                address(this)
            )
        );
        bytes32 structHash = keccak256(
            abi.encode(DELEGATION_TYPEHASH, delegatee, nonce, expiry)
        );
        bytes32 digest = keccak256(
            abi.encodePacked("\x19\x01", domainSeparator, structHash)
        );

        // .....
    }

```

When `delegatee == address(0)` in the `delegateBySig` function, the` _delegates[delegator]` will be set to` address(0)`, and when `getCurrentVotes` is called, it will always return 0. This can be problematic if the user votes for another address or tries to transfer their NFT, because the `_moveDelegates` function will fail due to an overflow.

The mentioned issue was found in a previous audit of NausDao, where the VotingEscrow contract is forked. [link](https://code4rena.com/reports/2022-08-nounsdao#h-01-erc721checkpointable-delegatebysig-allows-the-user-to-vote-to-address-0-which-causes-the-user-to-permanently-lose-his-vote-and-cannot-transfer-his-nft)

## Impact
Permanently lose of votes and prevent from transferring of NFT.

## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L1508

## Tool used

Manual Review

## Recommendation
Add a restriction to ensure `delegate != address(0)`