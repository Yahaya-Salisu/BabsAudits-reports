**Supply Function Uses Stale Exchange Rate, Leading to Inaccurate Minting**

_Bug Severity:_ High

_Target:_


**Summary:**
- The supply function does not call accrueInterest first! and it relies on exchangeRateStored() for price data, while exchangeRateStored() does not call accrueInterest or exchangeRateCurrent().
- Meaning that the supply function doesn't get latest price because exchangeRateStored() always give a stale price which is OutDated.

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

// @>>> This mint() accrues interest

      // Mint lTokens 
 require(LErc20Interface(_lToken).mint(_amount) == 0, "Mint failed");

```

- Lastly the supply function uses exchangeRateStored() to calculate actual 

```solidity
function supply(uint256 _amount, address _token) external {
        address _lToken = lendStorage.underlyingTolToken(_token);

        ... existing code...

        // Get exchange rate before mint
@audit-bug--> uint256 exchangeRateBefore = LTokenInterface(_lToken).exchangeRateStored(); // ⚠️ BUG: exchangeRateStored() doesn't call accrueInterest or exchangeRateCurrent(), hence, it gives an out dated price

        // Mint lTokens
        require(LErc20Interface(_lToken).mint(_amount) == 0, "Mint failed"); // ✅ This mint accrues interest

        // Calculate actual minted tokens using exchangeRate from before mint
@audit-bug--> uint256 mintTokens = (_amount * 1e18) / exchangeRateBefore; // ⚠️ BUG: Amount of mintTokens can be less than expected due to out dated price from exchangeRateStored()

```

**Venerability Details:**


**Recommendation:**


**Proof of concept (PoC)**
