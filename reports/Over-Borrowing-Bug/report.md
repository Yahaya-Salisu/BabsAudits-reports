A user can borrow amount beyond collateral limit in Lend-Protocol

Severity: High
Target:
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2%2Fsrc%2FLayerZero%2FCoreRouter.sol#L145
 

Summary:
The borrow function is used to borrow debt from the protocol, and the function does not check borrowed and collateral properly, the require checks preBorrowAmount ( currentBorrowBalance ) instead of postBorrowAmount ( currentBorrowBalance + _amount ) tis will allow borrowers to borrow more than their collateral
