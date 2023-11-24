# Overview

Original contest issue and links: 

|#|Issue|Original link|
|-|:-|:-:|
| [M-01] | A malicious strategy can permanently DoS all currently pending withdrawals that contain it | [Link](https://github.com/code-423n4/2023-04-eigenlayer-findings/issues/132) |
| [QA] | Quality Assurance (grade-A) | [Link](https://github.com/code-423n4/2023-04-eigenlayer-findings/blob/main/data/ABA-Q.md) |

Typos may have been fixed and, the discussion part, was added at the end where applicable.

# Medium Risk Findings (1)

# [M-01] A malicious strategy can permanently DoS all currently pending withdraws that contain it

## Impact

A malicious strategy can permanently DoS all currently pending withdrawals that contain it. 

### Vulnerability details

In order to withdrawal funds from the project a user has to:
1. queue a withdrawal (via `queueWithdrawal`)
2. complete a withdrawal (via `completeQueuedWithdrawal(s)`)

Queuing a withdrawal, via `queueWithdrawal` modifies all internal accounting to reflect this:

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L345-L346
```Solidity
        // modify delegated shares accordingly, if applicable
        delegation.decreaseDelegatedShares(msg.sender, strategies, shares);
```

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L370
```Solidity
    if (_removeShares(msg.sender, strategyIndexes[strategyIndexIndex], strategies[i], shares[i])) {
```

and saves the withdrawl hash in order to be used in `completeQueuedWithdrawal(s)`

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L400-L415
```Solidity
            queuedWithdrawal = QueuedWithdrawal({
                strategies: strategies,
                shares: shares,
                depositor: msg.sender,
                withdrawerAndNonce: withdrawerAndNonce,
                withdrawalStartBlock: uint32(block.number),
                delegatedAddress: delegatedAddress
            });


        }


        // calculate the withdrawal root
        bytes32 withdrawalRoot = calculateWithdrawalRoot(queuedWithdrawal);


        // mark withdrawal as pending
        withdrawalRootPending[withdrawalRoot] = true;
```

In other words, it is final (as there is no `cancelWithdrawl` mechanism implemented).

When executing`completeQueuedWithdrawal(s)`, the `withdraw` function of the strategy is called (if `receiveAsTokens` is set to true).

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L786-L789
```Solidity
    // tell the strategy to send the appropriate amount of funds to the depositor
    queuedWithdrawal.strategies[i].withdraw(
        msg.sender, tokens[i], queuedWithdrawal.shares[i]
    );
```

In this case, a malicious strategy can always revert, blocking the user from retrieving his tokens.

If a user sets `receiveAsTokens` to `false`, the other case, then the tokens will be added as shares to the delegation contract

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L797-L800
```Solidity
    for (uint256 i = 0; i < strategiesLength;) {
        _addShares(msg.sender, queuedWithdrawal.strategies[i], queuedWithdrawal.shares[i]);
        unchecked {
            ++i;
```

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L646-L647
```Solidity
        // if applicable, increase delegated shares accordingly
        delegation.increaseDelegatedShares(depositor, strategy, shares);
```

This still poses a problem because the `increaseDelegatedShares` function's counterpart, `decreaseDelegatedShares` is only callable by the strategy manager (so as for users to indirectly get theier rewards worth back via this workaround)

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/DelegationManager.sol#L168-L179
```Solidity
    /**
     * @notice Decreases the `staker`'s delegated shares in each entry of `strategies` by its respective `shares[i]`, typically called when the staker withdraws from EigenLayer
     * @dev Callable only by the StrategyManager
     */
    function decreaseDelegatedShares(
        address staker,
        IStrategy[] calldata strategies,
        uint256[] calldata shares
    )
        external
        onlyStrategyManager
    {
```

but in `StrategyManager` there are only 3 cases where this is called, none of which is beneficial for the user:
 - [`recordOvercommittedBeaconChainETH`](https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L205) - slashes user rewards in certain conditions
 - [`queueWithdrawal`](https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L346) - accounting side effects have already been done, will fail in `_removeShares`
 - [`slashShares`](https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L346) - slashes user rewards in certain conditions


In other words, the workaround (of setting `receiveAsTokens` to `false` in `completeQueuedWithdrawal(s)`) just leaves the funds accounted and stuck in `DelegationManager`.

In both cases all shares/tokens associated with other strategies in the pending withdrawal are permanently blocked.

## Proof of Concept

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L786-L789

## Tools Used

Manual analysis

## Recommended Mitigation Steps

Add a `cancelQueuedWithdrawl` function that will undo the accounting and cancel out any dependency on the malicious strategy. Although this solution may create a front-running opportunity for when their withdrawal will be slashed via `slashQueuedWithdrawal`, there may exist workarounds to this.

Another possibility is to implement a similar mechanism to how `slashQueuedWithdrawal` treats malicious strategies: adding a list of strategy indices to skip

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L532-L535
https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L560-L566
```Solidity
     * @param indicesToSkip Optional input parameter -- indices in the `strategies` array to skip (i.e. not call the 'withdraw' function on). This input exists
     * so that, e.g., if the slashed QueuedWithdrawal contains a malicious strategy in the `strategies` array which always reverts on calls to its 'withdraw' function,
     * then the malicious strategy can be skipped (with the shares in effect "burned"), while the non-malicious strategies are still called as normal.
     */
     
     ...

        for (uint256 i = 0; i < strategiesLength;) {
            // check if the index i matches one of the indices specified in the `indicesToSkip` array
            if (indicesToSkipIndex < indicesToSkip.length && indicesToSkip[indicesToSkipIndex] == i) {
                unchecked {
                    ++indicesToSkipIndex;
                }
            } else {     
```

## Discussions

(last 2 messages)

**ChaoticWalrus (EigenLayer) commented:**

It seems like the worst case scenario would be having to re-queue without the missing strategy which would require waiting for the withdrawal to unlock

Yes, agreed this is the impact + worst-case scenario. The existing functionality was designed, in part at least, to address concerns about malicious Strategies.

So the question left to answer is whether an additional wait period is acceptable

I suppose so, yes. We have deemed this acceptable ourselves, but I could see it being viewed differently. Regardless, the impact here is orders of magnitude less than the original claimed impact.

**Alex the Entreprenerd (judge) commented:**

The Warden has shown how, due to the possibility of queueing a withdrawal with multiple strategies, in the case in which one of the strategies stops working (reverts, paused, malicious), the withdrawal would be denied.

As the sponsor said, in those scenarios, the withdrawer would have to perform a second withdrawal, which would have to be re-queued.

Intuitively, a withdrawer that always withdraws a separate strategy would also never see their withdrawal denied (except for the malicious strategy).

As we can see, there are plenty of side steps to the risk shown. However, the functionality of the function is denied, even if temporarily, leading to it not working as intended, and for this reason I believe Medium Severity to be the most appropriate.

As shown above, the finding could be fixed. However, it seems to me like most users would want to plan around the scenarios described in this finding and its duplicates.

# QA report

## Overview
During the audit, 1 low, 4 non-critical and 3 refactoring issues were found.

### Low Risk Issues

Total: 1 instances over 1 issues

|#|Issue|Instances|
|-|:-|:-:|
| [L-01] | `StrategyManager._removeShares` does not check for 0 address; reverts with misleading message | 1 |

### Refactoring Issues

Total: 5 instances over 3 issues

|#|Issue|Instances|
|-|:-|:-:|
| [R-01]| Reuse `userShares` in `StrategyManager.recordOvercommittedBeaconChainETH` calculation | 1 |
| [R-02]| Use already defined `PAUSE_ALL` instead of `type(uint256).max` | 3 |
| [R-03]| Add `createIfNotExisting` flag on `EigenPodManager.stake` | 1 |

### Non-critical Issues

Total: 8 instances over 4 issues

|#|Issue|Instances|
|-|:-|:-:|
| [NC-01]| If `amount` is 0 the executions passes through in `StrateguManager.recordOvercommittedBeaconChainETH` | 1 |
| [NC-02]| Check that argument `_pauserRegistry` from `Pausable._initializePauser` is a contract | 1 |
| [NC-03]| Remove redundant functions that wrap over other functions from `StrategyBase` | 3 |
| [NC-04]| Misleading or incorrect documentation | 3 |

#
## Low Risk Issues (1)
#

### [L-01] `StrategyManager._removeShares` does not check for 0 address; reverts with misleading message
##### Description

Wehn withdrawing from the project, execution flow reaches `StrategyManager._removeShares`. 
This function takes an strategy object (address) as one of its parameter but it does not check if it is 0.

```Solidity
        // sanity checks on inputs
        require(depositor != address(0), "StrategyManager._removeShares: depositor cannot be zero address");
        require(shareAmount != 0, "StrategyManager._removeShares: shareAmount should not be zero!");
```

Execution will revert but only because of the 0 address strategy not being found in `_removeStrategyFromStakerStrategyList`

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L736
```Solidity
            require(j != stratsLength, "StrategyManager._removeStrategyFromStakerStrategyList: strategy not found");
```

`_removeShares` expects callers to check the address but not all flows do check this. Examples
    - `queueWithdrawal` -> **_removeShares**
    - `slashShares` -> **_removeShares**

As such, if a user mistakenly adds the 0 address in the list of strategies to withdraw from, when calling queueWithdrawal, or the admin when calling slashShares, 
then the misleading _strategy not found_ revert message will be seen.

##### Instances (1)

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L683-L685

##### Recommendation

Add 0 address check in `_removeShares`.

#

#### Refactoring Issues (3)
#

### [R-01] Reuse `userShares` in `StrategyManager.recordOvercommittedBeaconChainETH` calculation
##### Description

In `StrategyManager` at line 191 debt is calculate as so:

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L191
```Solidity
    uint256 debt = amount - userShares;
```

and at line 193 `amount` is then set to its new value:

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L193
```Solidity
    amount -= debt;
```

There is a redundant subtraction here that can be changed to

```Solidity
    amount = userShares;
```

##### Instances (1)

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L191-L193

##### Recommendation

Change line 193 to:

```Solidity
    amount = userShares;
```
#

### [R-02] Use already defined `PAUSE_ALL` instead of `type(uint256).max`
##### Description

The `Pausable` contract declares a `PAUSE_ALL` constant with the value `type(uint256).max`

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/permissions/Pausable.sol#L23
```Solidity
    uint256 constant internal PAUSE_ALL = type(uint256).max;
```

But further down in the contract, in the function `Pausable.pauseAll` that is used to pause all functionalities of a contract, it is not used.
```Solidity
    /**
     * @notice Alias for `pause(type(uint256).max)`.
     */
    function pauseAll() external onlyPauser {
        _paused = type(uint256).max;
        emit Paused(msg.sender, type(uint256).max);
    }
```

##### Instances (3)

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/permissions/Pausable.sol#L78-L84

##### Recommendation

Change `type(uint256).max)` to `PAUSE_ALL` in the contest of `Pausable.pauseAll`.

#

### [R-03] Add "createIfNotExisting" flag on `EigenPodManager.stake`
##### Description

In `EigenPodManager`, the function `stake` creates a pod if the one you want to stake in does not exists (as it's documented to do).
Although this is to be a QoL feature, when staking one does not consider naturally to also create the staking environment. 

Consider adding a flag that if true will create the pod if not existing.

##### Instances

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/pods/EigenPodManager.sol#L112

##### Recommendation

Add a flag that if true will create the pod if not existing, do not have this behavior as default.

#

## Non-critical Issues (4)

#

### [NC-01] If `amount` is 0 the executions passes through in `StrateguManager.recordOvercommittedBeaconChainETH`
##### Description

In `StrateguManager.recordOvercommittedBeaconChainETH` there is a check that if amount is not 0 then `_removeShares` should not be called.

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L196-L198
```
    if (amount != 0) {
        _removeShares(overcommittedPodOwner, beaconChainETHStrategyIndex, beaconChainETHStrategy, amount);            
    }
```

This check should include all the leftover code as it would call a `decreaseDelegatedShares` with an empty amount to withdrawal.

##### Instances (1)

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L196-L198

##### Recommendation

Add all the code after the check in the if block.

#

### [NC-02] Check that argument `_pauserRegistry` from `Pausable._initializePauser` is a contract
##### Description

The currently done checks in the function `Pausable._initializePauser` for the `_pauserRegistry` do not include a contract check.
As such, an EOA can be set.

```Solidity
    function _initializePauser(IPauserRegistry _pauserRegistry, uint256 initPausedStatus) internal {
        require(
            address(pauserRegistry) == address(0) && address(_pauserRegistry) != address(0),
            "Pausable._initializePauser: _initializePauser() can only be called once"
        );
```

##### Instances (1)

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/permissions/Pausable.sol#L55-L59

##### Recommendation

Add a check that `_pauserRegistry` should be a contract.

#

### [NC-03] Remove redundant functions that wrap over other functions from `StrategyBase`
##### Description

In `StrategyBase` there are several functions where the documentation indicates

`* @notice In contrast to <next_function>, this function **may** make state modifications`

while simply they simply call the function that, in its turn indicatesthat it does not modify state changes.

```Solidity
    /**
     * @notice Used to convert a number of shares to the equivalent amount of underlying tokens for this strategy.
     * @notice In contrast to `sharesToUnderlying`, this function guarantees no state modifications
     * @param amountShares is the amount of shares to calculate its conversion into the underlying token
     * @dev Implementation for these functions in particular may vary signifcantly for different strategies
     */
    function sharesToUnderlyingView(uint256 amountShares) public view virtual override returns (uint256) {
        if (totalShares == 0) {
            return amountShares;
        } else {
            return (_tokenBalance() * amountShares) / totalShares;
        }
    }


    /**
     * @notice Used to convert a number of shares to the equivalent amount of underlying tokens for this strategy.
     * @notice In contrast to `sharesToUnderlyingView`, this function **may** make state modifications
     * @param amountShares is the amount of shares to calculate its conversion into the underlying token
     * @dev Implementation for these functions in particular may vary signifcantly for different strategies
     */
    function sharesToUnderlying(uint256 amountShares) public view virtual override returns (uint256) {
        return sharesToUnderlyingView(amountShares);
    }
```

All these functions are view so they cannot change state.

##### Instances (3)

https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/strategies/StrategyBase.sol#L166-L188
https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/strategies/StrategyBase.sol#L190-L213
https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/strategies/StrategyBase.sol#L215-L229

##### Recommendation

Leave only one example of each (the one without the `View` in the name, as this is not standard).

#

### [NC-04] Misleading or incorrect documentation
##### Description

There are places in the project where there is either incorrect or misleading code documentation (either via NatSpec or inline).
These need to be corrected

##### Instances (3)

- `StrategyManager.depositBeaconChainETH` declares the `amount` param 2 times, the second time being a leftover from a copy-paste ([code](https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L161))

- in `StrategyManager` functions `addStrategiesToDepositWhitelist` and `removeStrategiesFromDepositWhitelist` document that they are `Owner-only function` when in fact they are executable only by the strategy whitelister (`onlyStrategyWhitelister`) ([code](https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L591-L619))

- in `BeaconChainProofs.getBalanceFromBalanceRoot` remove incorrect "is" from documentation ([code](https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/libraries/BeaconChainProofs.sol#L172))

##### Recommendation

Resolve the mentioned issues

#
