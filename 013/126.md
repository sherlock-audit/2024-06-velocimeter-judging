Boxy Plastic Turtle

Medium

# Users will be unable to delegate votes via signature in `VotingEscrow::delegateBySig()`

## Summary 

## Vulnerability Detail

The `VotingEscrow` contract implements a voting mechanism where users can delegate their voting power to other addresses. One of the key features is the ability to delegate votes using a signature, implemented in the [`delegateBySig()` function](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L1507-L1545). This function is designed to allow users to delegate their votes without directly interacting with the contract, by providing a signed message.

The delegation process relies on EIP-712 for structured data hashing and signing. A critical component of this process is the [`DOMAIN_TYPEHASH`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L1259), which is used to create a domain separator for the contract. This domain separator is then used in the signature verification process.

However, the current implementation of the `DOMAIN_TYPEHASH` in the VotingEscrow contract is incorrect:

```solidity
bytes32 public constant DOMAIN_TYPEHASH = keccak256("EIP712Domain(string name,uint256 chainId,address verifyingContract)");
```

This definition is missing the `string version` field, which is required by the EIP-712 standard. The `delegateBySig()` function, however, constructs the `domainSeparator` with the `version` field included:

```solidity
bytes32 domainSeparator = keccak256(
    abi.encode(
        DOMAIN_TYPEHASH,
        keccak256(bytes(name)),
        keccak256(bytes(version)),
        block.chainid,
        address(this)
    )
);
```

This mismatch between the `DOMAIN_TYPEHASH` definition and its usage in `domainSeparator` construction will cause all signature verifications in `delegateBySig()` to fail, effectively breaking the functionality of delegating votes via signature.

## Impact
Users will be unable to delegate their votes using signatures, as all calls to `delegateBySig()` will revert with an "invalid signature" error. This significantly impairs a core feature of the voting system, preventing users from delegating their votes without directly interacting with the contract. It could lead to reduced participation in the voting process, especially for users who rely on signature-based delegation for convenience or gas efficiency.

## Proof of Concept
1. Alice wants to delegate her votes to Bob without directly interacting with the contract.
2. Alice signs a message with her intention to delegate to Bob, following the EIP-712 standard.
3. Someone (could be Alice or a relayer) calls `delegateBySig()` with Alice's signed message.
4. The function attempts to verify the signature using the incorrectly constructed `domainSeparator`.
5. The signature verification fails due to the mismatch in the `DOMAIN_TYPEHASH`.
6. The transaction reverts with an "invalid signature" error.
7. Alice's votes remain undelegated, contrary to her intention.


## Code Snippet
- https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L1507-L1545
- https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L1259


## Tools Used
Manual review

## Recommendation
To fix this issue, update the `DOMAIN_TYPEHASH` to include the `string version` field and define `DOMAIN_SEPARATOR` as an immutable variable initialized in the constructor. Here's the recommended change:

```diff
- bytes32 public constant DOMAIN_TYPEHASH = keccak256("EIP712Domain(string name,uint256 chainId,address verifyingContract)");
+ bytes32 private immutable DOMAIN_SEPARATOR;

constructor(address base_token_addr, address lp_token_addr, address art_proxy, address _owner) {
    // ... existing code ...

+   bytes32 domainTypeHash = keccak256(
+       "EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"
+   );
+   DOMAIN_SEPARATOR = keccak256(
+       abi.encode(
+           domainTypeHash,
+           keccak256(bytes(name)),
+           keccak256(bytes(version)),
+           block.chainid,
+           address(this)
+       )
+   );

    // ... existing code ...
}

function delegateBySig(
    address delegatee,
    uint nonce,
    uint expiry,
    uint8 v,
    bytes32 r,
    bytes32 s
) public {
    bytes32 structHash = keccak256(
        abi.encode(DELEGATION_TYPEHASH, delegatee, nonce, expiry)
    );
    bytes32 digest = keccak256(
        abi.encodePacked("\x19\x01", DOMAIN_SEPARATOR, structHash)
    );

    // ... existing code ...
}
```