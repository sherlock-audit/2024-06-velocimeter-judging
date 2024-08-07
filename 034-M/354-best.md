Dizzy Fuzzy Nuthatch

Medium

# `Pair::_update` incorrectly saves observation's timestamp to current block.timestamp, possibly causing `Pair::sample` function to return for a longer hour range than expected.

## Summary

TWAP can be inconsistent due to `Pair::_update` updating newest observation's timestamp to the current block.timestamp.

## Vulnerability Detail

The `Pair::_update` function is responsible for updating the reserves after any interaction with the pool, it also handles the observations to make sure they are updated accordingly.

```solidity
function _update(
        uint balance0,
        uint balance1,
        uint _reserve0,
        uint _reserve1
    ) internal {
        uint blockTimestamp = block.timestamp;

        uint timeElapsed = blockTimestamp - blockTimestampLast; // overflow is desired
        if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
            reserve0CumulativeLast += _reserve0 * timeElapsed;
            reserve1CumulativeLast += _reserve1 * timeElapsed;
        }

        Observation memory _point = lastObservation();
        timeElapsed = blockTimestamp - _point.timestamp; // compare the last observation with current timestamp, if greater than 30 minutes, record a new event



        if (timeElapsed > periodSize) {
            observations.push(
                Observation(
                    blockTimestamp,
                    reserve0CumulativeLast,
                    reserve1CumulativeLast
                )
            );
        }
        reserve0 = balance0;
        reserve1 = balance1;
        blockTimestampLast = blockTimestamp;
        emit Sync(reserve0, reserve1);
    }
```

As we can see from the snippet above, if the time elapsed from the last point is greater than the period size (currently 30 minutes), we append a new observation with a the current timestamp.

The function is called every mint, burn and swap and sync to make sure every time anyone interacts in a way with the pair, if needed a new observation will be appended.

These observations are used by `Pair::sample` to get the Time weighted average price.

```solidity
    function sample(
        address tokenIn,
        uint amountIn,
        uint points,
        uint window
    ) public view returns (uint[] memory) {
        uint[] memory _prices = new uint[](points);

        uint length = observations.length - 1;
        uint i = length - (points * window);
        uint nextIndex = 0;
        uint index = 0;

        for (; i < length; i += window) {
            nextIndex = i + window;

            uint timeElapsed = observations[nextIndex].timestamp -
                observations[i].timestamp;

            uint _reserve0 = (observations[nextIndex].reserve0Cumulative -
                observations[i].reserve0Cumulative) / timeElapsed;
            uint _reserve1 = (observations[nextIndex].reserve1Cumulative -
                observations[i].reserve1Cumulative) / timeElapsed;

            _prices[index] = _getAmountOut(
                amountIn,
                tokenIn,
                _reserve0,
                _reserve1
            );
            // index < length; length cannot overflow
            unchecked {
                index = index + 1;
            }
        }
        return _prices;
    }
```

The way we decide how far behind we want to get the prices are points  (1 point == 30 minutes) and we loop over each point and get the the prices for it.

However, due to saving current timestamp into the new observation, not every observation will have a consistent gap of 30 minutes from the next/previous one, causing the `Pair::sample` function to return prices for period outside of the points provided.

## Proof of Concept

Consider the following updates and the calculations for TWAP:

1. **Initial Setup and Updates**:
    - **14:00**: `reserve0Cumulative = 100e18`, `reserve1Cumulative = 50e18`
    - **14:50**: `reserve0Cumulative = 220e18`, `reserve1Cumulative = 220e18`
    - **15:40**: `reserve0Cumulative = 500e18`, `reserve1Cumulative = 500e18`
    - **16:30**: `reserve0Cumulative = 1100e18`, `reserve1Cumulative = 1100e18`
    - **17:20**: `reserve0Cumulative = 2600e18`, `reserve1Cumulative = 2500e18`

2. **User calls `Pair::sample` at 17:30 passing 4 points expecting to receive average price from  15:30/17:30**:
    - **14:00 to 14:50 1st point (this should not get included but is)**: 
      - `timeElapsed = 50 minutes`
      - `reserve0 = (220e18 * 50 - 100e18 * 30) / 50 = 160e18`
      - `reserve1 = (220e18 * 50 - 50e18 * 30) / 50 = 190e18`
      - `Price = 120 / 170 = 0,84`
    - **14:50 to 15:40 2nd point**: 
      - `timeElapsed = 50 minutes`
      - `reserve0 = (500e18 * 50 - 220e18 * 50) / 50 = 280e18`
      - `reserve1 = (500e18 * 50 - 220e18 * 50) / 50 = 280e18`
      - `Price = 280 / 280 = 1.00`
    - **15:40 to 16:30 3rd point**: 
      - `timeElapsed = 50 minutes`
      - `reserve0 = (1100e18 * 50 - 500e18 * 50) / 50 = 600e18`
      - `reserve1 = (1100e18 * 50 - 500e18 * 50) / 50 = 600e18`
      - `Price = 600 / 600 = 1.00`
    - **16:30 to 17:20 4th point**: 
      - `timeElapsed = 50 minutes`
      - `reserve0 = (2600e18 * 50 - 1100e18 * 50) / 50 = 1500e18`
      - `reserve1 = (2500e18 * 50 - 1100e18 * 50) / 50 = 1400e18`
      - `Price = 1500 / 1400 = 1.07`
    - **Average Price** = `(0.70 + 1.00 + 1.00 + 1.07) / 4 = 0.9425`
    
However, as we can see from the calculations above, due to the inconsistent period saving, we are including the price from 14:00, which is and hour and a half, outside of the points window provided. 

This is especially an issue for more volatile pairs, if the price from 14:00 to 14:50 was much higher and then suddenly drops, the user will pay higher price than intended, due to that price being included.
 

## Impact

Inconsistent prices returned by `Pair::sample`, which is used heavily by `OptionTokenV4` to determine how much the user has to pay in order to excersise his option tokens.

## Code Snippet

https://github.com/sherlock-audit/2024-06-velocimeter/tree/main/v4-contracts/contracts/Pair.sol#L162-L179
https://github.com/sherlock-audit/2024-06-velocimeter/tree/main/v4-contracts/contracts/Pair.sol#L222-L246
https://github.com/sherlock-audit/2024-06-velocimeter/tree/main/v4-contracts/contracts/OptionTokenV4.sol#L323-L336
https://github.com/sherlock-audit/2024-06-velocimeter/tree/main/v4-contracts/contracts/OptionTokenV4.sol#L372-L388

## Tool used

Manual Review

## Recommendation

Consider modifying the `Pair::_update` function to change to observation timestamp to be `lastPoint,timestamp + periodSize`.

*NOTE*: It should also be modified to handle missed periods
