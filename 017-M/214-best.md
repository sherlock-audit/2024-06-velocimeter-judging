Slow Steel Seahorse

High

# Merging incorrectly increases contract's supply

## Summary
Merging incorrectly increases contract's supply 

## Vulnerability Detail
Within the `VotingEscrow` contract, user can merge two of their `veNFTs` into one. When doing so, one of them is burned and the necessary amount is added to the other one.

```solidity
    function merge(uint _from, uint _to) external {
        require(attachments[_from] == 0 && !voted[_from], "attached");
        require(_from != _to);
        require(_isApprovedOrOwner(msg.sender, _from));
        require(_isApprovedOrOwner(msg.sender, _to));

        LockedBalance memory _locked0 = locked[_from];
        LockedBalance memory _locked1 = locked[_to];
        uint value0 = uint(int256(_locked0.amount));
        uint end = _locked0.end >= _locked1.end ? _locked0.end : _locked1.end;

        locked[_from] = LockedBalance(0, 0);
        _checkpoint(_from, _locked0, LockedBalance(0, 0));
        _burn(_from);
        _deposit_for(_to, value0, end, _locked1, DepositType.MERGE_TYPE);
    }
```

The problem is that when doing so, the contract `supply` is increased.

```solidity
    function _deposit_for(
        uint _tokenId,
        uint _value,
        uint unlock_time,
        LockedBalance memory locked_balance,
        DepositType deposit_type
    ) internal {
        LockedBalance memory _locked = locked_balance;
        uint supply_before = supply;

        supply = supply_before + _value;
```

This is incorrect, as merging two `veNFTs` does not add any new tokens to the contract and hence the supply should not be increased.

Although, currently the `supply` value is never used, it should be used within the `Minter` contract in order to calculate the `circulatingSupply` (which it currently tries to do in a wrong way, check other issue).

By being able to inflate this value, user can cause DoS within the Minter due to underflow when calculating the `circulating_supply`

## Impact
DoS 

## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L787

## Tool used

Manual Review

## Recommendation
Do not increase contract's supply upon merging NFTs 