# Ether.Fi

The code under review can be found in [`here`](https://github.com/hats-finance/ether-fi-0x36c3b77853dec9c4a237a692623293223d4b9bc4/tree/180c708dc7cb3214d68ea9726f1999f67c3551c9).

## Findings Summary

| ID | Description | Severity |
| :-: | - | :-: |
| [L-01](#l-01-etherfiadmin.sol::pause()-function-unusable) | EtherFiAdmin.sol::pause() function unusable | Low |


## [L-01] EtherFiAdmin.sol::pause() function unusable

### Bug Description

In `EtherFiAdmin.sol::pause()`:
[EtherFiAdmin.sol#L75-L106](https://github.com/hats-finance/ether-fi-0x36c3b77853dec9c4a237a692623293223d4b9bc4/blob/180c708dc7cb3214d68ea9726f1999f67c3551c9/src/EtherFiAdmin.sol#L75-L106)

```solidity
    function pause(bool _etherFiOracle, bool _stakingManager, bool _auctionManager, bool _etherFiNodesManager, bool _liquidityPool, bool _membershipManager) external isAdmin() {
        if (_etherFiOracle) {
            etherFiOracle.pauseContract();
        } else {
            etherFiOracle.unPauseContract();
        }
        if (_stakingManager) {
            stakingManager.pauseContract();
        } else {
            stakingManager.unPauseContract();
        }
        if (_auctionManager) {
            auctionManager.pauseContract();
        } else {
            auctionManager.unPauseContract();
        }
        if (_etherFiNodesManager) {
            etherFiNodesManager.pauseContract();
        } else {
            etherFiNodesManager.unPauseContract();
        }
        if (_liquidityPool) {
            liquidityPool.pauseContract();
        } else {
            liquidityPool.unPauseContract();
        }
        if (_membershipManager) {
            membershipManager.pauseContract();
        } else {
            membershipManager.unPauseContract();
        }
    }
```

The function `EtherFiAdmin.sol::pause()` is unusable because of `onlyOwner` and `whenPaused` modifiers.


### Attack Scenario

Let's suppose owner tries to pause only the `EtherFiOracle.sol` contract, he will call `EtherFiAdmin.sol::pause(true,false,false,false,false,false)`. However, the call will revert because of `onlyOwner` modifier. The only caller that can call `theEtherFiOracle.sol::pauseContract()` is the deployer of `EtherFiOracle.sol`. Even if it is called by the right user, the call will still revert because of `whenPaused` modifier in `PausableUpgradeable.sol`. This is because the contract `EtherFiAdmin.sol` tries to unpause the contracts in `else` statement even it is not in paused status.

### Impact

As a result, admins calling `EtherFiAdmin.sol::pause()` function will get reverted unexpectedly and this will be worse in case of an emergency scenario such as when part of the contracts must be pause due to attacks etc.


### Proof Of Concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "./TestSetup.sol";
import "forge-std/console2.sol";

contract EtherFiAdminTest is TestSetup {

    function setUp() public {

        setUpTests();

    }


    function test_cannotPauseProperly() public {

        vm.startPrank(owner);
        etherFiAdminInstance.updateAdmin(owner,true);
        vm.expectRevert("Pausable: not paused");
        etherFiOracleInstance.unPauseContract(); //@audit demonstration of a revert scenario when contract not in paused status.

        vm.expectRevert("Ownable: caller is not the owner");
        etherFiAdminInstance.pause(true,false,false,false,false,false); //@audit this will fails due to caller is not owner.
        vm.stopPrank();
    

    }
```


### Recommended Mitigation

Check whether the contract(s) are in pause status first instead of directly calling `unpauseContract()` in the else statements.

