Fit Burgundy Narwhal

Medium

# Gauge's reward rate and period duration can be griefed

## Summary
Anybody can send reward tokens to a gauge via `notifyRewardAmount()` when there're little rewards left or when the current period finish has ended and extend the reward period by another week and dilute the reward rate.
## Vulnerability Detail
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/GaugeV4.sol#L563-L599
```solidity
    function notifyRewardAmount(address token, uint amount) external lock {
        require(token != stake);
        require(amount > 0);
        if (!isReward[token]) {
            require(IVoter(voter).isWhitelisted(token), "rewards tokens must be whitelisted");
            require(rewards.length < MAX_REWARD_TOKENS, "too many rewards tokens");
        }
        if (rewardRate[token] == 0) _writeRewardPerTokenCheckpoint(token, 0, block.timestamp);
        (rewardPerTokenStored[token], lastUpdateTime[token]) = _updateRewardPerToken(token, type(uint).max, true);

        if (block.timestamp >= periodFinish[token]) {
            uint256 balanceBefore = IERC20(token).balanceOf(address(this));
            _safeTransferFrom(token, msg.sender, address(this), amount);
            uint256 balanceAfter = IERC20(token).balanceOf(address(this));
            amount = balanceAfter - balanceBefore;
            rewardRate[token] = amount / DURATION;
        } else {
            uint _remaining = periodFinish[token] - block.timestamp;
            uint _left = _remaining * rewardRate[token];
            require(amount > _left); 
            uint256 balanceBefore = IERC20(token).balanceOf(address(this));
            _safeTransferFrom(token, msg.sender, address(this), amount);
            uint256 balanceAfter = IERC20(token).balanceOf(address(this));
            amount = balanceAfter - balanceBefore;
            rewardRate[token] = (amount + _left) / DURATION;
        }
        require(rewardRate[token] > 0);
        uint balance = IERC20(token).balanceOf(address(this));
        require(rewardRate[token] <= balance / DURATION, "Provided reward too high");
        periodFinish[token] = block.timestamp + DURATION;
        if (!isReward[token]) {
            isReward[token] = true;
            rewards.push(token);
        }

        emit NotifyReward(msg.sender, token, amount);
    }
```

Whether the current period has finished or not, anybody can deposit rewards to a gauge and dilute the reward rate and extend the period finish time.
## Impact
Diluted reward rate for current stakers in the gauge. Also extended waiting times for current period to finish. 
## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/GaugeV4.sol#L563-L599
## Tool used
Manual Review
## Recommendation
Consider implementing validation to ensure that this function is callable by anyone only if the previous reward period has already expired. Otherwise, when there is an active reward period, it may only be called by the designated reward distributor account.
