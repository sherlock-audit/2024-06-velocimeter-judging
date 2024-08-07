Furry Clear Chinchilla

High

# If someone deposits with a lock and still has a locked balance, it will be deleted

## Summary

If someone deposits with a lock and still has a locked balance, it will be deleted:

```solidity
    function depositWithLock(address account, uint256 amount, uint256 _lockDuration) external lock {
        require(msg.sender == account || isOToken[msg.sender],"Not allowed to deposit with lock");
        _deposit(account, amount, 0);

        if(block.timestamp >= lockEnd[account]) { // if the current lock is expired relased the tokens from that lock before loking again
            delete lockEnd[account];
            delete balanceWithLock[account]; 
        }
```

## Vulnerability Detail

One of the new features in `Gaugev4` is that a user can now deposit tokens with a lock:

```solidity
    function depositWithLock(address account, uint256 amount, uint256 _lockDuration) external lock {
        require(msg.sender == account || isOToken[msg.sender],"Not allowed to deposit with lock");
        _deposit(account, amount, 0);

        if(block.timestamp >= lockEnd[account]) { // if the current lock is expired relased the tokens from that lock before loking again
            delete lockEnd[account];
            delete balanceWithLock[account]; 
        }
```

But we have a problem here because if a user decides to deposit tokens with a lock and his previous period has expired, they balance are reset, and thus, the user loses all his tokens:

```solidity
if(block.timestamp >= lockEnd[account]) { // if the current lock is expired relased the tokens from that lock before loking again
            delete lockEnd[account];
            delete balanceWithLock[account]; 
        }
```
## Impact

The user will lose all his tokens from the previous lock period.
## Code Snippet

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/GaugeV4.sol#L447-L450

## Tool used

Manual Review

## Recommendation

Make it so that if the user has tokens, they accumulate in the new lock period and are not deleted.