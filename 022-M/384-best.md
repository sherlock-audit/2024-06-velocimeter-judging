Loud Inky Shark

High

# pause or killed gauges doesn't decrement the activeGaugeNumber causing more rewards emitted

## Summary
Many Solidly forks have the ability to pause the system in case of suspicious activity. V4 enhances this by allowing the pausing of trading on a single pool. This improves security and enables VMCoupons to maintain a consistent exercise price for all holders. Additionally, it supports scenarios where a pool's price needs to be maintained for a period of time. Even when trading is paused, LP providers can still freely enter and exit their positions.

However, the problem lies in updating the total amount of gauges.

## Vulnerability Detail

In V4, emissions are determined by the number of gauges. More gauges result in more emissions being emitted. 

`Voter::distribute` function can be called by anyone and it updates accounting for gauges. It is called weekly by keeper bot. When the gauge is paused by the council, the total `activeGaugeNumber` remains as is. The `Voter::distribute` function calls the `IMinter(minter).update_period();`, this can only be updated once due to the if statement. There is the same problem in `Voter::killGaugeTotally`. Hence, for that week, more rewards will be emitted since the `activeGaugeNumber` remains the same.

```solidity
// Voter.sol
    function distribute(address _gauge) public lock {
        IMinter(minter).update_period(); //@audit
    -- SNIP --
```
```solidity
// Minter.sol
    function update_period() external returns (uint) {
        uint _period = active_period;
        if (block.timestamp >= _period + WEEK && initializer == address(0)) { // only trigger if new week
            _period = (block.timestamp / WEEK) * WEEK;
            active_period = _period;
            uint256 weekly = weekly_emission();
            uint _teamEmissions = (teamRate * weekly) /
                (PRECISION - teamRate);
            uint _required =  weekly + _teamEmissions;
            uint _balanceOf = _flow.balanceOf(address(this));
            if (_balanceOf < _required) {
                _flow.mint(address(this), _required - _balanceOf);
            }
            require(_flow.transfer(teamEmissions, _teamEmissions));
            _checkpointRewardsDistributors();
            _flow.approve(address(_voter), weekly);
            _voter.notifyRewardAmount(weekly);
            emit Mint(msg.sender, weekly, circulating_supply());
        }
        return _period;
    }

    function weekly_emission() public view returns (uint) {
        uint256 numberOfGauges = _voter.activeGaugeNumber(); // should decrease however it was not
        if(numberOfGauges == 0) { 
            return weeklyPerGauge;
        }
        return weeklyPerGauge * numberOfGauges;
    }
```

<details><summary> Run: forge test --match-test testPoCDistribute_reset_to1 -vv </summary>
<p>

