# Overview

Original contest issue and links: 

|#|Issue|Original link|
|-|:-|:-:|
| [M-01] | Funding cycles that use `JBXBuybackDelegate` as a redeem data source (or any derived class) will slash all redeemable tokens | [Link]() |
| [QA] | Quality Assurance (grade-A) | [Link]() |

Typos may have been fixed and, the discussion part, was added at the end where applicable.

# Medium Risk Findings (1)

# [M-01] Funding cycles that use `JBXBuybackDelegate` as a redeem data source (or any derived class) will slash all redeemable tokens

## Impact

For a funding cycle that is set to use the data source for redeem and the data source is `JBXBuybackDelegate`, any and all redeemed tokens is 0 because of the returned 0 values from the empty `redeemParams` implementation. This means the beneficiary would wrongly receive 0 tokens instead of an intended amount.

While `JBXBuybackDelegate` was not designed to be used for redeem also, per protocol logic, this function should of been a pass-through (if no redeem such functionality was desired) because a redeem logic is by default used if not overwrite by `redeemParams`, as per [the documentation](https://docs.juicebox.money/dev/build/treasury-extensions/data-source/) (further detailed below).
Impact is further increased as the function is not marked as `virtual`, as such, no inheriting class can modify the faulty implementation if any such extension is desired.

## Vulnerability details

Project logic dictates that a contract can become a treasury data source by adhering to `IJBFundingCycleDataSource`, such as `JBXBuybackDelegate` is and
that there are two functions that must be implemented, `payParams(...)` and `redeemParams(...)`. 
Either one can be **left empty** if the intent is to only extend the treasury's pay functionality or redeem functionality.

But by left empty, it is specifically required to pass the input variables, not a 0 default value

https://docs.juicebox.money/dev/build/treasury-extensions/data-source/

> Return the JBRedeemParamsData.reclaimAmount value if no altered functionality is desired.
> Return the JBRedeemParamsData.memo value if no altered functionality is desired.
> Return the zero address if no additional functionality is desired.

What should of been done:

```Solidity
  // This is unused but needs to be included to fulfill IJBFundingCycleDataSource.
  function redeemParams(JBRedeemParamsData calldata _data)
    external
    pure
    override
    returns (
      uint256 reclaimAmount,
      string memory memo,
      IJBRedemptionDelegate delegate
    )
  {
    // Return the default values.
    return (_data.reclaimAmount.value, _data.memo, IJBRedemptionDelegate(address(0)));
  }
```

What is implemented:

https://github.com/code-423n4/2023-05-juicebox/blob/main/juice-buyback/contracts/JBXBuybackDelegate.sol#L235-L239
```Solidity
    function redeemParams(JBRedeemParamsData calldata _data)
        external
        override
        returns (uint256 reclaimAmount, string memory memo, JBRedemptionDelegateAllocation[] memory delegateAllocations)
    {}
```

While an overwritten `memo` with a empty default string is not such a large issue, and incidentally `delegateAllocations` is actually returned correctly as a zero address, 
the issue lies with the `reclaimAmount` amount which not is returned as 0.

Per documentation:

> `reclaimAmount` is the amount of tokens in the treasury that the terminal should send out to the redemption beneficiary as a result of burning the amount of project tokens tokens

and in case of redemptions, by default, the value is

> a reclaim amount determined by the standard protocol bonding curve based on the redemption rate the project has configured into its current funding cycle which is provided to the data source function in JBRedeemParamsData.reclaimAmount

but in this case, it will be overwritten with 0 thus the redemption beneficiary will be deprived of funds.

`redeemParams` is also lacking the `virtual` keyword, as such, no inheriting class can modify the faulty implementation if any such extension is desired.

## Tools Used

Manual analysis

## Recommended Mitigation Steps

Change the implementation of the function to respect protocol logic and add the `virtual` keyword, example:

```Solidity
  // This is unused but needs to be included to fulfill IJBFundingCycleDataSource.
  function redeemParams(JBRedeemParamsData calldata _data)
    external
    pure
    override
    virtual
    returns (
      uint256 reclaimAmount,
      string memory memo,
      IJBRedemptionDelegate delegate
    )
  {
    // Return the default values.
    return (_data.reclaimAmount.value, _data.memo, IJBRedemptionDelegate(address(0)));
  }
```

## Note for judging

Any user/developer that wishes to extend the tool cannot or, at worst, uses a faulty implementation by mistake. Historically, medium severity was awarded to various issues which rely on some user error (in this case using a faulty implementation because docs indicate that all non-redeemable data sources are either pass-through, reverts or custom implementation), are easy to check/fix and present material risk. I respect this line of thought and for the sake of consistency I believe this submission should be judged similarly. 


# QA report

## Overview
During the audit, 1 low, 2 refactoring and 5 non-critical issues were found.

### Low Risk Issues

Total: 1 instance over 1 issues

|#|Issue|Instances|
|-|:-|:-:|
| [L-01] | `JBXBuybackDelegate` deployer should decide if held fees should be refunded | 1 |

### Refactoring Issues

Total: 2 instances over 2 issues

|#|Issue|Instances|
|-|:-|:-:|
| [R-01]| Better and documentation naming for `_projectTokenIsZero`  | 1 |
| [R-02]| Use named function calls | 1 |

### Non-critical Issues

Total: 13 instances over 5 issues

|#|Issue|Instances|
|-|:-|:-:|
| [NC-01]| Use `JBTokenAmount::decimals` instead of hardcoding it in `payParams` | 1 |
| [NC-02]| Memo is not passed in all cases | 3 |
| [NC-03]| Missing, misleading, incorrect, or incomplete comments | 7 |
| [NC-04]| Events missing key information | 1 |
| [NC-05]| Empty methods should be thoroughly documented | 1 |

#
## Low Risk Issues (1)
#

### [L-01] `JBXBuybackDelegate` deployer should decide if held fees should be refunded
##### Description

When minting of tokens is done, via `JBXBuybackDelegate::_mint`, ETH is sent back to the terminal via a call to `addToBalanceOf`.

There are 2 versions of `addToBalanceOf`. One that [accepts a flag `_shouldRefundHeldFees` indicating if held fees should be refunded based on the amount being added](https://github.com/jbx-protocol/juice-contracts-v3/blob/main/contracts/abstract/JBPayoutRedemptionPaymentTerminal3_1.sol#L686-L696).

and other, with one less argument, that is used by our protocol that [sets it to false by default](https://github.com/jbx-protocol/juice-contracts-v3/blob/main/contracts/abstract/JBPayoutRedemptionPaymentTerminal3_1.sol#L559-L578).

This option should not be hardcoded as it is, a contract deployer should determine if he opts in or not.

##### Instances (1)

https://github.com/code-423n4/2023-05-juicebox/blob/main/juice-buyback/contracts/JBXBuybackDelegate.sol#L347-L350

##### Recommendation

Add a constructor parameter that is then used to with the `addToBalanceOf` when `_mint` is called.

#

#### Refactoring Issues (2)
#

### [R-01] Better and documentation naming for `_projectTokenIsZero` 
##### Description

A variable named `_projectTokenIsZero` is used to indicate if the project token address is less then the address terminal token by sort order.
It describes it simply as:
```
    /**
     * @notice Address project token < address terminal token ?
     */
    bool private immutable _projectTokenIsZero;
```
then it is used to identify which swap return amount is to be used.
```Solidity
            _amountReceived = uint256(-(_projectTokenIsZero ? amount0 : amount1));
```

This is needed because Uniswap `IUniswapV3Pool::pool` returns the token amounts corresponding to the token sorted addresses order. 
This is implicit because pools are created with the tokens as such ([IUniswapV3Factory#PoolCreated](https://docs.uniswap.org/contracts/v3/reference/core/interfaces/IUniswapV3Factory#poolcreated))

The naming and description can be changed to better describe the need for this flag variable.

##### Instances (1)

https://github.com/code-423n4/2023-05-juicebox/blob/main/juice-buyback/contracts/JBXBuybackDelegate.sol#L60-L63

##### Recommendation

Add a more descriptive name and documentation. Example:
```
    /**
     * @notice sores if: address project token < address terminal token
     * @dev used to identify which pair token (0 or 1) is the project token in a swap result
     */
    bool private immutable _projectTokenIsPairToken0;
```

#

### [R-02] Use named function calls
##### Description

Code base has an extensive use of named function calls, but it somehow missed one instance where this would be appropriate.

```Solidity
        jbxTerminal.addToBalanceOf{value: _data.amount.value}(
            _data.projectId, _data.amount.value, JBTokens.ETH, "", new bytes(0)
        );
```

([addToBalanceOf](https://github.com/jbx-protocol/juice-contracts-v3/blob/main/contracts/abstract/JBPayoutRedemptionPaymentTerminal3_1.sol#L569-L574))

##### Instances (1)

https://github.com/code-423n4/2023-05-juicebox/blob/main/juice-buyback/contracts/JBXBuybackDelegate.sol#L348-L350

##### Recommendation

Use named function calls on function call, as such:
```Solidity
    jbxTerminal.addToBalanceOf{value: _data.amount.value}({
        _projectId: _data.projectId,
        _amount: _data.amount.value,
        _token: JBTokens.ETH,
        _memo: "",
        _metadata: new bytes(0)
    });
```

#

## Non-critical Issues (5)
#

### [NC-01] Use `JBTokenAmount::decimals` instead of hardcoding it in `payParams`
##### Description

In `payParams` the number of tokens to mint is determined by multiplying value with weight and dividing by *1e18*.

```Solidity
        // Find the total number of tokens to mint, as a fixed point number with 18 decimals
        uint256 _tokenCount = PRBMath.mulDiv(_data.amount.value, _data.weight, 10 ** 18);
```

As the token to be use by the project has 18 decimals this is not an issue for now. Any other project token to be added will require this modified in order to keep the
premise that the total number of tokens to mint is a fixed point number with 18 decimals.

Out of the formula, `_data.weight` has 18 decimals

https://github.com/jbx-protocol/juice-contracts-v3/blob/12d852f28d372dd44987586f8009c56b0fe247a9/contracts/JBSingleTokenPaymentTerminalStore3_1.sol#L351-L352
```Solidity
    // The weight according to which new token supply is to be minted, as a fixed point number with 18 decimals.
    uint256 _weight;
```

so the the 1e18 must be the decimal count of the `JBTokenAmount` in order to be fully safe.

##### Instances (1)

https://github.com/code-423n4/2023-05-juicebox/blob/main/juice-buyback/contracts/JBXBuybackDelegate.sol#L149-L150

##### Recommendation

Use the decimals propriety of `JBTokenAmount` instead of a hardcoded *1e18*:
```Solidity
        uint256 _tokenCount = PRBMath.mulDiv(_data.amount.value, _data.weight, 10 ** _data.amount.decimals);
```

#

### [NC-02] Memo is not passed in all cases
##### Description

All controller operations of `mintTokensOf` and `burnTokensOf` or terminal operation of `addToBalanceOf` have a `_memo` parameter. 
The information in the memo will be passed to an event. The event is sent regardless so adding the memo will make off-chain analytics clearer.

##### Instances (3)

https://github.com/code-423n4/2023-05-juicebox/blob/main/juice-buyback/contracts/JBXBuybackDelegate.sol#L297
https://github.com/code-423n4/2023-05-juicebox/blob/main/juice-buyback/contracts/JBXBuybackDelegate.sol#L319
https://github.com/code-423n4/2023-05-juicebox/blob/main/juice-buyback/contracts/JBXBuybackDelegate.sol#L349 

##### Recommendation

Pass the memo information to the mentioned calls

#

### [NC-03] Missing, misleading, incorrect, or incomplete comments
##### Description
There are cases where the comments are either missing, incomplete or incorrect throughout the codebase.

##### Instances (7)

- `_swap`: 
    - at line 246 *toke_beforeTransferTon* should be *token_beforeTransferToken* ([code](https://github.com/code-423n4/2023-05-juicebox/blob/main/juice-buyback/contracts/JBXBuybackDelegate.sol#L246))
    - at line 251 *fc* should be properly described as *funding cycle* ([code](https://github.com/code-423n4/2023-05-juicebox/blob/main/juice-buyback/contracts/JBXBuybackDelegate.sol#L251))
    - at line 301 the comments after `result: ` are ambiguous `_amountReceived-reserved here, reservedToken in reserve`, consider making them more clear, example: `(_amountReceived - reserved) here, (reserved (token)) in reserve` ([code](https://github.com/code-423n4/2023-05-juicebox/blob/main/juice-buyback/contracts/JBXBuybackDelegate.sol#L301))
    - at lines 250 and 252 *ie* should be *i.e.* ([code](https://github.com/code-423n4/2023-05-juicebox/blob/main/juice-buyback/contracts/JBXBuybackDelegate.sol#L250-L252))
- `_mint`: at line 329 there is a double *in in*, remove one *in* ([code](https://github.com/code-423n4/2023-05-juicebox/blob/main/juice-buyback/contracts/JBXBuybackDelegate.sol#L329))
- `uniswapV3SwapCallback`: 
    - at line 214 typo in `controle`, an extra *e*, remove the extra *e* ([code](https://github.com/code-423n4/2023-05-juicebox/blob/main/juice-buyback/contracts/JBXBuybackDelegate.sol#L214))
    - at line 212 typo, "should happen" or rephrase to "happens" ([code](https://github.com/code-423n4/2023-05-juicebox/blob/main/juice-buyback/contracts/JBXBuybackDelegate.sol#L212))

##### Recommendation

Resolve the mentioned issues.

#

### [NC-04] Events missing key information
##### Description

Some events are missing key information when emitted.

##### Instances (1)

- in `_mint` the `JBXBuybackDelegate_Mint` event should contain the minted amount and beneficiary to whom it was minted ([code](https://github.com/code-423n4/2023-05-juicebox/blob/main/juice-buyback/contracts/JBXBuybackDelegate.sol#L352))

##### Recommendation

Add the missing information.

#

### [NC-05] Empty methods should be thoroughly documented
##### Description

The function `redeemParams` variable has no code implementation. 
It is both lacking any NatSpec and any internal comments to indicate what is its purpose.

##### Instances (1)

https://github.com/code-423n4/2023-05-juicebox/blob/main/juice-buyback/contracts/JBXBuybackDelegate.sol#L235-L239

##### Recommendation

Properly document the method, why it is needed and why it is empty.

#