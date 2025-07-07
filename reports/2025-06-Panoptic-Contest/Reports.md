### [L-01] Redundant storage update in executeDeposit function

_Severity:_ Low

_Target:_
https://github.com/code-423n4/2025-06-panoptic/blob/main/src%2FHypoVault.sol#L310-L343



### Summary:

`executeDeposit()` converts an active pending deposit into shares. the function saves user's deposit amount in `queuedDepositAmount` memory, and also the function updates user's balance in the storage

```solidity
    function executeDeposit(address user, uint256 epoch) external {
        if (epoch >= depositEpoch) revert EpochNotFulfilled();

        uint256 queuedDepositAmount = queuedDeposit[user][epoch];
        queuedDeposit[user][epoch] = 0; // update storage balance
```

Since user's balance is updated to 0 in the storage and it has already written in the memery, that means the function will keep reading the balance from memory instead of the storage.

The function also re-updates the storage after minting shares in the same function, which is unnecessary and gas wasted.

```solidity
        _mintVirtual(user, sharesReceived);

        userBasis[user] += userAssetsDeposited;

        queuedDeposit[user][epoch] = 0; // Unnecessarily re-update the storage 
```



### Impact:

Updating storage balance twice in the same function increases gas usage unnecessarily and removing it can help optimize the contract for better performance and lower user costs.



### Recommendation:

`queuedDeposit[user][epoch]` is unnecessarily written to 0 twice. Consider removing the redundant write to optimize gas usage.

### Tools Used:
Manual code review.
