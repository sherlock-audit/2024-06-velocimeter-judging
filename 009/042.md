Decent Mandarin Pangolin

Medium

# Due to using approve instead of _clearApproval, the merge and withdraw functions cannot be called by delegated operators in some ocassions (causing a permanent DoS for single-token-delegatees operators)

## Summary
The `merge` and `withdraw` functions cannot be called by operators *if these operators were granted allowances via the `approve(operatorAddress, tokenId)` functions*. This causes a DoS upon accessing the `withdraw` and `merge` functions (basically all functions that call `_burn` inside them, which currently are only these two aforementioned ones) by the single-token-delegatee operators.

In order to clear the delegated approvals after `burn`'ing a token, instead of calling `_clearApproval` (which should be called instead to fix the error!), the `_burn` function initiates a call to a publicly-exposed `approve(...)` function...
```solidity
function _burn(uint _tokenId) internal {
        require(_isApprovedOrOwner(msg.sender, _tokenId), "caller is not owner nor approved"); // << this check implies that the _burn function CAN be called by any approved operator, whether it's an operator that has approval to operate on all tokens of a particular user, or solely on this particular token.

        address tokenOwner = ownerOf(_tokenId);

        // Clear approval
@@      approve(address(0), _tokenId); // < here's the problem!
```


However, the `approve` function restricts the call to only the `isApprovedForAll` `ownerToOperators[_owner])[_operator]` operator, or the `owner` of the `tokenId`, which is by design.


The `_burn` function is called from within the `merge` and `withdraw` functions.

The `merge` flow:
```solidity
    function merge(uint _from, uint _to) external {
        require(attachments[_from] == 0 && !voted[_from], "attached");
        require(_from != _to);
        require(_isApprovedOrOwner(msg.sender, _from));
        require(_isApprovedOrOwner(msg.sender, _to));

        LockedBalance memory _locked0 = locked[_from];
        LockedBalance memory _locked1 = locked[_to];
        uint value0 = uint(int256(_locked0.amount));
        uint end = _locked0.end >= _locked1.end ? _locked0.end : _locked1.end;

        locked[_from] = LockedBalance(0, 0);
        _checkpoint(_from, _locked0, LockedBalance(0, 0));
@@      _burn(_from);
        _deposit_for(_to, value0, end, _locked1, DepositType.MERGE_TYPE);
    }
```
And the `withdraw` flow is the following:
```solidity
    /// @notice Withdraw all tokens for `_tokenId`
    /// @dev Only possible if the lock has expired
    function withdraw(uint _tokenId) external nonreentrant {
        assert(_isApprovedOrOwner(msg.sender, _tokenId));
        require(attachments[_tokenId] == 0 && !voted[_tokenId], "attached");

        LockedBalance memory _locked = locked[_tokenId];
        require(block.timestamp >= _locked.end, "The lock didn't expire");
        uint value = uint(int256(_locked.amount));

        locked[_tokenId] = LockedBalance(0,0);
        uint supply_before = supply;
        supply = supply_before - value;

        // old_locked can have either expired <= timestamp or zero end
        // _locked has only 0 end
        // Both can have >= 0 amount
        _checkpoint(_tokenId, _locked, LockedBalance(0,0));

        assert(IERC20(lpToken).transfer(msg.sender, value));

        // Burn the NFT
@@      _burn(_tokenId);

        emit Withdraw(msg.sender, _tokenId, value, block.timestamp);
        emit Supply(supply_before, supply_before - value);
    }
```

## Vulnerability Detail
Due to the attempt to clear the approval for a *just-burned `_tokenId`* by calling the `approve` function, the whole `_burn` function gets DoS'ed if the initiator (`msg.sender`) is a solo-token operator (i.e. was approved using the `approve(operator, TOKEN_ID)` call.

