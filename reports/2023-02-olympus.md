# Overview

Original contest issue and links: 

|#|Issue|Original link|
|-|:-|:-:|
| [H&#x2011;01] | claiming internal rewards via `internalRewardsForToken` fails in some cases even when users are entitled to internal rewards | [#68](https://github.com/sherlock-audit/2023-02-olympus-judging/issues/68) |
| [H&#x2011;02] | `_claimExternalRewards` does not update contract state after claiming, allowing for extra external reward token claims via `claimRewards` | [#14](https://github.com/sherlock-audit/2023-02-olympus-judging/issues/14) |
| [H&#x2011;03] | `userRewardDebts` is not correctly deducted from `cachedUserRewards` in `_withdrawUpdateRewardState` leading to extra external rewards by withdrawing | [#23](https://github.com/sherlock-audit/2023-02-olympus-judging/issues/23) + [#24](https://github.com/sherlock-audit/2023-02-olympus-judging/issues/24) |
| [M&#x2011;01] | claiming external rewards via `externalRewardsForToken` fails in some cases even when users are entitled to external rewards | [#56](https://github.com/sherlock-audit/2023-02-olympus-judging/issues/56) |

Typos may have been fixed and, the discussion part, was added at the end where applicable.

# High Risk Findings (3)

# [H&#x2011;01] claiming internal rewards via `internalRewardsForToken` fails in some cases even when users are entitled to internal rewards

## Summary

Claiming internal rewards via `internalRewardsForToken` (in `SingleSidedLiquidityVault.sol`) fails in some cases even when users are entitled to internal rewards. The code path starts from `claimRewards` and is possible due to a misplaced calculation.

## Vulnerability Detail

`claimRewards` calls `_claimInternalRewards` to get valid internal rewards for user. This function in turn, calls `internalRewardsForToken` to get the available amount to share.
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L625

Rewards are calculate by adding 2 sources:
1) accumulated rewards generated from having shares of the pools
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L355-L369
    - from the resulting amount itself it deducts the `userRewardDebts[user_][rewardToken.token]` value

2) rewards that have been accumulated but not claimed (`cachedUserRewards`)

The issue lies with `totalAccumulatedRewards` as in the way it's calculated causes, in some cases, a negative result, and since it is `uint256` results in an revert.
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L368-L369

`totalAccumulatedRewards` is calculate as the gains from the users' LP positions (which increase over time) minus the considered user reward debt (which is increasable in other areas of the code). This way of calculating it allows for negative values to possibly appear.

## Impact

Users are left with internal rewards they are entitled to but cannot access without needing to further deposit liquidity into the pool or wait when they should have not.

## Code Snippet

POC that shows a normal user use case and revert follows (for `WstethLiquidityVaultMock.t`).
```Solidity
    function testCorrectness_internalRewardsRevertsOnSubtraction(address user_) public {
        
        vm.assume(user_ != address(0) && user_ != alice && user_ != address(liquidityVault));

        // Setup
        _withdrawTimeSetUp();

        // Add second depositor
        vm.startPrank(user_);
        wsteth.mint(user_, 1e18);
        wsteth.approve(address(liquidityVault), 1e18);
        liquidityVault.deposit(1e18, 1e18);
        vm.stopPrank();
        vm.warp(block.timestamp + 10); // Increase time by 10 seconds

        // Trigger external rewards accumulation
        vm.prank(alice);
        liquidityVault.withdraw(1e18, minTokenAmounts_, false);

        // Alice's rewards should be 15 REWARD tokens _per claim_
        // 10 for the first 10 blocks and 5 for the second 10 blocks
        // User's rewards should be 5 REWARD tokens _per claim_

        // Verify initial state
        assertEq(reward.balanceOf(alice), 0);

        // normal user activity
        vm.prank(alice);
        liquidityVault.claimRewards();
        
        // simply checking the internalRewardsForToken function reverts 
        vm.expectRevert();
        liquidityVault.internalRewardsForToken(0, alice);    
        
        // user waits and want so claim again
        vm.expectRevert();
        vm.prank(alice);
        liquidityVault.claimRewards();
    }
```

## Tool used

Manual Review

## Recommendation

Move the problematic operations in the final calculation of the return statement
```
         }
 
         // This correctly uses 1e18 because the LP tokens of all major DEXs have 18 decimals
-        uint256 totalAccumulatedRewards = (lpPositions[user_] * accumulatedRewardsPerShare) -
-            userRewardDebts[user_][rewardToken.token];
-
-        return (cachedUserRewards[user_][rewardToken.token] + totalAccumulatedRewards) / 1e18;
+        return (cachedUserRewards[user_][rewardToken.token] + (lpPositions[user_] * accumulatedRewardsPerShare) -
+            userRewardDebts[user_][rewardToken.token]) / 1e18;
     }

```
or declare `totalAccumulatedRewards` as `int256`

# [H&#x2011;02] `_claimExternalRewards` does not update contract state after claiming, allowing for extra external reward token claims via `claimRewards`

## Summary

Externally callable function `claimRewards()` from `SingleSidedLiquidityVault.sol`  gets external rewards via calling the `_claimExternalRewards`. `_claimExternalRewards` itself determines the reward token quantity via `externalRewardsForToken` but does not update the extracted rewards, nor does `claimRewards`. As such, continously calling `claimRewards()` will drain pool of all **external** rewards

## Vulnerability Detail

Vulnerability execution flow, from the entry point:

`claimRewards()`
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L288

function calls `_claimExternalRewards` on each external rewards token (`_updateExternalRewardState` only updates the `accumulatedRewardsPerShare` of the external reward token).
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L302-L309

function `_claimExternalRewards` retrieves the total available rewards via a call to `externalRewardsForToken`
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L638

subsequently updates state variables `userRewardDebts`, `accumulatedFees` and `rewardToken.lastBalance`.

There are no other states updated regarding the rewards, but this is an issue because of the way `externalRewardsForToken` calculates the available rewards.

Rewards are calculate by adding 2 sources:
1) accumulated rewards generated from having shares of the pools
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L374-L386
    - taking the coressponding LP generated rewards and removing owed user debts `userRewardDebts[user_][rewardToken.token]`
    - because of this subtraction, the exploit can be used until the subtraction reverts with underflow (can be considered a separate issue on it's own)

2) rewards that have been accumulated but not claimed (`cachedUserRewards`)

During this entire execution flow `cachedUserRewards`  __is never reset__

Because of the way `externalRewardsForToken` does the subtaction, calls through `claimRewards` will only survive as long as the function does not revert.

## Impact

A malicious actor can call `claimRewards()` gaining extra rewards, on each call.
Or the simplest case, each claim is paid extra, leaving the system with possibly bad debt. 

## Code Snippet

A partially implemented, actually failing, POC follows (for `WstethLiquidityVaultMock.t`).

Coincidently, the code fails because of the `externalRewardsForToken` reverting on the subtraction in 
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L382-L383
(which is an issue in itself). Without it, the POC would function.

```
    function testCorrectness_ExploitInfiniteClaimExternalRewards(address user_) public {
        
        vm.assume(user_ != address(0) && user_ != alice && user_ != address(liquidityVault));

        // Setup
        _withdrawTimeSetUp();

        // Add second depositor
        vm.startPrank(user_);
        wsteth.mint(user_, 1e18);
        wsteth.approve(address(liquidityVault), 1e18);
        liquidityVault.deposit(1e18, 1e18);
        vm.stopPrank();
        vm.warp(block.timestamp + 10); // Increase time by 10 seconds

        // Trigger external rewards accumulation
        vm.prank(alice);
        liquidityVault.withdraw(1e18, minTokenAmounts_, false);

        // Alice's rewards should be 1.5 external tokens and her current balance 0
        assertEq(liquidityVault.externalRewardsForToken(0, alice), 15e17);
        assertEq(externalReward.balanceOf(alice), 0);

        // Exploiter Alice's claimes multiple times rewards
        vm.startPrank(alice);
        liquidityVault.claimRewards();
        assertEq(liquidityVault.externalRewardsForToken(0, alice), 15e17);
        liquidityVault.claimRewards();
            
        //assertApproxEqRel(externalReward.balanceOf(alice), 15e17, 1e16); // 1e16 = 1%
        vm.stopPrank();

        // // Verify end state
        // assertApproxEqRel(externalReward.balanceOf(alice), 18e17, 1e16); // 1e16 = 1%
    }
```

## Tool used

Manual Review

## Recommendation

Modify `_claimExternalRewards` to correctly update variables. Example implementation:
```Solidity
diff --git a/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol b/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol
index 279b7e4..6cec65c 100644
--- a/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol
+++ b/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol
@@ -640,6 +640,7 @@ abstract contract SingleSidedLiquidityVault is Policy, ReentrancyGuard, RolesCon
 
         userRewardDebts[msg.sender][rewardToken.token] += reward;
         accumulatedFees[rewardToken.token] += fee;
+        cachedUserRewards[user_][rewardToken.token] = 0;
 
         if (reward > 0) ERC20(rewardToken.token).safeTransfer(msg.sender, reward - fee);
         rewardToken.lastBalance = ERC20(rewardToken.token).balanceOf(address(this));

```


# [H&#x2011;03] `userRewardDebts` is not correctly deducted from `cachedUserRewards` in `_withdrawUpdateRewardState` leading to extra external rewards by withdrawing

## Summary

When users execute a `withdraw` the overall __external__ rewards owned to them,  `cachedUserRewards`, is not correctly calculated. `userRewardDebts` is not deducted from `cachedUserRewards` in `_withdrawUpdateRewardState` leading to extra rewards in withdraw.

## Vulnerability Detail

To withdraw their liquidity, users call `withdraw` which calls `_withdrawUpdateRewardState`
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L263

in this function, when user has more external rewards then debt, reward should be set to the difference between the debt and value, but `userRewardDebts[msg.sender][rewardToken.token]` is set to 0 __before__ the subtraction (next line), this needs to be done after
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L607-L610

## Impact

User claims more rewards then he is entitled to (by calling `withdraw` or `claimRewars`). 
Severly degrades the liquidity of the pool.

## Code Snippet

The order of these 2 attributions should be changed
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L607-L610

## Tool used

Manual Review

## Recommendation

Reverse the attributions in lines 607-610 from 
```diff
diff --git a/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol b/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol
index 279b7e4..a83eedf 100644
--- a/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol
+++ b/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol
@@ -604,10 +604,10 @@ abstract contract SingleSidedLiquidityVault is Policy, ReentrancyGuard, RolesCon
             uint256 rewardDebtDiff = lpAmount_ * rewardToken.accumulatedRewardsPerShare;
 
             if (rewardDebtDiff > userRewardDebts[msg.sender][rewardToken.token]) {
-                userRewardDebts[msg.sender][rewardToken.token] = 0;
                 cachedUserRewards[msg.sender][rewardToken.token] +=
                     rewardDebtDiff -
                     userRewardDebts[msg.sender][rewardToken.token];
+                userRewardDebts[msg.sender][rewardToken.token] = 0;
             } else {
                 userRewardDebts[msg.sender][rewardToken.token] -= rewardDebtDiff;
             }

```

# Medium Risk Findings (1)

# [M&#x2011;01] claiming external rewards via `externalRewardsForToken` fails in some cases even when users are entitled to external rewards

## Summary

Claiming external rewards via `externalRewardsForToken` (in `SingleSidedLiquidityVault.sol`) fails in some cases even when users are entitled to external rewards. The code path starts from `claimRewards` and is possible due to a misplaced calculation.

## Vulnerability Detail

`claimRewards` calls `_claimExternalRewards` to get valid external rewards for user. This function in turn, calls `externalRewardsForToken` to get the available amount to share.
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L638

Here extermal rewards are calculate by adding 2 sources:
1) accumulated rewards generated from having shares of the pools
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L379-L383
    - taking the coressponding LP generated rewards and removing owed user debts `userRewardDebts[user_][rewardToken.token]`
