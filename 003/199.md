Helpful Oily Corgi

Medium

# `OptionTokenV4.exerciseLP`'s `addLiquidity` lack of slippage can be abused to make victims exercise for a lower liquidity than intended

## Summary

`OptionTokenV4.exerciseLP` uses spot reserves and a fixed `_amount` by sandwiching an exercise operation, as well as due to the pool being imbalanced, the depositor can receive less liquidity than intended, burning more OptionTokens for less LP tokens

https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/OptionTokenV4.sol#L690-L700

```solidity
        (, , lpAmount) = IRouter(router).addLiquidity( /// @audit I need to do the math here to see the gain when
            underlyingToken,
            paymentToken,
            false,
            _amount,
            paymentAmountToAddLiquidity,
            1,
            1,
            address(this),
            block.timestamp
        );
```

## Vulnerability Detail

`OptionTokenV4.exerciseLP` has a slippage check on the maximum price paid to exercise the option

But there is no check that the `lpAmount` is within the bounds of what the user intended

The `Pool.mint` formula for liquidity to be minted is as follows:

https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/Pair.sol#L262-L263

```solidity
            liquidity = Math.min(_amount0 * _totalSupply / _reserve0, _amount1 * _totalSupply / _reserve1);

```

To calculate the correct amount of `paymentReserve` to add to the pool, spot reserves are checked

https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/OptionTokenV4.sol#L354-L355


```solidity
        (uint256 underlyingReserve, uint256 paymentReserve) = IRouter(router).getReserves(underlyingToken, paymentToken, false);
 paymentAmountToAddLiquidity = (_amount * paymentReserve) / underlyingReserve;
```

This means that spot reserves are read and are supplied in a proportional way, this is rational and superficially correct

However, `_amount` for the OptionToken is a fixed value, meaning that the amount of liquidity we will get is directly related to how "imbalanced the pool is"

When a pool is perfectly balance (e.g. both reserves are in the same proportion), we will have the following math:

Start balances
tokenA: 1000000000000000000 (1e18)
tokenB: 1000000000000000000 (1e18)

New Deposit: 1000000000000000000 (1e18)
New tokens minted: 1000000000000000000 (1e18)

Meaning we get a proportional amount

However, if we start imbalancing the pool by adding more `underlyingToken`, then the amount of `paymentAmountToAddLiquidity` will be reduced, meaning we will be using the same `_amount` of `underlying` but we will receive less total LP tokens

This can happen naturally, if the pool is imbalanced and can also be exploited by an attacker to cause the ExerciseLP to be less effective than intended

## Impact

Less LP tokens will be produced from burning the OptionTokens, resulting in a loss of the value of the OptionToken

## Code Snippet

Run this POC to see how a deposit of `amount = 1e18` will result in very different amounts of liquidity out

By purposefully front-running and imbalancing the pool, an attacker can make the exercised options massively less valuable, in this example the result is 9% of the unmanipulated value

```solidity
address a;
    address b;
    function getInitializable() external view returns (address, address, bool) {
        return (a, b, false);
    }
    function getFee(address ) external view returns (uint256) {
        return 25;
    }
    function isPaused(address ) external view returns (bool) {
        return false;
    }



    // forge test --match-test test_swapAndSee -vv
     function test_swapAndSee() public {
        MockERC20 tokenA = new MockERC20("a", "A", 18);
        MockERC20 tokenB = new MockERC20("b", "B", 18);
        a = address(tokenA);
        b = address(tokenB);

        tokenA.mint(address(this), 1_000_000e18);
        tokenB.mint(address(this), 1_000_000e18);

        // Setup Pool
        Pair pool = new Pair();



        // Classic stable Pool
        // TODO
        // pool.initialize(address(tokenA), address(tokenB), false);

        uint256 initial = 100e18;
        tokenA.transfer(address(pool), initial);
        tokenB.transfer(address(pool), initial);
        pool.mint(address(this));

        // We assume we'll deposit 1e18 from the option
        // We'll take spot of the other amount
        // And see how much liquidity we get
        uint256 snapshot = vm.snapshot();
        (uint256 underlyingReserve, uint256 paymentReserve, ) = pool.getReserves();

        // Amt * payRes / underlyingRes
        uint256 paymentAmountToAddLiquidity = (1e18 * paymentReserve) / underlyingReserve;
        console2.log("paymentAmountToAddLiquidity Initial", paymentAmountToAddLiquidity);
        
        tokenA.transfer(address(pool), 1e18);
        tokenB.transfer(address(pool), paymentAmountToAddLiquidity);
        uint256 balanceB4 = pool.balanceOf(address(this));
        pool.mint(address(this));
        uint256 poolMinted = pool.balanceOf(address(this)) - balanceB4; 
        console2.log("poolMinted Initial", poolMinted);
        vm.revertTo(snapshot);

        // swap
        uint256 counter = 1000;
        while(counter > 0) {
            // By swapping more of underlyingReserve, we make `paymentReserve` cheaper and we make them get less liquidity
            // this wastes their `_amount` which is limited

            // Swap 0
            tokenA.transfer(address(pool), 1e18);
            uint256 toSwapOut = pool.getAmountOut(1e18, address(tokenA));
            pool.swap(0, toSwapOut, address(this), hex"");

            --counter;
        }

        // Basically same as above, but with altered reserves
        (underlyingReserve, paymentReserve, ) = pool.getReserves();

        // Amt * payRes / underlyingRes
        paymentAmountToAddLiquidity = (1e18 * paymentReserve) / underlyingReserve;
        console2.log("paymentAmountToAddLiquidity After", paymentAmountToAddLiquidity);
        
        tokenA.transfer(address(pool), 1e18);
        tokenB.transfer(address(pool), paymentAmountToAddLiquidity);
        balanceB4 = pool.balanceOf(address(this));
        pool.mint(address(this));
        poolMinted = pool.balanceOf(address(this)) - balanceB4; 
        console2.log("poolMinted After", poolMinted);
    }
```

Full File is here:
https://gist.github.com/GalloDaSballo/d40d7a1d1b2a481450f44ebade421d14

## Tool used

Manual Review

## Recommendation

Add an additional slippage check for `exerciseLP` to check that `lpAmount` is above a slippage threshold
