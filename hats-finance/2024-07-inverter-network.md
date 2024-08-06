# Inverter Network

The code under review can be found in [`here`](https://github.com/hats-finance/Inverter-Network-0xe47e52c4fea05e555920f1dcdcc6fb8eca103eeb).

## Findings Summary

| ID | Description | Severity |
| :-: | - | :-: |
| [L-01](#l-01-missing-event-emission-in-aut_ext_votingroles_v1solcastvote) | Missing event emission in `AUT_EXT_VotingRoles_v1.sol::castVote`.| Low |
| [L-02](#l-02-incorrect-time-check-in-srcmodulespaymentprocessorpp_streaming_v1solvalidtimes) | Incorrect time check in `src/modules/paymentProcessor/PP_Streaming_v1.sol::validTimes` | Low |
| [L-03](#l-03-indexed-keyword-in-events-causes-data-loss-for-dynamic-array-variables) | `indexed` keyword in Events Causes Data Loss for Dynamic Array Variables | Low |
| [L-04](#l-04-rounding-issue-in-rewards-calculation-in-lm_pc_kpirewarder_v1solassertionresolvedcallback) | Rounding issue in rewards calculation in `LM_PC_KPIRewarder_v1.sol::assertionResolvedCallback`  | Low |

## [L-01] Missing event emission in `AUT_EXT_VotingRoles_v1.sol::castVote`.

### Bug Description

The function `AUT_EXT_VotingRoles_v1.sol::castVote` is missing event emission. Emitting events when these actions occur is essential for transparency and informing users about important changes. Adding events for these functions will provide users with a clear record of when the votes has been casted.

### Attack Scenario

The absence of events for these critical functions may result in a lack of transparency, making it difficult for users to track important changes in the contract. Emitting events can ensures that users are informed.



### Recommended Mitigation

Consider adding event emission in `AUT_EXT_VotingRoles_v1.sol::castVote`.

## [L-02] Incorrect time check in `src/modules/paymentProcessor/PP_Streaming_v1.sol::validTimes`

### Bug Description

The time validation check is incorrectly implemented in `src/modules/paymentProcessor/PP_Streaming_v1.sol::validTimes`.

In `src/modules/paymentProcessor/PP_Streaming_v1.sol::validTimes`:
```javascript
/// @notice validate uint start input.
    /// @param _start uint to validate.
    /// @param _cliff uint to validate.
    /// @param _end uint to validate.
    /// @return True if uint is valid.
    function validTimes(uint _start, uint _cliff, uint _end)
        internal
        pure
        returns (bool)
    {
        return !(_start >= type(uint).max && _start + _cliff > _end);
    }
```
In this function, `||` should be used instead of `&&` operator. As a result, invalid times will be treated as valid.



### Attack Scenario

Invalid times being treated as valid, this will put protocol in an unexpected state.


### Proof of Concept

Let's take a look at the equation of `src/modules/paymentProcessor/PP_Streaming_v1.sol::validTimes`, in the current implementation:
```javascript
 return !(_start >= type(uint).max && _start + _cliff > _end);
```

As long as the value of `_start` < `type(uint).max`, the times will be treated as valid (Because of the `&&` operator), which is incorrect.


### Recommended Mitigation

Consider making the following changes in `src/modules/paymentProcessor/PP_Streaming_v1.sol::validTimes`:

```diff
/// @notice validate uint start input.
    /// @param _start uint to validate.
    /// @param _cliff uint to validate.
    /// @param _end uint to validate.
    /// @return True if uint is valid.
    function validTimes(uint _start, uint _cliff, uint _end)
        internal
        pure
        returns (bool)
    {
--      return !(_start >= type(uint).max && _start + _cliff > _end);
++      return !(_start >= type(uint).max || _start + _cliff > _end);
   
 }
```




## [L-03] `indexed` keyword in Events Causes Data Loss for Dynamic Array Variables 


### Bug Description

when the `indexed` keyword is used for reference type variables such as dynamic arrays or strings, it will return the hash of the mentioned variables.
Thus, the event which is supposed to inform all of the applications subscribed to its emitting transaction (e.g. front-end of the DApp, or the backend listeners to that event), would get a meaningless and obscure 32 bytes that correspond to keccak256 of an encoded string. This may cause some problems on the DApp side and even lead to data loss. For more information about the indexed events, check here:
(https://docs.soliditylang.org/en/v0.8.17/abi-spec.html?highlight=indexed#events)

The problem exists inside the `LM_PC_Bounties_v1` contract. The `eventClaimAdded` is defined in such a way that the dynamical array of `Contributor[]` is indexed. With doing so, the expected parameters wouldn't be emitted properly and front-end would get meaningless one-way hashes.


### Attack Scenario

Meaningless and obscure 32 bytes that correspond to keccak256 of an encoded string events will be emitted.

### Proof of Concept

Paste the following test in `test/modules/logicModule/paymentClient/LM_PC_Bounties_v1.t.sol`:

```javascript
event ClaimAdded1(
        uint indexed claimId,
        uint indexed bountyId,
        ILM_PC_Bounties_v1.Contributor[] contributors,
        bytes details
    );

    function test_emitter() public {
        ILM_PC_Bounties_v1.Contributor[] memory contributors = new ILM_PC_Bounties_v1.Contributor[](3);
        contributors[0].addr = address(123);
        contributors[0].claimAmount = 100;
        contributors[1].addr = address(321);
        contributors[1].claimAmount = 200;
        contributors[2].addr = address(111);
        contributors[2].claimAmount = 300;

        emit ClaimAdded(1, 1, contributors,new bytes(0));
        emit ClaimAdded1(1, 1, contributors,new bytes(0));
    }
```

Result:
```javascript
 [9453] LM_PC_BountiesV1Test::test_emitter()
    ├─ emit ClaimAdded(claimId: 1, bountyId: 1, contributors: 0xa300377439e337de50ee8f0f7713c580a4e168467afdeeece93c0ddb83553143, details: 0x)
    ├─ emit ClaimAdded1(claimId: 1, bountyId: 1, contributors: [Contributor({ addr: 0x000000000000000000000000000000000000007B, claimAmount: 100 }), Contributor({ addr: 0x0000000000000000000000000000000000000141, claimAmount: 200 }), Contributor({ addr: 0x000000000000000000000000000000000000006F, claimAmount: 300 })], details: 0x)
    └─ ← ()
```

From the result, the contributors in ClaimAdded is a meaningless and obscure 32 bytes that correspond to keccak256 of contributors value.


### Recommended Mitigation

Consider make the following changes in events `ClaimAdded` & `ClaimContributorsUpdated`:
```diff
 event ClaimAdded(
        uint indexed claimId,
        uint indexed bountyId,
--      Contributor[] indexed contributors,
++      Contributor[] contributors,
        bytes details
    );
```

```diff
event ClaimContributorsUpdated(
--        uint indexed claimId, Contributor[] indexed contributors
++        uint indexed claimId, Contributor[] contributors
    );
```

## [L-04] Rounding issue in rewards calculation in `LM_PC_KPIRewarder_v1.sol::assertionResolvedCallback`.

### Bug Description


In `LM_PC_KPIRewarder_v1.sol::assertionResolvedCallback`:
```javascript
 function assertionResolvedCallback(
        bytes32 assertionId,
        bool assertedTruthfully
    ) public override {
        // First, we perform checks and state management on the parent function.
        super.assertionResolvedCallback(assertionId, assertedTruthfully);

        // If the assertion was true, we calculate the rewards and distribute them.
        if (assertedTruthfully) {
            // SECURITY NOTE: this will add the value, but provides no guarantee that the fundingmanager actually holds those funds.

            // Calculate rewardamount from assertionId value
            KPI memory resolvedKPI =
                registryOfKPIs[assertionConfig[assertionId].KpiToUse];
            uint rewardAmount;

            for (uint i; i < resolvedKPI.numOfTranches; i++) {
                if (
                    resolvedKPI.trancheValues[i]
                        <= assertionConfig[assertionId].assertedValue
                ) {
                    // the asserted value is above tranche end
                    rewardAmount += resolvedKPI.trancheRewards[i];
                } else {
                    // tranche was not completed
                    if (resolvedKPI.continuous) {
                        // continuous distribution
                        uint trancheRewardValue = resolvedKPI.trancheRewards[i];
                        uint trancheStart =
                            i == 0 ? 0 : resolvedKPI.trancheValues[i - 1];

                        uint achievedReward = assertionConfig[assertionId]
                            .assertedValue - trancheStart;
                        uint trancheEnd =
                            resolvedKPI.trancheValues[i] - trancheStart;

                        rewardAmount +=
                            achievedReward * (trancheRewardValue / trancheEnd); // since the trancheRewardValue will be a very big number.
                    }
                    // else -> no reward

                    // exit the loop
                    break;
                }
            }

            _setRewards(rewardAmount, 1);
            assertionConfig[assertionId].distributed = true;
        } else {
            // To keep in line with the upstream contract. If the assertion was false, we delete the corresponding assertionConfig from storage.
            delete assertionConfig[assertionId];
        }

        // Independently of the fact that the assertion resolved true or not, new assertions can now be posted.
        assertionPending = false;
    }
```
Notice the line `rewardAmount += achievedReward * (trancheRewardValue / trancheEnd);` contains curly braces, that means if `trancheRewardValue < trancheEnd`, the value will be rounded down to zero. As a result, the rewards will be lost.


### Attack Scenario
If `trancheRewardValue < trancheEnd`, rewards will be rounded down to zero. As a result, rewards will be lost. Additionally, the function will revert due to the validAmount modifier in `_setRewards` function as the rewardAmount is 0.

Not only that, it will also prevent new assertions from being posted (as assertionPending still remains true), putting the protocol in the state of denial of service.

### Proof of Concept

Add `test_RewardsRoundedDownZero` test to `LM_PC_KPIRewarder_v1.t.sol` with the following content:
```javascript
function test_RewardsRoundedDownZero() public {
        uint totalUserFunds;
        address[] memory stakers = new address[](3);
        uint[] memory amounts = new uint[](3);

        for(uint i; i < stakers.length; i++) {
            stakers[i] = address(uint160(uint256(keccak256(abi.encodePacked(i)))));
            amounts[i] = 1 + i;
        }

        for (uint i = 0; i < stakers.length; i++) {
            stakingToken.mint(stakers[i], amounts[i]);
            vm.startPrank(stakers[i]);
            stakingToken.approve(address(kpiManager), amounts[i]);
            kpiManager.stake(amounts[i]);
            totalUserFunds += amounts[i];
            vm.stopPrank();
        }


        //create KPI
        uint[] memory trancheValues = new uint[](2);
        uint[] memory trancheRewards = new uint[](2);

        trancheValues[0] = 15;
        trancheValues[1] = 20;
        //trancheValues[2] = 300;

        trancheRewards[0] = 5;
        trancheRewards[1] = 10;
        //trancheRewards[2] = 3;

        kpiManager.createKPI(true, trancheValues, trancheRewards);

        // prepare  bond and asserter authorization
        kpiManager.grantModuleRole(
            kpiManager.ASSERTER_ROLE(), MOCK_ASSERTER_ADDRESS
        );
        feeToken.mint(
            address(MOCK_ASSERTER_ADDRESS),
            ooV3.getMinimumBond(address(feeToken))
        ); //
        vm.startPrank(address(MOCK_ASSERTER_ADDRESS));
        feeToken.approve(
            address(kpiManager), ooV3.getMinimumBond(address(feeToken))
        );
        vm.stopPrank();


        vm.startPrank(address(MOCK_ASSERTER_ADDRESS));
         bytes32 assertionId = kpiManager.postAssertion(
            MOCK_ASSERTION_DATA_ID, 10, MOCK_ASSERTER_ADDRESS, 0
        );
        vm.stopPrank();

         // state after
        assertEq(kpiManager.getStakingQueue().length, 0);
        assertEq(kpiManager.totalQueuedFunds(), 0);

        for (uint i = 0; i < stakers.length; i++) {
            assertEq(stakingToken.balanceOf(stakers[i]), 0);
        }

        assertEq(feeToken.balanceOf(MOCK_ASSERTER_ADDRESS), 0);

        vm.warp(block.timestamp + DEFAULT_LIVENESS); //fast forward to settle assertion
        ooV3.settleAssertion(assertionId);



    }
```
Run the test with `forge test --mt test_RewardsRoundedDownZero -vvvv`:

Result 1:
```javascript
Running 1 test for test/modules/logicModule/LM_PC_KPIRewarder_v1.t.sol:LM_PC_KPIRewarder_v1_postAssertionTest
[FAIL. Reason: Module__ERC20PaymentClientBase__InvalidAmount()] test_RewardsRoundedDownZero() (gas: 1439480)
Traces:
  [15679213] LM_PC_KPIRewarder_v1_postAssertionTest::setUp()

//REDACTED

 ├─ [30505] OptimisticOracleV3Mock::settleAssertion(0xebb7ec143087972b093c50b75d1a3d2e307d389ad7f9b7b9c36a29f6c95c0751)
    │   ├─ [20535] ERC20Mock::transfer(0x0000000000000000000000000000000000000001, 1000000000000000000 [1e18])
    │   │   ├─ emit Transfer(from: OptimisticOracleV3Mock: [0xA4AD4f68d0b91CFD19687c881e50f3A00242828c], to: 0x0000000000000000000000000000000000000001, value: 1000000000000000000 [1e18])
    │   │   └─ ← true
    │   ├─ [8769] 0xD6BbDE9174b1CdAa358d2Cf4D57D1a9F7178FBfF::assertionResolvedCallback(0xebb7ec143087972b093c50b75d1a3d2e307d389ad7f9b7b9c36a29f6c95c0751, true)
    │   │   ├─ [8592] LM_PC_KPIRewarder_v1::assertionResolvedCallback(0xebb7ec143087972b093c50b75d1a3d2e307d389ad7f9b7b9c36a29f6c95c0751, true) [delegatecall]
    │   │   │   ├─ [776] 0x2a07706473244BC757E10F2a9E86fB532828afe3::isTrustedForwarder(OptimisticOracleV3Mock: [0xA4AD4f68d0b91CFD19687c881e50f3A00242828c]) [staticcall]
    │   │   │   │   ├─ [604] OrchestratorV1Mock::isTrustedForwarder(OptimisticOracleV3Mock: [0xA4AD4f68d0b91CFD19687c881e50f3A00242828c]) [delegatecall]
    │   │   │   │   │   └─ ← false
    │   │   │   │   └─ ← false
    │   │   │   ├─ emit DataAssertionResolved(assertedTruthfully: true, dataId: 0x3078313233340000000000000000000000000000000000000000000000000000, data: 0x000000000000000000000000000000000000000000000000000000000000000a, asserter: 0x0000000000000000000000000000000000000001, assertionId: 0xebb7ec143087972b093c50b75d1a3d2e307d389ad7f9b7b9c36a29f6c95c0751)
    │   │   │   └─ ← Module__ERC20PaymentClientBase__InvalidAmount()
    │   │   └─ ← Module__ERC20PaymentClientBase__InvalidAmount()
    │   └─ ← Module__ERC20PaymentClientBase__InvalidAmount()
    └─ ← Module__ERC20PaymentClientBase__InvalidAmount()

Test result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 10.35ms
 
Ran 1 test suites: 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/modules/logicModule/LM_PC_KPIRewarder_v1.t.sol:LM_PC_KPIRewarder_v1_postAssertionTest
[FAIL. Reason: Module__ERC20PaymentClientBase__InvalidAmount()] test_RewardsRoundedDownZero() (gas: 1439480)
```

Remove the curly braces in the `LM_PC_KPIRewarder_v1.sol::assertionResolvedCallback#L386` and run the same test again, this time you will notice the test pass with rewardAmount = 3:

Result 2:
```javascript
Running 1 test for test/modules/logicModule/LM_PC_KPIRewarder_v1.t.sol:LM_PC_KPIRewarder_v1_postAssertionTest
[PASS] test_RewardsRoundedDownZero() (gas: 1217083)
Traces:
  [1221296] LM_PC_KPIRewarder_v1_postAssertionTest::test_RewardsRoundedDownZero()
    ├─ [46856] ERC20Mock::mint(0x88386Fc84bA6bC95484008F6362F93160eF3e563, 1)
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: 0x88386Fc84bA6bC95484008F6362F93160eF3e563, value: 1)
    │   └─ ← ()
    ├─ [0] VM::startPrank(0x88386Fc84bA6bC95484008F6362F93160eF3e563)
    │   └─ ← ()
    ├─ [24784] ERC20Mock::approve(0xD6BbDE9174b1CdAa358d2Cf4D57D1a9F7178FBfF, 1)
    │   ├─ emit Approval(owner: 0x88386Fc84bA6bC95484008F6362F93160eF3e563, spender: 0xD6BbDE9174b1CdAa358d2Cf4D57D1a9F7178FBfF, value: 1)
    │   └─ ← true
    ├─ [130117] 0xD6BbDE9174b1CdAa358d2Cf4D57D1a9F7178FBfF::stake(1)
    │   ├─ [127982] LM_PC_KPIRewarder_v1::stake(1) [delegatecall]
//REDACTED

├─ [100208] OptimisticOracleV3Mock::settleAssertion(0xebb7ec143087972b093c50b75d1a3d2e307d389ad7f9b7b9c36a29f6c95c0751)
    │   ├─ [20535] ERC20Mock::transfer(0x0000000000000000000000000000000000000001, 1000000000000000000 [1e18])
    │   │   ├─ emit Transfer(from: OptimisticOracleV3Mock: [0xA4AD4f68d0b91CFD19687c881e50f3A00242828c], to: 0x0000000000000000000000000000000000000001, value: 1000000000000000000 [1e18])
    │   │   └─ ← true
    │   ├─ [74709] 0xD6BbDE9174b1CdAa358d2Cf4D57D1a9F7178FBfF::assertionResolvedCallback(0xebb7ec143087972b093c50b75d1a3d2e307d389ad7f9b7b9c36a29f6c95c0751, true)
    │   │   ├─ [74569] LM_PC_KPIRewarder_v1::assertionResolvedCallback(0xebb7ec143087972b093c50b75d1a3d2e307d389ad7f9b7b9c36a29f6c95c0751, true) [delegatecall]
    │   │   │   ├─ [776] 0x2a07706473244BC757E10F2a9E86fB532828afe3::isTrustedForwarder(OptimisticOracleV3Mock: [0xA4AD4f68d0b91CFD19687c881e50f3A00242828c]) [staticcall]
    │   │   │   │   ├─ [604] OrchestratorV1Mock::isTrustedForwarder(OptimisticOracleV3Mock: [0xA4AD4f68d0b91CFD19687c881e50f3A00242828c]) [delegatecall]
    │   │   │   │   │   └─ ← false
    │   │   │   │   └─ ← false
    │   │   │   ├─ emit DataAssertionResolved(assertedTruthfully: true, dataId: 0x3078313233340000000000000000000000000000000000000000000000000000, data: 0x000000000000000000000000000000000000000000000000000000000000000a, asserter: 0x0000000000000000000000000000000000000001, assertionId: 0xebb7ec143087972b093c50b75d1a3d2e307d389ad7f9b7b9c36a29f6c95c0751)
    │   │   │   ├─ emit RewardSet(rewardAmount: 3, duration: 1, newRewardRate: 3, newRewardsEnd: 25002 [2.5e4])
    │   │   │   └─ ← ()
    │   │   └─ ← ()
    │   ├─ emit AssertionSettled(assertionId: 0xebb7ec143087972b093c50b75d1a3d2e307d389ad7f9b7b9c36a29f6c95c0751, bondRecipient: 0x0000000000000000000000000000000000000001, disputed: false, settlementResolution: true, settleCaller: LM_PC_KPIRewarder_v1_postAssertionTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   └─ ← ()
    └─ ← ()

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 10.12ms
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Recommended Mitigation

In mathematics, multiplication and division can be done interchangeably, therefore the curly braces is not needed to prevent value getting rounded down to zero. Consider make the following changes in `LM_PC_KPIRewarder_v1.sol::assertionResolvedCallback Line 386`:
```diff
rewardAmount +=
--                            achievedReward * (trancheRewardValue / trancheEnd); 
++                           achievedReward * trancheRewardValue / trancheEnd; 
```