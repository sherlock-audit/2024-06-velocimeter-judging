Immense Maroon Tiger

High

# _deposit_for in VotingEscrow do not check the _value lead to symbol flip

## Summary
Issue High: _deposit_for in VotingEscrow do not check the _value lead to symbol flip

## Vulnerability Detail

In the contract VotingEscrow.sol, the function `_deposit_for` do not check the value of `_value`，so a big uint can lead to symbol flip

the `_value` is send from function `deposit_for`, which can be controled by attacker. and in line `_locked.amount += int128(int256(_value));`,`_value` will be convert to int128, if the `_value` is larger than 2^127-1 the `_locked.amount` will decrease

in this case ,the supply will increase and amount of `_locked` is decreased
 
[VotingEscrow](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L777-L811)
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
        LockedBalance memory old_locked;
        (old_locked.amount, old_locked.end) = (_locked.amount, _locked.end);
        // Adding to existing lock, or if a lock is expired - creating a new one
        _locked.amount += int128(int256(_value)); // if the _value is larger than 2^127-1 the _locked.amount will decrease
        if (unlock_time != 0) {
            _locked.end = unlock_time;
        }
        locked[_tokenId] = _locked;

        // Possibilities:
        // Both old_locked.end could be current or expired (>/< block.timestamp)
        // value == 0 (extend lock) or value > 0 (add to lock or extend lock)
        // _locked.end > block.timestamp (always)
        _checkpoint(_tokenId, old_locked, _locked);

        address from = msg.sender;
        if (_value != 0 && deposit_type != DepositType.MERGE_TYPE && deposit_type != DepositType.SPLIT_TYPE) {
            assert(IERC20(lpToken).transferFrom(from, address(this), _value));
        }

        emit Deposit(from, _tokenId, _value, _locked.end, deposit_type, block.timestamp);
        emit Supply(supply_before, supply_before + _value);
    }
```


## Impact

property loss


## Code Snippet

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L777-L811

## Tool used
Manual Review

## Recommendation

Check the validity of the _value.
