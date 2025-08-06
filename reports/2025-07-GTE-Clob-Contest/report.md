### [M-01] claimFees() Function Deletes Unclaimed Fees Without Transferring To Recipient, Causing Potential Funds Loss

_Target:_ https://github.com/code-423n4/2025-07-gte-clob/blob/main/contracts%2Fclob%2Ftypes%2FFeeData.sol#L134-L140


### Summary
The `claimFees()` function fetches the unclaimed fee amount from `unclaimedFees[token]` and then deletes it without performing any token transfer to the caller or recipient.

As a result, fees may be permanently lost, since only the internal state is updated and an event is emitted, but no transfer is made.

```solidity
    /// @dev Claims fees for a given token
    function claimFees(FeeData storage self, address token) internal returns (uint256 fees) {
        fees = self.unclaimedFees[token];
        delete self.unclaimedFees[token]; // deletes fee without transferring it to the recipient

        emit FeesClaimed(FeeDataEventNonce.inc(), token, fees);
    }
```


### Impact
User or protocol may permanently lose access to fees.

No actual claim occurs because only state mutation and event emission happen in the function

If used carelessly, attacker can empty unclaimed fees.

## Expected Behavior
Any `claimFees()` function is expected to:
1. Fetch the unclaimed fee amount,
2. Transfer the fee to the intended recipient (e.g., caller or designated address),
3. Then update internal state (e.g., zero out the fee balance and emit an event).

## Actual Behavior
This implementation:
1. Fetches the unclaimed fee amount,
2. Immediately deletes the fee record (`delete self.unclaimedFees[token]`),
3. Emits an event,
4. But **fails to perform any token transfer** to the recipient.

As a result, the fee is effectively "claimed" from storage but never delivered, which may lead to permanent fund loss.


### Proof of concept 
The `FeeData.sol` is a types library and lacks direct tests file in the repo, a runnable PoC is not feasible. However, the bug can still be triggered when any function integrates this logic and calls `claimFees()`. 

Below is a conceptual PoC:

1. User or admin calls claimFees() function 
```solidity
claimFees(FeeData storage self, address token, address to) internal returns (uint256 fees)
```

2. The function fetches the unclaimed fees from `unclaimedFees[token]`
```solidity
fees = self.unclaimedFees[token];
```

3. And the function deletes the unclaimed fee and emited the event
```solidity
fees = self.unclaimedFees[token];
delete self.unclaimedFees[token]; // deletes the fee without transferring it to the recipient

emit FeesClaimed(FeeDataEventNonce.inc(), token, fees);
    }
```

4. Finally, the fees balance is updated, but the fee transfer does not occur in any way, causing potential funds loss.


### Recommendation
Ensure fees transfer is made to the proper recipient before deleting the entry.

Also include a `require(fees > 0)` to prevent meaningless 0-value claims

```solidity
function claimFees(FeeData storage self, address token, address to) internal returns (uint256 fees) {
    fees = self.unclaimedFees[token];
    require(fees > 0, "No fees to claim"); // this prevents zero amount claiming

    delete self.unclaimedFees[token];

    // Transfer the tokens to the recipient
    IERC20(token).safeTransfer(account, fees);

    emit FeesClaimed(FeeDataEventNonce.inc(), token, fees);
}
```
