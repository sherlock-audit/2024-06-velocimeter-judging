Active Lace Hippo

Medium

# Partial Withdrawals From A `Gauge` Result In Token Tradeability

## Summary

By withdrawing just `0 wei` of underlying [`stake`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/Gauge.sol#L15C30-L15C35) from a [`Gauge`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/Gauge.sol), an account regains complete control of their supposedly locked [`VotingEscrow`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol) token whilst maintaining exposure to incentivization rewards.

## Vulnerability Detail

When depositing into a [`Gauge`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/Gauge.sol) (or [`GaugeV4`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/GaugeV4.sol)), depositors may receive token incentives in exchange for locking a  [`VotingEscrow`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol).

Locking is achieved by incrementing the [`attachments`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/VotingEscrow.sol#L1185C5-L1193C6) prop. In the presence of non-zero [`attachments`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/VotingEscrow.sol#L1167), the following restrictions are imposed on a [`VotingEscrow`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/VotingEscrow.sol) token instance:

1. The token cannot be [transferred](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/VotingEscrow.sol#L321).
2. The token cannot be [withdrawn](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/VotingEscrow.sol#L957).
3. The token cannot be [merged](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/VotingEscrow.sol#L1196C9-L1196C71).
4. The token cannot be [split](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/VotingEscrow.sol#L1220).

However, these constraints are easily subverted.

Notice that on [`withdrawToken(uint256,uint256)`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/Gauge.sol#L482C14-L482C54), regardless of the amount of underlying [`stake`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/Gauge.sol#L15C30-L15C35) that is withdrawn, the [`attachments`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/VotingEscrow.sol#L1167) for the locked token will continue to be removed:

```solidity
function withdrawToken(uint amount /* @audit i.e. `0 wei` */, uint tokenId) public lock {
    _updateRewardForAllTokens();

    totalSupply -= amount;
    balanceOf[msg.sender] -= amount; /// @audit balanceOf_could_be_far_in_excess_of_withdrawn_amount
    _safeTransfer(stake, msg.sender, amount);

@>  if (tokenId > 0) {
@>      require(tokenId == tokenIds[msg.sender]);
@>      tokenIds[msg.sender] = 0;
@>      IVoter(voter).detachTokenFromGauge(tokenId, msg.sender); /// @audit token_is_detached_even_though_majority_stake_remains
@>  } else {
        tokenId = tokenIds[msg.sender];
    }

    uint _derivedBalance = derivedBalances[msg.sender];
    derivedSupply -= _derivedBalance;
    _derivedBalance = derivedBalance(msg.sender);
    derivedBalances[msg.sender] = _derivedBalance;
    derivedSupply += _derivedBalance;

    _writeCheckpoint(msg.sender, derivedBalances[msg.sender]);
    _writeSupplyCheckpoint();

    IVoter(voter).emitWithdraw(tokenId, msg.sender, amount);
    emit Withdraw(msg.sender, tokenId, amount);
}
```

An attacker may therefore make a significant stake to a [`VotingEscrow`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol) token, then immediately withdraw negligible size of that stake to gain exposure to locking rewards whilst gaining immunity from the intended opportunity cost of locking the token.

## Impact

The [`attachments`]() mechanism can be trivially subverted to regain full access to a previously-attached [`VotingEscrow`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol) token whilst continuing to be registered for rewards.

This weakens the incentive structure, since accounts continue to receive rewards even though they may not be long-term aligned with the protocol (for example, the deposited [`VotingEscrow`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol) token could be burned and the initial depositor would continue to be rewarded as if they were staked).

## Code Snippet

```solidity
function withdrawToken(uint amount, uint tokenId) public lock {
    _updateRewardForAllTokens();

    totalSupply -= amount;
    balanceOf[msg.sender] -= amount;
    _safeTransfer(stake, msg.sender, amount);

    if (tokenId > 0) {
        require(tokenId == tokenIds[msg.sender]);
        tokenIds[msg.sender] = 0;
        IVoter(voter).detachTokenFromGauge(tokenId, msg.sender);
    } else {
        tokenId = tokenIds[msg.sender];
    }

    uint _derivedBalance = derivedBalances[msg.sender];
    derivedSupply -= _derivedBalance;
    _derivedBalance = derivedBalance(msg.sender);
    derivedBalances[msg.sender] = _derivedBalance;
    derivedSupply += _derivedBalance;

    _writeCheckpoint(msg.sender, derivedBalances[msg.sender]);
    _writeSupplyCheckpoint();

    IVoter(voter).emitWithdraw(tokenId, msg.sender, amount);
    emit Withdraw(msg.sender, tokenId, amount);
}
```

https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/Gauge.sol#L482C14-L482C54

## Tool used

Manual Review

## Recommendation

Only execute [`detachTokenFromGauge(uint256,address)`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/Voter.sol#L444C14-L444C65) once the position is completely closed:

```diff
balanceOf[msg.sender] -= amount;
_safeTransfer(stake, msg.sender, amount);

if (tokenId > 0) {
    require(tokenId == tokenIds[msg.sender]);
+
+   /// @notice Tokens may only be detached once they
+   /// @notice represent no claim to underlying stake.
+   if (balanceOf[msg.sender] == 0) {
+       tokenIds[msg.sender] = 0;
+       IVoter(voter).detachTokenFromGauge(tokenId, msg.sender);
+   }
-   tokenIds[msg.sender] = 0;
-   IVoter(voter).detachTokenFromGauge(tokenId, msg.sender);
} else {
    tokenId = tokenIds[msg.sender];
}
```