As you can guess by looking at the `_isApprovedOrOwner` function's implementation *(the description of which also gives us a hint!)*, it's logical to assume that the `_burn`, `merge` and `withdraw` methods can be called by any delegated operator, ***no matter whether it's an `operator` that has approval for all tokens of the `owner` (like if it is the `isApprovedForAll(owner, operator ) == true`, `ownerToOperators[_owner])[_operator]` account), or it is the operator that has approval <ins>solely</ins> for a particular `tokenId`***:
```solidity
    /// @dev Returns whether the given spender can transfer a given token ID
    /// @param _spender address of the spender to query
    /// @param _tokenId uint ID of the token to be transferred
    /// @return bool whether the msg.sender is approved for the given token ID, is an operator of the owner, or is the owner of the token
    function _isApprovedOrOwner(address _spender, uint _tokenId) internal returns (bool) {
        max_lock(_tokenId);

        address tokenOwner = idToOwner[_tokenId];
        bool spenderIsOwner = tokenOwner == _spender;
        bool spenderIsApproved = _spender == idToApprovals[_tokenId];
        bool spenderIsApprovedForAll = (ownerToOperators[tokenOwner])[_spender];
        return spenderIsOwner || spenderIsApproved || spenderIsApprovedForAll;
    }
```

## Impact
Due to mishandling of the logic that resets the approval, the whole `merge` and `withdraw` flows will be DoS'ed for operators, unless they were granted approvals to operate on all `owner`'s tokens.

As a result, this bug will force the users (`owner`s) to grant full approval **(which implies the right of a 3rd-party entity to operate on all of their tokens!)** to operators. Otherwise, there'll be no other way to delegate operational access for `withdraw` and `merge` functions.

***I believe that DoS is a <ins>Medium</ins> impact in terms of severity.***

### To summarize:
- If users want to delegate operational rights to other users, they'll have no choice other than granting the approval for operating on **ALL** of their token assets, otherwise the grantees won't be able to operate with `merge` and `withdraw` function on the behalf of the tokens that were granted to them.
- In force major circumstances, the user might want to give himself the ownership to his 2nd wallet, for instance, if his wallet may be compromised, but he doesn't have enough gas to cover the transfer fees on his own EOA wallet. The user won't be able to withdraw with his second account.

## PoC
Please add this function to the `test/Imbalance.t.sol` test file and run `forge test --match-contract ImbalanceTest --vvvv`:
```solidity
    function test_votingEscrowMerge() public {
        createLock();

        flowDaiPair.approve(address(escrow), TOKEN_1);
        escrow.create_lock(TOKEN_1, FIFTY_TWO_WEEKS);

        address operator1 = makeAddr("operator1");

        escrow.approve(operator1, 1);
        escrow.approve(operator1, 2);

        vm.startPrank(operator1);

        vm.expectRevert();
        escrow.merge(2, 1); // this will revert on the following lines:
       // in the approve() function that is called from within the _burn function
       // And the burn function is called in merge
       /*
        bool senderIsOwner = (idToOwner[_tokenId] == msg.sender);
        bool senderIsApprovedForAll = (ownerToOperators[tokenOwner])[msg.sender];
        require(senderIsOwner || senderIsApprovedForAll); // check here!!
       */

        
        vm.stopPrank();

        escrow.setApprovalForAll(operator1, true);

        vm.prank(operator1);

        escrow.merge(2, 1);
    }
```

### You'll see the following logs:

<details>

<summary>
<strong>OPEN ME:</strong>
</summary>

