Itchy Blonde Cuckoo

Medium

# `abi.encodePacked()` should not be used with dynamic types when passing the result to a hash function such as `keccak256()`

## Summary
Using abi.encodePacked() can lead to hash collisions, potentially causing critical issues in smart contracts. This vulnerability is mitigated by using abi.encode(), which pads items to 32 bytes, preventing collisions.

## Vulnerability Detail
When using abi.encodePacked(), items are concatenated without padding. This can result in hash collisions where different input values produce the same hash output. For example:

abi.encodePacked(0x123, 0x456) results in 0x123456
abi.encodePacked(0x1, 0x23456) also results in 0x123456
In contrast, abi.encode() pads each item to 32 bytes, ensuring unique hashes for different inputs:

abi.encode(0x123, 0x456) results in distinct 32-byte padded values.

## Impact
Hash collisions can lead to significant vulnerabilities, such as unauthorized access or incorrect function execution, due to the misinterpretation of hashed values. This can compromise the integrity and security of the smart contract, leading to potential financial losses or unintended behaviors.


## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Pair.sol#L87-L91
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VeArtProxy.sol#L31-L34

## Tool used

Manual Review

## Recommendation
To prevent hash collisions:

- Use abi.encode() instead of abi.encodePacked(), as it pads each item to 32 bytes, ensuring unique hash outputs.
- Single Argument Handling: If there is only one argument to abi.encodePacked(), consider casting it to bytes() or bytes32().
- String/Bytes Concatenation: If all arguments are strings or bytes, use bytes.concat().