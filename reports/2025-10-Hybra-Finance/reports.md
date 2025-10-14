| ID                                                                                                               | Title                                                                                                        |
| ---------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| [L-01](#l-01-Incorrect-Mint-Amount-Or-Comment-Mismatch-In-HYBR.InitalMint()-Function)                       | Incorrect Mint Amount Or Comment Mismatch In HYBR.InitalMint() Function                                |
| [L-02](#l-02-Logical-error-in-GaugeV2.setInternalBribe,-allows-setting-internal_bribe-to-zero-address)                              | Logical error in GaugeV2.setInternalBribe, allows setting internal_bribe to zero address                              |
| [L-03](#l-03-Logical-error-in-GaugeCL.setInternalBribe,-the-require-condition-does-not-prevent-zero-address-as-expected)                                | Logical error in GaugeCL.setInternalBribe, the require condition does not prevent zero address as expected                |
| [L-04](#l-04-Unsafe-token-approval-pattern-in-GaugeV2.getReward()-can-lead-to-reverts-or-stuck-rewards)                             | Unsafe token approval pattern in GaugeV2.getReward() can lead to reverts or stuck rewards                              |
| [L-05](#l-05-Missing-zero-amount-validation-in-emergencyWithdrawAmount()-allows-zero-value-or-invalid-withdrawals)                | Missing zero amount validation in emergencyWithdrawAmount() allows zero value or invalid withdrawals  |
| [L-05](#l-05-Missing-event-emission-in-emergencyWithdraw()-function)                | Missing event emission in emergencyWithdraw() function


# [L-01] Incorrect Mint Amount Or Comment Mismatch In HYBR.InitalMint() Function

_Severity:_ Low

_Target:_

## Description
The comment above the `initialMint()` function states “Initial mint: total `50M`”, but the actual minted amount in the code is `500 * 1e6 * 1e18`, which equals 500 million tokens.
This represents a 10× difference between the documented and actual mint supply.
```solidity
    // Initial mint: total 50M    
    function initialMint(address _recipient) external {
        require(msg.sender == minter && !initialMinted);
        initialMinted = true;
        _mint(_recipient, 500 * 1e6 * 1e18);
    }
```

## Impact:
May lead to confusion or incorrect assumptions by external reviewers, integrators, or token holders, and could indicate a potential misconfiguration or leftover test value.

## Recommendation
Confirm the intended total supply with the protocol team.
Update either the comment or the mint amount to ensure consistency.

# [L-02] Logical error in GaugeV2.setInternalBribe, allows setting internal_bribe to zero address

_Severity:_ Low

_Target:_ https://github.com/code-423n4/2025-10-hybra-finance/blob/main/ve33%2Fcontracts%2FGaugeV2.sol#L131-L134

## Summary
`setInternalBribe` uses `require(_int >= address(0), "ZA");` which does not prevent the zero address from being set. The check is incorrect; it should be `require(_int != address(0), "ZA");`. As written, an admin can set `internal_bribe to address(0)`, causing payments intended for internal bribes to be irreversibly lost.

## Description
The function
```solidity
function setInternalBribe(address _int) external onlyOwner {
    require(_int >= address(0), "ZA");
    internal_bribe = _int;
}
```

compares addresses using >= address(0) which is always true for any valid address (including address(0)), so the require does not prevent the zero address. If internal_bribe becomes address(0), subsequent transfers intended for the bribe contract would be sent to the burn/null address.

## Impact
**Permanent loss of funds:** fees/bribes routed to internal_bribe could be sent to `address(0)` and become irrecoverable.

**Broken reward/bribe distribution:** expected flows and accounting will be disrupted; users expecting bribes/rewards may not receive them.

**Reputational & financial damage:** especially if an admin accidentally sets this or if an admin key is compromised.

## PoC
1. If the owner mistakenly or intentionally calls
```solidity
gaugeV2.setInternalBribe("0x0000000000000000000000000000000000000000")
```

2. Zero address validation bypasses and the internal bribe could became address zero, meanwhile the function is expected to revert with "ZA" but it will not revert.

3. Trigger or wait for a function that sends tokens/fees to internal_bribe (e.g., fee distribution). Tokens will be transferred to address(0) and are irrecoverable.

## Recommendation / Fix
Replace the incorrect require with a check that disallows the zero address.

```solidity
-    require(_int >= address(0), "ZA");
+    require(_int != address(0), "ZA");
     internal_bribe = _int;
```
The function will now behave as expected.
```solidity
function setInternalBribe(address _int) external onlyOwner {
    require(_int != address(0), "ZA");
    internal_bribe = _int;
}
```

# [L-03] Logical error in GaugeCL.setInternalBribe, the require condition does not prevent zero address as expected

_Target:_ https://github.com/code-423n4/2025-10-hybra-finance/blob/main/ve33%2Fcontracts%2FCLGauge%2FGaugeCL.sol#L342-L346

## Summary
The `setInternalBribe()` function allows setting `internal_bribe` to the zero address, which can lead to loss of reward/bribe functionality.

## Description
The function validates `_int >= address(0)` instead of `_int != address(0)`, allowing the internal bribe address to be set to zero.  
If the owner mistakenly calls this with `address(0)`, fees intended for the bribe contract may be burned or the gauge logic may break.

## Impact
This misconfiguration could disable internal bribe distribution, resulting in reward malfunction and fees/bribes routed to internal_bribe could be sent to `address(0)` and become irrecoverable.

**Broken reward/bribe distribution:** expected flows and accounting will be disrupted; users expecting bribes/rewards may not receive them.

**Reputational & financial damage:** especially if an admin accidentally sets this or if an admin key is compromised.

## PoC
1. If the owner mistakenly or intentionally calls
```solidity
gaugeV2.setInternalBribe("0x0000000000000000000000000000000000000000")
```

2. Zero address validation bypasses and the internal bribe could became address zero, meanwhile the function is expected to revert with "ZA" but it will not revert.

3. Trigger or wait for a function that sends tokens/fees to internal_bribe (e.g., fee distribution). Tokens will be transferred to address(0) and are irrecoverable.

## Recommendation
Use a strict zero-address check
```solidity
-    require(_int >= address(0), "ZA");
+    require(_int != address(0), "ZA");
     internal_bribe = _int;
```
The function will now behave as expected.
```solidity
function setInternalBribe(address _int) external onlyOwner {
    require(_int != address(0), "ZA");
    internal_bribe = _int;
}
```

# [L-04] Unsafe token approval pattern in GaugeV2.getReward() can lead to reverts or stuck rewards

_Severity:_ Low

_Target:_ gaugeV2.sol#L307-308 and #L323-324

## Summary
The contract defines two overloaded `getReward()` functions with duplicated logic that both call `IERC20(rewardToken).safeApprove(rHYBR, reward)` without first resetting allowance to zero.
This can cause reward claim failures, revert on certain ERC20 tokens, and unnecessary complexity due to redundant code paths.

## Description
Both functions
```solidity
function getReward(address _user, uint8 _redeemType)
function getReward(uint8 _redeemType)
```
contain identical reward distribution logic and the same approval pattern:
```solidity
IERC20(rewardToken).safeApprove(rHYBR, reward);
IRHYBR(rHYBR).depostionEmissionsToken(reward);
IRHYBR(rHYBR).redeemFor(reward, _redeemType, _user);
```

However, this implementation has two key logical flaws:

1. Unsafe approval flow
Calling `safeApprove(spender, amount)` directly after a previous non-zero allowance violates the ERC20 spec.
Some tokens (like USDT) require the allowance to first be reset to zero before setting a new value, otherwise the transaction reverts.
If `rHYBR` retains an existing allowance, reward claiming may fail unexpectedly.

2. Duplicated logic
Maintaining two identical versions of `getReward()` with only a small parameter difference (`msg.sender` vs `_user`) increases maintenance risk and inconsistency potential.
A future update to one version may leave the other outdated, leading to diverging reward logic or inconsistent accounting.

## Impact
For user-level getReward(), harvesting may fail entirely, preventing users from claiming rewards.

For distribution-level getReward(), system may fail to distribute rewards across multiple gauges.

Causes stuck rewards, failed transactions, and reward distribution inconsistencies.

## PoC
1. A user stakes in the gauge.

2. When claiming rewards, the contract calls:

IERC20(rewardToken).safeApprove(rHYBR, reward);

3. If rewardToken is USDT-like (requires reset to 0), this will revert because the allowance was non-zero.

4. The user cannot harvest until contract logic is fixed or allowance manually reset.

## Recommendation
A. Use safe approval pattern
Reset the allowance before re-approval to avoid ERC20 reverts:
```solidity
IERC20(rewardToken).safeApprove(rHYBR, 0);
IERC20(rewardToken).safeApprove(rHYBR, reward);
```

# [L-05] Missing zero amount validation in emergencyWithdrawAmount() allows zero value or invalid withdrawals

_Severity:_ Low

_Target:_ 

## Summary
Unlike other withdrawal functions that enforce require(_amount > 0), the emergencyWithdrawAmount() function does not validate the _amount parameter.
As a result, it allows calls with _amount == 0, which can lead to unnecessary gas consumption and event emissions without state changes — or, in certain misconfigured states, could cause inconsistent accounting.
```solidity
function emergencyWithdrawAmount(uint256 _amount) external nonReentrant {
    require(emergency, "EMER");
    _totalSupply = _totalSupply - _amount;
    _balances[msg.sender] = _balances[msg.sender] - _amount;
    TOKEN.safeTransfer(msg.sender, _amount);
    emit Withdraw(msg.sender, _amount);
}
```

## Description
Most functions that handle token transfers or accounting modifications include a sanity check
```solidity
require(_amount > 0, "Zero amount");
```
However, this function lacks that validation, allowing an arbitrary user to:

1. Call `emergencyWithdrawAmount(0)`, and triggering a useless token transfer of 0 tokens.

2. A misleading Withdraw event emission with `_amount = 0`.

3. Unnecessary gas usage and potential confusion in off chain monitoring tools.

4. If the _amount value is incorrectly computed elsewhere, the lack of validation may also result in unintended accounting inconsistencies when _amount is zero or improperly large.

## Impact
1. **Inconsistent accounting:** Without proper validation, the function may alter _totalSupply and _balances incorrectly if _amount is zero or invalid.

2. **Event noise:** Emission of misleading zero-value Withdraw events.

3. **Operational risk:** Possible edge cases if other parts of the system rely on nonzero _amount assumptions.

##Recommendation
Add a strict validation check before updating balances:
```solidity
    require(_amount > 0, "ZV");
```

# [L-06] Missing event emission in emergencyWithdraw() function

_Severity:_ Low

_Target:_

## Description
The `emergencyWithdraw()` function performs a sensitive operation (withdrawal of funds) but does not emit an event.

## Impact
Lack of event logging can hinder transparency and off-chain monitoring of fund withdrawals.

## Recommendation
Emit an event after a successful withdrawal
```solidity
event EmergencyWithdraw(address indexed token, address indexed to, uint256 amount);
```
```solidity
function emergencyWithdraw(address token, address to, uint256 amount) external override onlyOwner {
    require(to != address(0), "Invalid recipient");
    if (token == address(0)) {
        (bool success, ) = to.call{value: amount}("");
        require(success, "Transfer failed");
    } else {
        IERC20(token).safeTransfer(to, amount);
    }
    emit EmergencyWithdraw(token, to, amount);
}
```