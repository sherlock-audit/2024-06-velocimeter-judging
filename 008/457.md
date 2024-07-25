Macho Snowy Shetland

Medium

# Reentrancy Vulnerability in `notifyRewardAmount` Function

## Summary
A potential reentrancy vulnerability was identified in the [notifyRewardAmount](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L485) function of the smart contract. This vulnerability can be exploited by an attacker to manipulate the index value, which impacts the reward distribution logic in functions such as `_updateFor`, `claimRewards`, and `getReward`. Proper reentrancy protection mechanisms need to be implemented to mitigate this risk.
## Vulnerability Detail
The `notifyRewardAmount` function lacks a reentrancy guard, making it susceptible to reentrancy attacks. During the execution of this function, an attacker can reenter and manipulate the index value. This manipulation can lead to inflated rewards, as the index is a critical parameter in calculating the distribution of rewards across different gauges.
## Impact
The impact of this vulnerability is significant. If an attacker successfully exploits the reentrancy vulnerability in `notifyRewardAmount`, they can artificially increase the `index` value. This increase will affect the reward calculations in the following functions:

`_updateFor`: Accrues rewards based on the manipulated index.
`claimRewards`: Claims rewards for gauges based on the manipulated index.
`getReward`: Calculates and distributes rewards to users based on the manipulated index.
By inflating the index, an attacker can disproportionately increase their rewards, leading to an unfair distribution of rewards and potential financial loss for other participants.
## Code Snippet
```solidity
function notifyRewardAmount(uint amount) external {
        require(msg.sender == minter,"not a minter");
        activeGaugeNumber = 0;
        currentEpochRewardAmount = amount;
    @>  _safeTransferFrom(base, msg.sender, address(this), amount); // transfer the distro in
        uint256 _ratio = amount * 1e18 / totalWeight; // 1e18 adjustment is removed during claim
        if (_ratio > 0) {
            index += _ratio;
        }
        emit NotifyReward(msg.sender, base, amount);
    }
```

```solidity
function _updateFor(address _gauge) internal {
    address _pool = poolForGauge[_gauge];
    uint256 _supplied = weights[_pool];
    if (_supplied > 0) {
        uint _supplyIndex = supplyIndex[_gauge];
        uint _index = index; // get global index0 for accumulated distro
        supplyIndex[_gauge] = _index; // update _gauge current position to global position
        uint _delta = _index - _supplyIndex; // see if there is any difference that need to be accrued
        if (_delta > 0) {
            uint _share = uint(_supplied) * _delta / 1e18; // add accrued difference for each supplied token
            if (isAlive[_gauge]) {
                claimable[_gauge] += _share;
            }
        }
    } else {
        supplyIndex[_gauge] = index; // new users are set to the default global state
    }
}
```
```solidity
function claimRewards(address[] memory _gauges, address[][] memory _tokens) external {
    for (uint i = 0; i < _gauges.length; i++) {
        IGauge(_gauges[i]).getReward(msg.sender, _tokens[i]);
    }
}
```

```solidity
function getReward(address account, address[] memory tokens) external lock {
    require(msg.sender == account || msg.sender == voter);
    _unlocked = 1;
    IVoter(voter).distribute(address(this));
    _unlocked = 2;

    for (uint i = 0; i < tokens.length; i++) {
        (rewardPerTokenStored[tokens[i]], lastUpdateTime[tokens[i]]) = _updateRewardPerToken(tokens[i], type(uint).max, true);

        uint _reward = earned(tokens[i], account);
        lastEarn[tokens[i]][account] = block.timestamp;
        userRewardPerTokenStored[tokens[i]][account] = rewardPerTokenStored[tokens[i]];
        if (_reward > 0) {
            if (tokens[i] == flow) {
                try IOptionToken(oFlow).mint(account, _reward) {} catch {
                    _safeTransfer(tokens[i], account, _reward);
                }
            } else {
                _safeTransfer(tokens[i], account, _reward);
            }
        }

        emit ClaimRewards(msg.sender, tokens[i], _reward);
    }

    uint _derivedBalance = derivedBalances[account];
    derivedSupply -= _derivedBalance;
    _derivedBalance = derivedBalance(account);
    derivedBalances[account] = _derivedBalance;
    derivedSupply += _derivedBalance;

    _writeCheckpoint(account, derivedBalances[account]);
    _writeSupplyCheckpoint();
}
```

## Tool used

Manual Review

## Recommendation

To mitigate the risk of reentrancy attacks, it is recommended to implement a reentrancy guard in the `notifyRewardAmount` function.