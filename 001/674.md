Fluffy Sage Okapi

Medium

# Restarting the gauge after killing it will make some important functions to revert


## Summary &Vulnerability Detail

In Voter.sol when calling killGaugeTotally() will delete the gauge from the mapping isGauge
" delete isGauge[_gauge];"
Then, when restarting the same gauge again it is not set back to true.
This value is  crucial in setting setExternalBribeFor() for the gauge as it is required to be true
```solidity
 function setExternalBribeFor(address _gauge, address _external) external onlyEmergencyCouncil {
        require(isGauge[_gauge]);
        _setExternalBribe(_gauge, _external);
    }
```
and it is checked also for :
-  _vote() so if it is not set to true that will stop voting to that gauge.
-  emitDeposit()
- attachTokenToGauge()
So, not setting this value right will make all these functions revert. 
## Impact
This bug will cause many functionalities to revert harming both the users and the protocol.

## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L394C5-L406C1

## Tool used

Manual Review

## Recommendation
In restartGauge() 
```diff
function restartGauge(address _gauge) external {
        if (msg.sender != emergencyCouncil) {
            require(
                IGaugePlugin(gaugePlugin).checkGaugeRestartAllowance(msg.sender, _gauge)
            , "Restart gauge not allowed");
        }
        require(!isAlive[_gauge], "gauge already alive");
        isAlive[_gauge] = true;
+      isGauge[_gauge] = true;
        address _pair = IGauge(_gauge).stake(); // TODO: add test cases
        try IPair(_pair).setHasGauge(true) {} catch {}
        emit GaugeRestarted(_gauge);
    }
```

