**Over-repayment possible in repayBorrow due to missing upper bound check on _amount**

_Bug Severity:_ High

_Target:_


**Summary:**

The repayBorrow function attempts to handle both full and partial repayments. If a user supplies _amount == type(uint256).max, the contract assumes full repayment and uses borrowedAmount as the repay amount. However, when _amount is a specific value, there is no upper bound check to ensure it is not greater than borrowedAmount, allowing over-repayment.

```solidity
function repayBorrowInternal(address borrower, address liquidator, uint256 _amount, address _lToken, bool _isSameChain) internal {  
       
      ... existing code ...   
     
@audit-bug--> uint256 repayAmountFinal = _amount == type(uint256).max ? borrowedAmount : _amount; // ⚠️ BUG: Potential over-repayment if `_amount` is set to a value greater than `borrowedAmount` the function will still proceed.
```

When _amount is explicitly specified (i.e., not type(uint256).max), the contract will proceed to transfer _amount without checking if it exceeds the actual debt (borrowedAmount). This can lead to scenarios where users accidentally or unknowingly over-repay beyond what they owe.


**Vulnerability Details:**

- A user borrows 1000e18.
- Later, the user wants to repay by calling `repayBorrowInternal()`.
- If the allowance is unlimited (`type(uint256).max`), the contract correctly limits repayment to `borrowedAmount`.
- But if the user provides a specific `_amount`, **even if greater than the debt**, the contract proceeds to transfer `_amount` instead of capping at `borrowedAmount`.


**Impact:**

- Users may over-repay and lose funds permanently.
- The protocol will unjustly collect tokens it is not entitled to.
- Could cause accounting inconsistencies in protocol’s internal state.
- May be exploited in MEV/front-running scenarios where users are tricked into over-repaying.


**Recommendation**

The require should also ensure that the _amount <= borrowedAmount.

```solidity
uint256 repayAmountFinal = _amount == type(uint256).max ? borrowedAmount : _amount > borrowedAmount ? borrowedAmount : _amount;

// Or add a require:

require(_amount <= borrowedAmount, "Cannot repay more than owed");
```

This will check if the _amount > borrowAmount, and if so, the protocol will transfer borrowAmount only, or it will revert ( if you use require ).


**Proof of concept (PoC)**

The below PoC shows how the borrower borrowed 1_000e18 and later repaid 10_000e18, meaning that the user directly lost 9_000e18

```solidity
function testOverRepayment() public {
        vm.startPrank(User);
        
        // 1. Approve and supply collateral
        IERC20(USDC).approve(CoreRouter, type(uint256).max);
        (bool success, bytes memory data) = address(CoreRouter).call(
            abi.encodeWithSignature("supply(uint256,address)", 1_500e6, USDC)
        );
        require(success, string(data));
        
        // 2. Borrow some amount
        (success, data) = address(CoreRouter).call(
            abi.encodeWithSignature("borrow(address,uint256)", LToken, 1_000e18)
        );
        require(success, string(data));
        
        // 3. Wait to accrue interest
        vm.warp(block.timestamp + 5 days);
        
        // 4. Attempt over-repayment (repay more than borrowed)
        IERC20(USDC).approve(CoreRouter, 10_000e6);
        (success, data) = address(CoreRouter).call(
            abi.encodeWithSignature("repayBorrow(address,uint256)", LToken, 10_000e18)
        );
        
        // Should either succeed or revert with specific reason
        if (!success) {
            console2.log("Repayment reverted with reason:", string(data));
            // Check if protocol allows over-repayment
            assertEq(
                keccak256(data),
                keccak256("Cannot repay more than borrowed"), // Expected revert message
                "Unexpected revert reason"
            );
        } else {
            console2.log("Over-repayment succeeded");
            // Verify the actual repaid amount
            uint256 balanceAfter = IERC20(USDC).balanceOf(User);
            console2.log("Remaining USDC balance:", balanceAfter);
        }
        
        vm.stopPrank();
    }
}
```

**PoC Output:**

![PoC](https://github.com/user-attachments/assets/f7221988-244e-42e7-af3c-d5f0ad7c2285)
