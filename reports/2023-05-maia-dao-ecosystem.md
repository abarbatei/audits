# Overview

Original contest issue and links: 

|#|Issue|Original link|
|-|:-|:-:|
| [H-01] | `BaseV2Minter` DAO reward shares are calculated wrong | [Link](https://github.com/code-423n4/2023-05-maia-findings/issues/104) |
| [H-02] | Removing a BribeFlywheel from a Gauge does not remove the reward asset from the rewards depo, making it impossible to add a new Flywheel with the same reward token | [Link](https://github.com/code-423n4/2023-05-maia-findings/issues/214) |
| [M-01] | If `HERMES` gauge rewards are not queued for distribution every week, they are slashed | [Link](https://github.com/code-423n4/2023-05-maia-findings/issues/411) |
| [M-02] | Unstaking `vMAIA` tokens on the first Tuesday of the month can be offset | [Link](https://github.com/code-423n4/2023-05-maia-findings/issues/469) |
| [M-03] | `BribesFactory::createBribeFlywheel` can be completely blocked from creating any Flywheel by a malicious actor | [Link](https://github.com/code-423n4/2023-05-maia-findings/issues/362) |
| [M-04] | Removing a `UniswapV3Gauge` via `UniswapV3GaugeFactory` does not actually remove it from the `UniswapV3Staker`; Gauge still gains rewards, can be staked to (even though deprecated) plus old stakers game the rewards of new stakers | [Link](https://github.com/code-423n4/2023-05-maia-findings/issues/277) |
| [Analysis] | Codebase Analysis (grade-A) | [Link](https://github.com/code-423n4/2023-05-maia-findings/blob/main/data/ABA-Analysis.md) |

Typos may have been fixed and, the discussion part, was added at the end where applicable.

# High Risk Findings (2)

# [H-01] `BaseV2Minter` DAO reward shares are calculated wrong

## Impact

In `BaseV2Minter` when calculating the DAO shares out of the weekly emissions, the current implementation wrongly also takes into consideration the extra `bHERMES` growth tokens (to the locked) thus is allocated a larger value then intended. This also has an indirect effect of increasing protocol inflation if `HERMES` [needs to be minted in order to reach the required token amount](https://github.com/code-423n4/2023-05-maia/blob/main/src/hermes/minters/BaseV2Minter.sol#L138-L141).

### Issue details

Token DAO shares (`share` variable) is calculated in `BaseV2Minter::updatePeriod` as such:

https://github.com/code-423n4/2023-05-maia/blob/main/src/hermes/minters/BaseV2Minter.sol#L133-L137
```Solidity
    uint256 _growth = calculateGrowth(newWeeklyEmission);
    uint256 _required = _growth + newWeeklyEmission;
    /// @dev share of newWeeklyEmission emissions sent to DAO.
    uint256 share = (_required * daoShare) / base;
    _required += share;
```

We actually do see that the original developer intention (confirmed by sponsor) was that the share value to be calculated relative to `newWeeklyEmission`, not to (`_required = newWeeklyEmission + _growth`)

```Solidity
    /// @dev share of newWeeklyEmission emissions sent to DAO.
```

Also, it is [documented that DAO shares should be calculated as part of weekly emissions](https://v2-docs.maiadao.io/protocols/Hermes/overview/tokenomics/emissions#dao-emissions):

> Up to 30% of weekly emissions can be allocated to the DAO.

## Proof of Concept

DAO shares value is not calculated relative to `newWeeklyEmission`

https://github.com/code-423n4/2023-05-maia/blob/main/src/hermes/minters/BaseV2Minter.sol#L134-L136

## Tools Used

Manual analysis

## Recommended Mitigation Steps

Change the implementation to reflect intention

```diff
diff --git a/src/hermes/minters/BaseV2Minter.sol b/src/hermes/minters/BaseV2Minter.sol
index 7d7f013..217a028 100644
--- a/src/hermes/minters/BaseV2Minter.sol
+++ b/src/hermes/minters/BaseV2Minter.sol
@@ -133,7 +133,7 @@ contract BaseV2Minter is Ownable, IBaseV2Minter {
             uint256 _growth = calculateGrowth(newWeeklyEmission);
             uint256 _required = _growth + newWeeklyEmission;
             /// @dev share of newWeeklyEmission emissions sent to DAO.
-            uint256 share = (_required * daoShare) / base;
+            uint256 share = (newWeeklyEmission * daoShare) / base;
             _required += share;
             uint256 _balanceOf = underlying.balanceOf(address(this));
             if (_balanceOf < _required) {

```

# [H-02] Removing a BribeFlywheel from a Gauge does not remove the reward asset from the rewards depo, making it impossible to add a new Flywheel with the same reward token

Removing a BribeFlywheel from a Gauge does not remove the reward asset from the rewards depo, making it impossible to add a new Flywheel with the same reward token

## Impact

Removing a bribe Flywheel (`FlywheelCore`) from a Gauge (via `BaseV2Gauge::removeBribeFlywheel`) does not remove the reward asset (call `MultiRewardsDepot::removeAsset`) from the rewards depo (`BaseV2Gauge::multiRewardsDepot`), making it impossible to add a new Flywheel (by calling `BaseV2Gauge::addBribeFlywheel`) with the same reward token (because `MultiRewardsDepot::addAsset` reverts as the assets already exists).

Impact is limiting protocol functionality in unwanted ways, possibly impacting gains in the long run (example due to incentives loves by not having a specific token bribe reward).

## Proof of Concept

_Observation: a `BribeFlywheel` is a `FlywheelCore` with a `FlywheelBribeRewards` set as the `FlywheelRewards`, typically created using the `BribesFactory::createBribeFlywheel`_

### Scenario and execution flow

- project decides to add an initial  `BribeFlywheel` to the recently deployed `UniswapV3Gauge` contract.
- this is done by calling [`UniswapV3GaugeFactory::BaseV2GaugeFactory::addBribeToGauge`](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/factories/BaseV2GaugeFactory.sol#L144-L148)
    - executions further goes to `BaseV2Gauge::addGaugetoFlywheel` where [the bribe flywheel reward token is added](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/BaseV2Gauge.sol#L135) to the multi reward depo
- project decides, for whatever reason (a bug in the contract, an exploit, a decommission, a more profitable wheel that would use the same rewards token), that they want to replace the old flywheel with a new one
- removing is done via calling [`UniswapV3GaugeFactory::BaseV2GaugeFactory::removeBribeFromGauge`](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/factories/BaseV2GaugeFactory.sol#L151-L154)
    - executions further goes to `BaseV2Gauge::removeBribeFlywheel` where the flywheel is removed but the reward token asset [is not remove from the multi reward depo](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/BaseV2Gauge.sol#L144-L152), there is no call to [`MultiRewardsDepot::removeAsset`](https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/depots/MultiRewardsDepot.sol#L57-L65):

```Solidity
    function removeBribeFlywheel(FlywheelCore bribeFlywheel) external onlyOwner {
        /// @dev Can only remove active flywheels
        if (!isActive[bribeFlywheel]) revert FlywheelNotActive();

        /// @dev This is permanent; can't be re-added
        delete isActive[bribeFlywheel];

        emit RemoveBribeFlywheel(bribeFlywheel);
    }
```
- after removal, when trying to add a new flywheel with the same rewards token, execution fails with `ErrorAddingAsset` since the [`addAsset`](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/BaseV2Gauge.sol#L135) call reverts since [the rewards token was not removed](https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/depots/MultiRewardsDepot.sol#L48) with the previous call to `BaseV2Gauge::removeBribeFlywheel`.

## Tools Used

Manual analysis

## Recommended Mitigation Steps

when `BaseV2Gauge::removeBribeFlywheel` is called for a particular flywheel, also remove it's corresponding reward depo token.

Example implementation
```diff
diff --git a/src/gauges/BaseV2Gauge.sol b/src/gauges/BaseV2Gauge.sol
index c2793a7..8ea6c1e 100644
--- a/src/gauges/BaseV2Gauge.sol
+++ b/src/gauges/BaseV2Gauge.sol
@@ -148,6 +148,9 @@ abstract contract BaseV2Gauge is Ownable, IBaseV2Gauge {
         /// @dev This is permanent; can't be re-added
         delete isActive[bribeFlywheel];
 
+        address flyWheelRewards = address(bribeFlywheel.flywheelRewards());        
+        multiRewardsDepot.removeAsset(flyWheelRewards);
+
         emit RemoveBribeFlywheel(bribeFlywheel);
     }
 

```

# Medium Risk Findings (4)

# [M-01] If `HERMES` gauge rewards are not queued for distribution every week, they are slashed

## Impact

In order to queue weekly `HERMES` rewards for distribution, `FlywheelGaugeRewards::queueRewardsForCycle` must be called during the next cycle (week). If a cycle has passed and no one calls `queueRewardsForCycle` to queue rewards, cycle gauge rewards are lost as internal accounting does not take into consideration time passing, only last processed cycle.

### Issue details

The minter kicks off a new epoch via calling `BaseV2Minter::updatePeriod`. The execution flow goes to `FlywheelGaugeRewards::queueRewardsForCycle` -> `FlywheelGaugeRewards::_queueRewards` where after several checks the [rewards are queued](https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/rewards/FlywheelGaugeRewards.sol#L189-L193) in order for them to be retrieved via a call to [`FlywheelGaugeRewards::getAccruedRewards`](https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/rewards/FlywheelGaugeRewards.sol#L200) from [`BaseV2Gauge::newEpoch`](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/BaseV2Gauge.sol#L89).

Reward queuing logic revolves around the [current and previously saved gauge cycle](https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/rewards/FlywheelGaugeRewards.sol#L78-L80):

```Solidity
        // next cycle is always the next even divisor of the cycle length above current block timestamp.
        uint32 currentCycle = (block.timestamp.toUint32() / gaugeCycleLength) * gaugeCycleLength;
        uint32 lastCycle = gaugeCycle;
```

This way of noting cycles ([and further checks done](https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/rewards/FlywheelGaugeRewards.sol#L82-L85)) does not take into consideration any intermediary cycles, only that new cycle is after old cycle. If `queueRewardsForCycle` is not called for a number of cycles then rewards will be lost for those cycles

## Proof of Concept

Rewards are calculate for current cycle and last stored cycle only, with no intermediary accounting:
https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/rewards/FlywheelGaugeRewards.sol#L78-L80

Visual example:
```
 0 1 2 3 4 5 6  (epoch/cycle)
+-+-+-+-+-+-+-+
|Q|Q|Q| | |Q|Q|
+-+-+-+-+-+-+-+
```

Up until epoch 2 `queueRewardsForCycle` (Q) was called, for cycle 3 and 4 nobody calls, on cycle 5 `queueRewardsForCycle` is called again but cycle 3 and 4 rewards are not taken into consideration.


## Tools Used

Manual analysis

## Recommended Mitigation Steps

Because of the way the entire MaiaDAO ecosystem is set up, the premise is that someone will call `BaseV2Minter::updatePeriod` (which calls `FlywheelGaugeRewards::queueRewardsForCycle`) as there is incentive for users (or project) to do so. Realistically this _should_ always happen, but unforeseen events may lead to this event.

It is difficult from an architectural point of view, regarding how MaiaDAO is constructed, to offer a solution. A generic suggestion would be to implemented a snapshot mechanism / dynamic accounting of each cycle but then the issue would be who triggers that snapshot event? 

This issue is real but mitigating it is not straightforward or evident in web3 context.


# [M-02] Unstaking `vMAIA` tokens on the first Tuesday of the month can be offset

## Impact

According to [project documentation](https://v2-docs.maiadao.io/protocols/overview/tokenomics/vMaia) and [natspec](https://github.com/code-423n4/2023-05-maia/blob/main/src/maia/vMaia.sol#L23-L24):
> Users can stake their MAIA tokens at any time, but can only withdraw their staked tokens on the first Tuesday of each month. 

> NOTE: Withdraw is only allowed once per month, during the 1st Tuesday (UTC+0) of the month.

The implementation that keeps the above invariant true is dependent on at least one user attempting to unstake their `vMAIA` on the first chronological Tuesday of the month. But if nobody unstakes on the first Tuesday, then, on the second Tuesday of the month, the conditions are met and users can unstake then. Again, if no one unstakes on the second Tuesday, then the next Tuesday after that will be valid. So on and so forth.

Not respecting declared withdraw/unstaking period and limitation is a sever protocol issue in itself. The case is also not that improbable to happen, if good enough incentives are present there will be odd Tuesdays where nobody will unstake, thus creating this loophole.

### Issue details

`vMAIA` is an `ERC4626` vault compliant contract ([`vMAIA`](https://github.com/code-423n4/2023-05-maia/blob/main/src/maia/vMaia.sol#L26) -> [`ERC4626PartnerManager`](https://github.com/code-423n4/2023-05-maia/blob/main/src/maia/tokens/ERC4626PartnerManager.sol#L22) -> [`ERC4626`](https://github.com/code-423n4/2023-05-maia/blob/main/src/erc-4626/ERC4626.sol)). `ERC4626::withdraw` has a [beforeWithdraw](https://github.com/code-423n4/2023-05-maia/blob/main/src/erc-4626/ERC4626.sol#L70) hook callback that is overwritten/implemented in [vMAIA::beforeWithdraw](https://github.com/code-423n4/2023-05-maia/blob/main/src/maia/vMaia.sol#L97-L114)

```Solidity
    /**
     * @notice Function that performs the necessary verifications before a user can withdraw from their vMaia position.
     *  Checks if we're inside the unstaked period, if so then the user is able to withdraw.
     * If we're not in the unstake period, then there will be checks to determine if this is the beginning of the month.
     */
    function beforeWithdraw(uint256, uint256) internal override {
        /// @dev Check if unstake period has not ended yet, continue if it is the case.
        if (unstakePeriodEnd >= block.timestamp) return;


        uint256 _currentMonth = DateTimeLib.getMonth(block.timestamp);
        if (_currentMonth == currentMonth) revert UnstakePeriodNotLive();


        (bool isTuesday, uint256 _unstakePeriodStart) = DateTimeLib.isTuesday(block.timestamp);
        if (!isTuesday) revert UnstakePeriodNotLive();


        currentMonth = _currentMonth;
        unstakePeriodEnd = _unstakePeriodStart + 1 days;
    }
```

By thoroughly analyzing the function we can see that:

- it first checks if unstake period has not ended. The unstake period is 24h since the start of Tuesday. On the first call for the contract this is 0, so execution continues
```Solidity
        /// @dev Check if unstake period has not ended yet, continue if it is the case.
        if (unstakePeriodEnd >= block.timestamp) return;
```

- it then gets the current month and compares it to the last saved "currentMonth". The `currentMonth` is set only after the Tuesday condition is meet. Doing it this way they assure that after a Tuesday was validated then no further unstakes can happen in the same month
```Solidity
        uint256 _currentMonth = DateTimeLib.getMonth(block.timestamp);
        if (_currentMonth == currentMonth) revert UnstakePeriodNotLive();
```

- the next operation is to determine if now is a Tuesday and also return the start of the current day (this is to be used in determining the unstake period). To note here is that the check is only it is a Tuesday, **not the first Tuesday of the month**, rather, up until now, the check is that  _this is the first Tuesday in a month that was noted by this execution_
```Solidity
        (bool isTuesday, uint256 _unstakePeriodStart) = DateTimeLib.isTuesday(block.timestamp);
        if (!isTuesday) revert UnstakePeriodNotLive();
```

- after checking that we are in _the first **marked** Tuesday of this month_ the current month is noted (saved to `currentMonth`) and the unstake period is defined as the entire day (24h since the start of Tuesday)
```Solidity
        currentMonth = _currentMonth;
        unstakePeriodEnd = _unstakePeriodStart + 1 days;
```

To conclude the flow: the withdraw limitation is actually: _in a given month, on the first Tuesday where users attempt to withdraw and only on that Tuesday will withdrawals be allowed. It can be the last Tuesday of the month or the first Tuesday of the month_.

## Proof of Concept

Add the following coded POC to `test\maia\vMaiaTest.t.sol` and run it with `forge test --match-test testWithdrawMaiaWorksOnAnyThursday -vvv`
```Solidity

    import {DateTimeLib as MaiaDateTimeLib} from "@maia/Libraries/DateTimeLib.sol"; // add this next to the other imports

    function testWithdrawMaiaWorksOnAnyThursday() public {
        testDepositMaia();
        uint256 amount = 100 ether;

        // we now are in the first Tuesday of the month (ignore the name, getFirstDayOfNextMonthUnix gets the first Tuesday of the month)
        hevm.warp(getFirstDayOfNextMonthUnix());

        // sanity check that we are actually in a Tuesday
        (bool isTuesday_, ) = MaiaDateTimeLib.isTuesday(block.timestamp);
        assertTrue(isTuesday_);

        // no withdraw is done, and then the next Tuesday comes
        hevm.warp(block.timestamp + 7 days);

        // sanity check that we are actually in a Tuesday, again
        (isTuesday_, ) = MaiaDateTimeLib.isTuesday(block.timestamp);
        assertTrue(isTuesday_);

        // withdraw succeeds even if we are NOT in the first Tuesday of the month, but in the second one
        vmaia.withdraw(amount, address(this), address(this));

        assertEq(maia.balanceOf(address(vmaia)), 0);
        assertEq(vmaia.balanceOf(address(this)), 0);
    }
```

## Tools Used

Manual analysis and ChatGPT for the `isFirstTuesdayOfMonth` function optimizations.

## Recommended Mitigation Steps

Modify the `isTuesday` function into a `isFirstTuesdayOfMonth` function, a function that checks that the given timestamp is in the first Tuesday of it's containing month.

Example implementation:

```Solidity
    /// @dev Returns if the provided timestamp is in the first Tuesday of it's corresponding month (result) and (startOfDay);
    ///      startOfDay will always by the timestamp of the first Tuesday found searching from the given timestamp,     
    ///      regardless if it's the first of the month or not, so always check result if using it
    function isFirstTuesdayOfMonth(uint256 timestamp) internal pure returns (bool result, uint256 startOfDay) {
        uint256 month = getMonth(timestamp);
        uint256 firstDayOfMonth = timestamp - ((timestamp % 86400) + 1) * 86400;
        
        uint256 dayIndex = ((firstDayOfMonth / 86400 + 3) % 7) + 1; // Monday: 1, Tuesday: 2, ....., Sunday: 7.
        uint256 daysToAddToReachNextTuesday = (9 - dayIndex) % 7;

        startOfDay = firstDayOfMonth + daysToAddToReachNextTuesday * 86400;
        result = (startOfDay <= timestamp && timestamp < startOfDay + 86400) && month == getMonth(startOfDay);
    }

```

# [M-03] `BribesFactory::createBribeFlywheel` can be completely blocked from creating any Flywheel by a malicious actor

## Impact
A malicious actor can completely block the creation of any bribe flywheel that is created via `BribesFactory::createBribeFlywheel` because of the way the `FlywheelBribeRewards` parameter is set.
Initially it is set [to the zero address in its constructor](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/factories/BribesFactory.sol#L84) and then reset to [a different address via the `FlywheelCore::setFlywheelRewards` call](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/factories/BribesFactory.sol#L96) (in the same transaction). 

```Solidity
    function createBribeFlywheel(address bribeToken) public {
        // ...

        FlywheelCore flywheel = new FlywheelCore(
            bribeToken,
            FlywheelBribeRewards(address(0)),
            flywheelGaugeWeightBooster,
            address(this)
        );

        // ...

        flywheel.setFlywheelRewards(address(new FlywheelBribeRewards(flywheel, rewardsCycleLength)));
        
        // ...
    }
```

The `FlywheelCore::setFlywheelRewards` function verifies if the current `flywheelRewards` address has any balance of the provided reward token and, if so, [transfers it to the new `flywheelRewards` address](https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/base/FlywheelCore.sol#L125-L129).

```Solidity
    function setFlywheelRewards(address newFlywheelRewards) external onlyOwner {
        uint256 oldRewardBalance = rewardToken.balanceOf(address(flywheelRewards));
        if (oldRewardBalance > 0) {
            rewardToken.safeTransferFrom(address(flywheelRewards), address(newFlywheelRewards), oldRewardBalance);
        }
```

The issue is that `FlywheelCore::setFlywheelRewards` does not check if the current `FlywheelCore::flywheelRewards` address is 0 and thus attempts to transfer from 0 address if that address has any reward token in it. 
A malicious actor can simply send 1 wei of `rewardToken` to the zero address and all `BribesFactory::createBribeFlywheel` will fail because of the attempted transfer of tokens from the 0 address.

This is also an issue for any 3rd party project, that wishes to use MaiaDAO's `BribesFactory` implementation, and uses a burnable reward token because most likely normal users (non-malicious) have already burned (sent to zero address) tokens so the creating of bribe factories would fail by default.

Another observation is that, because all MaiaDAO project tokens use the Solmate ERC20 implementation they [all can transfer to 0 (burn)](https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC20.sol#L76-L83) which makes this scenario real even if using project tokens as reward tokens.

## Proof of Concept

A coded POC follows, add it to `test\gauges\factories\BribesFactoryTest.t.sol`:

```Solidity
    import {stdError} from "forge-std/Test.sol";

    function testDosCreateBribeFlywheel() public {
        MockERC20 bribeToken3 = new MockERC20("Bribe Token3", "BRIBE3", 18);
        bribeToken3.mint(address(this), 1000);
        
        // transfer 1 wei to zero address (or "burn" on other contracts)
        bribeToken3.transfer(address(0), 1);
        assertEq(bribeToken3.balanceOf(address(0)), 1);
                
        // hevm.expectRevert(stdError.arithmeticError); // for some reason this does not work, foundry error        
        // function reverts regardless with "Arithmetic over/underflow" because the way Solmate ERC20::transferFrom is implemented
        factory.createBribeFlywheel(address(bribeToken3)); 
    }
```

Observation: because the `MockERC20` contract uses Solmate ERC20 implementation, the error is `"Arithmetic over/underflow"` since `address(0)` did not pre-approve the token swap (evidently).

## Tools Used

Manual analysis

## Recommended Mitigation Steps

- if project tokens are to be used as reward tokens, consider using OpenZeppelin ERC20 implementation (as [it does not allow transfer to 0 address](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol#L225-L233) if burn is not intended) or add checks to all project token contracts that transfer `to` argument must never be `address(0)`.
- modify `FlywheelCore::setFlywheelRewards` to not attempt any token transfer if previous `flywheelRewards` is `address(0)`. Example implementation:

```diff
diff --git a/src/rewards/base/FlywheelCore.sol b/src/rewards/base/FlywheelCore.sol
index 308b804..eaa0093 100644
--- a/src/rewards/base/FlywheelCore.sol
+++ b/src/rewards/base/FlywheelCore.sol
@@ -123,9 +123,11 @@ abstract contract FlywheelCore is Ownable, IFlywheelCore {
 
     /// @inheritdoc IFlywheelCore
     function setFlywheelRewards(address newFlywheelRewards) external onlyOwner {
-        uint256 oldRewardBalance = rewardToken.balanceOf(address(flywheelRewards));
-        if (oldRewardBalance > 0) {
-            rewardToken.safeTransferFrom(address(flywheelRewards), address(newFlywheelRewards), oldRewardBalance);
+        if (address(flywheelRewards) != address(0)) {
+            uint256 oldRewardBalance = rewardToken.balanceOf(address(flywheelRewards));
+            if (oldRewardBalance > 0) {
+                rewardToken.safeTransferFrom(address(flywheelRewards), address(newFlywheelRewards), oldRewardBalance);
+            }
         }
 
         flywheelRewards = newFlywheelRewards;

```

# [M-04] Removing a `UniswapV3Gauge` via `UniswapV3GaugeFactory` does not actually remove it from the `UniswapV3Staker`; Gauge still gains rewards, can be staked to (even though deprecated) plus old stakers game the rewards of new stakers

## Impact

Gauge factories have a [`BaseV2GaugeFactory::removeGauge`](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/factories/BaseV2GaugeFactory.sol#L130-L137) that removes the indicate gauge and marks it as deprecated for the corresponding [`bHermesGauges` and `bHermesBoost`](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/factories/BaseV2GaugeManager.sol#L100-L103) token contracts.

However, removing a `UniswapV3Gauge` with `UniswapV3GaugeFactory` does not actually remove it from the `UniswapV3Staker`; Gauge still remains and existing users that staked still gain the exact same benefits from it.
What is worse that staking to the gauge can still happen and any new users that stakes cannot receive a share of the generated fees (plus boost), as it is impossible to vote for the deprecated gauge.

### Issue detailed explanation

When a `UniswapV3Gauge` is created via `UniswapV3GaugeFactory` it is also attached to a `UniswapV3Staker` via the [`BaseV2GaugeFactory::afterCreateGauge`](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/factories/BaseV2GaugeFactory.sol#L122-L125) callback [implementation](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/factories/UniswapV3GaugeFactory.sol#L87-L91):

```Solidity
    /// @notice Adds Gauge to UniswapV3Staker
    /// @dev Updates the UniswapV3 staker with bribe and minimum width information
    function afterCreateGauge(address strategy, bytes memory) internal override {
        uniswapV3Staker.updateGauges(IUniswapV3Pool(strategy));
    }
```

However there is no `afterCreateRemoved` mechanism implemented in `BaseV2GaugeFactory`, as such, the `UniswapV3Staker` contract is never updated about the removed gauge.
This creates a situation in which existing users/stakes benefit while new stakes lose out on bribes, gaming the system.

This is because: 

- new users can stake to the deprecated gauge, as there is no mechanism to check if the gauge they are staking to is active or not (similar to how [`UniswapV3Staker::updateGauges` checks](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/uni-v3-staker/UniswapV3Staker.sol#L526-L529))

```Solidity
    function updateGauges(IUniswapV3Pool uniswapV3Pool) external {
        address uniswapV3Gauge = address(uniswapV3GaugeFactory.strategyGauges(address(uniswapV3Pool)));


        if (uniswapV3Gauge == address(0)) revert InvalidGauge();
```

- **but** new users that stake to the deprecated gauge do not receive a portion of fees that are generated and sent to the bribe deposit. Although users that have staked to the _now_ deprecated gauge **beforehand** still gain the fees generated by the staked positions. 

This happens because when gauge removal happened in [BaseV2GaugeManager::removeGauge](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/factories/BaseV2GaugeManager.sol#L100-L103) the indicated gauge is marked as deprecated ([bHermesGauges::ERC20Gauges::_removeGauge](https://github.com/code-423n4/2023-05-maia/blob/main/src/erc-20/ERC20Gauges.sol#L435). Users that have already voted to the deprecated gauge still get the bribe rewards when `BaseV2Gauge::accrueBribes` is called. 

Rewards flows is:

- [BaseV2Gauge::accrueBribes](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/BaseV2Gauge.sol#L111-L121)
    - [FlywheelCore::accrue](https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/base/FlywheelCore.sol#L69-L72)
        - [FlywheelCore::_accrue](https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/base/FlywheelCore.sol#L74-L81)
            - [FlywheelCore::accrueUser](https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/base/FlywheelCore.sol#L196-L198) (influences reward calculation)
                - [FlywheelBoosterGaugeWeight::boostedBalanceOf](https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/booster/FlywheelBoosterGaugeWeight.sol#L58-L61)
                    - [bHermesGauges::ERC20Gauges::getUserGaugeWeight](https://github.com/code-423n4/2023-05-maia/blob/main/src/erc-20/interfaces/IERC20Gauges.sol#L62-L65)
                        - and `ERC20Gauges::getUserGaugeWeight` is only [increasable if the gauge is not deprecated](https://github.com/code-423n4/2023-05-maia/blob/main/src/erc-20/ERC20Gauges.sol#L203)


To be noted that the action of _unstaking_ (calling [UniswapV3Staker::_unstakeToken](https://github.com/code-423n4/2023-05-maia/blob/main/src/uni-v3-staker/UniswapV3Staker.sol#L377-L390)) sends rewards to the gauge bribe deposit:
```Solidity
            // scope for bribeAddress, avoids stack too deep errors
            address bribeAddress = bribeDepots[key.pool];


            if (bribeAddress != address(0)) {
                nonfungiblePositionManager.collect(
                    INonfungiblePositionManager.CollectParams({
                        tokenId: tokenId,
                        recipient: bribeAddress,
                        amount0Max: type(uint128).max,
                        amount1Max: type(uint128).max
                    })
                );
            }
        }
```

from where it is then transferred to those that already delegated to the (now deprecated) gauge following the already mentioned execution flow.


Note, deprecated gauges [still have the boosting bonus associated with `bHermesBoost`](https://github.com/code-423n4/2023-05-maia/blob/main/src/uni-v3-staker/UniswapV3Staker.sol#L401-L424) where the same issue as above appears, already existing users get the boost, new users cannot. 

```Solidity
        // ...
        
        // get boost amount and total supply
        (boostAmount, boostTotalSupply) = hermesGaugeBoost.getUserGaugeBoost(owner, address(gauge));
        
        // ...

        secondsInsideX128 = RewardMath.computeBoostedSecondsInsideX128(
            // ...
            uint128(boostAmount),
            uint128(boostTotalSupply),
            // ...

        );

        // ...

        uint256 reward = RewardMath.computeBoostedRewardAmount(
            // ...
            secondsInsideX128,
            // ...
        );

        }
```

## Proof of Concept

A step by step execution flow was shown above in **Issue detailed explanation**

Lack of active gauge check can be observed in any of the staking flow functions:
- [restakeToken](https://github.com/code-423n4/2023-05-maia/blob/main/src/uni-v3-staker/UniswapV3Staker.sol#L340-L348)
- [stakeToken](https://github.com/code-423n4/2023-05-maia/blob/main/src/uni-v3-staker/UniswapV3Staker.sol#L465-L473)
- [_stakeToken](https://github.com/code-423n4/2023-05-maia/blob/main/src/uni-v3-staker/UniswapV3Staker.sol#L475-L519) (called by the above 2)

Also no [`BaseV2GaugeFactory::afterCreateRemoved`](https://github.com/code-423n4/2023-05-maia/blob/main/src/gauges/factories/BaseV2GaugeFactory.sol#L125-L137) type callback existing

A theoretical POC would by as:
- a `UniswapV3Gauge` is created via `UniswapV3GaugeFactory` (it is also automatically attached to the existing `UniswapV3Staker`)
- users vote for it via `bHermesGauges`
- team decides to remove the existing `UniswapV3Gauge` for a newer version
- team calls `BaseV2GaugeFactory::removeGauge` that does not remove the gauge from the `UniswapV3Staker` while also deprecating it in `bHermesGauges`
- the now deprecated and faulty removed `UniswapV3Gauge` still receives fees from the `UniswapV3Staker`
- new users stake to the removed `UniswapV3Gauge` but will not receive any bribe rewards creating a situation where the first depositors gain the later ones

## Tools Used

Manual analysis and, **the most important factor**: a very good, active and helping project team!

## Recommended Mitigation Steps

As the system is complex we must take into consideration a few observations:

- we cannot remove the gauge from `UniswapV3Staker` because already existing incentives would become bricked and worthless
- removing a gauge completely from the `UniswapV3Staker` means loosing potential rewards deposited by users
    - a `UniswapV3Staker` without the `bHermesGauges` mechanism is similar to a normal `UniswapV3Staker` so it does still work has some incentive
- leaving the gauge open to be staked and added incentives would allow old stakers to pray on new stakes and new stakers will not receive any fees generated by the staked positions
- refunding potential emissions (rewards) deposited by users (or protocols) adds storage overhead
- a `BaseV2GaugeFactory::afterCreateRemoved` mechanism is required regardless for any future gauge that needs post remove operations
- deprecated gauges still have the boosting bonus associated with `bHermesBoost` where the same issue as above appears, already existing users get the boost, new users cannot

A possible solution can be a mix of the above:
1. create a `BaseV2GaugeFactory::afterCreateRemoved` mechanism
2. add a function `UniswapV3GaugeFactory::afterCreateRemoved` that overrides the above that calls `UniswapV3Staker::updateBribeDepot`
3. in `UniswapV3Staker::updateBribeDepot` check if the strategy associated with the `IUniswapV3Pool` is active and if not, then set the bribeDepot (`bribeDepots[uniswapV3Pool]`) of that pool's gauge to the zero address so that no new rewards are sent to the deprecated gauge

The above is a minimum suggested regardless. Extra:

3. add a check for  `UniswapV3Staker::_stakeToken` if the gauge is active or not and revert; i.e. do not allow any further staking into inactive gauges
4. Consider decreasing the gain value of `bHermesBoost` if the gauge is deprecated in `_unstakeToken`. Some more consideration should be taken if implementing this as any reward bonuses that were not collected before the removal/deprecation will also be lost (that in itself is an issue that must not happen).



# Codebase Analysis of the MaiaDAO ecosystem codebase

## Codebase overview

Maia DAO Ecosystem [codebase](https://github.com/code-423n4/2023-05-maia/tree/main/src) is separated into the following directories:
```
├───erc-20
├───erc-4626
├───gauges
├───governance
├───hermes
├───maia
├───rewards
├───talos
├───ulysses-amm
├───ulysses-omnichain
└───uni-v3-staker
```
An observation for new developers or auditors, some contracts are grouped on _functionality similarity_ others on _component of which they belong to_ which can prove a bit difficult to follow at first

**Grouped based on similar functionality**
- erc-20
- erc-4626
- gauges
- governance
- rewards
- uni-v3-staker

**Grouped based on belonging project**
- hermes
- maia
- talos
- ulysses-amm
- ulysses-omnichain

Those grouped by project often use common code from the other directories. In particular `erc-20`, `erc-4626` and `rewards` are heavily used.

**Other codebase observations**
- contracts and variables need better naming as in some cases the name has nothing to do with the functionality. Example: [FlywheelBoosterGaugeWeight](https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/booster/FlywheelBoosterGaugeWeight.sol) does not actually boost anything, it stores the user gauge related balances
- it would be better to group all code by component where possible, instead of by name
- code documentation is at a minimum accepted level and can always improve; the same with project testing
- nice ASCII art on some contracts!


## Patterns and reused code

Several MaiaDAO components are actually extensions, forks or inspired by existing projects. In total MaiaDAO borrows code from at least 3 different major projects:
- **Hermes** is a [Solidly](http://web.archive.org/web/20220422051938/https://github.com/solidlyexchange/solidly) fork
- **Talos** is based on [popsicle.finance](https://github.com/Popsicle-Finance)
- **Flywheel reward logic** is inspired by [Tribe DAO Contracts](https://github.com/fei-protocol/flywheel-v2)

and a few smaller code components are borrowed from (or inspired by) popular libraries such as:
- **ERC4626 vaults** implementation is extended from [Solmate](https://github.com/transmissions11/solmate/blob/main/src/mixins/ERC4626.sol)
- **UniswapV3Staker** (uni-v3-staker) is a modified implementation of the original [UniswapV3Staker](https://github.com/Uniswap/v3-staker/blob/main/contracts/UniswapV3Staker.sol)
- **vMAIA DateTimeLib** utility is a stripped down version of [Solady](https://github.com/vectorized/solady/blob/main/src/utils/DateTimeLib.sol)'s DateTimeLib

## Project unique attributes

From a unique point of view (at least subjectively to the author of this description) MaiaDAO codebase:
- manages to combine a large number of complex and complicated components into one, this is a feat into itself
- uses extensive, _and I do mean extensive_, use of factories for almost every logic contract there is (there are in total 20 different factory contracts)
- uses peculiar time constraints; for example, `vMAIA` unstaking can happen only on the first Tuesday of the month, why not on the first Friday? or Saturday?

## Learning resources

In order to help any future auditor or developer of the project, this section was created to provide resources (or aggregate existing resources) in that direction

- A [high level description of each project contract Solidity file](https://github.com/code-423n4/2023-05-maia#contracts) was provided by the MaiaDAO team. 

- The [code4ena MaiaDAO documentation](https://github.com/code-423n4/2023-05-maia#overview) also aggregates all team provided resources up to this point and can serve as a good initial read for people wishing to understand the system as a whole

For future auditors of the project, the following advice may be of help:
- when staring an audit, with regards to Hermes, start from either rewards logic ([FlywheelCore](https://github.com/code-423n4/2023-05-maia/blob/main/src/rewards/base/FlywheelCore.sol)) or Hermes Minter ([BaseV2Minter](https://github.com/code-423n4/2023-05-maia/blob/main/src/hermes/minters/BaseV2Minter.sol)) and work your way from there
 - read the project documentation and attempt to make a summary specifically for your usage or understanding
 - after going through the documentation ask yourself questions about assumptions that you read
- make a high level component interaction diagram. Recommended tool: [excalidraw](https://excalidraw.com/), [miro](https://miro.com/) or [draw.io](https://app.diagrams.net/)


The following link: [**ABA MaiaDAO overview**](https://ethereal-net-2b6.notion.site/ABA-MaiaDAO-overview-bc57e0924a3745b48ef2f6b5f61bdb3a),
 is this authors's own attempt to summaries MaiaDAOs documentation. At the end of the article you will find sections helpful for future auditors:
- **Questions to answer when auditing the codebase**: after reading the documentation, what type of questions, as an auditor should you ask yourself
- **Containing components and their respective audits**: a section with information about projects from which code was borrowed (or is used) and their respective audits. It is possible some issue may reappear or you can use POCs or the mental model the auditors have conveyed in the audit


# Architecture feedback

## Current Architecture

- The architecture in itself is complicated and at places also complex. It tries to follow the 4 major components (protocols) of the MaiaDAO ecosystem: Maia, Hermes, Talos and Ulysses

- Extensive usage of factories also add an extra layer of complexity but that is unfortunately unavoidable due to the nature of the project.

- Rewards logic has at its core the bHermes token, which presents an interesting multi-use scenarios for the ERC20 token. 

- Project also uses the [ve(3,3) model](https://andrecronje.medium.com/ve-3-3-44466eaa088b) as an observation

The following link shows a partial architecture diagram that contains the rewards logic for Hermes and UniswapV3Staker gauge logic:
**https://link.excalidraw.com/readonly/JnTC9HB2yLadiFxKzlwA?darkMode=true**

## Future considerations

- Some issues with the current architecture are regarding the removal of an element. Removals are not entirely atomic, several different "remove/disable" actions need to be done separately across the protocol to fully remove a component from it. Consider making a more thorough and documented removal system.

- in case of a migration to a V3 (similar to how now the project is migrating to a V2), current architecture and V2 implementation overall does not take this into consideration. If this is not properly treated it can cause project problems in the future. Example, with current implementation you cannot stop the UniswapV3 Staker or migrate th

- not necessarily an architectural issue, but project uses `Solmate ERC20` implementation which [allows for transfer to 0 address](https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC20.sol#L76-L110). This may have negative repercussions in the future. As a suggestion, add checks that no transfers can go to zero address or user OpenZeppelin's ERC20 implementation.

# Centralization risks

- Trust assumptions need to be respected for the project to survive. A malicious (or compromised) ownership can remove liquidity from almost the entire project. This can be done either by adding malicious strategies (Gauges) to existing project reward components or by adding malicious reward tokens.

- The following statement may be controversial: due to the highly complicated nature of the project, even it anyone can kickoff system critical and time dependent actions on-chain (no access control limitations), these are still mostly done by the team. An example is queuing weekly pending bHermes rewards. In that respect, if the team does not setup proper automation then protocol health may plummet and rewards may be lost.

# Systemic risks

- MaiaDAO has one of the most prolific development team that the author has seen. That being said, a major issue lies in the fact that the core developers is a team of 3. This results in a [project bus factor of 3](https://en.wikipedia.org/wiki/Bus_factor). If anything happens to one of the team members, development will be drastically stalled or even project may be closed down. Add more team members.


- MaiaDAO uses Multichain's [`Anycall`](https://docs.multichain.org/developer-guide/anycall-v7) implementation for inter-chain communication. Recent events regarding the [Multichain team being arrested](https://twitter.com/BoringSleuth/status/1661399578313211905) and having needed to [shutdown a part of their supported bridges](https://twitter.com/MultichainOrg/status/1663941611380965376) raises concerns regarding the future of the protocol. It is best to also explore alternatives for `Anycall`.

- From a token/liquidity point of view, the protocol does have any more of a system risk that other DeFi products. Its spread to V3 Uniswap liquidity mining, staking, and liquidity marketplaces gives the project a better changes of surviving any black swan type events that may occur.


# Other recommendations

### **Protocol needs more developers**

The rate of code development outpaces the security insight and mental attention to details that the team can handle. MaiaDAO developers have proven incredibly good developers but growing the system, in a relatively short time span, with such a small team has meant that fine details are missed out. This can be seen even by the presence of typos in the code that, extending on the [broken windows theory](https://en.wikipedia.org/wiki/Broken_windows_theory), also implies hidden code smells and issues. 

A more evident example is the simple _protocol fee calculation reversal_ that was seen [in a previous audit](https://github.com/Maia-DAO/maia-ecosystem-monorepo/blob/1-audit/audits/Maia%20DAO%20February%202023%20-%20Zellic%20Audit%20Report.pdf). This type of issue is identified with a simple test. It's presence, among others, enforce the idea that the team needs more developers and eyes on the project.

### **Implement proper events monitoring** 

MaiaDAO ecosystem implementation leaves the subtle impression that events are not used to their full potential. There are several cases where events are missing and even some that seem to be more likely emitted because "it's good practice", rather that actually being good for the team. Consider using events to their full potential.

### **User more of our auditing tools**

The majority of auditors use Visual Studio Code with some specific plugins. A recomendation for the project developers is to use VC with the plugins:
- [Solidity Visual Developer](https://marketplace.visualstudio.com/items?itemName=tintinweb.solidity-visual-auditor)
- [Code Spell Checker](https://marketplace.visualstudio.com/items?itemName=streetsidesoftware.code-spell-checker) and [make it work with Solidity](https://twitter.com/abarbatei/status/1655492604736274435)
    - do not underestimate typos, they are akin to code how [canaries are to a coal mine](https://en.wiktionary.org/wiki/canary_in_a_coal_mine)

### **Export entire project documentation to a text file to be given as input to ChatGPT**

As AI technology becomes more powerful so do we must embrace and use it. Export the entire project documentation in an input file that can be feed to a ChatGPT session. This way you allow anyone to ask questions to it instead of you as developers.

### **Keep on being beautiful human beings**

_You can put a price on a Code4rena audit contest but you can't put a price on humanity_ (quote by ABA). The MaiaDAO team has gracefully and patiently interacted and answered each and every question the auditor community had for them. There were times where "Hi, I sent you a DM" was the only type of messages you saw in the discord channel but the team always answered. Thank you for being beautiful human beings and please keep being so!
