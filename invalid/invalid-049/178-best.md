Pet Stone Pelican

Medium

# Missing of dupplication check in `Voter.sol#addFactory()` function

## Summary
Since the dupplication check is missed from the `Voter.sol#addFactory()` function, it cannot be removed if the factory is exploited.
## Vulnerability Detail
The admin uses the `addFactory()` function to add `_pairFactory` and `_gaugeFactory`.
```solidity
    function addFactory(address _pairFactory, address _gaugeFactory) external onlyEmergencyCouncil {
        require(_pairFactory != address(0), 'addr 0');
        require(_gaugeFactory != address(0), 'addr 0');
158:    require(!isGaugeFactory[_gaugeFactory], 'g.fact true');

        factories.push(_pairFactory);
        gaugeFactories.push(_gaugeFactory);
        isFactory[_pairFactory] = true;
        isGaugeFactory[_gaugeFactory] = true;

        emit FactoryAdded(msg.sender, _pairFactory, _gaugeFactory);
    }
```
As you can see, in the provided code snippet, only the check for `_gaugeFactory` is performed, and the check for `_pairFactory` is not performed.

On the other hand, if the factory is exploited or similar, the administrator can use the `removeFactory()` function to remove the factory.
```solidity
    function removeFactory(uint256 _pos) external onlyEmergencyCouncil {
        require(_pos < factoryLength() && _pos < gaugeFactoriesLength(), '_pos out of range');
        address oldPF = factories[_pos];
        address oldGF = gaugeFactories[_pos];
191:    require(isFactory[oldPF], 'factory false');
192:    require(isGaugeFactory[oldGF], 'g.fact false');
        factories[_pos] = address(0);
        gaugeFactories[_pos] = address(0);
195:    isFactory[oldPF] = false;
196:    isGaugeFactory[oldGF] = false;

        emit FactoryRemoved(msg.sender, _pos);
    }
```
Here, unexpected errors may occur due to incorrect processing of the `addFactory()` function.

Attack Vector
  1.The administrator adds `pairFactoryA` and `gaugeFactoryA` at pos=1.
    At this time, `isFactory[pairFactoryA]` and `isGaugeFactory[gaugeFactoryA]` are set to `true`.
  2.Next, Let's say an administrator accidentally or intentionally adds `pairFactoryA` and `gaugeFactoryB` at pos=3.
    In #L158, `isGaugeFactory[gaugeFactoryB]` is `false`, so the function is not reverted and `isGaugeFactory[gaugeFactoryB]` is set to `true`.
  3.After, Let's say the administrator uses the `removeFactory(1)` function to remove `pairFactoryA` and `gaugeFactoryA`.
    At this time, `isFactory[pairFactoryA]` and `isGaugeFactory[gaugeFactoryA]` are set to `false` in #L195 and #L196.
  4.Then, Let’s say the administrator calls `removeFactory(3)` function.
    At this time, Since `isFactory[pairFactoryA]` was set to `false` in step3, it is reverted in #L191.
    As a result, the factory cannot be removed.
## Impact
The `removeFactory()` function may not work as intended, and as a result, if the factory is abused or similar, it may not be removed unexpectedly.
## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L155-L166
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L187-L199
## Tool used

Manual Review

## Recommendation
It is recommended to modify the `addFactory()` function as follows:
```solidity
    function addFactory(address _pairFactory, address _gaugeFactory) external onlyEmergencyCouncil {
        require(_pairFactory != address(0), 'addr 0');
        require(_gaugeFactory != address(0), 'addr 0');
+++     require(!isFactory[_pairFactory], 'factory true');
        require(!isGaugeFactory[_gaugeFactory], 'g.fact true');

        SNIP...
    }
```