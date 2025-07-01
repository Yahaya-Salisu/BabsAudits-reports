| ID                                                                                                               | Title                                                                                                        |
| ---------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| [H-01](#h-01-A-user-can-borrow-amount-beyond-his-collateral-limit-in-borrow-function)                       | A user can borrow amount beyond collateral limit in borrow function                                |
| [H-02](#h-02-Over-repayment-possible-in-repayBorrow-due-to-missing-upper-bound-check-on-_amount)                              | Over-repayment possible in repayBorrow due to missing upper bound check on _amount                              |
| [H-03](#h-03-repayBorrowInternal-allows-arbitrary-third-party-to-repay-on-behalf-of-borrower-without-authorization)                                | repayBorrowInternal allows arbitrary third-party to repay on behalf of borrower without authorization                                          |
| [H-04](#h-04-Supply-Function-Uses-Stale-Exchange-Rate,-Leading-To-Inaccurate-Minting)                             | Supply Function Uses Stale Exchange Rate, Leading to Inaccurate Minting                              |
| [M-01](#m-01-Redeem-function-does-not-call-accrueInterest-leading-to-loss-of-user-interests)                | Redeem function does not call accrueInterest, leading to loss of user interests


___



## [H-01] A user can borrow amount beyond collateral limit in borrow function 


_Target:_
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2%2Fsrc%2FLayerZero%2FCoreRouter.sol#L145
 


### Summary:

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




### Description:

A. A User deposited $1000 for example, and the collateral factor (LTV) is 0.8e18 (80%), that means the User has a chance to borrow $800

B. The User borrowed $500, the remaining borrowable balance = $300

C. The User tried to borrow $400 even though his available amount is $300

D. The function checks if the borrowAmount which $500 is equal or less than collateral.

E. The borrow function has indeed identified that the collateral > $500

F. The borrower has received $400, while his currentBorrowBalance is $500. that means the borrower has borrowed $900 instead of $800,that extra $100 will fall the user's debt underCollateralized.




### Impact:

- Loss of protocol and Liquidators funds because the user's collateral can not pay the debt

- The protocol has to take responsibilities of these losses

- While the Liquidators will not receive their incentives.



### Proof of concept (PoC):

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


### Output:

![PoC Output](https://github.com/user-attachments/assets/38c81c75-7017-48b6-8d54-b24e8a9df4bc)



### Recommendation:

Change borrowAmount in require

```solidity

// from this
require(collateral >= borrowAmount, "Insufficient collateral");

// to this
require(collateral >= borrowed, "Insufficient collateral");

// borrowAmount = currentBorrowBalance e.g $500
// borrowed = currentBorrowBalance + _amount e.g $500 + $400 = $900.
```


___


## [H-02] Over-repayment possible in repayBorrow due to missing upper bound check on _amount


_Target:_



### Summary:

The repayBorrow function attempts to handle both full and partial repayments. If a user supplies _amount == type(uint256).max, the contract assumes full repayment and uses borrowedAmount as the repay amount. However, when _amount is a specific value, there is no upper bound check to ensure it is not greater than borrowedAmount, allowing over-repayment.

```solidity
function repayBorrowInternal(address borrower, address liquidator, uint256 _amount, address _lToken, bool _isSameChain) internal {  
       
      ... existing code ...   
     
@audit-bug--> uint256 repayAmountFinal = _amount == type(uint256).max ? borrowedAmount : _amount; // ⚠️ BUG: Potential over-repayment if `_amount` is set to a value greater than `borrowedAmount` the function will still proceed.
```

When _amount is explicitly specified (i.e., not type(uint256).max), the contract will proceed to transfer _amount without checking if it exceeds the actual debt (borrowedAmount). This can lead to scenarios where users accidentally or unknowingly over-repay beyond what they owe.



### Description:

- A user borrows 1000e18.
- Later, the user wants to repay by calling `repayBorrowInternal()`.
- If the allowance is unlimited (`type(uint256).max`), the contract correctly limits repayment to `borrowedAmount`.
- But if the user provides a specific `_amount`, **even if greater than the debt**, the contract proceeds to transfer `_amount` instead of capping at `borrowedAmount`.



### Impact:

- Users may over-repay and lose funds permanently.
- The protocol will unjustly collect tokens it is not entitled to.
- Could cause accounting inconsistencies in protocol’s internal state.
- May be exploited in MEV/front-running scenarios where users are tricked into over-repaying.




### Proof of concept (PoC)

The below PoC shows how the borrower borrowed 1_000e18 and later repaid 10_000e18, meaning that the user directly lost 9_000e18

```solidity
function testOverRepayment() public {
        vm.startPrank(User);
        
        // 1. Approve and supply collateral
        IERC20(USDC).approve(CoreRouter, type(uint256).max);
        (bool success, bytes memory data) = address(CoreRouter).call(
            abi.encodeWithSignature("supply(uint256,address)", 1_500e6, USDC)
        );
        require(success, string(data));
        
        // 2. Borrow some amount
        (success, data) = address(CoreRouter).call(
            abi.encodeWithSignature("borrow(address,uint256)", LToken, 1_000e18)
        );
        require(success, string(data));
        
        // 3. Wait to accrue interest
        vm.warp(block.timestamp + 5 days);
        
        // 4. Attempt over-repayment (repay more than borrowed)
        IERC20(USDC).approve(CoreRouter, 10_000e6);
        (success, data) = address(CoreRouter).call(
            abi.encodeWithSignature("repayBorrow(address,uint256)", LToken, 10_000e18)
        );
        
        // Should either succeed or revert with specific reason
        if (!success) {
            console2.log("Repayment reverted with reason:", string(data));
            // Check if protocol allows over-repayment
            assertEq(
                keccak256(data),
                keccak256("Cannot repay more than borrowed"), // Expected revert message
                "Unexpected revert reason"
            );
        } else {
            console2.log("Over-repayment succeeded");
            // Verify the actual repaid amount
            uint256 balanceAfter = IERC20(USDC).balanceOf(User);
            console2.log("Remaining USDC balance:", balanceAfter);
        }
        
        vm.stopPrank();
    }
}
```

### Output:

![PoC](https://github.com/user-attachments/assets/f7221988-244e-42e7-af3c-d5f0ad7c2285)




### Recommendation:

The require should also ensure that the _amount <= borrowedAmount.

```solidity
uint256 repayAmountFinal = _amount == type(uint256).max ? borrowedAmount : _amount > borrowedAmount ? borrowedAmount : _amount;

// Or add a require:

require(_amount <= borrowedAmount, "Cannot repay more than owed");
```

This will check if the _amount > borrowAmount, and if so, the protocol will transfer borrowAmount only, or it will revert ( if you use require ).



---


## [H-03] repayBorrowInternal allows arbitrary third-party to repay on behalf of borrower without authorization


_Target:_ https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2%2Fsrc%2FLayerZero%2FCoreRouter.sol#L459


### Summary:

The repayBorrow in a LEND protocol has incorrect transferFrom parameter, the function uses an improper transferFrom that unintentionally deducts funds from the liquidator even when the borrower calls repayBorrow.




### Description:

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



### Impact:

- Liquidators will always be charged for debt they never borrowed.
- Actual borrowers will get free debt, because when they borrowed the debts and wanted to repay, the protocol will not charges them, instead, the protocol will deduct the Amount from liquiditors and also update the borrower's balance.



### Proof of concept (PoC)

The below PoC shows how a User borrowed 1_000e18, and later wanted to repay, then he calls repayBorrow, but the repayBorrow function didn't charge the amount from borrower but from a liquiditor, and after debt is cleared, the borrower's balance is still the same.

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

### Output:
![PoC](https://github.com/user-attachments/assets/171deb23-f349-4cd8-af7f-999e0e959fb2)


### Recommendation:

```solidity
// coreRouter.sol

function repayBorrowInternal(address borrower, address liquidator, uint256 _amount, address _lToken, bool _isSameChain) internal {
     
     ... existing code ....

    // Tokens transferFrom borrower to the contract
@audit-fix--> IERC20(_token).safeTransferFrom(msg.sender, address(this), repayAmountFinal);
```


---


## [H-04]Supply Function Uses Stale Exchange Rate, Leading to Inaccurate Minting


_Target:_ https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2%2Fsrc%2FLayerZero%2FCoreRouter.sol#L61


### Summary:

- The supply() function fails to call accrueInterest() before relying on exchangeRateStored() for price data.

However, exchangeRateStored() does not internally call accrueInterest() or exchangeRateCurrent(), and therefore returns a stale (outdated) exchange rate.

As a result, the calculation for minted tokens is based on outdated pricing, which can lead to under- or over-minting of lTokens.

```solidity
// CoreRouter.sol
function supply(uint256 _amount, address _token) external {
        address _lToken = lendStorage.underlyingTolToken(_token);

        ... existing code...

        // Get exchange rate before mint
@audit-bug--> uint256 exchangeRateBefore = LTokenInterface(_lToken).exchangeRateStored(); // ⚠️ BUG: exchangeRateStored() doesn't call accrueInterest or exchangeRateCurrent(), hence, it gives an out dated price
```

- Furthermore, the mint function calls accrueInterest while Minting the supply tokens.

```solidity
function supply(uint256 _amount, address _token) external {
        address _lToken = lendStorage.underlyingTolToken(_token);

        ... existing code...

  
// Mint lTokens

@audit-safe--> require(LErc20Interface(_lToken).mint(_amount) == 0, "Mint failed"); // ✅ This mint() accrues interest internally

```

- After calling mint(), the actual number of lTokens to credit is calculated using the old stale exchange rate captured before interest was accrued:

```solidity
function supply(uint256 _amount, address _token) external {
        address _lToken = lendStorage.underlyingTolToken(_token);

        ... existing code...

        // Get exchange rate before mint
@audit-bug--> uint256 exchangeRateBefore = LTokenInterface(_lToken).exchangeRateStored(); // ⚠️ BUG: exchangeRateStored() doesn't call accrueInterest or exchangeRateCurrent(), hence, it gives an out dated price

        // Mint lTokens
@audit-safe--> require(LErc20Interface(_lToken).mint(_amount) == 0, "Mint failed"); // ✅ This mint accrues interest internally

        // Calculate actual minted tokens using exchangeRate from before mint
@audit-bug--> uint256 mintTokens = (_amount * 1e18) / exchangeRateBefore; // ⚠️ BUG: Uses outdated exchangeRate, leading to inaccurate minting
```


### Description:

This leads to miscalculation in the number of lTokens minted for the user, creating an inconsistent accounting between real-time token value and actual supply.



### Impact:

- In edge cases, this could allow:

- Over-minting: protocol suffers loss

- Under-minting: user suffers loss




### Proof of concept (PoC)

The below PoC shows how supply uses stale price in different time of supplying...

- In supply-1 we get the amount of **1e11** lTokens, while after 3 days ( using warp ) the price is still the same, even though there is a difference of **1.717e9** but the price remains the same as 1e11 in supply-2. 
- This clearly shows that the supply function and exchangeRateStored() do not call accrueInterest or exchangeRateCurrent() during supplying
- Despite 3 days passing and interest being accumulated,
the `supply()` still relies on the old stored rate instead of the updated one,
because `exchangeRateStored()` was used before mint.


```solidity
   ... existing code ...
function test_supply1() public {
        vm.startPrank(User);

       (bool success, bytes memory data) = address(CoreRouter).call(
    abi.encodeWithSignature("supply(uint256,address)", 10_000e6, USDC)
        );
     if (!success) console2.log("Revert reason:", string(data));

     console2.log("User supplied-1:", ICoreRouter(USDC).balanceOf(User));

     vm.stopPrank();
    }
     

     function test_supply2() public {
        vm.startPrank(User);

// warp by 3 days to simulate interest accrual over time
        vm.warp(block.timestamp + 3 days);

       (bool success, bytes memory data) = address(CoreRouter).call(
    abi.encodeWithSignature("supply(uint256,address)", 10_000e6, USDC)
        );
     if (!success) console2.log("Revert reason:", string(data));

     console2.log("User supplied-2:", ICoreRouter(USDC).balanceOf(User));

     vm.stopPrank();

    }
}
```


**PoC OutPut:**
![PoC](https://github.com/user-attachments/assets/be3e499b-305e-4aab-9303-1a1c5aca5dff)


### Recommendation:

Replace exchangeRateStored() with a call to exchangeRateCurrent() or manually call accrueInterest() before reading the exchange rate.

This ensures that minting calculations use the latest, interest-accrued exchange rate.



---


## [M-01] Redeem function does not call accrueInterest leading to loss of user interests


_Target:_ (https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2%2Fsrc%2FLayerZero%2FCoreRouter.sol#L100)



### Summary:

Rdeem function fails to call accrueInterest entirely, this could result in loss of users interests because whenever the users tried to redeem their LToken to underlying asset, the redeem function will not accumulate the interests, meaning that the users will receive exact their principal amount without interests.

```solidity
function redeem(uint256 _amount, address payable _lToken) external returns (uint256) {
        // Redeem lTokens
        address _token = lendStorage.lTokenToUnderlying(_lToken);

@audit-bug--> // ⚠️ BUG: missing accrueInterest in redeem

       require(_amount > 0, "Zero redeem amount");

        // Check if user has enough balance before any calculations
        require(lendStorage.totalInvestment(msg.sender, _lToken) >= _amount, "Insufficient balance");

        // Check liquidity
        (uint256 borrowed, uint256 collateral) =
 lendStorage.getHypotheticalAccountLiquidityCollateral(msg.sender, LToken(_lToken), _amount, 0);
        require(collateral >= borrowed, "Insufficient liquidity");

        // Get exchange rate before redeem

@audit-bug--> uint256 exchangeRateBefore = LTokenInterface(_lToken).exchangeRateStored(); // ⚠️ BUG: potential price miscalculation because exchangeRateStored() does not accrueInterest

        // Calculate expected underlying tokens
@audit-bug--> uint256 expectedUnderlying = (_amount * exchangeRateBefore) / 1e18; // ⚠️ BUG: `expectedUnderlying` also relies on outdated exchange rate, leading to over/under amount.

        // Perform redeem
        require(LErc20Interface(_lToken).redeem(_amount) == 0, "Redeem failed");
```

- Apart from missing accrueInterest, the redeem function relies on exchangeRateStored() for price data, and the `exchangeRatestored()` does not call `accrueInterest()` or `exchangeRateCurrent()`.

```solidity
function redeem(uint256 _amount, address payable _lToken) external returns (uint256) {
        
       ... existing code ...

        // Get exchange rate before redeem

@audit-bug--> uint256 exchangeRateBefore = LTokenInterface(_lToken).exchangeRateStored(); // ⚠️ BUG: potential price miscalculation because exchangeRateStored() does not accrueInterest
```


- Again, the redeem function calculates `expected underlying` based on outdated exchange rate, leading to over/under amount.

```solidity
// Calculate expected underlying tokens
        uint256 expectedUnderlying = (_amount * exchangeRateBefore) / 1e18;

        // Perform redeem
        require(LErc20Interface(_lToken).redeem(_amount) == 0, "Redeem failed");
```



### Description:

- If a user supplied 1_000e6 USDC for example, after 30 days the user wants to redeem/withdraw his token, the protocol will proceed only exact amount the user supplied without accumulating interest.

- Furthermore, the function uses exchangeRateStored to calculate expected underlying of users, meaning that the user's underlying can be over/under amount.

**Impact:**
- Users will not receive their interests because the function does not call accrueInterest at all.
- Inaccurate redemption, the function calculates exchange rate based on outdated price.
- User's amount could be more or less than expected because the function uses exchangeRateStored() to calculate expected underlying.




### Proof of concept (PoC)

The below PoC shows how a user supplied 10_000e6 USDC, after 30 days redeems all of the token and received exact amount he supplied (principal amount) without interest.

```solidity
 function setUp() public {
        vm.createSelectFork(vm.envString("ETH_RPC_URL"), 19999999);

        deal(USDC, User, 10_000e6);

        // 1. Approve and supply collateral

        IERC20(USDC).approve(CoreRouter, type(uint256).max);

        (bool success, bytes memory data) = address(CoreRouter).call(
            abi.encodeWithSignature("supply(uint256,address)", 10_000e6, USDC)
        );
        require(success, string(data));

        // 3. Wait to accrue interest
        vm.warp(block.timestamp + 30 days);
    }

    function test_missingAccrueInterest() public {
        vm.startPrank(User);
        
        // 2. redeem the amount

        (bool success, bytes memory data) = address(CoreRouter).call(
            abi.encodeWithSignature("redeem(uint256 redeemTokens)", IERC20(USDC).balanceOf(User))
        );
        require(success, string(data));

    }
```



### Output:
![PoC](https://github.com/user-attachments/assets/8c02d364-3588-4764-8020-f72780ddcd81)



### Recommendation:
- For missing accrueInterest

```solidity
function redeem(uint256 _amount, address payable _lToken) external returns (uint256) {
        // Redeem lTokens
        address _token = lendStorage.lTokenToUnderlying(_lToken);

@audit-fix--> LTokenInterface(_lToken).accrueInterest(); // add this.

       require(_amount > 0, "Zero redeem amount");
```

- For calculating expected underlying

```solidity
// use exchangeRateCurrent() instead
uint256 exchangeRateBefore = LTokenInterface(_lToken).exchangeRateStored();

// like this
uint256 exchangeRateBefore = LTokenInterface(_lToken). exchangeRateCurrent();
```
