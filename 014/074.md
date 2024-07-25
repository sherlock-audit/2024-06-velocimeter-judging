Dandy Shamrock Sheep

High

# Missed Emissions for Skipped Weeks in Minter Contract

## Summary
The Minter contract's update_period function does not account for scenarios where multiple weeks pass between calls, potentially leading to missed token emissions and inconsistent reward distribution.

## Vulnerability Detail
The update_period function is designed to distribute weekly emissions of FLOW tokens. However, it only emits tokens for a single week, even if multiple weeks have passed since the last update. This can result in significant loss of emissions if the function is not called every week.
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Minter.sol#L114-L117

## Impact
This vulnerability can lead to:
* Fewer tokens being minted than intended
* Inconsistent reward distribution
* Potential economic imbalance in the protocol

## Code Snippet
POC:
```solidity
function testMissedEmissionsForSkippedWeeks() public {
    initializeVotingEscrow();

    // Initial state
    uint256 initialSupply = FLOW.totalSupply();
    uint256 weeklyEmission = minter.weekly_emission();

    // Simulate 3 weeks passing
    vm.warp(block.timestamp + 3 * ONE_WEEK);

    // Call update_period
    minter.update_period();

    // Check the new supply
    uint256 newSupply = FLOW.totalSupply();

    // The supply should have increased by 3 * weeklyEmission, but it only increased by weeklyEmission
    assertEq(newSupply, initialSupply + weeklyEmission, "Supply should have increased by one week's emission");
    assertTrue(newSupply < initialSupply + 3 * weeklyEmission, "Supply should have increased by less than three weeks' emission");

    console.log("Initial Supply:", initialSupply);
    console.log("New Supply:", newSupply);
    console.log("Actual Increase:", newSupply - initialSupply);
    console.log("Expected Increase:", 3 * weeklyEmission);
}
```

## Tool used

Manual Review

## Recommendation
Modify the update_period function to account for multiple weeks:
```solidity
function update_period() external returns (uint) {
    uint _period = active_period;
    if (block.timestamp >= _period + WEEK && initializer == address(0)) {
        uint256 weeksElapsed = (block.timestamp - _period) / WEEK;
        _period = _period + (weeksElapsed * WEEK);
        active_period = _period;
        
        for (uint i = 0; i < weeksElapsed; i++) {
            uint256 weekly = weekly_emission();
            uint _teamEmissions = (teamRate * weekly) / (PRECISION - teamRate);
            uint _required =  weekly + _teamEmissions;
            uint _balanceOf = _flow.balanceOf(address(this));
            if (_balanceOf < _required) {
                _flow.mint(address(this), _required - _balanceOf);
            }
            require(_flow.transfer(teamEmissions, _teamEmissions));
            _checkpointRewardsDistributors();
            _flow.approve(address(_voter), weekly);
            _voter.notifyRewardAmount(weekly);
            emit Mint(msg.sender, weekly, circulating_supply());
        }
    }
    return _period;
}
```