```solidity
// 1:1 with Hardhat test
pragma solidity 0.8.13;
import './BaseTest.sol';
import '../contracts/GaugeV4.sol';
contract PoCTest is BaseTest {
    VotingEscrow escrow;
    GaugeFactory gaugeFactory;
    BribeFactory bribeFactory;
    Voter voter;
    RewardsDistributor distributor;
    Minter minter;
    function deployBase() public {
        vm.warp(block.timestamp + 1 weeks); // put some initial time in
        deployOwners();
        deployCoins();
        mintStables();
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 1e25;
        mintFlow(owners, amounts);
        VeArtProxy artProxy = new VeArtProxy();
        deployPairFactoryAndRouter();
        deployMainPairWithOwner(address(owner));
        escrow = new VotingEscrow(address(FLOW), address(flowDaiPair),address(artProxy), owners[0]);
        gaugeFactory = new GaugeFactory();
        bribeFactory = new BribeFactory();
        gaugePlugin = new GaugePlugin(address(FLOW), address(WETH), owners[0]);
        voter = new Voter(address(escrow), address(factory), address(gaugeFactory), address(bribeFactory), address(gaugePlugin));
        factory.setVoter(address(voter));
        flowDaiPair.setVoter();
        deployPairWithOwner(address(owner));
        deployOptionTokenWithOwner(address(owner), address(gaugeFactory));
        gaugeFactory.setOFlow(address(oFlow));
        address[] memory tokens = new address[](2);
        tokens[0] = address(FRAX);
        tokens[1] = address(FLOW);
        flowDaiPair.approve(address(escrow), TOKEN_1);
        escrow.create_lock(TOKEN_1, FIFTY_TWO_WEEKS);
        distributor = new RewardsDistributor(address(escrow));
        escrow.setVoter(address(voter));
        minter = new Minter(address(voter), address(escrow), address(distributor));
        voter.initialize(tokens, address(minter));
        distributor.setDepositor(address(minter));
        FLOW.setMinter(address(minter));
        FLOW.approve(address(router), TOKEN_1);
        FRAX.approve(address(router), TOKEN_1);
        router.addLiquidity(address(FRAX), address(FLOW), false, TOKEN_1, TOKEN_1, 0, 0, address(owner), block.timestamp);
        address pair = router.pairFor(address(FRAX), address(FLOW), false);
        FLOW.approve(address(voter), 5 * TOKEN_100K);
        address old_gauge = voter.createGauge(pair, 0);
        vm.roll(block.number + 1); // fwd 1 block because escrow.balanceOfNFT() returns 0 in same block
        assertGt(escrow.balanceOfNFT(1), 995063075414519385);
        assertEq(flowDaiPair.balanceOf(address(escrow)), TOKEN_1);
        address[] memory pools = new address[](1);
        pools[0] = pair;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;
        voter.vote(1, pools, weights);
    }

    function initializeVotingEscrow() public {
        deployBase();

        Minter.Claim[] memory claims = new Minter.Claim[](1);
        claims[0] = Minter.Claim({
            claimant: address(owner),
            amount: TOKEN_100K,
            lockTime: FIFTY_TWO_WEEKS
        });
        //minter.initialMintAndLock(claims, 2 * TOKEN_100K);
        FLOW.transfer(address(minter), TOKEN_100K);
        flowDaiPair.approve(address(escrow), TOKEN_1);
        escrow.create_lock(TOKEN_1,FIFTY_TWO_WEEKS);
        minter.startActivePeriod();

        assertEq(escrow.ownerOf(2), address(owner));
        assertEq(escrow.ownerOf(3), address(0));
        vm.roll(block.number + 1);
        assertEq(FLOW.balanceOf(address(minter)), 1 * TOKEN_100K);
    }
    function testPoCPause_weeklyEmission() public {
        initializeVotingEscrow();

        FLOW.approve(address(router), TOKEN_1);
        FRAX.approve(address(router), TOKEN_1);
        router.addLiquidity(address(FRAX), address(FLOW), false, TOKEN_1, TOKEN_1, 0, 0, address(owner), block.timestamp);
        address pair1 = router.pairFor(address(FRAX), address(FLOW), false);
        address pair2 = router.pairFor(address(DAI), address(FLOW), false);
        uint pool_length = voter.length();
        console.log("Total pool gauge created:", pool_length); //should remain 0 below comment

        address new_gauge = voter.createGauge(pair2, 0); // create second gauges
      
        assertEq(minter.weekly_emission(), 2000e18);
        uint pool_length2 = voter.length();
        console.log("Create new gauge. Total pool gauge created: ", pool_length2); //should remain 0 below comment

        uint activeGaugeNumber1 = voter.activeGaugeNumber();

        console.log("Didn't call vote, Active Gauge: ", activeGaugeNumber1);

        _elapseOneWeek();

        address[] memory pools = new address[](2);
        pools[0] = pair1; // vote on first gauge
        pools[1] = pair2;// vote on second gauge
        uint256[] memory weights = new uint256[](2);
        weights[0] = 9899;
        weights[1] = 101;

        // SECOND VOTE EMISSION TO POOLS [first vote is in deployBase()]
        voter.vote(1, pools, weights);

        // Variables for new gauge

        // Variables for old Gauge
        address[][] memory old_reward_tokens = new address[][](1);
        address[] memory old_token = new address[](1); // used for reward_tokens
        address[] memory oldGaugeArray = new address[](1);
        address[] memory newGaugeArray = new address[](1);

        // mapping(address => address) public gauges; // pool => gauge
        address get_oldGauge = voter.gauges(pair1); // FRAX/FLOW
        oldGaugeArray[0] = address(get_oldGauge);// FRAX/FLOW gauge
        old_token[0] = address(FLOW); //reward token
        old_reward_tokens[0] = old_token;
        
        newGaugeArray[0] = address(new_gauge); // DAI/FLOW Gauge

        
        uint activeGaugeNumber2 = voter.activeGaugeNumber(); // 0

        console.log("After first vote and before claiming rewards, Active Gauge: ", activeGaugeNumber2);

        _elapseOneWeek();
        
        GaugeV4 gauge_instance = GaugeV4(new_gauge);
        voter.distribute();

        voter.claimRewards(oldGaugeArray,old_reward_tokens); // rewards claimed 1
        voter.claimRewards(newGaugeArray,old_reward_tokens); // rewards claimed 1

        uint activeGaugeNumber3= voter.activeGaugeNumber();
        assertEq(minter.weekly_emission(), 4000e18); //should return 4000e18 instead of 2000e18 because of 2 active gauges.

        console.log("After first vote and after claiming rewards, Active Gauge:", activeGaugeNumber3);

        // Main problem, another week later
        _elapseOneWeek();
        
        voter.distribute();

        voter.pauseGauge(new_gauge);

        //@audit still able to claim more rewards for both gauges
        voter.claimRewards(oldGaugeArray,old_reward_tokens); // rewards claimed 
        voter.claimRewards(newGaugeArray,old_reward_tokens); // rewards claimed 

        uint activeGaugeNumber4= voter.activeGaugeNumber();

        //@audit-issue
        assertEq(minter.weekly_emission(), 4000e18); //should return 2000e18 instead of 4000e18 because the other gauge is paused

        console.log("After paused: Active Gauge: ", activeGaugeNumber4);
    }
    function _elapseOneWeek() private {
        vm.warp(block.timestamp + ONE_WEEK);
        vm.roll(block.number + 1);
    }
}
```
</p>
</details> 