2) rewards that have been accumulated but not claimed (`cachedUserRewards`)
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L385


The issue lies with `totalAccumulatedRewards` as in the way it's calculated causes, in some cases, a negative result, and since it is `uint256` results in an revert.
https://github.com/sherlock-audit/2023-02-olympus/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L382-L383

`totalAccumulatedRewards` is calculate as the gains from the users' LP positions (which only increase when the user deposits more collateral) minus the considered user reward debt (which is increasable in other areas of the code).

## Impact

Users are left with rewards they are entitled to but cannot access without needing to further deposit liquidity into the pool.

## Code Snippet

POC that shows a normal user use case and revert follows (for `WstethLiquidityVaultMock.t`).

```Solidity
    function testCorrectness_externalRewardsRevertsOnSubtraction(address user_) public {
        
        vm.assume(user_ != address(0) && user_ != alice && user_ != address(liquidityVault));

        // Setup
        _withdrawTimeSetUp();

        // Add second depositor
        vm.startPrank(user_);
        wsteth.mint(user_, 1e18);
        wsteth.approve(address(liquidityVault), 1e18);
        liquidityVault.deposit(1e18, 1e18);
        vm.stopPrank();
        vm.warp(block.timestamp + 10); // Increase time by 10 seconds

        // Trigger external rewards accumulation
        vm.prank(alice);
        liquidityVault.withdraw(1e18, minTokenAmounts_, false);

        // Alice's rewards should be 1.5 external tokens and her current balance 0
        assertEq(liquidityVault.externalRewardsForToken(0, alice), 15e17);
        assertEq(externalReward.balanceOf(alice), 0);

        // normal user activity
        vm.prank(alice);
        liquidityVault.claimRewards();
        
        // simply checking the externalRewardsForToken function reverts 
        vm.expectRevert();
        liquidityVault.externalRewardsForToken(0, alice);    
        
        // user waits and want so claim again
        vm.expectRevert();
        vm.prank(alice);
        liquidityVault.claimRewards();
    }
```

## Tool used

Manual Review

## Recommendation

Move the problematic operations in the final calculation of the return statement

```Solidity
         ExternalRewardToken memory rewardToken = externalRewardTokens[id_];
 
         // This correctly uses 1e18 because the LP tokens of all major DEXs have 18 decimals
-        uint256 totalAccumulatedRewards = (lpPositions[user_] *
-            rewardToken.accumulatedRewardsPerShare) - userRewardDebts[user_][rewardToken.token];
-
-        return (cachedUserRewards[user_][rewardToken.token] + totalAccumulatedRewards) / 1e18;
+        return (cachedUserRewards[user_][rewardToken.token] + (lpPositions[user_] *
+            rewardToken.accumulatedRewardsPerShare) - userRewardDebts[user_][rewardToken.token]) / 1e18;
     }
 
```



