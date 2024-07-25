Square Arctic Chicken

High

# First  epoch weekly Emission does not factor in all active gauges in the protocol leading to dilution of emission rewards

## Summary

per the protocol [docs](https://paragraph.xyz/@velocimeter/velocimeter-v4#h-emissions-grow-with-growth) is
> Emissions are now a factor of how many gauges there are.

The first epochs emission does not factor in the number of active gauges in the protocol leading to dilution of the minted FLOW emission for the first epoch


## Vulnerability Detail

The weekly emissions is evaluated based on the `activeGaugeNumber` variable, which is increased whenever the keeper calls `Voter::distribute()` because the rewards are sent to all gauges that are alive. The variable is updated only after only after the distribution is done because the `claimable[_gauge]` is zero for all gauges in the first epoch (and will remain zero in the first epoch even if anyone calls `distribute()` before the end of the first epoch)

```solidity
    function distribute(address _gauge) public lock {
        IMinter(minter).update_period();
        _updateFor(_gauge); // should set claimable to 0 if killed
        uint _claimable = claimable[_gauge];
        if (_claimable > IGauge(_gauge).left(base) && _claimable / DURATION > 0) {
            claimable[_gauge] = 0;
  ->        if((_claimable * 1e18) / currentEpochRewardAmount > minShareForActiveGauge) {
  ->            activeGaugeNumber += 1;
            }

            IGauge(_gauge).notifyRewardAmount(base, _claimable);
            emit DistributeReward(msg.sender, _gauge, _claimable);
        }
    }
```

Since, during the first epoch, the `activeGaugeNumber`  is zero, when the keeper calls `Voter::distribute()` at the end of the first epoch, the weekly emissions is just the `weeklyPerGauge` which is currently set to 2000 * 1e18. Hence diluting the amount of emissions received by each for gauge for the first epoch and causing each gauge to receive less than the promised weekly emissions for a gauge. 
It becomes severe if in the first epoch there are at least 2 gauges (the more the number of gauges, the fewer the reward)


```solidity
File: Minter.sol
098:     function weekly_emission() public view returns (uint) {
099:         uint256 numberOfGauges = _voter.activeGaugeNumber();
100:         if(numberOfGauges == 0) { 
101:  ->         return weeklyPerGauge;
102:         }
103:  ->     return weeklyPerGauge * numberOfGauges;
104:     }

```

**CODED POC**


To run the POC, i set up the test suite to have more than one gauge hence
- Modify the `BaseTest.sol` file as shown below
- Create a file in the `test` folder and paste the `BrokenEmission.t.sol` file
- in your terminal run:
    - `forge test --mt testFirstEpochEmissionsIsBroken -vvv`

<details>
<summary>Modify the `BaseTest.sol` file as shown below</summary>

```diff
File: BaseTest.sol
01: pragma solidity 0.8.13;
02: 
03: import "forge-std/Test.sol";
04: import "forge-std/console2.sol";
05: import "solmate/test/utils/mocks/MockERC20.sol";

SNIP...

30: import "utils/TestWETH.sol";
31: 
32: abstract contract BaseTest is Test, TestOwner {
33:     uint256 constant USDC_1 = 1e6;


SNIP...

64:     Pair pair3;
65:     Pair flowDaiPair;
+ 66:     Pair fraxDaiPair;

+154:     function deployCustomPairWithOwner(address _owner) public {
+ 155:         vm.startPrank(_owner, _owner);
+ 156:         FLOW.approve(address(router), 300*TOKEN_1);
+ 157:         DAI.approve(address(router), 300*TOKEN_1);
+ 158:         router.addLiquidity(address(FLOW), address(DAI), false, TOKEN_1, TOKEN_1, 0, 0, address(_owner), block.timestamp);
+ 159: 
+ 160:         FRAX.approve(address(router), TOKEN_1);
+ 161:         DAI.approve(address(router), TOKEN_1);
+ 162:         router.addLiquidity(address(FRAX), address(DAI), false, TOKEN_1, TOKEN_1, 0, 0, address(_owner), block.timestamp);
+ 163:         vm.stopPrank();
+ 164: 
+ 165:         address address4 = factory.getPair(address(FLOW), address(DAI), false);
+ 166:         flowDaiPair = Pair(address4);
+ 167: 
+ 168:         address address5 = factory.getPair(address(FRAX), address(DAI), false);
+ 169:         fraxDaiPair = Pair(address5);
+ 170:     }


```
</details>

<details>
<summary>Create a file in the test  folder and  paste the `BrokenEmission.t.sol` file</summary>

```solidity
pragma solidity 0.8.13;

import "./BaseTest.sol";
import "contracts/factories/GaugeFactoryV4.sol";

contract BrokenEmission is BaseTest {
    VotingEscrow escrow;
    GaugeFactoryV4 gaugeFactory;
    BribeFactory bribeFactory;
    Voter voter;
    GaugeV4 gauge;
    GaugeV4 gauge2; // added
    RewardsDistributor distributor;
    Minter minter; 
    address alice = address(0x03);
    address bob = address(0x04);

    
    function setUp() public {
        deployOwners(); // deploy 3 owners
        deployCoins(); // deploy USDC, DAI, FRAX, FLOW, LR, WETH, stake
        mintStables(); // mint 1 USDC, 1 DAI and 1 FRAX to owners above

        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 2 * TOKEN_1M; // use 1/2 for veNFT position
        amounts[1] = TOKEN_1M;
        mintFlow(owners, amounts); // mint these amount of flow to the first two owners

        FLOW.mint(alice, TOKEN_100K);
        FLOW.mint(bob, TOKEN_100K);
        DAI.mint(alice, TOKEN_100K);
        DAI.mint(bob, TOKEN_100K);

        address[] memory tokens = new address[](4);
        tokens[0] = address(USDC);
        tokens[1] = address(FRAX);
        tokens[2] = address(DAI);
        tokens[3] = address(FLOW);

        deployPairFactoryAndRouter();
        deployCustomPairWithOwner(address(owner));

        VeArtProxy artProxy = new VeArtProxy();
        escrow = new VotingEscrow(address(FLOW), address(flowDaiPair), address(artProxy), owners[0]);

        distributor = new RewardsDistributor(address(escrow));


        gaugeFactory = new GaugeFactoryV4();
        bribeFactory = new BribeFactory();
        gaugePlugin = new GaugePlugin(address(FLOW), address(WETH), owners[0]);
        voter = new Voter(address(escrow), address(factory), address(gaugeFactory), address(bribeFactory), address(gaugePlugin));
        factory.setVoter(address(voter));
  
        
        escrow.setVoter(address(voter));

        minter = new Minter(address(voter), address(escrow), address(distributor)); // added

        // initialize voter with minter
        voter.initialize(tokens, address(minter)); // whitelist tokens and the minter

        
        distributor.setDepositor(address(minter)); // added
        FLOW.setMinter(address(minter)); // added

        deployOptionTokenV3WithOwner( // oFlowV3 deployed
            address(owner),
            address(gaugeFactory),
            address(voter),
            address(escrow)
        );


        gaugeFactory.setOFlow(address(oFlowV3));

        // deployPairWithOwner(address(owner));
        
        address address1 = factory.getPair(address(FLOW), address(DAI), false);
        address address2 = factory.getPair(address(FRAX), address(DAI), false);

        pair = Pair(address1);
        pair2 = Pair(address2); // added

        assertEq(address(pair2), address(fraxDaiPair));
        

        flowDaiPair.setVoter(); // set voter for the pairs
        fraxDaiPair.setVoter(); // set voter for the pairs
        assertEq(address(voter), flowDaiPair.voter());

        address gaugeAddress = voter.createGauge(address(flowDaiPair), 0);
        address gaugeAddress2 = voter.createGauge(address(fraxDaiPair), 0);

        gauge = GaugeV4(gaugeAddress);
        gauge2 = GaugeV4(gaugeAddress2);
        
        oFlowV3.setGauge(address(gauge));
        
    }


    function createLock(uint256 value, uint256 time, bool max) public returns (uint _tokenCreated) {
        flowDaiPair.approve(address(escrow), value); // approve escrow to spend value
     
        if (max) {
            _tokenCreated = escrow.create_lock(value, FIFTY_TWO_WEEKS);
        }
        else {
            require(time >= 7 days, "lock time too small");
            _tokenCreated = escrow.create_lock(value, time);
         } // tokenId is created
        vm.roll(block.number + 1); // fwd 1 block because escrow.balanceOfNFT() returns 0 in same block
    }

    function voteForPool(Pair poolPair, uint256 weight, uint tokenId) public {
        address[] memory pools = new address[](1);
        pools[0] = address(poolPair);
        uint256[] memory weights = new uint256[](1);
        weights[0] = weight;
        voter.vote(tokenId, pools, weights);
    }

    function testCreateLockGauge() public {
        flowDaiPair.approve(address(escrow), TOKEN_1);
        uint256 lockDuration = 7 * 24 * 3600; // 1 week

        // Balance should be zero before and 1 after creating the lock
        assertEq(escrow.balanceOf(address(owner)), 0);
        escrow.create_lock(TOKEN_1, lockDuration);
        assertEq(escrow.currentTokenId(), 1);
        assertEq(escrow.ownerOf(1), address(owner));
        assertEq(escrow.balanceOf(address(owner)), 1);
    }

    function claimEmissonsReward(GaugeV4 _gauge, address account) public {
        address[] memory tokens = new address[](1);
        tokens[0] = address(FLOW);
        _gauge.getReward(account, tokens);
    }


    function testFirstEpochEmissionsIsBroken() public {
        // get the gauge bribes
        address gaugeBribe1 = voter.external_bribes(address(gauge));
        address gaugeBribe2 = voter.external_bribes(address(gauge2));

        vm.warp(((block.timestamp / ONE_WEEK )* ONE_WEEK) + ONE_WEEK);
        //start active period
        vm.prank(minter.team());
        minter.startActivePeriod();


        //  set up a pool1, trade and deposit into a gauge1
        vm.startPrank(address(owner));
        washTradeNonStableFlowPool(flowDaiPair, FLOW, DAI, owner);
        flowDaiPair.approve(address(gauge), flowDaiPair.balanceOf(address(owner))); // approve gauge to spend all flowDaiPair lp tokens

        uint256 lpBalanceBeforeDepositOwner = flowDaiPair.balanceOf(address(owner));
        gauge.depositFor(address(owner), lpBalanceBeforeDepositOwner / 2); // deposit for owner
        assertEq(gauge.balanceOf(address(owner)), lpBalanceBeforeDepositOwner / 2);
        vm.stopPrank();

        //  set up a pool2, trade and deposit into a gauge2
        vm.startPrank(address(owner2));
        washTradeNonStablePool(fraxDaiPair, FRAX, DAI, owner2);
        fraxDaiPair.approve(address(gauge2), fraxDaiPair.balanceOf(address(owner2))); // approve gauge2 to spend all fraxDaiPair lp tokens
        uint256 lpBalanceBeforeDepositOwner2 = fraxDaiPair.balanceOf(address(owner2));
        gauge2.depositFor(address(owner2), (lpBalanceBeforeDepositOwner2 / 2) ); // deposit for owner2
        assertEq(gauge2.balanceOf(address(owner2)), lpBalanceBeforeDepositOwner2 / 2);
        vm.stopPrank();

        // Alice provide liquidity into flowdaipair so they can stake in and vote for their gauge of choice
        vm.startPrank(alice);
        provideLiquidityNonStableFlowPoolEOA(flowDaiPair, FLOW, DAI, alice);
        uint256 aliceFlowDaiLpBalance = flowDaiPair.balanceOf(alice);
        flowDaiPair.approve(address(gauge), (aliceFlowDaiLpBalance / 4));
        gauge.depositFor(alice, (aliceFlowDaiLpBalance / 4) );
        uint aliceToken = createLock(aliceFlowDaiLpBalance / 4, 0, true);
        // uint aliceToken = createLock(aliceFlowDaiLpBalance / 4, 7 days, false);
        voteForPool(flowDaiPair, flowDaiPair.balanceOf(alice), aliceToken);
        vm.stopPrank();

        // Bob provide liquidity into flowdaipair so they can stake in and vote for their gauge of choice
        vm.startPrank(bob);
        provideLiquidityNonStableFlowPoolEOA(flowDaiPair, FLOW, DAI, bob);
        uint256 bobFlowDaiLpBalance = flowDaiPair.balanceOf(bob);
        flowDaiPair.approve(address(gauge), (aliceFlowDaiLpBalance / 4));
        gauge.depositFor(bob, (bobFlowDaiLpBalance / 4) );
        uint bobToken = createLock(bobFlowDaiLpBalance / 4, 0, true);
        voteForPool(fraxDaiPair, flowDaiPair.balanceOf(bob), bobToken);
        vm.stopPrank();

        // // 1 epoch ends
        vm.warp(((block.timestamp / ONE_WEEK )* ONE_WEEK) + ONE_WEEK); // after 1 week
        emit log_named_decimal_uint("FLOW in voter before distribution", FLOW.balanceOf(address(voter)), 18);
        emit log_named_decimal_uint("FLOW in gauge before distribution", FLOW.balanceOf(address(gauge)), 18);
        emit log_named_decimal_uint("FLOW in gauge2 before distribution", FLOW.balanceOf(address(gauge2)), 18);

        // keeper calls distribute
        voter.distribute(); 
        emit log_named_decimal_uint("FLOW in voter after distribution", FLOW.balanceOf(address(voter)), 18);
        emit log_named_decimal_uint("FLOW in gauge after distribution", FLOW.balanceOf(address(gauge)), 18);
        emit log_named_decimal_uint("FLOW in gauge2 after distribution", FLOW.balanceOf(address(gauge2)), 18);
        uint epochOneDistributedAmount = FLOW.balanceOf(address(voter)) + FLOW.balanceOf(address(gauge)) + FLOW.balanceOf(address(gauge2));
        assertEq(epochOneDistributedAmount, 2000*1e18);

        emit log_named_decimal_uint("Total distributed emission", epochOneDistributedAmount, 18);

    }
    

    function washTradeNonStableFlowPool(Pair poolPair, Flow token0, MockERC20 token1, TestOwner recipient) public {

        provideLiquidityNonStableFlowPool(poolPair, token0, token1, recipient);

        Router.route[] memory routes = new Router.route[](1);
        routes[0] = Router.route(address(token0), address(token1), false);
        Router.route[] memory routes2 = new Router.route[](1);
        routes2[0] = Router.route(address(token1), address(token0), false);

        uint256 i;
        for (i = 0; i < 10; i++) {
            vm.warp(block.timestamp + 1801);
            assertEq(
                router.getAmountsOut(TOKEN_1, routes)[1],
                poolPair.getAmountOut(TOKEN_1, address(token0))
            );

            uint256[] memory expectedOutput = router.getAmountsOut(
                TOKEN_1,
                routes
            );
            token0.approve(address(router), TOKEN_1);
            router.swapExactTokensForTokens(
                TOKEN_1,
                expectedOutput[1],
                routes,
                address(recipient),
                block.timestamp
            );

            assertEq(
                router.getAmountsOut(TOKEN_1, routes2)[1],
                poolPair.getAmountOut(TOKEN_1, address(token1))
            );

            uint256[] memory expectedOutput2 = router.getAmountsOut(
                TOKEN_1,
                routes2
            );
            token1.approve(address(router), TOKEN_1);
            router.swapExactTokensForTokens(
                TOKEN_1,
                expectedOutput2[1],
                routes2,
                address(recipient),
                block.timestamp
            );
        }
    
    }

    function washTradeNonStablePool(Pair poolPair, MockERC20 token0, MockERC20 token1, TestOwner recipient) public {

        provideLiquidityNonStablePool(poolPair, token0, token1, recipient);

        Router.route[] memory routes = new Router.route[](1);
        routes[0] = Router.route(address(token0), address(token1), false);
        Router.route[] memory routes2 = new Router.route[](1);
        routes2[0] = Router.route(address(token1), address(token0), false);

        uint256 i;
        for (i = 0; i < 10; i++) {
            vm.warp(block.timestamp + 1801);
            assertEq(
                router.getAmountsOut(TOKEN_1, routes)[1],
                poolPair.getAmountOut(TOKEN_1, address(token0))
            );

            uint256[] memory expectedOutput = router.getAmountsOut(
                TOKEN_1,
                routes
            );
            token0.approve(address(router), TOKEN_1);
            router.swapExactTokensForTokens(
                TOKEN_1,
                expectedOutput[1],
                routes,
                address(recipient),
                block.timestamp
            );

            assertEq(
                router.getAmountsOut(TOKEN_1, routes2)[1],
                poolPair.getAmountOut(TOKEN_1, address(token1))
            );

            uint256[] memory expectedOutput2 = router.getAmountsOut(
                TOKEN_1,
                routes2
            );
            token1.approve(address(router), TOKEN_1);
            router.swapExactTokensForTokens(
                TOKEN_1,
                expectedOutput2[1],
                routes2,
                address(recipient),
                block.timestamp
            );
        }
    
    }

    function provideLiquidityNonStablePool(Pair poolPair, MockERC20 token0, MockERC20 token1, TestOwner recipient) public {
        token0.approve(address(router), TOKEN_100K); // 1e23 with 18 decimals
        token1.approve(address(router), TOKEN_100K); // 1e23 with 18 decimals
        router.addLiquidity(
            address(token0),
            address(token1),
            false,
            TOKEN_100K,
            TOKEN_100K,
            0,
            0,
            address(recipient),
            block.timestamp
        );
    }

    function provideLiquidityNonStableFlowPool(Pair poolPair, Flow token0, MockERC20 token1, TestOwner recipient) public {
        token0.approve(address(router), TOKEN_100K); // 1e23 with 18 decimals
        token1.approve(address(router), TOKEN_100K); // 1e23 with 18 decimals
        router.addLiquidity(
            address(token0),
            address(token1),
            false,
            TOKEN_100K,
            TOKEN_100K,
            0,
            0,
            address(recipient),
            block.timestamp
        );
    }

    function provideLiquidityNonStableFlowPoolEOA(Pair poolPair, Flow token0, MockERC20 token1, address recipient) public {
        token0.approve(address(router), TOKEN_100K); // 1e23 with 18 decimals
        token1.approve(address(router), TOKEN_100K); // 1e23 with 18 decimals
        router.addLiquidity(
            address(token0),
            address(token1),
            false,
            TOKEN_100K,
            TOKEN_100K,
            0,
            0,
            recipient,
            block.timestamp
        );
    }

    function washTrades() public {
        FLOW.approve(address(router), TOKEN_100K); // 1e23 with 18 decimals
        DAI.approve(address(router), TOKEN_100K); // 1e23 with 18 decimals
        router.addLiquidity(
            address(FLOW),
            address(DAI),
            false,
            TOKEN_100K,
            TOKEN_100K,
            0,
            0,
            address(owner),
            block.timestamp
        );

        Router.route[] memory routes = new Router.route[](1);
        routes[0] = Router.route(address(FLOW), address(DAI), false);
        Router.route[] memory routes2 = new Router.route[](1);
        routes2[0] = Router.route(address(DAI), address(FLOW), false);

        uint256 i;
        for (i = 0; i < 10; i++) {
            vm.warp(block.timestamp + 1801);
            assertEq(
                router.getAmountsOut(TOKEN_1, routes)[1],
                flowDaiPair.getAmountOut(TOKEN_1, address(FLOW))
            );

            uint256[] memory expectedOutput = router.getAmountsOut(
                TOKEN_1,
                routes
            );
            FLOW.approve(address(router), TOKEN_1);
            router.swapExactTokensForTokens(
                TOKEN_1,
                expectedOutput[1],
                routes,
                address(owner),
                block.timestamp
            );

            assertEq(
                router.getAmountsOut(TOKEN_1, routes2)[1],
                flowDaiPair.getAmountOut(TOKEN_1, address(DAI))
            );

            uint256[] memory expectedOutput2 = router.getAmountsOut(
                TOKEN_1,
                routes2
            );
            DAI.approve(address(router), TOKEN_1);
            router.swapExactTokensForTokens(
                TOKEN_1,
                expectedOutput2[1],
                routes2,
                address(owner),
                block.timestamp
            );
        }
    }

    function washTradesFraxDai() public {
        FRAX.approve(address(router), TOKEN_100K); // 1e23 with 18 decimals
        DAI.approve(address(router), TOKEN_100K); // 1e23 with 18 decimals
        router.addLiquidity(
            address(FRAX),
            address(DAI),
            false,
            TOKEN_100K,
            TOKEN_100K,
            0,
            0,
            address(owner2),
            block.timestamp
        );

        Router.route[] memory routes = new Router.route[](1);
        routes[0] = Router.route(address(FRAX), address(DAI), false);
        Router.route[] memory routes2 = new Router.route[](1);
        routes2[0] = Router.route(address(DAI), address(FRAX), false);

     

        uint256 i;
        for (i = 0; i < 10; i++) {
            vm.warp(block.timestamp + 1801);
            assertEq(
                router.getAmountsOut(TOKEN_1, routes)[1],
                fraxDaiPair.getAmountOut(TOKEN_1, address(FRAX))
            );

            uint256[] memory expectedOutput = router.getAmountsOut(
                TOKEN_1,
                routes
            );
            FRAX.approve(address(router), TOKEN_1);
            router.swapExactTokensForTokens(
                TOKEN_1,
                expectedOutput[1],
                routes,
                address(owner2),
                block.timestamp
            );

            assertEq(
                router.getAmountsOut(TOKEN_1, routes2)[1],
                fraxDaiPair.getAmountOut(TOKEN_1, address(DAI))
            );

            uint256[] memory expectedOutput2 = router.getAmountsOut(
                TOKEN_1,
                routes2
            );
            DAI.approve(address(router), TOKEN_1);
            router.swapExactTokensForTokens(
                TOKEN_1,
                expectedOutput2[1],
                routes2,
                address(owner2),
                block.timestamp
            );
        }
    }


}

```

</details>

The following logs are emitted
```solidity
  FLOW in voter before distribution: 0.000000000000000000
  FLOW in gauge before distribution: 0.000000000000000000
  FLOW in gauge2 before distribution: 0.000000000000000000
  FLOW in voter after distribution: 0.000000000000045744
  FLOW in gauge after distribution: 999.999999999999977128
  FLOW in gauge2 after distribution: 999.999999999999977128
  Total distibuted emission: 2000.000000000000000000
```
## Impact
Dilution of rewards received between active gauges hence not delivering promised returns


## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Voter.sol#L548-L562

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Minter.sol#L98-L104

https://github.com/sherlock-audit/2024-06-velocimeter/blob/main/v4-contracts/contracts/Minter.sol#L117

## Tool used

Manual Review
Foundry test

## Recommendation
Implement a mechanism that would return active gauges for only the first epoch and subsequently use `activeGaugeNumber` after the first epoch