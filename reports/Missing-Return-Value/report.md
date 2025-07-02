**[M-01] Missing Return Value for Non-Compliant ERC20 Tokens in CToken.sol**

_Severity:_ Medium

_Target:_
https://github.com/compound-finance/compound-protocol/blob/master/contracts%2FCToken.sol#L111


**Summary:**

The contract uses transfer() and transferFrom() on non-compliant EERC20 tokens such as cUSDC or cDAI, which do not correctly return a boolean value. This can lead to situations where failed transfers are treated as successful. For example if a user tried to transfer cUSDC or cDAI and the transferAmount > userBalance the transaction will not revert, it will look like the transfer is successful while it didn't. 



**Impact:**

- If token is a non-compliant ERC20 like cUSDC or cDAI, the transfer may always pass even if the transfer fails, because the return value is undefined.
- Funds may fail to transfer without reverting
- Logic that assumes a successful transfer may proceed incorrectly.

  

**Proof of concept (PoC)**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../src/interfaces/IcERC20.sol";

contract MissingReturnValue is Test {
 address constant cUSDC = 0x39AA39c021dfbaE8faC545936693aC917d5E7563;
 address constant User = 0x90581aFC83520a649376852166B3df92153cEE20;

IcERC20 cusdc = IcERC20(cUSDC);

function setUp() public {
    vm.createSelectFork(vm.envString("RPC_URL"), 19600000);
    vm.deal(address(this), 1000e6);
}

function test_MissingReturnValue() public {

    uint256 balance = cusdc.balanceOf(address(this));
    uint256 OverAmount = balance + 1;

    cusdc.transfer(User, OverAmount);
    
    uint256 userBalance = cusdc.balanceOf(User);
emit log_named_uint("User Balance After Transfer", userBalance);
assertEq(userBalance, 0, "Transfer failed silently but acting as it succe, missing return value!");
}
}
```

***PoC Output:***

![PoC](https://github.com/user-attachments/assets/5077f239-7e2c-4a2e-87a0-de1a07035cb8)


**Recommendation:**

Use safeTreansfer and safeTransferFrom to support non ERC20 standerd tokens
