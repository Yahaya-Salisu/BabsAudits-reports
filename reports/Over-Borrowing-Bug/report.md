**A user can borrow amount beyond collateral limit in Lend-Protocol**

_Bug Severity: High_

_Target:_
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2%2Fsrc%2FLayerZero%2FCoreRouter.sol#L145
 

**Summary:**

The borrow function is used to borrow debt from the protocol, and the function does not check borrowed and collateral properly, the require checks preBorrowAmount ( currentBorrowBalance ) instead of postBorrowAmount ( currentBorrowBalance + _amount ) this will allow borrowers to borrow debt more than their collateral


```solidity
// CoreRouter.sol
    function borrow(uint256 _amount, address _token) external {
    
     // ... existing code...

        (uint256 borrowed, uint256 collateral) =
            lendStorage.getHypotheticalAccountLiquidityCollateral(msg.sender, LToken(payable(_lToken)), 0, _amount);

        uint256 borrowAmount = currentBorrow.borrowIndex != 0
            ? ((borrowed * LTokenInterface(_lToken).borrowIndex()) / currentBorrow.borrowIndex)
            : 0;


@audit-bug--> require(collateral >= borrowAmount, "Insufficient collateral"); // ⚠️ This require will allow over borrowing because it checks preBorrowAmount instead of postBorrowAmount.

       // ... existing code...

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


// ... existing code...


    function test_borrow1() public {
     vm.startPrank(User);
        // borrow
        (bool success, bytes memory data) = address(CoreRouter).call(
    abi.encodeWithSignature("borrow(address,uint256)", LToken, 5_000e18)
        );
    if (!success) console2.log("Revert reason:", string(data));
    vm.stopPrank();
    }

    function test_borrow2() public {
     vm.startPrank(User);
        // borrow
        (bool success, bytes memory data) = address(CoreRouter).call(
    abi.encodeWithSignature("borrow(address,uint256)", LToken, 4_000e18)
        );
    if (!success) console2.log("Revert reason:", string(data));
    vm.stopPrank();
    }
}
```

![PoC Output](https://github.com/user-attachments/assets/38c81c75-7017-48b6-8d54-b24e8a9df4bc)

