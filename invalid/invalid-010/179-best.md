Pet Stone Pelican

Medium

# The `GaugeV4.sol#swapOutRewardToken()` function does not check whether the reward token is whitelisted.

## Summary
Since the `GaugeV4.sol#swapOutRewardToken()` function does not check whether the token is whitelisted, an any token may be added as a reward token, which may cause unexpected actions.
## Vulnerability Detail
The `GaugeV4.sol#swapOutRewardToken()` function is used to swap the reward token.
```solidity
    function swapOutRewardToken(uint i, address oldToken, address newToken) external {
        require(msg.sender == IVotingEscrow(_ve).team(), 'only team');
        require(rewards[i] == oldToken);
        isReward[oldToken] = false;
        isReward[newToken] = true;
        rewards[i] = newToken;
    }
```
As you can see above this code snippet, this function does not check whether the `newToken` is whitelisted.

As shown in the `GaugeV4.sol#notifyRewardAmount()` function, only whitelisted tokens are added as reward tokens to the `rewards` list.
```solidity
    function notifyRewardAmount(address token, uint amount) external lock {
        SNIP...
        if (!isReward[token]) {
            require(IVoter(voter).isWhitelisted(token), "rewards tokens must be whitelisted");
            require(rewards.length < MAX_REWARD_TOKENS, "too many rewards tokens");
        }
        SNIP...
        if (!isReward[token]) {
            isReward[token] = true;
            rewards.push(token);
        }

        ...
    }
```

Additionally, Sherlock ReadMe clearly states that tokens used as reward/bribe must be registered in the whitelist.

> Tokens that are used for rewards/bribes that are not part of the pool needs to be whitelisted by protocol.

However, since the `swapOutRewardToken()` function does not check whether `newToken` is whitelisted, weird tokens may be added to the `rewards` list, which may have a negative impact on the protocol as a result.
## Impact
Weird tokens may be added to the `rewards` list, which may eventually have a negative impact on the protocol.
## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/GaugeV4.sol#L601-L607
## Tool used

Manual Review

## Recommendation
It is recommended to modify the `swapOutRewardToken()` function as follows:
```solidity
    function swapOutRewardToken(uint i, address oldToken, address newToken) external {
        require(msg.sender == IVotingEscrow(_ve).team(), 'only team');
        require(rewards[i] == oldToken);
+++     require(IVoter(voter).isWhitelisted(newToken), "rewards tokens must be whitelisted");
        isReward[oldToken] = false;
        isReward[newToken] = true;
        rewards[i] = newToken;
    }
```