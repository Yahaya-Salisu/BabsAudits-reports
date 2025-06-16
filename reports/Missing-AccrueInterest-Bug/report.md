**Redeem function does not call accrueInterest, leading to loss of user interests**

_Bug Severity:_ Medium 

_Target:_


**Summary:**

Rdeem function fails to call accrueInterest entirely, this could result in loss of users interests because whenever the users tried to redeem their LToken to underlying asset, the redeem function will not accumulate the interests, meaning that the users will receive exact amount they have supplied without interest.

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



**Venerability Details:**

- If a user supplied 1_000e6 USDC for example, after 30 days the user wants to redeem/withdraw his token, the protocol will proceed only exact amount the user supplied without accumulating interest.

- Furthermore, the function uses exchangeRateStored to calculate expected underlying of users, meaning that the user's underlying can be over/under amount.

**Impact:**
- Users will not receive their interests because the function does not call accrueInterest at all.
- Inaccurate redemption, the function calculates exchange rate based on outdated price.
- User's amount could be more or less than expected because the function uses exchangeRateStored() to calculate expected underlying.



**Recommendation:**

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



**Proof of concept (PoC)**

The below PoC shows how a user supplied 1_000e6 USDC, after 30 days redeems the token and received exact amount he supplied ( 1_000e6 ) without interest.

```solidity

```