## Impact
Unfair amounts of rewards are sent to users from different gauges.

## Code Snippet
https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/Voter.sol#L549C1-L562C6

https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/Minter.sol#L112C1-L137C6

https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/Voter.sol#L407C1-L429C6

https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/Voter.sol#L380-L392C6
## Tool used

Manual Review

## Recommendation
```diff
    function pauseGauge(address _gauge) external {
        if (msg.sender != emergencyCouncil) {
            require(
                IGaugePlugin(gaugePlugin).checkGaugePauseAllowance(msg.sender, _gauge)
            , "Pause gauge not allowed");
        }
        require(isAlive[_gauge], "gauge already dead");
        isAlive[_gauge] = false;
        claimable[_gauge] = 0;
+       activeGaugeNumber -= 1;
        address _pair = IGauge(_gauge).stake(); // TODO: add test cases
        try IPair(_pair).setHasGauge(false) {} catch {}
        emit GaugePaused(_gauge);

    function killGaugeTotally(address _gauge) external {
        if (msg.sender != emergencyCouncil) {
            require(
                IGaugePlugin(gaugePlugin).checkGaugeKillAllowance(msg.sender, _gauge)
            , "Restart gauge not allowed");
        }
        require(isAlive[_gauge], "gauge already dead");
        address _pool = poolForGauge[_gauge];
        delete isAlive[_gauge];
        delete external_bribes[_gauge];
        delete poolForGauge[_gauge];
        delete isGauge[_gauge];
        delete claimable[_gauge];
        delete supplyIndex[_gauge];
        delete gauges[_pool];
        try IPair(_pool).setHasGauge(false) {} catch {}
        killedGauges.push(_gauge);
+       activeGaugeNumber -= 1;
        emit GaugeKilledTotally(_gauge);
    }
```