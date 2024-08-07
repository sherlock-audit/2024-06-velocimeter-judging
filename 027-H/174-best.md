Loud Inky Shark

High

# paymentAmountToAddLiquidity is manipulatable, this allow LP stakers to get more tokens if oTokens are exercised

## Summary
 oTokens can be exercised to obtain either FLOW or veNFT based on the choice they made. They have to pay a premium with payment token to exercise the option. The `OptionTokenv4::getPaymentTokenAmountForExerciseLp()` function uses the reserves of the pool to calculate the price of FLOW and WETH. The prices is manipulatable, which affects the payment amount.
 
 From README.md:
 > Are there any limitations on values set by admins (or other roles) in the codebase, including restrictions on array lengths?
No
Example deployment script with the default start settings is available in our repository in script folder.
It is used as a template for our deployments.
It has two parts - first Deployment.s.sol script needs to be run then OFlowDeployment.s.sol script.

```solidity
        oFlow.setPairAndPaymentToken(IPair(pair), WETH);
```
Hence, the payment token is WETH.

## Vulnerability Detail
 The protocol adapts from Uniswapv2 pair contract, according to that docs:
 > In order to calculate these amounts, a contract must look up the current reserves of a pair, in order to understand what the current price is. However, it is not safe to perform this lookup and rely on the results without access to an external price.
 
In exercising oTokens to LP or veNFT the `OptionTokenv4::getPaymentTokenAmountForExerciseLp` is called:
```solidity
     // @notice Returns the amount in paymentTokens for a given amount of options tokens required for the LP exercise lp
    /// @param _amount The amount of options tokens to exercise
    /// @param _discount The discount amount
    function getPaymentTokenAmountForExerciseLp(uint256 _amount,uint256 _discount) public view returns (uint256 paymentAmount, uint256 paymentAmountToAddLiquidity)
    {
        paymentAmount = _discount == 0 ? 0 : getLpDiscountedPrice(_amount, _discount);
        //@audit-issue 
        (uint256 underlyingReserve, uint256 paymentReserve) = IRouter(router).getReserves(underlyingToken, paymentToken, false);
        paymentAmountToAddLiquidity = (_amount * paymentReserve) / underlyingReserve; 
    }
```
However, using `getReserves` as reserves of FLOW and WETH positions can fluctuate when the price of the pool changes. Therefore, the returned price `paymentAmountToAddLiquidity` of this function can be manipulated by swapping a significant amount of tokens into the pool. An attacker can utilize a flash loan to initiate a swap, thereby changing the price either upwards or downwards, and subsequently swapping back to repay the flash loan.

Example Lesser Fees Scenario:
Initial Pool: 1000 FLOW and 10 WETH (1 WETH = 100 FLOW)
1) Alice frontrun and borrow 10 WETH: Using a flash loan.
2) Within that same transaction Alice swap 10 WETH for FLOW: Pool now has 500 FLOW and 20 WETH.
3) Bob Exercise option with the drastically low payment amount
4) Alice payback flashloan
5) Protocol and liquidity providers don't earn as many fees

This can be escalated to steal the "exerciser" WETH tokens:

You can do the opposite including frontrunning to drastically increase the price, hence increasing WETH revenue earned draining user WETH. Since the attacker has LP in the protocol, the LP is worth more.

Second Steal Scenario

1) Alice wallet has a 10 staked LP, 10_000 FLOW in wallet
2) Bob exercise option
3) Alice swap 10_000 FLOW for WETH: Pool now has 500 FLOW and 1 WETH
4) Bob has 100 WETH in his wallet
5) With the manipulated pool, Bob pays 100 WETH as payment
6) Protocol earns more, meaning Alice LP worth more
7) Alice withdraw LP tokens

## Impact
1) The user interacting with the protocol will use lose WETH.
2) Lesser WETH revenue is earned for liquidity providers
3) Velocimiter get lesser WETH revenue sent to treasury


## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/OptionTokenV4.sol#L354
## Tool used

Manual Review

## Recommendation
One could build a TWAP-style contract that tracks a time-weighted-average reserve amount (instead of the price in traditional TWAPs). An alternative is to use Chainlink Oracle to get the price.