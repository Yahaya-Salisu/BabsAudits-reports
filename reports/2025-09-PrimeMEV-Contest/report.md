## Silent bid reduction in openBid can cause unexpected behavior and settlement mismatches

_Severity_: Medium
_Likelihood_: Medium

### Description
The openBid function allows a bidder to submit a bid (bidAmt) greater than their current availableAmount. Instead of reverting, the contract silently reduces bidAmt to deposit.availableAmount and proceeds.

This silent adjustment breaks the assumption that the committed bid is always honored on-chain, and introduces inconsistencies between off-chain coordination (commit/reveal, winner selection) and on-chain state.

### Impact:
The openBid function currently allows a situation where a bidder specifies a bidAmt greater than their availableAmount. Instead of reverting, the function silently reduces bidAmt to deposit.availableAmount and continues execution:
```solidity
if (deposit.availableAmount < bidAmt) {
    bidAmt = deposit.availableAmount;
    emit BidAmountReduced(bidder, provider, bidAmt);
}
```

This behavior introduces several risks:

1. Commit/Reveal mismatch: Off-chain systems (builders/validators) may assume the committed bid value is the one used, but the on-chain contract reduces it.

This creates inconsistencies between committed bids vs actual escrowed amounts.

2. Unexpected winner selection: A bidder might win based on their committed bidAmt, but the actual escrow is reduced.

This could distort incentives or leave the system vulnerable to accounting errors during reveal or distribution.

3. Silent degradation: Instead of failing loudly (via revert), the contract continues in a partially valid state, making it harder to debug and potentially exploitable by adversaries relying on silent reductions.

### Likelihood
Medium: Triggering this requires a bidder to intentionally or accidentally submit a bidAmt greater than their deposit. While not a trivial everyday occurrence, it is realistic in adversarial environments. 

### Proof of Concept (PoC)
1. Bidder deposits 150 ETH with a provider.

2. Bidder attempts to open a bid of 200 ETH (bidAmt = 200).

3. Contract silently reduces bid to 150 ETH and escrows 150 ETH, but off-chain logic may still treat the bid as 200 ETH.

```solidity
if (deposit.availableAmount < bidAmt) {
    bidAmt = deposit.availableAmount;
    emit BidAmountReduced(bidder, provider, bidAmt);
}
```

### Recommended Mitigation:
1. Attempt auto top-up via DepositManager.

2. If after top-up the balance is still insufficient, revert with a clear error.

```solidity
error BidExceedsAvailable(uint256 available, uint256 requested);
```
```solidity
if (deposit.availableAmount < bidAmt) {
    // attempt top-up if possible...
    if (deposit.availableAmount < bidAmt) {
        revert BidExceedsAvailable(deposit.availableAmount, bidAmt);
    }
}
```