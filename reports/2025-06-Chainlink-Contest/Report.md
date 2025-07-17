### [M-01] Updating balance before external call allows false deposits

_Severity_ Medium 

_Source:_ https://github.com/code-423n4/2025-06-chainlink/blob/main/src%2FBUILDClaim.sol#L84-L102

#### Summary:
The `deposit()` in `BUILDClaim.sol` updates balance before external call, this could lead to a situation where a projectAdmin can deposit amount more than allowance they approved.

#### Description:
ProjectAdmin is always required to approve allowance when performing deposit in `BUILClaim.sol` but the deposit always updates balance before external call, meaning that if the projectAdmin approved 100 tokens, and call deposit of 10,000 tokens, the deposit function will update the balance first before attempting to call `transferFrom()`, after transferFrom() is called the entire deposit will revert due to  'Insufficient allowance' but the balance was updated already.

```solidity
// BULDClaim.sol
function deposit(
    uint256 amount
  ) external override nonReentrant whenClaimNotPaused onlyProjectAdmin {

    ... existing code ...

@audit-bug--> uint256 totalDeposited = i_factory.addTotalDeposited(address(i_token), amount); // ⚠️ BUG: Balance is updated before external call.
    i_token.safeTransferFrom(msg.sender, address(this), amount); // external call after balance is updated.
    uint256 balanceAfter = i_token.balanceOf(address(this));
    if (balanceBefore + amount != balanceAfter) {
      revert InvalidDeposit(balanceBefore, balanceAfter);
    }
```

`Deposit()` calls `i_factory.addTotalDeposited()` first before attempting to call `transferFrom()`, and this function `i_factory.addTotalDeposited()` is updating balance and then return `newTotalDeposited` and later the deposit will attempt to call `transferFrom()` which will fail whenever amount > Allowance.

```solidity
// BULDFactory.sol
  function addTotalDeposited(address token, uint256 amount) external override returns (uint256) {
    _requireRegisteredClaim(token);
    if (amount == 0) {
      revert InvalidAmount();
    }
    TokenAmounts storage tokenAmounts = s_tokenAmounts[token];
    uint256 newTotalDeposited = tokenAmounts.totalDeposited + amount; // amount is added
    tokenAmounts.totalDeposited = newTotalDeposited; // balance is updated
    emit ProjectTotalDepositedIncreased(token, msg.sender, amount, newTotalDeposited);
    return newTotalDeposited;
  }
```

#### Impact:
Attacker can exploit the reward system using less approve amount and high deposit amount, meaning that an attacker will get double benefits (`addTotalDeposited()` is updated and Allowance reverted)

#### Proof of concept (POC)
A. ProjectAdmin approved = 1.5e20, and calls deposit of = 3e20.

B. Deposit calls `addTotalDeposited()` before external call, also balance is updated and event was emited.

C. Balance before = 4.5e20.

D. Balance after = 7.5e20 (4.5e20 + 3e20).

F. The `transferFrom()` revert but the balance is updated already.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.26;

import {BaseTest} from "./BaseTest.t.sol";
import {BUILDFactory} from "../src/BUILDFactory.sol";
import {BUILDClaim} from "../src/BUILDClaim.sol";
import "forge-std/console2.sol";

