Square Arctic Chicken

High

# Paused Gauges can claim rewards for the period in which they are paused especially if there is at least one killed gauge

## Summary
When a gauge is paused, ideally, emissions are not directed toward the gauge. But owing to constraints that exists when there are killed gauges in the protocol (see POC).

## Vulnerability Detail
The main issue is that the paused gauge index (`supplyIndex[_gauge]`) is not updated when a pause gauge is restarted leaving a path for the gauge to claim reward from the point where it was paused and emissions are directed to the gauge as though it was never paused.

```solidity
    function restartGauge(address _gauge) external {
        if (msg.sender != emergencyCouncil) {
            require(
                IGaugePlugin(gaugePlugin).checkGaugeRestartAllowance(msg.sender, _gauge)
            , "Restart gauge not allowed");
        }
        require(!isAlive[_gauge], "gauge already alive");
        isAlive[_gauge] = true; // _pair is the LP token that needs to be staked for rewards
        address _pair = IGauge(_gauge).stake(); // TODO: add test cases
        try IPair(_pair).setHasGauge(true) {} catch {}
        emit GaugeRestarted(_gauge);
    }
```

To understand this better, when a gauge is paused the `supplyIndex[_gauge]` is not cleared. However when the keeper calls `distribute()` at the beginning of every epoch the `supplyIndex[_gauge] is updated preventing the gauge from claiming past reward from the last epoch (although the weight of the paused gauge still contributes to this) when the gauge is restarted.

```solidity

    function _updateFor(address _gauge) internal { // it claims the reward for _gauge
        address _pool = poolForGauge[_gauge];
        uint256 _supplied = weights[_pool]; //  weight of votes in this pool
        if (_supplied > 0) {
            uint _supplyIndex = supplyIndex[_gauge]; // rewards amount for specific gauge tracker
            uint _index = index; // get global index0 for accumulated distro
  ->        supplyIndex[_gauge] = _index; // update _gauge current position to global position
            uint _delta = _index - _supplyIndex; // see if there is any difference that need to be accrued
            if (_delta > 0) {
                uint _share = uint(_supplied) * _delta / 1e18; // add accrued difference for each supplied token (share of deposited reward for the gauge)
                if (isAlive[_gauge]) {
                    claimable[_gauge] += _share;
                } 
            }
        } else {
->          supplyIndex[_gauge] = index; // new users are set to the default global state
        }
```

The problem is certain to occur when there is at least one killed gauge in the protocol, because `distribute()` will revert (as shown in this Low/info finding I submitted in  #9 . This is important because the keeper will need to call `distribute(uint start, uint finish)` excluding the paused (since emissions are not directed toward the paused gauge) and killed gauges  instead of `distribute()` and _there is no guarantee that anyone will call `distribute(paused gauge)` or [`updateGauge(paused gauge)`](https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/Voter.sol#L513-L515) to ensure `supplyIndex[paused gauge]`_ of the paused gauge(s) is updated during in any of the epochs that the gauge may be paused for. `distribute(uint start, uint finish)` will be called with only `alive` gauges and as such `supplyIndex[paused gauge]` will not be updated for paused gauge leaving the paused gauge leeway to claim from the last time it's supplyIndex was updated.

For simplicity, consider there are 3 gauges
- gauge1 is killed after the first epoch
- gauge2 paused after the third epoch (in the fourth epoch) leaving only gauge3 active
- keeper calls `distribute(gauge3)` in the 4th epoch, emissions are made 
- gauge2 is restarted  in the 6th epoch
- `supplyIndex[gauge2]` is not updated because nobody calls `updateGauge(gauge2)` 
- because gauge2 is now alive, keeper calls `distribute( [gauge2, gauge3] ), gauge2 gets all the claimable funds from the time it was paused


**CODED POC**

In the POC there are 2 gauges, `gauge2` is paused in the second epoch an restarted in the fourth epoch and it still claims all rewards that were due to it.

- modify the `BaseTest.sol` as shown below

<details>
<summary> Modified basetest</summary>

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

SNIP...

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


- create a file in the `test` folder and paste the code block below and the run `forge test --mt testPausedGaugeCanAccessPastReward -vvv`
<details>
<summary> POC File</summary>

```solidity

pragma solidity 0.8.13;

import "./BaseTest.sol";
import "contracts/factories/GaugeFactoryV4.sol";

contract GaugeV4Test is BaseTest {
    VotingEscrow escrow;
    GaugeFactoryV4 gaugeFactory;
    BribeFactory bribeFactory;
    Voter voter;
    GaugeV4 gauge;
    GaugeV4 gauge2; // added
    RewardsDistributor distributor; // added
    Minter minter; // added

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
        emit log_address(factory.voter());

        
        escrow.setVoter(address(voter));

        minter = new Minter(address(voter), address(escrow), address(distributor)); // added

        // initialize voter with minter
        voter.initialize(tokens, address(minter)); // whitelist tokens and the minter

        
        distributor.setDepositor(address(minter)); // added
        emit log("Voter initializing");
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
        
        emit log_address(escrow.lpToken());

        flowDaiPair.setVoter(); // set voter for the pairs
        fraxDaiPair.setVoter(); // set voter for the pairs
        assertEq(address(voter), flowDaiPair.voter());

        address gaugeAddress = voter.createGauge(address(flowDaiPair), 0);
        address gaugeAddress2 = voter.createGauge(address(fraxDaiPair), 0);

        gauge = GaugeV4(gaugeAddress);
        gauge2 = GaugeV4(gaugeAddress2);
        
        oFlowV3.setGauge(address(gauge));
        // oFlowV32.setGauge(address(gauge2));
        
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

    function createLock(uint256 value, uint256 time, bool max) public returns (uint _tokenCreated) {
        flowDaiPair.approve(address(escrow), value); // approve escrow to spend value
        emit log("lock has started");
        if (max) {
            _tokenCreated = escrow.create_lock(value, FIFTY_TWO_WEEKS);
        }
        else {
            require(time >= ONE_WEEK, "lock time too small");
            _tokenCreated = escrow.create_lock(value, time);
         } 
        vm.roll(block.number + 1);
        emit log_named_uint("_tokenCreated", _tokenCreated);
    }

    function voteForPool(Pair poolPair, uint256 weight, uint tokenId) public {
        address[] memory pools = new address[](1);
        pools[0] = address(poolPair);
        uint256[] memory weights = new uint256[](1);
        weights[0] = weight;
        voter.vote(tokenId, pools, weights);
    }

    function claimEmissonsReward(GaugeV4 _gauge, address account) public {
        address[] memory tokens = new address[](1);
        tokens[0] = address(FLOW);
        _gauge.getReward(account, tokens);
    }
    function testPausedGaugeCanAccessPastReward() public {

        vm.warp(((block.timestamp / ONE_WEEK )* ONE_WEEK) + ONE_WEEK);
        //start active period
        vm.prank(minter.team());
        minter.startActivePeriod();

        // owner1 set up a add liquidity and perform wash trades
        vm.startPrank(address(owner));
        washTradeNonStableFlowPool(flowDaiPair, FLOW, DAI, owner);
        flowDaiPair.approve(address(gauge), flowDaiPair.balanceOf(address(owner)));
        uint256 lpBalanceBeforeDepositOwner = flowDaiPair.balanceOf(address(owner));
        gauge.depositFor(address(owner), lpBalanceBeforeDepositOwner / 2);
        assertEq(gauge.balanceOf(address(owner)), lpBalanceBeforeDepositOwner / 2);
        vm.stopPrank();

        // owner2 set up a pool2, trade and deposit into a gauge2
        vm.startPrank(address(owner2));
        washTradeNonStablePool(fraxDaiPair, FRAX, DAI, owner2);
        fraxDaiPair.approve(address(gauge2), fraxDaiPair.balanceOf(address(owner2))); // approve gauge2 to spend all fraxDaiPair lp tokens
        uint256 lpBalanceBeforeDepositOwner2 = fraxDaiPair.balanceOf(address(owner2));
        gauge2.depositFor(address(owner2), (lpBalanceBeforeDepositOwner2 / 2) ); // deposit for owner2
        assertEq(gauge2.balanceOf(address(owner2)), lpBalanceBeforeDepositOwner2 / 2);
        vm.stopPrank();

        // Alice provide liquidity into flowdaipair so they can stake in and vote for gauge1
        vm.startPrank(alice);
        provideLiquidityNonStableFlowPoolEOA(flowDaiPair, FLOW, DAI, alice);
        uint256 aliceFlowDaiLpBalance = flowDaiPair.balanceOf(alice);

        flowDaiPair.approve(address(gauge), (aliceFlowDaiLpBalance / 4));
        gauge.depositFor(alice, (aliceFlowDaiLpBalance / 4) );
        // uint aliceToken = createLock(aliceFlowDaiLpBalance / 4, ONE_WEEK, false); // Alice create lock for ONE_WEEK
        uint aliceToken = createLock(aliceFlowDaiLpBalance / 4, 0, true); 
        voteForPool(flowDaiPair, flowDaiPair.balanceOf(alice), aliceToken); // Alice vote on the flowDaiPair
        vm.stopPrank();

        // Bob provide liquidity into flowdaipair so they can stake in and vote for gauge2
        vm.startPrank(bob);
        provideLiquidityNonStableFlowPoolEOA(flowDaiPair, FLOW, DAI, bob);
        uint256 bobFlowDaiLpBalance = flowDaiPair.balanceOf(bob);

        flowDaiPair.approve(address(gauge), (aliceFlowDaiLpBalance / 4));
        gauge.depositFor(bob, (bobFlowDaiLpBalance / 4) );
        uint bobToken = createLock(bobFlowDaiLpBalance / 4, 0, true); // Bob create lock for FIFTY_TWO_WEEKS
        voteForPool(fraxDaiPair, flowDaiPair.balanceOf(bob), bobToken);
        vm.stopPrank();

        vm.warp(((block.timestamp / ONE_WEEK )* ONE_WEEK) + ONE_WEEK - 1 days); 

        emit log_named_uint("Weight of gauge before first epoch end beofore death", voter.weights(voter.poolForGauge(address(gauge))));
        emit log_named_uint("Weight of gauge2 before first epoch end", voter.weights(voter.poolForGauge(address(gauge2))));

        emit log("************************************                  ************************************");
        emit log("**                                                                                      **");
        emit log("************************************                  ************************************");


        // 1st epoch ends
        vm.warp(((block.timestamp / ONE_WEEK )* ONE_WEEK) + ONE_WEEK); // after 1 week
        voter.distribute();

        emit log_named_decimal_uint("FLOW in voter after first distribution", FLOW.balanceOf(address(voter)), 18);
        emit log_named_decimal_uint("FLOW in gauge after first distribution", FLOW.balanceOf(address(gauge)), 18);
        emit log_named_decimal_uint("FLOW in gauge2 after first distribution", FLOW.balanceOf(address(gauge2)), 18);

        // pause gauge
        vm.prank(voter.emergencyCouncil());
        voter.pauseGauge(address(gauge2));



        emit log("************************************                  ************************************");
        emit log("**                                  GAUGE2 IS PAUSED                                  **");
        emit log("************************************                  ************************************");

        for (uint i = 0; i < 3; i++) {
            vm.warp(((block.timestamp / ONE_WEEK )* ONE_WEEK) + ONE_WEEK); 
            voter.distribute(address(gauge));
            emit log_named_decimal_uint("FLOW in voter after }ith epoch distribution", FLOW.balanceOf(address(voter)), 18);
            emit log_named_decimal_uint("FLOW in gauge after ith epoch distribution", FLOW.balanceOf(address(gauge)), 18);
            emit log_named_decimal_uint("FLOW in gauge2 after ith epoch distribution", FLOW.balanceOf(address(gauge2)), 18);
            emit log("**************************                             ************************************");
            emit log_named_uint("**                        EPOCH**", i+2);
            emit log("**************************                             ************************************");
        }


        // restart gauge
        vm.prank(voter.emergencyCouncil());
        voter.restartGauge(address(gauge2));

        emit log("************************************                  ************************************");
        emit log("**                                  GAUGE2 IS UNPAUSED                                  **");
        emit log("************************************                  ************************************");

        // after 4th
        vm.warp(((block.timestamp / ONE_WEEK )* ONE_WEEK) + ONE_WEEK); 
        // keeper calls distribute
        voter.distribute(); // allgauges


        emit log_named_decimal_uint("FLOW in voter after fifth epoch distribution", FLOW.balanceOf(address(voter)), 18);
        emit log_named_decimal_uint("FLOW in gauge after fifth epoch distribution", FLOW.balanceOf(address(gauge)), 18);
        emit log_named_decimal_uint("FLOW in gauge2 after fifth epoch distribution", FLOW.balanceOf(address(gauge2)), 18);

        emit log("**************************                             ************************************");
        emit log_named_uint("**                        EPOCH**", 5);
        emit log("**************************                             ************************************");
    }
```

</details>


## Impact
- Rewards could be diluted for gauges that are alive and this could as well be a leak of value
- it breaks core protocol functionality because emissions are still directed toward paused gauges when they are unpaused
- it could break accounting for protocols that integrate with VM

## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/Voter.sol#L513-L515

https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/Voter.sol#L531-L533

https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/Voter.sol#L568-L576


## Tool used
Foundry test
Manual Review

## Recommendation
Modify the `restartGauge(...)` function to ensure the paused gauge can claim emissions distributed only after it was paused

```diff
    function restartGauge(address _gauge) external {
        if (msg.sender != emergencyCouncil) {
            require(
                IGaugePlugin(gaugePlugin).checkGaugeRestartAllowance(msg.sender, _gauge)
            , "Restart gauge not allowed");
        }
        require(!isAlive[_gauge], "gauge already alive");
+       supplyIndex[_gauge] = index;
        isAlive[_gauge] = true; // _pair is the LP token that needs to be staked for rewards
        address _pair = IGauge(_gauge).stake(); // TODO: add test cases
        try IPair(_pair).setHasGauge(true) {} catch {}
        emit GaugeRestarted(_gauge);
    }
```
