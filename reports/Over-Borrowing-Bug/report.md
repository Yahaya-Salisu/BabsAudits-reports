A user can borrow amount beyond collateral limit in Lend-Protocol

Severity: High
Target:
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2%2Fsrc%2FLayerZero%2FCoreRouter.sol#L145
 

Summary:
The borrow function is used to borrow debt from the protocol, and the function does not check borrowed and collateral properly, the require checks preBorrowAmount ( currentBorrowBalance ) instead of postBorrowAmount ( currentBorrowBalance + _amount ) tis will allow borrowers to borrow more than their collateral


```solidity
// CoreRouter.sol
    function borrow(uint256 _amount, address _token) external {
    
     // ... existing code

        (uint256 borrowed, uint256 collateral) =
            lendStorage.getHypotheticalAccountLiquidityCollateral(msg.sender, LToken(payable(_lToken)), 0, _amount);

        uint256 borrowAmount = currentBorrow.borrowIndex != 0
            ? ((borrowed * LTokenInterface(_lToken).borrowIndex()) / currentBorrow.borrowIndex)
            : 0;

@audit ---> This require will allow over borrowing because it checks only preBorrowAmount
         require(collateral >= borrowAmount, "Insufficient collateral");

       // ...

        // Emit BorrowSuccess event
        emit BorrowSuccess(msg.sender, _lToken, lendStorage.getBorrowBalance(msg.sender, _lToken).amount);
    }```