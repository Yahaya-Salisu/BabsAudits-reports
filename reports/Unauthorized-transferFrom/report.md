**repayBorrowInternal allows arbitrary third-party to repay on behalf of borrower without authorization**

_Bug Severity:_ High 

_Target:_


**Summary:**


**Impact:**


**Vulnerability Details:**



**Recommendation**


**Proof of concept (PoC)**

```solidity
vm.startPrank(borrower);

    // approve
    IERC20(USDC).approve(CoreRouter, type(uint256).max);
    console2.log("USDC Allowance for CoreRouter:", IERC20(USDC).allowance(borrower, CoreRouter));

       (bool success, bytes memory data) = address(CoreRouter).call(
    abi.encodeWithSignature("supply(uint256,address)", 10_000e6, USDC)
        );
     if (!success) console2.log("Revert reason:", string(data));

     console2.log("borrower supplied-1:", ICoreRouter(USDC).balanceOf(borrower));

     vm.stopPrank();

    }

    function test_borrow() public {
        vm.startPrank(borrower);

       (bool success, bytes memory data) = address(CoreRouter).call(
    abi.encodeWithSignature("borrow(uint256 borrowAmount)", 1_000e18)
        );
     if (!success) console2.log("Revert reason:", string(data));

     require(success, string(data));
     vm.stopPrank();
    }


    function test_rapayBorrow() public {

        vm.startPrank(borrower);

       (bool success, bytes memory data) = address(CoreRouter).call(
    abi.encodeWithSignature("repayBorrow(uint256 repayAmount)", 1_000e18)
        );
     if (!success) console2.log("Revert reason:", string(data));

     require(success, string(data));
     vm.stopPrank();

     assertEq(IERC20(USDC).balanceOf(borrower), 10_000e6);
    }
```

**PoC Output:**
![PoC](https://github.com/user-attachments/assets/171deb23-f349-4cd8-af7f-999e0e959fb2)
