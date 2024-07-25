Generous Malachite Rook

High

# createGauge is permissionless , allowing attackers to push numerous pools into the voter.

## Summary
[Voter.sol::createGauge](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L323-L378)  is permissionless , attackers to push numerous pools into the voter , result in  [Voter.sol::updateAll](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L509-L511) out of gas.

## Vulnerability Detail
From [Voter.sol::createGauge](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L349-L351) we can see
```solidity
  if (msg.sender != governor && msg.sender != emergencyCouncil) { // gov can create for any pool, even non-Velocimeter pairs
      require(isPair, "!_pool");
      require(IGaugePlugin(gaugePlugin).checkGaugeCreationAllowance(msg.sender, tokenA, tokenB), "!whitelistedForGaugeCreation");
  }
```

If current `msg.sender` is not governor and emergencyCouncil , protocol first check weather `pair` is created by `PairFactory.sol`. And then check `tokenA` and `tokenB` is WhitelistedForGaugeCreation.

[PairFactory.sol::createPair](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/factories/PairFactory.sol#L108-L124) is also  permissionless anyone can call `createPair` to create a `pair`. The only check is that `tokenA` and `tokenB` pair is not exist.
[PairFactory.sol::createPair](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/factories/PairFactory.sol#L111-L112)
```solidity
        require(token0 != address(0), 'ZA'); // Pair: ZERO_ADDRESS
        require(getPair[token0][token1][stable] == address(0), 'PE');
```

thus the last check and only check is [GaugePlugin.sol::checkGaugeCreationAllowance](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/GaugePlugin.sol#L55-L63)
```solidity
  function checkGaugeCreationAllowance(
      address caller,
      address tokenA,
      address tokenB
  ) external view returns (bool) {
      return
          isWhitelistedForGaugeCreation[tokenA] ||
          isWhitelistedForGaugeCreation[tokenB];   <@
  }
```
From the code, we can see that `checkGaugeCreationAllowance` requires only one token that is `isWhitelistedForGaugeCreation`. An attacker can exploit this by using a single whitelisted token along with any other token to create numerous pools, overwhelming the voter system and potentially causing a denial of service (DoS).

test:
```solidity
  function testPermissionlessCreateGauge() public {
      address random = makeAddr("random");
      vm.startPrank(random);

      for(uint256 i;i<10;i++){
          MockERC20 tokenB = new MockERC20("name","symbol",18);
          address pair = factory.createPair(address(FLOW),address(tokenB),false);
          voter.createGauge(pair,0);
      }

      console2.log(voter.length());
  }
```
out:
```shell
Ran 1 test for test/Voter.t.sol:VoterTest
[PASS] testPermissionlessCreateGauge() (gas: 87659391)
Logs:
  10

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 22.06ms (6.84ms CPU time)
```

## Impact
system dos

## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L323-L378
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/GaugePlugin.sol#L55-L63

## Tool used

Manual Review

## Recommendation
```diff
@@ -58,8 +58,8 @@ contract GaugePlugin is IGaugePlugin, Ownable {
         address tokenB
     ) external view returns (bool) {
         return
-            isWhitelistedForGaugeCreation[tokenA] ||
-            isWhitelistedForGaugeCreation[tokenB];
+            isWhitelistedForGaugeCreation[tokenA] &&
+            isWhitelistedForGaugeCreation[tokenB]; //@audit & ?
     }
```
