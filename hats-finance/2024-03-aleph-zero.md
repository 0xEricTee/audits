# Aleph Zero Bridge

The original submission can be found in [`here`](https://github.com/hats-finance/Most--Aleph-Zero-Bridge-0xab7c1d45ae21e7133574746b2985c58e0ae2e61d/issues/3).

## Findings Summary

| ID | Description | Severity |
| :-: | - | :-: |
| [L-01](#l-01-wrong-implementation-in-migrations-upgrade-causes-this-function-doesnt-work-as-expected) | Wrong implementation in Migrations::upgrade causes this function doesn't work as expected. | Low |


## [L-01] Wrong implementation in Migrations::upgrade causes this function doesn't work as expected.

### Bug Description

Wrong implementation in Migrations::upgrade causes this function doesn't work as expected.


### Attack Scenario

In `Migrations::upgrade`:
```javascript
   function upgrade(address new_address) public restricted {
        Migrations upgraded = Migrations(new_address);
        upgraded.setCompleted(last_completed_migration);
    }

```

This function let owner to specify `address new_address` and the contract will call `setCompleted` in that address and set the value to `last_completed_migration`. However, the value of `last_completed_migration` in `upgraded` address will not be able to set properly because of `restricted` modifier.

The issue arises as the old migration contract trying to call `setCompleted` function on new migration contract however the old migration contract is not the `deployer/owner` of the new migration contract. As a result, the `last_completed_migration` value in new migration contract cannot be set properly by calling `Migrations::upgrade` function.

Proof of concept is given below for more details.

### Proof Of Concept

* Install foundry.

* Create an empty folder named `aleph_zero`.

* Inside this folder, execute forge init.

* Add the `Migrations.sol` contract to the `/src` folder.

* Add the `Migrations.t.sol` contract inside `/test` folder with the following content:

```javascript
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console2 as console} from "forge-std/Test.sol";
import {Migrations} from "../src/Migrations.sol";

contract MigrationsTest is Test {
    Migrations public migration;

    function setUp() public {
        migration = new Migrations();
    }

    function test_UpgradeNotWorking() external {
        migration.setCompleted(1);
        require(migration.last_completed_migration() == uint(1));

        // Owner decides to migrate to new migration contract.

        Migrations migration2 = new Migrations();
        migration.upgrade(address(migration2));

        //Expect the last_completed_migration of migration2 to be 1 but in reality the value is 0.
        console.log("Expected last_completed_migration of Migration2: 1");
        console.log("Real last_completed_migration of Migration2: ", migration2.last_completed_migration());



    }
}
```
* Launch the foundry test with command: `forge test --match-test test_UpgradeNotWorking -vv`:

Foundry Result:
```javascript
Running 1 test for test/Migration.t.sol:MigrationsTest
[PASS] test_UpgradeNotWorking() (gas: 178266)
Logs:
  Expected last_completed_migration of Migration2: 1
  Real last_completed_migration of Migration2:  0

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 10.93ms
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```



