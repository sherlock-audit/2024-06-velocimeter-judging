Salty Azure Eagle

High

# Voting power does not decay when calculating shares of flow emissions if the user does not vote again.

## Summary

Voting power does not decay when calculating shares of flow emissions earned by a pool/gauge, if the user does not vote again. The votes are still counted, but assigned too much power.

## Vulnerability Detail

The documentation linked in the README states that voting power should decay linearly based on time to unlock. However, this decay is not taken into account when calculating shares of flow emissions in `Voter._updateFor()` if the user does not call `Voter.vote` again. Their votes are still counted, but with the weights unchanged.

This leads to an unfair distribution of weekly emissions and a loss of funds for the liquidity providers who do not get their fair share.

## Impact

Liquidity providers does not get their fair share of weekly emissions.

## Proof of Concept

Copy this to a new file anywhere within `v4-contracts/test` and run it with `forge test --match-contract "NoVoteNoDecay"`

Note: In this example, Bob votes for the same pool twice, to clearly show that the rewards are skewed. Hopefully it is clear from the description above and the code snippets below, that the result is the same, if he changes his vote to another pool/other pools.

<details>
<summary>Coded PoC</summary>

```solidity
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "lib/solmate/src/tokens/ERC20.sol";
import "lib/solmate/src/tokens/WETH.sol";
import "contracts/factories/PairFactory.sol";
import "contracts/factories/GaugeFactoryV4.sol";
import "contracts/factories/BribeFactory.sol";
import "contracts/Router.sol";
import "contracts/VotingEscrow.sol";
import "contracts/Voter.sol";
import "contracts/Pair.sol";
import "contracts/GaugeV4.sol";
import "contracts/Flow.sol";
import "contracts/RewardsDistributorV2.sol";
import "contracts/Minter.sol";
import "contracts/OptionTokenV4.sol";
import "contracts/interfaces/IERC20.sol";

contract Token is ERC20 {
    constructor(
        string memory _name,
        string memory _symbol,
        uint8 _decimals
    ) ERC20(_name, _symbol, _decimals) {}

    function mint(address to, uint amount) public {
        _mint(to, amount);
    }
}

contract NoVoteNoDecayTest is Test {
    address DEPLOYER = address(uint160(uint(keccak256("DEPLOYER"))));
    address ALICE = address(uint160(uint(keccak256("ALICE"))));
    address BOB = address(uint160(uint(keccak256("BOB"))));

    Flow flow;
    OptionTokenV4 oFlow;
    WETH weth;
    Pair flowWethPair;

    Token tokenA;
    Token tokenB;

    Pair pairA;
    Pair pairB;

    GaugeV4 gaugeA;
    GaugeV4 gaugeB;

    PairFactory pairFactory;
    Router router;
    VotingEscrow votingEscrow;
    Voter voter;
    Minter minter;

    function setUp() public {
        vm.deal(DEPLOYER, 100 ether);
        vm.deal(ALICE, 100 ether);
        vm.deal(BOB, 100 ether);

        vm.startPrank(DEPLOYER);

        flow = new Flow(DEPLOYER, 1e21);
        weth = new WETH();

        pairFactory = new PairFactory();
        GaugeFactoryV4 gaugeFactory = new GaugeFactoryV4();
        router = new Router(address(pairFactory), address(weth));

        _addFlowWethLiquidity(1e18, DEPLOYER);

        flowWethPair = Pair(
            pairFactory.getPair(address(flow), address(weth), false)
        );

        votingEscrow = new VotingEscrow(
            address(flow),
            address(flowWethPair),
            address(0),
            address(0)
        );

        voter = new Voter(
            address(votingEscrow),
            address(pairFactory),
            address(gaugeFactory),
            address(new BribeFactory()),
            address(0)
        );

        votingEscrow.setVoter(address(voter));
        pairFactory.setVoter(address(voter));

        RewardsDistributorV2 rewardsDistributorFlow = new RewardsDistributorV2(
            address(votingEscrow),
            address(flow)
        );

        minter = new Minter(
            address(voter),
            address(votingEscrow),
            address(rewardsDistributorFlow)
        );

        rewardsDistributorFlow.setDepositor(address(minter));

        address[] memory whitelistedTokens = new address[](1);
        whitelistedTokens[0] = address(flow);
        voter.initialize(whitelistedTokens, address(minter));

        flow.setMinter(address(minter));
        minter.startActivePeriod();

        oFlow = new OptionTokenV4(
            "Option to buy Flow",
            "oFlow",
            DEPLOYER,
            address(flow),
            DEPLOYER,
            address(voter),
            address(router),
            true,
            false,
            false,
            0
        );

        oFlow.setPairAndPaymentToken(flowWethPair, address(weth));
        oFlow.grantRole(oFlow.ADMIN_ROLE(), address(gaugeFactory));

        gaugeFactory.setOFlow(address(oFlow));

        tokenA = new Token("Token A", "A", 18);
        tokenB = new Token("Token B", "B", 18);

        tokenA.mint(ALICE, 1e19);
        tokenB.mint(BOB, 1e19);

        pairA = Pair(
            pairFactory.createPair(address(flow), address(tokenA), false)
        );

        pairB = Pair(
            pairFactory.createPair(address(flow), address(tokenB), false)
        );

        gaugeA = GaugeV4(voter.createGauge(address(pairA), 0));
        gaugeB = GaugeV4(voter.createGauge(address(pairB), 0));

        flow.transfer(ALICE, 1e20);
        flow.transfer(BOB, 1e20);

        vm.stopPrank();
    }

    function _addFlowWethLiquidity(uint amount, address to) internal {
        flow.approve(address(router), amount);
        router.addLiquidityETH{value: amount}(
            address(flow),
            false,
            amount,
            0,
            0,
            to,
            block.timestamp
        );
    }

    function _addFlowWethLiquidityAndMaxLock(uint amount, address to) internal {
        _addFlowWethLiquidity(1e18, to);
        flowWethPair.approve(address(votingEscrow), 1e18);
        votingEscrow.create_lock(1e18, 52 weeks);
    }

    function testNoVoteNoDecay() public {
        // Alice and Bob both lock 1e18 lp tokens for 52 weeks
        vm.startPrank(ALICE);
        _addFlowWethLiquidityAndMaxLock(1e18, ALICE);
        vm.stopPrank();

        vm.startPrank(BOB);
        _addFlowWethLiquidityAndMaxLock(1e18, BOB);
        vm.stopPrank();

        // Alice owns token 1
        assertEq(votingEscrow.ownerOf(1), ALICE);
        // Bob owns token 2
        assertEq(votingEscrow.ownerOf(2), BOB);

        vm.warp(block.timestamp + 1 weeks);

        address[] memory alicePools = new address[](1);
        address[] memory bobPools = new address[](1);
        uint[] memory weights = new uint[](1);
        alicePools[0] = address(pairA);
        bobPools[0] = address(pairB);
        weights[0] = 1;

        // Alice votes for pairA
        vm.prank(ALICE);
        voter.vote(1, alicePools, weights);

        // Bob votes for pairB
        vm.prank(BOB);
        voter.vote(2, bobPools, weights);

        vm.warp(block.timestamp + 1 weeks);
        voter.distribute();

        // Both gauges receive the same share of emissions
        assertEq(
            flow.balanceOf(address(gaugeA)),
            flow.balanceOf(address(gaugeB))
        );

        // Bob votes again
        vm.prank(BOB);
        voter.vote(2, bobPools, weights);

        (int128 aliceLockedAmount, uint aliceLockedEnd) = votingEscrow.locked(
            1
        );
        (int128 bobLockedAmount, uint bobLockedEnd) = votingEscrow.locked(2);

        // They both still have the same amount of locked lp tokens with the same lock duration
        assertEq(aliceLockedAmount, bobLockedAmount);
        assertEq(aliceLockedEnd, bobLockedEnd);

        vm.warp(block.timestamp + 1 weeks);
        voter.distribute();

        // gaugeA receives a larger amount of emissions, as only Bob's voting power has decayed
        assertGt(
            flow.balanceOf(address(gaugeA)),
            flow.balanceOf(address(gaugeB)) + 1e19 // adding 1e19 to emphasize, that it is not just a rounding error
        );
    }
}
```
</details>

## Code Snippet

`Voter.vote()` calls `Voter._vote`, which updates `weights[_pool]`. This is where the calculation of the voting power based on time to unlock happens.

https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/Voter.sol#L287-L292

https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/Voter.sol#L249-L285

`Voter._updateFor()` uses `weights[_pool]` to calculate the share of emissions earned by the pool/gauge. The user's contribution to `weights[_pool]` remains unchanged until the user calls `Voter.reset` or `Voter.vote`.

https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/Voter.sol#L517-L534

## Tool used

Manual Review

## Recommendation

Either
1. Count votes per epoch, so users are forced to vote again every week.
2. Calculate shares of emissions similarly to the logic in `RewardsDistributorV2` using `bias` and `slope`.