contract submissionValidity is BaseTest {
    function test_submissionValidity() external whenSeason1IsSetup {
    _changePrank(PROJECT_ADMIN);

    // Log balances before
    uint256 balanceBefore = s_token.balanceOf(address(s_claim));
    console2.log("Balance BEFORE:", balanceBefore);

    // Approve 1.5e20
s_token.approve(address(s_claim), TOKEN_AMOUNT_P1_S1);

  
s_claim.deposit(TOKEN_AMOUNT_P1_S1 * 2); // Call deposit of 3e20 (2× of Allowance)

    // Log balances after
    uint256 balanceAfter = s_token.balanceOf(address(s_claim));
    console2.log("Balance AFTER:", balanceAfter);

    assertEq(balanceAfter, balanceBefore);
    }
}
```

Test suits
```solidity
forge test --match-test submissionValidity -vvvv
```


Output
```solidity
[540] ProjectToken::balanceOf(BUILDClaim: [0xfeE3f359c00b1Ef7e9123D21ff056B7BF95e508B]) [staticcall]
    │   └─ ← [Return] 450000000000000000000 [4.5e20]
    ├─ [0] console::log("Balance BEFORE:", 450000000000000000000 [4.5e20]) [staticcall]
    │   └─ ← [Stop]
    ├─ [22639] ProjectToken::approve(BUILDClaim: [0xfeE3f359c00b1Ef7e9123D21ff056B7BF95e508B], 150000000000000000000 [1.5e20])
    │   ├─ emit Approval(owner: PROJECT_ADMIN: [0x000000000000000000000000000000000000000b], spender: BUILDClaim: [0xfeE3f359c00b1Ef7e9123D21ff056B7BF95e508B], value: 150000000000000000000 [1.5e20])
    │   └─ ← [Return] true
    ├─ [10257] BUILDClaim::deposit(300000000000000000000 [3e20])
    │   ├─ [787] BUILDFactory::isClaimContractPaused(ProjectToken: [0x2c7cF54991df665C90E8aDeb50b50f98Be4B74B9]) [staticcall]
    │   │   └─ ← [Return] false
    │   ├─ [977] BUILDFactory::getProjectConfig(ProjectToken: [0x2c7cF54991df665C90E8aDeb50b50f98Be4B74B9]) [staticcall]
    │   │   └─ ← [Return] ProjectConfig({ admin: 0x000000000000000000000000000000000000000b, claim: 0xfeE3f359c00b1Ef7e9123D21ff056B7BF95e508B })
    │   ├─ [372] BUILDFactory::isOpen() [staticcall]
    │   │   └─ ← [Return] true
    │   ├─ [540] ProjectToken::balanceOf(BUILDClaim: [0xfeE3f359c00b1Ef7e9123D21ff056B7BF95e508B]) [staticcall]
    │   │   └─ ← [Return] 450000000000000000000 [4.5e20]
    │   ├─ [3233] BUILDFactory::addTotalDeposited(ProjectToken: [0x2c7cF54991df665C90E8aDeb50b50f98Be4B74B9], 300000000000000000000 [3e20])
    │   │   ├─ emit ProjectTotalDepositedIncreased(token: ProjectToken: [0x2c7cF54991df665C90E8aDeb50b50f98Be4B74B9], sender: BUILDClaim: [0xfeE3f359c00b1Ef7e9123D21ff056B7BF95e508B], amount: 300000000000000000000 [3e20], totalDeposited: 750000000000000000000 [7.5e20])
    │   │   └─ ← [Return] 750000000000000000000 [7.5e20]
    │   ├─ [963] ProjectToken::transferFrom(PROJECT_ADMIN: [0x000000000000000000000000000000000000000b], BUILDClaim: [0xfeE3f359c00b1Ef7e9123D21ff056B7BF95e508B], 300000000000000000000 [3e20])
    │   │   └─ ← [Revert] ERC20InsufficientAllowance(0xfeE3f359c00b1Ef7e9123D21ff056B7BF95e508B, 150000000000000000000 [1.5e20], 300000000000000000000 [3e20])
    │   └─ ← [Revert] ERC20InsufficientAllowance(0xfeE3f359c00b1Ef7e9123D21ff056B7BF95e508B, 150000000000000000000 [1.5e20], 300000000000000000000 [3e20])
    └─ ← [Revert] ERC20InsufficientAllowance(0xfeE3f359c00b1Ef7e9123D21ff056B7BF95e508B, 150000000000000000000 [1.5e20], 300000000000000000000 [3e20])


Ran 2 test suites in 2.68s (118.78ms CPU time): 1 tests passed, 1 failed, 0 skipped (2 total tests)
```

#### Recommendation:
```solidity
  function deposit(
    uint256 amount
  ) external override nonReentrant whenClaimNotPaused onlyProjectAdmin {
    // only callable when factory contract is open
    if (!i_factory.isOpen()) {
      revert Closable.AlreadyClosed();
    }
    uint256 balanceBefore = i_token.balanceOf(address(this));
    i_token.safeTransferFrom(msg.sender, address(this), amount); // external call first.
    uint256 balanceAfter = i_token.balanceOf(address(this));
    if (balanceBefore + amount != balanceAfter) {
      revert InvalidDeposit(balanceBefore, balanceAfter);
    }
uint256 totalDeposited = i_factory.addTotalDeposited(address(i_token), amount); // then update balance.

    emit Deposited(address(i_token), msg.sender, amount, totalDeposited);
  }
```