```bash
[PASS] test_votingEscrowMerge() (gas: 25558829)
Traces:
  [25558829] ImbalanceTest::test_votingEscrowMerge()
    ├─ [997634] → new TestOwner@0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
    │   └─ ← [Return] 4983 bytes of code
    ├─ [997634] → new TestOwner@0x2e234DAe75C793f67A35089C9d99245E1C58470b
    │   └─ ← [Return] 4983 bytes of code
    ├─ [651216] → new MockERC20@0xF62849F9A0B5Bf2913b396098F7c7019b51A820a
    │   └─ ← [Return] 3018 bytes of code
    ├─ [651216] → new MockERC20@0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9
    │   └─ ← [Return] 3018 bytes of code
    ├─ [651216] → new MockERC20@0xc7183455a4C133Ae270771860664b6B7ec320bB1
    │   └─ ← [Return] 3018 bytes of code
    ├─ [427805] → new Flow@0xa0Cb889707d426A7A386870A03bc70d1b0697598
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: DefaultSender: [0x1804c8AB1F12E6bbf3894d4083f33e07309d1f38], value: 6000000000000000000000000 [6e24])
    │   └─ ← [Return] 1793 bytes of code
    ├─ [651216] → new MockERC20@0x1d1499e622D69689cdf9004d05Ec547d650Ff211
    │   └─ ← [Return] 3018 bytes of code
    ├─ [750210] → new TestWETH@0xA4AD4f68d0b91CFD19687c881e50f3A00242828c
    │   └─ ← [Return] 3518 bytes of code
    ├─ [775969] → new TestToken@0x03A6a84cD762D9707A21605b548aaaB891562aAb
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], value: 0)
    │   └─ ← [Return] 3277 bytes of code
    ├─ [46691] MockERC20::mint(ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 1000000000000000000 [1e18])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], value: 1000000000000000000 [1e18])
    │   └─ ← [Stop]
    ├─ [46691] MockERC20::mint(ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 1000000000000000000000000000000 [1e30])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], value: 1000000000000000000000000000000 [1e30])
    │   └─ ← [Stop]
    ├─ [46691] MockERC20::mint(ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 1000000000000000000000000000000 [1e30])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], value: 1000000000000000000000000000000 [1e30])
    │   └─ ← [Stop]
    ├─ [24791] MockERC20::mint(TestOwner: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 1000000000000000000 [1e18])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: TestOwner: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], value: 1000000000000000000 [1e18])
    │   └─ ← [Stop]
    ├─ [24791] MockERC20::mint(TestOwner: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 1000000000000000000000000000000 [1e30])   
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: TestOwner: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], value: 1000000000000000000000000000000 [1e30])
    │   └─ ← [Stop]
    ├─ [24791] MockERC20::mint(TestOwner: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 1000000000000000000000000000000 [1e30])   
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: TestOwner: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], value: 1000000000000000000000000000000 [1e30])
    │   └─ ← [Stop]
    ├─ [24791] MockERC20::mint(TestOwner: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 1000000000000000000 [1e18])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: TestOwner: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], value: 1000000000000000000 [1e18])
    │   └─ ← [Stop]
    ├─ [24791] MockERC20::mint(TestOwner: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 1000000000000000000000000000000 [1e30])   
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: TestOwner: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], value: 1000000000000000000000000000000 [1e30])
    │   └─ ← [Stop]
    ├─ [24791] MockERC20::mint(TestOwner: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 1000000000000000000000000000000 [1e30])   
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: TestOwner: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], value: 1000000000000000000000000000000 [1e30])
    │   └─ ← [Stop]
    ├─ [24933] Flow::mint(ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 10000000000000000000000000 [1e25])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], value: 10000000000000000000000000 [1e25])
    │   └─ ← [Return] true
    ├─ [521156] → new VeArtProxy@0xD6BbDE9174b1CdAa358d2Cf4D57D1a9F7178FBfF
    │   └─ ← [Return] 2603 bytes of code
    ├─ [4410084] → new PairFactory@0x15cF58144EF33af1e14b5208015d11F9143E27b9
    │   ├─ emit OwnershipTransferred(previousOwner: 0x0000000000000000000000000000000000000000, newOwner: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   └─ ← [Return] 21575 bytes of code
    ├─ [2360] PairFactory::allPairsLength() [staticcall]
    │   └─ ← [Return] 0
    ├─ [2460] PairFactory::setFee(true, 1)
    │   ├─ emit FeeSet(setter: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], stable: true, fee: 1)
    │   └─ ← [Stop]
    ├─ [2450] PairFactory::setFee(false, 1)
    │   ├─ emit FeeSet(setter: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], stable: false, fee: 1)
    │   └─ ← [Stop]
    ├─ [24193] PairFactory::setTank(DefaultSender: [0x1804c8AB1F12E6bbf3894d4083f33e07309d1f38])
    │   ├─ emit TankSet(setter: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], tank: DefaultSender: [0x1804c8AB1F12E6bbf3894d4083f33e07309d1f38])
    │   └─ ← [Stop]
    ├─ [3013438] → new Router@0x212224D2F2d262cd093eE13240ca4873fcCBbA3C
    │   ├─ [7613] PairFactory::pairCodeHash() [staticcall]
    │   │   └─ ← [Return] 0x4744b566107323e5f0ed37adee846c09f8741bf059af84cb30db54539bd25331
    │   └─ ← [Return] 15007 bytes of code
    ├─ [260] Router::factory() [staticcall]
    │   └─ ← [Return] PairFactory: [0x15cF58144EF33af1e14b5208015d11F9143E27b9]
    ├─ [788692] → new VelocimeterLibrary@0x2a07706473244BC757E10F2a9E86fB532828afe3
    │   └─ ← [Return] 3938 bytes of code
    ├─ [0] VM::startPrank(ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   └─ ← [Return]
    ├─ [24544] Flow::approve(Router: [0x212224D2F2d262cd093eE13240ca4873fcCBbA3C], 300000000000000000000 [3e20])
    │   ├─ emit Approval(owner: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], spender: Router: [0x212224D2F2d262cd093eE13240ca4873fcCBbA3C], value: 300000000000000000000 [3e20])
    │   └─ ← [Return] true
    ├─ [24545] MockERC20::approve(Router: [0x212224D2F2d262cd093eE13240ca4873fcCBbA3C], 300000000000000000000 [3e20])
    │   ├─ emit Approval(owner: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], spender: Router: [0x212224D2F2d262cd093eE13240ca4873fcCBbA3C], value: 300000000000000000000 [3e20])
    │   └─ ← [Return] true
    ├─ [3354904] Router::addLiquidity(Flow: [0xa0Cb889707d426A7A386870A03bc70d1b0697598], MockERC20: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], false, 300000000000000000000 [3e20], 300000000000000000000 [3e20], 0, 0, ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 1)
    │   ├─ [2916] PairFactory::getPair(Flow: [0xa0Cb889707d426A7A386870A03bc70d1b0697598], MockERC20: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], false) [staticcall]
    │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000
    │   ├─ [3148782] PairFactory::createPair(Flow: [0xa0Cb889707d426A7A386870A03bc70d1b0697598], MockERC20: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], false)
    │   │   ├─ [2954049] → new Pair@0x796C3a295293a6c8F4ef1B012AfF248063e9199B
    │   │   │   ├─ [2416] PairFactory::voter() [staticcall]
    │   │   │   │   └─ ← [Return] 0x0000000000000000000000000000000000000000
    │   │   │   ├─ [544] PairFactory::getInitializable() [staticcall]
    │   │   │   │   └─ ← [Return] Flow: [0xa0Cb889707d426A7A386870A03bc70d1b0697598], MockERC20: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], false
    │   │   │   ├─ [527] Flow::symbol() [staticcall]
    │   │   │   │   └─ ← [Return] "BVM"
    │   │   │   ├─ [1258] MockERC20::symbol() [staticcall]
    │   │   │   │   └─ ← [Return] "DAI"
    │   │   │   ├─ [527] Flow::symbol() [staticcall]
    │   │   │   │   └─ ← [Return] "BVM"
    │   │   │   ├─ [1258] MockERC20::symbol() [staticcall]
    │   │   │   │   └─ ← [Return] "DAI"
    │   │   │   ├─ [315] Flow::decimals() [staticcall]
    │   │   │   │   └─ ← [Return] 18
    │   │   │   ├─ [249] MockERC20::decimals() [staticcall]
    │   │   │   │   └─ ← [Return] 18
    │   │   │   └─ ← [Return] 14079 bytes of code
    │   │   ├─ emit PairCreated(token0: Flow: [0xa0Cb889707d426A7A386870A03bc70d1b0697598], token1: MockERC20: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], stable: false, pair: Pair: [0x796C3a295293a6c8F4ef1B012AfF248063e9199B], : 1)
    │   │   └─ ← [Return] Pair: [0x796C3a295293a6c8F4ef1B012AfF248063e9199B]
    │   ├─ [6620] Pair::getReserves() [staticcall]
    │   │   └─ ← [Return] 0, 0, 0
    │   ├─ [25812] Flow::transferFrom(ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], Pair: [0x796C3a295293a6c8F4ef1B012AfF248063e9199B], 300000000000000000000 [3e20])
    │   │   ├─ emit Transfer(from: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], to: Pair: [0x796C3a295293a6c8F4ef1B012AfF248063e9199B], value: 300000000000000000000 [3e20])
    │   │   └─ ← [Return] true
    │   ├─ [25601] MockERC20::transferFrom(ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], Pair: [0x796C3a295293a6c8F4ef1B012AfF248063e9199B], 300000000000000000000 [3e20])
    │   │   ├─ emit Transfer(from: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], to: Pair: [0x796C3a295293a6c8F4ef1B012AfF248063e9199B], value: 300000000000000000000 [3e20])
    │   │   └─ ← [Return] true
    │   ├─ [137439] Pair::mint(ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   │   ├─ [586] Flow::balanceOf(Pair: [0x796C3a295293a6c8F4ef1B012AfF248063e9199B]) [staticcall]
    │   │   │   └─ ← [Return] 300000000000000000000 [3e20]
    │   │   ├─ [542] MockERC20::balanceOf(Pair: [0x796C3a295293a6c8F4ef1B012AfF248063e9199B]) [staticcall]
    │   │   │   └─ ← [Return] 300000000000000000000 [3e20]
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x0000000000000000000000000000000000000000, value: 1000)
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], value: 299999999999999999000 [2.999e20])
    │   │   ├─ emit Sync(reserve0: 300000000000000000000 [3e20], reserve1: 300000000000000000000 [3e20])
    │   │   ├─ emit Mint(sender: Router: [0x212224D2F2d262cd093eE13240ca4873fcCBbA3C], weekly: 300000000000000000000 [3e20], circulating_supply: 300000000000000000000 [3e20])
    │   │   └─ ← [Return] 299999999999999999000 [2.999e20]
    │   └─ ← [Return] 300000000000000000000 [3e20], 300000000000000000000 [3e20], 299999999999999999000 [2.999e20]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [916] PairFactory::getPair(Flow: [0xa0Cb889707d426A7A386870A03bc70d1b0697598], MockERC20: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], false) [staticcall]
    │   └─ ← [Return] Pair: [0x796C3a295293a6c8F4ef1B012AfF248063e9199B]
    ├─ [4411439] → new VotingEscrow@0x3D7Ebc40AF7092E3F1C81F2e996cbA5Cae2090d7
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: VotingEscrow: [0x3D7Ebc40AF7092E3F1C81F2e996cbA5Cae2090d7], tokenId: 0)
    │   ├─ emit Transfer(from: VotingEscrow: [0x3D7Ebc40AF7092E3F1C81F2e996cbA5Cae2090d7], to: 0x0000000000000000000000000000000000000000, tokenId: 0)
    │   └─ ← [Return] 20893 bytes of code
    ├─ [24607] Pair::approve(VotingEscrow: [0x3D7Ebc40AF7092E3F1C81F2e996cbA5Cae2090d7], 1000000000000000000 [1e18])
    │   ├─ emit Approval(owner: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], spender: VotingEscrow: [0x3D7Ebc40AF7092E3F1C81F2e996cbA5Cae2090d7], value: 1000000000000000000 [1e18])
    │   └─ ← [Return] true
    ├─ [469652] VotingEscrow::create_lock(1000000000000000000 [1e18], 31449600 [3.144e7])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], tokenId: 1)
    │   ├─ [27634] Pair::transferFrom(ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], VotingEscrow: [0x3D7Ebc40AF7092E3F1C81F2e996cbA5Cae2090d7], 1000000000000000000 [1e18])
    │   │   ├─ emit Approval(owner: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], spender: VotingEscrow: [0x3D7Ebc40AF7092E3F1C81F2e996cbA5Cae2090d7], value: 0)
    │   │   ├─ emit Transfer(from: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], to: VotingEscrow: [0x3D7Ebc40AF7092E3F1C81F2e996cbA5Cae2090d7], value: 1000000000000000000 [1e18])
    │   │   └─ ← [Return] true
    │   ├─ emit Deposit(provider: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], tokenId: 1, value: 1000000000000000000 [1e18], locktime: 31449600 [3.144e7], deposit_type: 1, ts: 1)
    │   ├─ emit Supply(prevSupply: 0, supply: 1000000000000000000 [1e18])
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000001
    ├─ [0] VM::warp(1)
    │   └─ ← [Return]
    ├─ [4196] VotingEscrow::balanceOfNFT(1) [staticcall]
    │   └─ ← [Return] 999999968174574804 [9.999e17]
    ├─ [559] Pair::balanceOf(VotingEscrow: [0x3D7Ebc40AF7092E3F1C81F2e996cbA5Cae2090d7]) [staticcall]
    │   └─ ← [Return] 1000000000000000000 [1e18]
    ├─ [22507] Pair::approve(VotingEscrow: [0x3D7Ebc40AF7092E3F1C81F2e996cbA5Cae2090d7], 1000000000000000000 [1e18])
    │   ├─ emit Approval(owner: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], spender: VotingEscrow: [0x3D7Ebc40AF7092E3F1C81F2e996cbA5Cae2090d7], value: 1000000000000000000 [1e18])
    │   └─ ← [Return] true
    ├─ [360157] VotingEscrow::create_lock(1000000000000000000 [1e18], 31449600 [3.144e7])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], tokenId: 2)
    │   ├─ [5734] Pair::transferFrom(ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], VotingEscrow: [0x3D7Ebc40AF7092E3F1C81F2e996cbA5Cae2090d7], 1000000000000000000 [1e18])
    │   │   ├─ emit Approval(owner: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], spender: VotingEscrow: [0x3D7Ebc40AF7092E3F1C81F2e996cbA5Cae2090d7], value: 0)
    │   │   ├─ emit Transfer(from: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], to: VotingEscrow: [0x3D7Ebc40AF7092E3F1C81F2e996cbA5Cae2090d7], value: 1000000000000000000 [1e18])
    │   │   └─ ← [Return] true
    │   ├─ emit Deposit(provider: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], tokenId: 2, value: 1000000000000000000 [1e18], locktime: 31449600 [3.144e7], deposit_type: 1, ts: 1)
    │   ├─ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 2000000000000000000 [2e18])
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000002
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← [Return] operator1: [0x204c495D35F172f9c22ACb5B8E340805DAF9482e]
    ├─ [0] VM::label(operator1: [0x204c495D35F172f9c22ACb5B8E340805DAF9482e], "operator1")
    │   └─ ← [Return]
    ├─ [27378] VotingEscrow::approve(operator1: [0x204c495D35F172f9c22ACb5B8E340805DAF9482e], 1)
    │   ├─ emit Approval(owner: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], approved: operator1: [0x204c495D35F172f9c22ACb5B8E340805DAF9482e], tokenId: 1)
    │   └─ ← [Stop]
    ├─ [25378] VotingEscrow::approve(operator1: [0x204c495D35F172f9c22ACb5B8E340805DAF9482e], 2)
    │   ├─ emit Approval(owner: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], approved: operator1: [0x204c495D35F172f9c22ACb5B8E340805DAF9482e], tokenId: 2)
    │   └─ ← [Stop]
    ├─ [0] VM::startPrank(operator1: [0x204c495D35F172f9c22ACb5B8E340805DAF9482e])
    │   └─ ← [Return]
    ├─ [0] VM::expectRevert(custom error f4844814:)
    │   └─ ← [Return]
    ├─ [135316] VotingEscrow::merge(2, 1)
    │   └─ ← [Revert] EvmError: Revert
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [24621] VotingEscrow::setApprovalForAll(operator1: [0x204c495D35F172f9c22ACb5B8E340805DAF9482e], true)
    │   ├─ emit ApprovalForAll(owner: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], operator: operator1: [0x204c495D35F172f9c22ACb5B8E340805DAF9482e], approved: true)
    │   └─ ← [Stop]
    ├─ [0] VM::prank(operator1: [0x204c495D35F172f9c22ACb5B8E340805DAF9482e])
    │   └─ ← [Return]
    ├─ [334727] VotingEscrow::merge(2, 1)
    │   ├─ emit Approval(owner: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], approved: 0x0000000000000000000000000000000000000000, tokenId: 2)
    │   ├─ emit Transfer(from: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], to: 0x0000000000000000000000000000000000000000, tokenId: 2)
    │   ├─ emit Deposit(provider: operator1: [0x204c495D35F172f9c22ACb5B8E340805DAF9482e], tokenId: 1, value: 1000000000000000000 [1e18], locktime: 31449600 [3.144e7], deposit_type: 4, ts: 1)
    │   ├─ emit Supply(prevSupply: 2000000000000000000 [2e18], supply: 3000000000000000000 [3e18])
    │   └─ ← [Stop]
    └─ ← [Stop]
```

