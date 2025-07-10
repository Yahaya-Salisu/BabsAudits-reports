 ID                                                                                                               | Title                                                                                                        |
| ---------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| [M-01](#m-01-Fee-is-charged-even-if-swap-failed,-leading-to-permanent-tokens-loss)                       |  Fee is charged even if swap failed, leading to permanent tokens loss                               |
| [M-02](#m-02-Premature-fee-deduction-in-DexSwap.sol-may-lead-to-unrecoverable-token-loss-if-swap-fails)                              | Premature fee deduction in DexSwap.sol may lead to unrecoverable token loss if swap fails 
| [L-01](#l-01-Missing-zero-amount-protection-may-lead-to-gas-wastage-or-unexpected-executor-calls)                       |  Missing zero-amount protection may lead to gas wastage or unexpected executor calls.                               |
| [L-02](#l-02-Missing-zero-swap-check-may-lead-to-gas-wastage-or-unexpected-executor-calls)                              | Missing zero-swap check may lead to gas wastage or unexpected executor calls


---



### [M-01] Fee is charged even if swap failed, leading to permanent tokens loss 

_Severity:_ Medium

#### Source: https://github.com/sherlock-audit/2025-07-debank/blob/main/swap-router-v1%2Fsrc%2Frouter%2FRouter.sol#L71-L74



### Summary: 
Calling `chargeFee()` before ensuring that if `fromTokenAmount` will be successfully transfered to the executor, can lead to permanent loss of user feeAmount if `safeTransferFrom()` failed or any error happens after fee deduction.

```solidity
    function swap(
        address fromToken,
        uint256 fromTokenAmount,
        address toToken,
        uint256 minAmountOut,
        bool feeOnFromToken,
        uint256 feeRate,
        address feeReceiver,
        Utils.MultiPath[] calldata paths
    ) external payable whenNotPaused nonReentrant {

         ... existing code ...

        // chargeFee() deducts feeAmount before ensuring if the fromTokenAmount will be successfully transferred to the executor contract.
        if (feeOnFromToken) {
            (fromTokenAmount, feeAmount) = chargeFee(fromToken, feeOnFromToken, fromTokenAmount, feeRate, feeReceiver);
        }
        // deposit to executor
        if (fromToken != UniversalERC20.ETH) {

// If this safeTransferFrom failed or any error happens, the swap will revert, but feeAmount will not be refunded to user.
 IERC20(fromToken).safeTransferFrom(msg.sender, address(executor), fromTokenAmount);
        }
```



### Description:
When performing a token swap, the user is required to approve `fromTokenAmount` to the contract. However, the `swap()` function charges the fee by calling `chargeFee()` before ensuring that the `fromTokenAmount` will be successfully transferred to the executor.

This can lead to a situation where the fee is deducted, but the main `safeTransferFrom()` reverts, leaving the user with reduced balance and no refund mechanism.

### Example:
A. User approves $1000 to the contract.

B. feeRate = 5%, feeOnFromToken = true (feeAmount = $50).

C. The swap function deducts $50 firstly as a fee using 'universalTransferFrom()` in `_chargeFee()`.

D. Remaining balance becomes $950.


E. In this case if the `safeTransferFrom()` fails or any error happens after fee deduction, the entire swap will revert but the fee was already deducted and sent to the feeReceiver.

F. User ends up losing $50 with no refund.



### Impact:
User can lose funds (fee) since there's no fee refund mechanism if swap did not proceed



### Proof of concept:
A. Approve feeAmount only.

B. Call swap.

C. Observe feeReceiver gets tokens.

D. Swap reverts before actual execution and user lost feeAmount.



### Recommendation: 
A. Only call `chargeFee()` after ensuring all following steps will succeed.

B. Collect fromTokenAmount first, then distribute feeAmount and swapAmount internally.

C. Provide fee refund mechanism that can refund the feeAmount to user whenever the swap failed.



### Tools Used:
Manual Code review.


---



### [M-02] Premature fee deduction in DexSwap.sol may lead to unrecoverable token loss if swap fails


_Severity:_ Medium

#### Source:https://github.com/sherlock-audit/2025-07-debank/blob/main/swap-router-v1%2Fsrc%2FaggregatorRouter%2FDexSwap.sol#L116-L139


### Summary: 
Similar to `M-01 (found in router/Router.sol)`, this issue occurs in `AggregatorRouter/DexSwap.sol`, using a separate execution path...

The `_swap()` function in `DexSwap.sol` deducts feeAmount using `universalTransferFrom()` in `_chargeFee()` before attempting to transfer `fromTokenAmount` to the swap executor, and If the `safeTransferFrom()` fails (e.g. insufficient allowance or if any error happens), the entire swap will revert but fee is already deducted, leading to unrecoverable user loss.

```solidity
        function swap(SwapParams memory params) external payable whenNotPaused nonReentrant {
        _swap(params);
    }

    function _swap(SwapParams memory params) internal {
        Adapter storage adapter = adapters[params.aggregatorId];

        
        _validateSwapParams(params, adapter);

        uint256 feeAmount;
        uint256 receivedAmount;

        //  charge fee on fromToken if needed
        if (params.feeOnFromToken) {
            (params.fromTokenAmount, feeAmount) = _chargeFee(
                params.fromToken, params.feeOnFromToken, params.fromTokenAmount, params.feeRate, params.feeReceiver
            );
        }

        //  transfer fromToken
        if (params.fromToken != UniversalERC20.ETH) {
            IERC20(params.fromToken).safeTransferFrom(msg.sender, address(spender), params.fromTokenAmount);
        }
```

Fee is charged, and feeReceiver balance is already updated, also there's no fee refund mechanism.
```solidity
    function _chargeFee(address token, bool feeOnFromToken, uint256 amount, uint256 feeRate, address feeReceiver)
        internal
        returns (uint256, uint256)
    {
        uint256 feeAmount = amount.decimalMul(feeRate);
        if (feeRate > 0) {
            if (feeOnFromToken) {
                IERC20(token).universalTransferFrom(msg.sender, payable(feeReceiver), feeAmount);
            } else {
                IERC20(token).universalTransfer(payable(feeReceiver), feeAmount);
            }
        }
        return (amount -= feeAmount, feeAmount);
    }
```

Though the logic looks very similar in the `router/Router.sol`, but this issue is distinct from `M-01` as it resides in `DexSwap.sol`, affecting a different execution path `(spender.swap)` through separate contract architecture.


### Description:
The `swap()` function calls `_chargeFee()` first before performing token swap, and if swap fails e.g safeTransferFrom may fail due to insufficient allowance or any error, and this can lead to a situation where the fee is deducted, but the main `safeTransferFrom()` reverts, leaving the user with reduced balance and no refund mechanism.


### Example:
A. User approves $1000 to the contract.

B. feeRate = 5%, feeOnFromToken = true (feeAmount = $50).

C. The swap function first deducts $50 as a fee using `_chargeFee()`.

D. Remaining balance becomes $950.


E. If the second transferFrom fails or any error happens after fee deduction, the entire swap will revert but the fee was already deducted and sent to the feeReceiver.

F. User ends up losing $50 with no refund.



### Impact:
User can lose funds (fee) and there's no fee refund mechanism if swap reverted.



### Proof of concept:
A. Approve specific amount.

B. Call swap of over-amount

C. Observe feeReceiver gets tokens.

D. Swap may reverts due to insufficient allowance or if any error happens, the entire swap will revert but feeReceiver already received fee.



### Recommendation: 
A. Avoid calling `_chargeFee()` until token transfer and swap parameters are validated.

B. Consider collecting total `fromTokenAmount`, then internally split into fee + swap amount to avoid double `transferFrom()` risk.

C. If fee must be collected early, introduce a try-catch or revert-proof refund path for `feeAmount` on failure.



### Tools Used:
Manual Code review.



---



### [L-01] Missing zero-amount protection may lead to gas wastage or unexpected executor calls

Yahaya Salisu.

_Severity:_ Low

_Target:_ https://github.com/sherlock-audit/2025-07-debank/blob/main/swap-router-v1%2Fsrc%2Frouter%2FRouter.sol#L56-L100

#### Summary:

The `swap()` function in `Router.sol` lacks a validation to prevent zero-amount swaps. If `fromTokenAmount == 0`, the function proceeds normally and calls the external `executeMegaSwap()` function on the executor contract.

This can result in unnecessary gas consumption and potential unexpected behavior within the executor, depending on its internal handling of zero-value swaps.

```solidity
function swap(
        address fromToken,
        uint256 fromTokenAmount,
        address toToken,
        uint256 minAmountOut,
        bool feeOnFromToken,
        uint256 feeRate,
        address feeReceiver,
        Utils.MultiPath[] calldata paths
    ) external payable whenNotPaused nonReentrant {
     ... existing code ...

// no check for fromTokenAmount > 0
executor.executeMegaSwap{value: fromToken == UniversalERC20.ETH ? fromTokenAmount : 0}(
    IERC20(fromToken),
    IERC20(toToken),
    paths
); // could be called with 0 amount
```

Even though underflows are prevented in Solidity ^0.8.0, the function allows execution with `fromTokenAmount = 0`, which might affect gas estimation or third-party integrations relying on expected behavior.


### Impact:
a. Unnecessary gas costs for the user and contract.

b. May cause unexpected behavior depending on how 'executor.executeMegaSwap()' handles 0-amount input.


### Recommendation:
Add a check to prevent zero-amount swaps before proceeding with transfer and execution.
```solidity
require(fromTokenAmount > 0, "Router: zero amount");
```
Place the check before any fee logic or external calls (especially transferFrom or executeMegaSwap) to ensure safe and intentional execution.


### Tools Used:
Manual Code review.


---



### [L-02] Missing zero-swap check may lead to gas wastage or unexpected executor calls


_Severity:_ Low

#### Source: https://github.com/sherlock-audit/2025-07-debank/blob/main/swap-router-v1%2Fsrc%2FaggregatorRouter%2FDexSwap.sol#L174-L182

#### Summary:
Swap function in `aggregatorRouter/DexSwap.sol` calls `_validateSwapParams()` and this function does not check 0 amount swap.

#### Description:
The role of _validateSwapParams is to validate the authenticity of swap parameters such as,
A. feeReceiver != address(this)
B. maxFeeRate > feeRate
C. fromToken != toToken
D. universalETH or ERC tokens.
But the function does not check 0 amount, that means if a user calls swap of zero-amount, the swap will still proceed.
```solidity
// Swap function calls _validateSwapParams()
    function swap(SwapParams memory params) external payable whenNotPaused nonReentrant {
        _swap(params);
    }

    function _swap(SwapParams memory params) internal {
        Adapter storage adapter = adapters[params.aggregatorId];

        // 1. check params
        _validateSwapParams(params, adapter);

        uint256 feeAmount;
        uint256 receivedAmount;

        // 2. charge fee on fromToken if needed
        if (params.feeOnFromToken) {
            (params.fromTokenAmount, feeAmount) = _chargeFee()


// _validateSwapParams() does not check 0 amount swap
    function _validateSwapParams(SwapParams memory params, Adapter storage adapter) internal view {
        if (params.feeReceiver == address(this)) revert RouterError.IncorrectFeeReceiver();
        if (params.feeRate > maxFeeRate) revert RouterError.FeeRateTooBig();
        if (params.fromToken == params.toToken) revert RouterError.TokenPairInvalid();
        if (!adapter.isRegistered) revert RouterError.AdapterDoesNotExist();
        if (msg.value != (params.fromToken == UniversalERC20.ETH ? params.fromTokenAmount : 0)) {
            revert RouterError.IncorrectMsgValue();
        }
    }
```
Similar to `M-01 (found in router/Router.sol)`, this issue occurs in `AggregatorRouter/DexSwap.sol`, using a separate execution path...


### Impact:
A. Unnecessary gas costs for the user and contract.

B. May cause unexpected behavior depending on how 'spender.swap()' handles 0-amount input.


### Recommendation:
```solidity
if (
    (params.fromToken == UniversalERC20.ETH && msg.value != params.fromTokenAmount) ||
    (params.fromToken != UniversalERC20.ETH && msg.value != 0)
) {
    revert RouterError.IncorrectMsgValue();
}
```
#### Tools Used:
Manual Code review.
