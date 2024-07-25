Sweet Lemonade Lynx

Medium

# Potential DoS in Pair.sol Core Functions Due to Overflow Vulnerability in _update Function

## Summary

The `_update` function in `Pair.sol` is prone to an overflow condition, which is intentionally desired for certain calculations. This can lead to a denial of service (DoS) in the `mint`, `burn`, and `swap` functions, potentially locking all funds in the contract. This issue arises because the contract appears to be a clone of Uniswap v2, which uses Solidity 0.6.6 where arithmetic operations overflow by default. However, the current contract uses Solidity 0.8.0, where such operations revert on overflow.

## Vulnerability Detail

The `_update` function in `Pair.sol` contains arithmetic operations that rely on overflow behavior, inherited from Uniswap v2, which was developed using Solidity 0.5.16. In Solidity ^0.8.0, arithmetic operations revert on overflow, causing the function to fail and disrupt core functionalities such as `mint`, `burn`, and `swap`.

The relevant code is as [follows](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Pair.sol#L163-L168):

In this function, the calculation of `timeElapsed` and the subsequent updates to cumulative reserves are designed to overflow:

```solidity
uint timeElapsed = blockTimestamp - blockTimestampLast; // overflow is desired
if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
    reserve0CumulativeLast += _reserve0 * timeElapsed;
    reserve1CumulativeLast += _reserve1 * timeElapsed;
}
```

Due to Solidity 0.8.0's default behavior of reverting on overflow, this will cause the function to revert, breaking the contract's functionality.


This issue is similar to one described in detail in a report [here](https://solodit.xyz/issues/m-3-jalapair-potential-permanent-dos-due-to-overflow-sherlock-jala-swap-git).


## Impact

This issue can cause a permanent denial of service (DoS) for the pool. The core functionalities such as `mint`, `burn`, and `swap` would fail, leading to all funds being locked within the contract. This effectively renders the contract unusable.

## Code Snippet

```solidity
function _update(uint balance0, uint balance1, uint _reserve0, uint _reserve1) internal {
    uint blockTimestamp = block.timestamp;
    uint timeElapsed = blockTimestamp - blockTimestampLast; // overflow is desired
    if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
        reserve0CumulativeLast += _reserve0 * timeElapsed;
        reserve1CumulativeLast += _reserve1 * timeElapsed;
    }

    Observation memory _point = lastObservation();
    timeElapsed = blockTimestamp - _point.timestamp; // compare the last observation with current timestamp, if greater than 30 minutes, record a new event
    if (timeElapsed > periodSize) {
        observations.push(Observation(blockTimestamp, reserve0CumulativeLast, reserve1CumulativeLast));
    }
    reserve0 = balance0;
    reserve1 = balance1;
    blockTimestampLast = blockTimestamp;
    emit Sync(reserve0, reserve1);
}
```

## Tool Used

Manual Review

## Recommendation

Use an `unchecked` block to allow for the desired overflow behavior without causing a revert. This ensures the function continues to operate as intended without breaking due to Solidity 0.8.0's overflow checks.