</details>

## Code Snippet
For reference, the callstack flow that causes the revert is the following, in this specific order:
- https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/VotingEscrow.sol#L975 (← the `withdraw` entrypoint);
- https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/VotingEscrow.sol#L1208 (← the `merge` entrypoint);
- https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/VotingEscrow.sol#L537C1-L549C6 (← the `_burn` function's filling);
- https://github.com/sherlock-audit/2024-06-velocimeter/blob/63818925987a5115a80eff4bd12578146a844cfd/v4-contracts/contracts/VotingEscrow.sol#L251C1-L264C6 (← the `approve` function).

## Tool used
Manual human review.

## Recommendation
Consider calling `_clearApproval` in the `_burn` function instead:

```diff
    function _burn(uint _tokenId) internal {
        require(_isApprovedOrOwner(msg.sender, _tokenId), "caller is not owner nor approved");

        address tokenOwner = ownerOf(_tokenId);

        // Clear approval
-      approve(address(0), _tokenId);
+      clearApproval(tokenOwner, _tokenId);
        // checkpoint for gov
        _moveTokenDelegates(delegates(tokenOwner), address(0), _tokenId);
        // Remove token
        _removeTokenFrom(tokenOwner, _tokenId);
        emit Transfer(tokenOwner, address(0), _tokenId);
    }
```

