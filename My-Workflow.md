### My Auditing Workflow for Complex DeFi Protocols (DEX, Lending, Staking, NFT)

This document outlines my personal step-by-step approach to auditing complex DeFi protocols effectively.



###  Protocol Architecture Review
1. **Read documentation**  
   I start by fully understanding the protocol’s design, use cases, and assumptions.

2. **Check past audits (if forked)**  
   Idea: Developers often repeat mistakes from the original protocol.

3. **If it's a DEX, identify the AMM type**  
   e.g., x*y=k, Constant Product, Stableswap, Curve-style, etc.

4. **Check proxy pattern**  
   e.g., UUPSUpgradeable, Transparent Proxy, Beacon Proxy, etc.

5. **Understand unique designs**  
   Idea: I always review any novel features before touching the code. Unique logic often introduces unique bugs.

6. **Study state variables and mappings**  
   Idea: Protocols use different naming conventions. Understanding variable roles is essential to map logic properly.

7. **Decode complex function names**  
   Idea: some protocol uses complex function names, decoding them helps me understand the function purposes, e.g `epoch` → `vault` → `pool`.

8. **Draw function & asset flow**  
   Idea: Visualizing expected vs actual asset movement helps catch logic flaws early.  
   *(Most real bugs hide where behavior deviates from expectation.)*

9. **Run Slither for static analysis** (when needed)  
   Then I start manual review, comparing **Expected vs Actual Code Behavior**.



###  Logic-Level Review
1. **LP Functions**  
`supply() → claim() → withdraw()`.

2. **User Functions**  
`swap(), stake() → unstake(), borrow() → repay() → liquidate()`.

3. **Math Functions**  
`interest math, liquidation math, TWAP and Oracle, sharePrice(), getSqrtRatioAtTick()`, etc.

4. **Admin Functions**  
`initialize() → setFee() → setCollateralFactor() → setAdmin()` etc.

5. **System Functions**  
`deposit() → mint() → transfer() → transferFrom() → burn()`.



### Proof of Concepts
1. I use **Foundry** to write reproducible tests for vulnerabilities.  
2. I build **unit + fuzzing** tests to simulate real-world abuse.  
3. I submit the report with reproduction steps or deliver it to the project/client.



### Tools I Use
1. **Slither**: For static analysis  
2. **Manual review**: I rely on it the most  
3. **Foundry**: For PoC, unit tests, and fuzzing.



### Conclusion
This is how I break down and deeply analyze complex DeFi protocols.  
As I continue learning and auditing more, I improve this workflow continuously.