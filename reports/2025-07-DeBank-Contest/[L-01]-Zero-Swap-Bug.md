### [L-01] Missing zero-amount protection may lead to gas wastage or unexpected executor calls.

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