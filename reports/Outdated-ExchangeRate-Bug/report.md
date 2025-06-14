**Supply Function Uses Stale Exchange Rate, Leading to Inaccurate Minting**

_Bug Severity:_ High

_Target:_


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
        @>>> require(LErc20Interface(_lToken).mint(_amount) == 0, "Mint failed"); // ✅ This mint accrues interest

        // Calculate actual minted tokens using exchangeRate from before mint
@audit-bug--> uint256 mintTokens = (_amount * 1e18) / exchangeRateBefore; // ⚠️ BUG: Amount of mintTokens can be less than expected due to out dated price from exchangeRateStored()

```

**Venerability Details:**

This leads to miscalculation in the number of lTokens minted for the user, creating an inconsistent accounting between real-time token value and actual supply.

- In edge cases, this could allow:

- Over-minting: protocol suffers loss

- Under-minting: user suffers loss


**Recommendation:**

Replace exchangeRateStored() with a call to exchangeRateCurrent() or manually call accrueInterest() before reading the exchange rate.

This ensures that minting calculations use the latest, interest-accrued exchange rate.


**Proof of concept (PoC)**
