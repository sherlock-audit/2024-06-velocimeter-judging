Stale Sangria Troll

Medium

# Desync between bribes being paid and gauge distribution allows voters to receive bribes without triggering emissions

## Summary
The desynchronization between the awarding of bribes by the `ExternalBribe` contract and the distribution of emissions by the `Voter` contract allows voters to receive bribes without their gauge receiving the corresponding emissions.

## Vulnerability Detail
`ExternalBribe` awards bribes based on the voting power at `EPOCH_END - 1`. However, `Minter.update_period()` and `Voter.distribute()` can be called sometime after this epoch ends.
This creates a window where voters can manipulate their votes, resulting in the following scenario
Some voters may switch their vote before their weight influences emissions, causing the voters to receive bribes, but the bribing protocols to not have their gauge receive the emissions.

For example, a project `XYZ` offers the best yield for 3 weeks. 
A voter can vote for `XYZ` during this period and then switch their vote to another gauge immediately after the third voting period ends, allowing them to:
- Receive the third week's bribes for `XYZ`.
- Redirect their vote to another gauge for emissions, resulting in a misalignment between bribes received and emissions distributed.

## Impact
This desynchronization issue leads to:
- Misallocation of rewards, where bribes are paid out without corresponding emissions.
- Potential loss for bribing protocols, as they do not receive the expected gauge emissions.
- Manipulation of the voting and reward distribution system, undermining its integrity and fairness.

## Code Snippet
distribute; https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L550
_vote;  https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L276
earned; https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/ExternalBribe.sol#L212

## Tool used
Manual Review

## Recommendation
It is recommended to introduce a post-voting window before emissions distribution. Specifically, allowing a `1-hour` post-voting window for protocols to trigger distribution can mitigate the desynchronization issue.
This ensures that:
- Votes cannot be changed immediately after the reward epoch ends.
- The alignment between bribe payments and emissions distribution is maintained.
- The integrity of the voting and reward distribution system is preserved, ensuring fair allocation of rewards.