Because the `_clearApproval` function doesn't require any specific rights and is just an internal utility:
```solidity
// ....

    /* TRANSFER FUNCTIONS */
    /// @dev Clear an approval of a given address
    ///      Throws if `_owner` is not the current owner.
    function _clearApproval(address _owner, uint _tokenId) internal {
        // Throws if `_owner` is not the current owner
        assert(idToOwner[_tokenId] == _owner);
        if (idToApprovals[_tokenId] != address(0)) {
            // Reset approvals
            idToApprovals[_tokenId] = address(0);
        }
    }
```

Then you can verify that the bug is fixed by running the test case again:
```bash
  │   ├─ emit Supply(prevSupply: 1000000000000000000 [1e18], supply: 2000000000000000000 [2e18])
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000000000000000000000000002
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← [Return] operator1: [0x204c495D35F172f9c22ACb5B8E340805DAF9482e]
    ├─ [0] VM::label(operator1: [0x204c495D35F172f9c22ACb5B8E340805DAF9482e], "operator1")
    │   └─ ← [Return]
    ├─ [27378] VotingEscrow::approve(operator1: [0x204c495D35F172f9c22ACb5B8E340805DAF9482e], 1)
    │   ├─ emit Approval(owner: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], approved: operator1: [0x204c495D35F172f9c22ACb5B8E340805DAF9482e], tokenId: 1)
    │   └─ ← [Stop]
    ├─ [25378] VotingEscrow::approve(operator1: [0x204c495D35F172f9c22ACb5B8E340805DAF9482e], 2)
    │   ├─ emit Approval(owner: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], approved: operator1: [0x204c495D35F172f9c22ACb5B8E340805DAF9482e], tokenId: 2)
    │   └─ ← [Stop]
    ├─ [0] VM::startPrank(operator1: [0x204c495D35F172f9c22ACb5B8E340805DAF9482e])
    │   └─ ← [Return]
    ├─ [0] VM::expectRevert(custom error f4844814:)
    │   └─ ← [Return]
    ├─ [334441] VotingEscrow::merge(2, 1)
    │   ├─ emit Transfer(from: ImbalanceTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], to: 0x0000000000000000000000000000000000000000, tokenId: 2)
    │   ├─ emit Deposit(provider: operator1: [0x204c495D35F172f9c22ACb5B8E340805DAF9482e], tokenId: 1, value: 1000000000000000000 [1e18], locktime: 31449600 [3.144e7], deposit_type: 4, ts: 1)
    │   ├─ emit Supply(prevSupply: 2000000000000000000 [2e18], supply: 3000000000000000000 [3e18])
    │   └─ ← [Stop]
    └─ ← [Revert] call did not revert as expected

Suite result: FAILED. 1 passed; 1 failed; 0 skipped; finished in 42.15ms (37.43ms CPU time)

Ran 1 test suite in 1.44s (42.15ms CPU time): 1 tests passed, 1 failed, 0 skipped (2 total tests)

Failing tests:
Encountered 1 failing test in test/Imbalance.t.sol:ImbalanceTest
[FAIL. Reason: call did not revert as expected] test_votingEscrowMerge() (gas: 25635490)
```

## TL;DR;
This causes a permanent unresolvable DoS upon accessing the `withdraw` and `merge` functions by the single-token-delegatees operators.