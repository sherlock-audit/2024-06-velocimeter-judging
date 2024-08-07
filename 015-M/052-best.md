Keen Black Dalmatian

Medium

# swap may be reverted if the input amount is not large enough, especially for low decimal tokens

## Summary
The swap fees will be sent to the `externalBribe`.  If the calculated swap fee is round down to zero, possible in low decimal tokens, the swap transaction will be reverted because `externalBribe` does not accept 0 fee.

## Vulnerability Detail
In swap(), the swap fees will be calculated based on the token's input amount. If the pool has one gauge, the swap fees will be sent to the `externalBribe::notifyRewardAmount()`. 
The vulnerability is that function `notifyRewardAmount` will be reverted if the fee amount is zero and the pool contract will send the swap fee if the inputAmount is larger than 0. So if the `amount0In` or `amount1In` is larger than 0 and the calculated swap fee is 0, the swap will be reverted.

The above scenario is unlikely triggered when the input token's decimal is high, for example 18. But when it comes to low decimal, it's possible.
For example:
GUSD, as one stable coin, it's decimal is 2. Checking the default swap fee ratio from the pariFactory, the default stable pool's swap fee ratio is 0.03%. Imagine we swap 30 dollar GUSD(3000GUSD) into another token, the swap fee will be zero.

```javascript
    function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external lock {
        ...
        if (hasGauge){
            if (amount0In != 0) _sendTokenFees(token0, fee0);
            if (amount1In != 0) _sendTokenFees(token1, fee1);
        } 
       ...
    }
    function notifyRewardAmount(address token, uint amount) external lock {
        require(amount > 0);
        ...
    }
contract PairFactory is IPairFactory, Ownable {

    constructor() {
        stableFee = 3; // 0.03%
        volatileFee = 25; // 0.25%
        deployer = msg.sender;
    }
    ...
}
```
### Poc
Add the below test case into FeesToBribes.t.sol. The test case will be reverted.
```javascript
    function testSwapAndClaimFees() public {
        createLock();
        vm.warp(block.timestamp + 1 weeks);

        voter.createGauge(address(pair), 0);
        address gaugeAddress = voter.gauges(address(pair));
        address xBribeAddress = voter.external_bribes(gaugeAddress);
        xbribe = ExternalBribe(xBribeAddress);

        Router.route[] memory routes = new Router.route[](1);
        routes[0] = Router.route(address(USDC), address(FRAX), true);

        assertEq(
            router.getAmountsOut(USDC_1, routes)[1],
            pair.getAmountOut(USDC_1, address(USDC))
        );

        uint256[] memory assertedOutput = router.getAmountsOut(3e3, routes);
        console.log("USDC Amount: ", USDC_1);
        USDC.approve(address(router), USDC_1);
        router.swapExactTokensForTokens(
            3e3,
            assertedOutput[1],
            routes,
            address(owner),
            block.timestamp
        );
}
```
## Impact
Pools with low decimal tokens may be reverted if the swap amount is not large enough.

## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Pair.sol#L295-L336
## Tool used

Manual Review

## Recommendation
If the calculated fee is 0, do not need to send fees to the `externalBribe`