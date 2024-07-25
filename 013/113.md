Digital Daisy Skunk

Medium

# No Owner Check

## Summary
In the `VotingEscrow::delegateBySig` function, there is no verification to ensure the recovered key matches the expected signer's key. This omission can lead to unauthorized actions by allowing signatures from incorrect or malicious signers to be accepted.

## Vulnerability Detail
The `delegateBySig` function performs various checks on the provided signature, such as ensuring the nonce is correct, the signature is not expired, and the signatory address is not zero. However, it does not check whether the recovered signatory address matches the expected owner address. This gap can result in mismatched public keys, indicating an incorrect or malicious signer.
This issue is part of Solodit's checklist.

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L1507-L1544

## Impact
Without verifying that the recovered signatory address matches the expected owner address, an attacker could potentially use a valid signature from a different address to perform unauthorized delegations. This could lead to unauthorized actions within the system.

## Code Snippet
```solidity
function delegateBySig(
	address delegatee,
	uint nonce,
	uint expiry,
	uint8 v,
	bytes32 r,
	bytes32 s
) public {
	bytes32 domainSeparator = keccak256(
		abi.encode(
			DOMAIN_TYPEHASH,
			keccak256(bytes(name)),
			keccak256(bytes(version)),
			block.chainid,
			address(this)
		)
	);
	bytes32 structHash = keccak256(
		abi.encode(DELEGATION_TYPEHASH, delegatee, nonce, expiry)
	);
	bytes32 digest = keccak256(
		abi.encodePacked("\x19\x01", domainSeparator, structHash)
	);
	address signatory = ecrecover(digest, v, r, s);
	require(
		signatory != address(0),
		"VotingEscrow::delegateBySig: invalid signature"
	);
	require(
		nonce == nonces[signatory]++,
		"VotingEscrow::delegateBySig: invalid nonce"
	);
	require(
		block.timestamp <= expiry,
		"VotingEscrow::delegateBySig: signature expired"
	);
	return _delegate(signatory, delegatee);
}
```

## Tool used
Manual Review

## Recommendation
Add an `owner` address parameter to the function signature and implement a check to ensure the public key derived from the signature matches the expected signer's public key (owner). For example:

```diff
function delegateBySig(
+	address owner,
	address delegatee,
	uint nonce,
	uint expiry,
	uint8 v,
	bytes32 r,
	bytes32 s
) public {
	bytes32 domainSeparator = keccak256(
		abi.encode(
			DOMAIN_TYPEHASH,
			keccak256(bytes(name)),
			keccak256(bytes(version)),
			block.chainid,
			address(this)
		)
	);
	bytes32 structHash = keccak256(
		abi.encode(DELEGATION_TYPEHASH, delegatee, nonce, expiry)
	);
	bytes32 digest = keccak256(
		abi.encodePacked("\x19\x01", domainSeparator, structHash)
	);
	address signatory = ecrecover(digest, v, r, s);
	require(
		signatory != address(0),
		"VotingEscrow::delegateBySig: invalid signature"
	);
+	require(
+		signatory == owner,
+		"VotingEscrow::delegateBySig: unauthorized signer"
+	);
	require(
		nonce == nonces[signatory]++,
		"VotingEscrow::delegateBySig: invalid nonce"
	);
	require(
		block.timestamp <= expiry,
		"VotingEscrow::delegateBySig: signature expired"
	);
	return _delegate(signatory, delegatee);
}
```