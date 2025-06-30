### My Auditing Workflow for Complex DeFi Protocols (DEX, Lending, Staking, NFT)

This document outlines my personal step-by-step approach when auditing complex DeFi protocols.

#### Coverage
 Architecture review.
 LPs & Supplier functions.
 Traders & user Functions.
 Math model evaluations.
 Admin & system functions.
 Proof of concept testing.


#### A. Protocol architecture review
```
1. Read documentation

2. If it's forked, i check past audit reports of the original protocol
 idea: Developers may repeat the same mistakes and if so, i can quickly catch the bugs.

3. Identify AMM type (DEX):
 e.g x*y=k | Constant product | Stableswap | etc

4. Check proxy type:
e.g UUPSUpgradeable, transparent proxy, beacon proxy, etc.

5. Identify unique designs
 idea: If the protocol has unique features, i understand them first before reviewing the code

6. Read state variables & mappings
 Idea: State variables and mappings names can vary from protocol to protocol, and some protocols may use complex names, so, understanding them helps me read every logic

7. Decode complex function names
 Idea: some protocol uses complex function names, decoding them helps me understand the function purposes, e.g `epoch` means `vault` means `pool`

8. Draw function & asset flow
 Idea: Drawing functions and assets flowchart helps me check expected vs actual code behavior

9. Run Slither scan
```

#### B. LPs & Supplier Functions
```solidity
// DEX
addLiquidity(), removeLiquidity(), collect()

// Lending
supply(), deposit(), redeem(), withdraw()

// NFT / Staking
stake(), claim(), unstake()
```

#### C. Traders & User Functions
```solidity
// DEX
swapExactTokenForTokens(), swapTokensForExactTokens(), swap()

// Lending
borrow(), repay(), liquidate()

// Staking
claimRewards(), delegate()
```

#### D. Math model evaluation
```solidity
// DEX
getAmountIn(), getAmountOut(), getReserves()

// Lending
interest calculation, liquidation math, utilization rate

// Staking
reward rate, time-weighted logic
```

#### E. Admin and system functions
```solidity
// DEX
sync(), initialize(), skim(), safeTransfer(), safeTransferFrom(), fee management

// Lending
oracle feeds, access control, pause/unpause

// NFT / Staking
admin withdrawals, setRewardRate()
```

#### F. Proof Of Concepts
```solidity
 Use Foundry to reproduce and test vulnerabilities.
 Write full unit + fuzz tests.
 Submit report with reproduction steps.
```

#### Conclusion
This is how I break down and deeply analyze complex DeFi protocols. I’m still improving this process as I learn, audit, and discover better ways.

#### Tools I Use
```
 Slither for static analysis.
 Manual code review (I use the most).
 Foundry for testing.
