### My Auditing Workflow From A-F For complex Dex Protocols

This document outlines my personal step-by-step approach when auditing complex Decentralized Exchanges (DEXs).

This is the way I read AMM contracts from both user & protocol perspectives, ensuring I catch logic bugs, economic exploits, and underflow/overflow bugs.

#### It Covers:
- Architecture review,
- LP logic,
- Trader/Swap mechanics,
- Math model evaluations,
- Admin/system functions,
- And ends with proof-of-concept testing.


#### A. Protocol architecture review
```
1. Read documentation

2. If it's forked, i check past audit reports of original protocol
- idea: Developers may repeat the same mistakes and if so, i can quickly catch the vulnerabilities.

3. Identify AMM type:
- e.g x*y=k | Constant product | Stableswap | etc

4. Identify unique designs
- idea: If the protocol has unique features, i understand them first before reviewing the code

5. Read state variables & mappings
- Idea: Each protocol uses variables and mappings with different names, understanding them first helps me read every logic

6. Decode complex function names
- Idea: some protocol uses complex function names, decoding them helps me understand the function purposes, e.g sync() means update()

7. Draw function & asset flow
- Idea: Drawing functions and assets flowchart helps me check expected vs actual code behavior

8. Run Slither scan
```

#### B. LPs Functions
```solidity
     1. addLiquidity() - mint()
     2. removeLiquidity() - burn()
     3. collect() - claimFees()
```

#### C. Trader/User Functions
```solidity
     1. swapExactTokenForTokens()
     2. swapTokensForExactTokens() 
     3. swap() low-level (with callbacks, ticks, steps)
```

#### D. Math Functions
```solidity
     1. getAmountOut()
     2. getAmountIn()
     3. getReserves()
     4. getSqrtRatioAtTick()
     5. getTickAtSqrtRatio()
```

#### E. Admin and system
```solidity
     1. sync() - updateReserves()
     2. safeTransfer(),
     3. safeTransferFrom()
     4. initialize()
     5. skim() – move excess tokens
     6. setFee(), setAdmin(), withdrawFees()
```

#### F. Proof Of Concepts ( 4 days )
```
- run foundry test of the bugs
- submit the reports and PoCs
```

#### Tools I Use
```
- Slither for static scanning
- Manual code review (I use the most)
- Foundry for fuzz and unit testing
