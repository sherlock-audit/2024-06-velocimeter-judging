Furry Clear Chinchilla

High

# Attackers can manipulate oracle by exploiting the initial block of a new period

## Summary

Attackers can manipulate oracle by exploiting the initial block of a new period. 

## Vulnerability Detail

**Note**: This vulnerability is inspired by this issue: https://solodit.xyz/issues/m-10-oracle-may-be-attacked-if-an-attacker-can-pump-the-tokens-for-the-entire-block-code4rena-canto-canto-git

The `_update` function in `Pair.sol` contract updates the cumulative reserves and logs an observation when the `periodSize` is exceeded:

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

The other function we need to look at is `current()`:

```solidity
    function current(address tokenIn, uint amountIn) external view returns (uint amountOut) {
        Observation memory _observation = lastObservation();
        (uint reserve0Cumulative, uint reserve1Cumulative,) = currentCumulativePrices();
        if (block.timestamp == _observation.timestamp) {
            _observation = observations[observations.length-2];
        }

        uint timeElapsed = block.timestamp - _observation.timestamp;
        uint _reserve0 = (reserve0Cumulative - _observation.reserve0Cumulative) / timeElapsed;
        uint _reserve1 = (reserve1Cumulative - _observation.reserve1Cumulative) / timeElapsed;
        amountOut = _getAmountOut(amountIn, tokenIn, _reserve0, _reserve1);
    }
```

This function retrieves the last observation and calculates the current price based on cumulative reserves. 

The problem here is that the attacker can manipulates the token price within the first block of a new period, and the TWAP can capture this manipulated price.

Let's see the following scenario:
- The `periodSize` is set to 1800 seconds, and we assume it’s the first block after this period.
- The attacker pumps the token price for one block.
- This causes `reserve0Cumulative` or `reserve1Cumulative` to skyrocket due to the pumped price.
- The next call to `current` uses the last observation, which includes the manipulated price.
- As the `timeElapsed` is less than one block, the TWAP calculation is based on this manipulated price, leading to inaccurate token valuations.

See how they did it in Uniswap [Uniswap v2 TWAP Example](https://github.com/Uniswap/v2-periphery/blob/master/contracts/examples/ExampleOracleSimple.sol).
## Impact

The attacker can pump the price by trading a large amount of tokens within a single block, thereby inflating the token price. This way he can manipulate the oracle and steal a large amount of tokens.
## Code Snippet

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Pair.sol#L161-L179
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Pair.sol#L197-L209

## Tool used

Manual Review

## Recommendation

Use cumulative reserves differences from the last update to the current time over the full `periodSize`, rather than a single block.

Refer to Uniswap v2 TWAP oracle implementation: [Uniswap v2 TWAP Example](https://github.com/Uniswap/v2-periphery/blob/master/contracts/examples/ExampleOracleSimple.sol).