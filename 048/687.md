Zany Sepia Millipede

Medium

# The current value of a Pair is not always returning a 30-minute TWAP and can be manipulated

## Summary
The current function returns a current TWAP. It fetches the last observation and calculates the
TWAP between the last observation. The observation is pushed every thirty minutes. However, the interval
between current timestamp and the last observation varies a lot. In most cases, the TWAP interval is shorter than
30 minutes.

## Vulnerability Detail
```solidity
// gives the current twap price measured from amountIn * tokenIn gives amountOut
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
If the last observation is newly updated, the timeElapsed will be much shorter than 30 minutes. The cost of price
manipulation is cheaper in this case.
Assume the last observation is updated at T. The exploiter can launch an attack at T + 30_MINUTES - 1
1. At T + 30_MINUTES - 1, the exploiter tries to manipulate the price of the pair. Assume the price is moved
to 100x.
2. At T + 30_MINUTES, the exploiter pokes the pair. The pair push an observation with the price = 100x.
3. At T + 30_MINUTES + 1, the exploiter tries to exploit external protocols. The current function fetches the
last observation and calculates the TWAP between the last observation and the current price. It ends up
calculating the two-block-interval TWAP.
## Impact
TWAP can be manipulated.Also pair won't always return a 30 minute TWAP.
## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Pair.sol#L197C5-L210C1
## Tool used

Manual Review

## Recommendation
A solution is a lastTWAP where always calculate the TWAP based on the last two observations. As
the time elapsed between two observations is always larger than 30 minutes, the manipulation cost is guaranteed
to be high.
