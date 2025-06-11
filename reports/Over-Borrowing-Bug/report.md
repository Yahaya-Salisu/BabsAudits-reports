A user can borrow amount beyond collateral limit in Lend-Protocol

Severity: High
Target:
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2%2Fsrc%2FLayerZero%2FCoreRouter.sol#L145
 

         Summary:

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

---

 ### ***Vulnerability Details:***

A. A User deposited $1000 for example, and the collateral factor (LTV) is 0.8e18 (80%), that means the User has a chance to borrow $800

B. The User borrowed $500, the remaining borrowable balance = $300

C. The User tried to borrow $400 even though his available amount is $300

D. The function checks if the borrowAmount which $500 is equal or less than collateral.

E. The borrow function has indeed identified that the collateral > $500

F. The borrower has received $400, while his currentBorrowBalance is $500. that means the borrower has borrowed $900 instead of $800,that extra $100 will fall the user's debt underCollateralized.

---

           Impact:

- Loss of protocol and Liquidators funds because the user's collateral can not pay the debt

- The protocol has to take responsibilities of these losses

- While the Liquidators will not receive their incentives.

---

       Recommendation:


```solidity
// Change borrowAmount in require

// from this
require(collateral >= borrowAmount, "Insufficient collateral");

// to this
require(collateral >= borrowed, "Insufficient collateral");

// borrowAmount = currentBorrowBalance e.g $500
// borrowed = currentBorrowBalance + _amount e.g $500 + $400 = $900.
```

