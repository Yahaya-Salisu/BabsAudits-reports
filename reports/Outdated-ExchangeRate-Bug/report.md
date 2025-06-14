**Supply Function Uses Stale Exchange Rate, Leading to Inaccurate Minting**

_Bug Severity:_ High

_Target:_ https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2%2Fsrc%2FLayerZero%2FCoreRouter.sol#L61


**Summary:**

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


**Venerability Details:**

This leads to miscalculation in the number of lTokens minted for the user, creating an inconsistent accounting between real-time token value and actual supply.



**Impact:**

- In edge cases, this could allow:

- Over-minting: protocol suffers loss

- Under-minting: user suffers loss



**Recommendation:**

Replace exchangeRateStored() with a call to exchangeRateCurrent() or manually call accrueInterest() before reading the exchange rate.

This ensures that minting calculations use the latest, interest-accrued exchange rate.



**Proof of concept (PoC)**

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
