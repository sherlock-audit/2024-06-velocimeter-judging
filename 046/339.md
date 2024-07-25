Atomic Citron Fly

Medium

# Arithmetic Error Risk due to Rounding Down in `VotingEscrow::_find_block_epoch()` Function

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L1002

## Summary
The VotingEscrow::_find_block_epoch() function suffers from an arithmetic error due to rounding down in the midpoint calculation during binary search. This is caused by the integer division in Solidity, which truncates results towards zero.

## Vulnerability Detail

The VotingEscrow::_find_block_epoch() function contains a critical arithmetic error caused by rounding down due to integer division.

```javascript
 function _find_block_epoch(
        uint _block,
        uint max_epoch
    ) internal view returns (uint) {
        // Binary search
        uint _min = 0;
        uint _max = max_epoch;
        for (uint i = 0; i < 128; ++i) {
            // Will be always enough for 128-bit numbers
            if (_min >= _max) {
                break;
            }
            uint _mid = (_min + _max + 1) / 2;
            if (point_history[_mid].blk <= _block) {
                _min = _mid;
            } else {
                _max = _mid - 1;
            }
        }
        return _min;
    }
```

based on this line

```javascript
uint _mid = (_min + _max + 1) / 2;
```

The problem here is that integer division in Solidity truncates the result towards zero. This behavior causes the midpoint (_mid) calculation to round down, which is inappropriate for the binary search algorithm used in this function. Proper binary search algorithms require rounding up to ensure correct convergence.

This error can lead to inaccurate estimates of the timestamp for a given block number. Specifically, the binary search algorithm in this function is designed to find the closest epoch in the `point_history` array based on the provided block number. If the calculations contain arithmetic errors, the function may return an incorrect epoch value.

Proof of Concept:

Consider the following scenario:

* _min = 3
* _max = 5
Using the current implementation:

```javascript
uint _mid = (_min + _max + 1) / 2; // (3 + 5 + 1) / 2 = 4
```
If integer division rounds down, the midpoint (_mid) is calculated incorrectly, potentially leading to an inaccurate search result.

for more info on binary see [here](https://ethereum.stackexchange.com/questions/7169/binary-search-in-solidity-arrays)


## Impact
Inaccurate midpoint calculations will lead to incorrect epoch estimates, affecting the accuracy of timestamp retrieval for a given block number.

## Code Snippet

```javascript
 function _find_block_epoch(uint _block, uint max_epoch) internal view returns (uint) {
        // Binary search
        uint _min = 0;
        uint _max = max_epoch;
        for (uint i = 0; i < 128; ++i) {
            // Will be always enough for 128-bit numbers
            if (_min >= _max) {
                break;
            }
            uint _mid = (_min + _max + 1) / 2;
            if (point_history[_mid].blk <= _block) {
                _min = _mid;
            } else {
                _max = _mid - 1;
            }
        }
        return _min;
    }
```

## Tool used

Manual Review

## Recommendation
To correct this error, modify the midpoint calculation to ensure it rounds up. This adjustment is crucial for the binary search algorithm to work accurately.

```javascript
uint _mid = (_min + _max) / 2 + 1;

```
