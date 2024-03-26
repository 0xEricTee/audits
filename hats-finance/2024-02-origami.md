# Origami

The github issue page can be found in [`here`](
https://github.com/hats-finance/Origami-0x998f1b716a5022be026ca6b919c0ddf45ca31abd/issues/7).

## Findings Summary

| ID | Description | Severity |
| :-: | - | :-: |
| [M-01](#m-01-origamiotoken-circulatingsupply-will-underflow-when-users-burn-their-tokens) | OrigamiOToken::circulatingSupply will underflow when users burn their tokens. | Medium |


## [M-01] OrigamiOToken::circulatingSupply will underflow when users burn their tokens.

### Bug Description

OrigamiOToken::circulatingSupply will underflow when users burn their tokens.



### Attack Scenario

OrigamiOToken::circulatingSupply will underflow when users burn their tokens.


### Proof Of Concept

Add the following test to `OrigamiOToken.t.sol`:

```javascript
function test_circulatingSupplyUnderflow() public {
        address exploiter = makeAddr("EXPLOITER");
        vm.prank(origamiMultisig);
        oToken.amoMint(exploiter, 100);
        console.log(oToken.circulatingSupply());
        vm.prank(exploiter);
        oToken.burn(100);
        console.log(oToken.circulatingSupply());
    }
```

Foundry Result:
```javascript
Running 1 test for test/foundry/unit/investments/OrigamiOToken.t.sol:OrigamiOTokenTestAccess
[PASS] test_circulatingSupplyUnderflow() (gas: 70396)
Logs:
  0
  115792089237316195423570985008687907853269984665640564039457584007913129639836

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.06ms
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```


### Recommended Mitigation

overrides `ERC20Burnable` functions if not used.