# Overview

Original contest issue and links: 

|#|Issue|Original link|
|-|:-|:-:|
| [H-01] | Rounding error in LP withdrawLP function leads to infinite rewards extraction without the need to have staked BYTES LP | [#165](https://github.com/code-423n4/2023-03-neotokyo-findings/issues/165) |
| [H-02] | Users reward withdraws may be frontrun, leading to reward loss; not claiming frequently leads to reward loss | [#185](https://github.com/code-423n4/2023-03-neotokyo-findings/issues/185) |
| [QA] | Quality Assurance (grade-b) | [#115](https://github.com/code-423n4/2023-03-neotokyo-findings/issues/115) + [#166](https://github.com/code-423n4/2023-03-neotokyo-findings/issues/166) |

Typos may have been fixed and, the discussion part, was added at the end where applicable.

# High Risk Findings (2)

# [H-01] Rounding error in LP withdrawLP function leads to infinite rewards extraction without the need to have staked BYTES LP

## Impact

When a user withdraws his LP, if the amount is of LP is smaller then 0.01 LP, then the LP is withdrawn but the bonus points still remain for user and for the pool as a whole.
This breaks internal accounting for both the pool and the user, both of witch will permanently have the excess bonus points.

By always having an excess of points via BYTES LP staking, even if actually nothing is staked, a malicious user can permanently extract rewards from the protocol as long as it exists.

For transparency, for this attack the economical incentive of an attacker is highly dependent on the price of the protocol assets and gas fees
- 3 transactions are required at minimum to set up the issue: 1 deposit and a minimum of 2 withdrawals to extract all the LPs (a detailed description follows as to why)
- the maximum LP amount that can be withdrawn this way is less then 0.01 LP. This means the higher amount of LP tokens initially staked (minimum of 0.01) the more operations are required to remove them
- the protocol is launched on Ethereum, meaning gas costs are a constant profit impediment.
- the BYTES bonus incentive is dependent on the number of staked basis points

All of the above operations are dependent indirectly on the price of `BYTESv2`, `ETH` (directly on the `BYTESv2-ETH-LP` tokens) and gas values at operation times.

In the right circumstances, this attack can become viable viable, as such the auditor feels that this issue is of a medium severity.

### Detailed description
The calculated bonus points to be deducted from the user, in a LP withdrawal, is calculated 

https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L1623
```Solidity
    uint256 points = amount * 100 / 1e18 * lpPosition.multiplier / _DIVISOR;
```
The above `points` can thus become 0 due to a rounding error, if the amount is small enough (amount < 1e16 (0.01)). With `points` as 0, the following internal accounting is broken:

https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L1625-L1630
```Solidity
    // Update the caller's LP token stake.
    lpPosition.amount -= amount;
    lpPosition.points -= points;


    // Update the pool point weights for rewards.
    pool.totalPoints -= points;
```

- `lpPosition.points` the user bonus points is not decreased, and will never be able to be decreased
- `pool.totalPoints` the overall pool points are also, permanently offset


### Exemplifying how an attack would be executed

- For the above calculation, the multiplier is the timelock multiplier, set when staking LPs:

https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L1144-L1145
```Solidity
    if (stakerLPPosition[msg.sender].multiplier == 0) {
        stakerLPPosition[msg.sender].multiplier = timelockMultiplier;
```
which will have one of the following values, depending on the locked time:
- for 30 days: 100
- for 90 days: 125
- for 180 days: 150
- for 360 days: 200
- for 720 days: 300

and where `_DIVISOR` is 100

https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L199-L200
```Solidity
    /// A constant divisor to calculate points and multipliers as basis points.
    uint256 constant private _DIVISOR = 100;
```

Also, when staking, the same calculation is used to determine the points, 

https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L1155

so for an initial deposit, user would need to deposit a minimum 0.01 LP. Example staking for 30 days and getting 1 basis point:
```
uint256 points = 1 * 1e16 * 100 / 1e18 * 100 / 100
               = 1 * 1e18 / 1e18 * 100 / 100
               = 1 * 100 / 100
               = 1 share basis point
```

The user then would proceed to withdrawal his LPs in N transactions but with values lower then 0.01 LP. Example: 0.009 and 0.001 LP.

```
uint256 points = 9 * 1e15 * 100 / 1e18 * 100 / 100
               = 9 * 1e17 / 1e18 * 100 / 100
               = 9 / 10 * 100 / 100
               = 0 * 100 / 100
               = 0
```
no points will be removed. User now has 1 permanent basis point that will generate yield as time passes.


## Proof of Concept

The following should have been a valid POC. Unfortunately, the assertation fails as calling `withdraw` has no side effect regardless of amount sent. There seems to be a fault in the project testing scripts which was failed be to identified in time due to audit time constraints.

POC can be added to `NeoTokyoStaker.test.js`, before test on line 561

https://github.com/code-423n4/2023-03-neotokyo/blob/main/test/NeoTokyoStaker.test.js#L561

```Solidity
    it('rounding error in withdrawLP ', async function () {
        
        const bobInitialLpBalance = await LPToken.balanceOf(bob.address);
        const initialStakeLPPosition = await NTStaking.stakerLPPosition(bob.address);

        // confirm that bob has no other LP staked for points
        initialStakeLPPosition.points.should.be.equal(0);

        // Get the time at which Bob staked.
        const priorBlockNumber = await ethers.provider.getBlockNumber();
        const priorBlock = await ethers.provider.getBlock(priorBlockNumber);
        const bobStakeTime = priorBlock.timestamp;
        
        // Stake minimum amount
        await NTStaking.connect(bob.signer).stake(
            ASSETS.LP.id,
            TIMELOCK_OPTION_IDS['30'],
            ethers.utils.parseEther('0.01'),
            0,
            0
        );

        // Jump to 30 days after Bob's stake.
        await ethers.provider.send('evm_setNextBlockTimestamp', [
            bobStakeTime + (60 * 60 * 24 * 30)
        ]);

        // // unstake amount below tolerance line 
        await NTStaking.connect(bob.signer).withdraw(
            ASSETS.LP.id,
            ethers.utils.parseEther('0.005')
        );

        // unstake amount below tolerance line
        await NTStaking.connect(bob.signer).withdraw(
            ASSETS.LP.id,
            ethers.utils.parseEther('0.005')
        );
        
        // get current state
        const currentStakeLPPosition =  await NTStaking.stakerLPPosition(bob.address);
        const bobCurrentlLpBalance = await LPToken.balanceOf(bob.address); 
        
        // observe that LP stake points have increased and bob has the same LP as before, gaining 1 share point forever
        currentStakeLPPosition.points.should.be.equal(initialStakeLPPosition.points.add(1));
        bobCurrentlLpBalance.should.be.equal(bobInitialLpBalance);

    });
```

## Tools Used

Manual analysis

## Recommended Mitigation Steps

2 suggested solutions:
- increase the precision of the bonus points (similar to how `shares` are guarded by `_PRECISION` constant)
- check if `points` is 0 after each calculation in stakeLP/unstakeLP and raise an error in that case


# [H-02] Users reward withdraws may be frontrun, leading to reward loss; not claiming frequently leads to reward loss

## Impact

There is an issue with this reward collecting mechanism related to timing. If a user has not claimed in a a long period and wishes to do so now, an ill intended attacker can frontron the `getReward` operation and deposit into the pool. This increases the total amount of points in the pool thus giving a lower then expected value to user. The less then expected value is relative to what the user would have gained if he would have regularly claimed rewards.

The attacker gains nothing in this, also he is further required to actually stake the assets, for a minimum of 30 days also. This however does cause damage to the user and loss of rewards. It is also very easy for an attack to execute it.

To further generalize this issue, the current reward distribution model requires frequent calls to `getReward` or risk losing entitled rewards. 

Having to frequently claim rewards is a sever gas consumption, as the protocol is on Ethereum, but not claiming (because of rewards being distributed only when `getReward` is called) leads to less rewards then expected. 

### Detailed description

Rewards are distributed to users by calling `getReward` from `BYTES2.sol`. This function further calls `claimReward` from `NeoTokyoStaker` to determin the user `BYTES` amount to be minted and the DAO's.

`claimReward` flows execution to `getPoolReward` where the amount is determined as:

https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L1388-L1392
```Solidity
    uint256 share = points * _PRECISION / pool.totalPoints * totalReward;
    uint256 daoShare = share * pool.daoTax / (100 * _DIVISOR);
    share /= _PRECISION;
    daoShare /= _PRECISION;
    return ((share - daoShare), daoShare);
```

we can see that user `BYTES` shares also depend on the `pool.totalPoints` size of the specific asset pool. The higher the value, the less `BYTES` are due, which is normal for a staking protocol. After each reward distribution, a time variable is also set, to mark the last user reward date.

Since rewards are only distributed when `getReward` is called, and the calculation takes into consideration only the state of the system at the current time, not how it passed, this leads to unexpectedly less tokens.

## Proof of Concept

An example scenario:
- `alice` deposits a substantial amount of assets and has 30% of an asset pool
- `alice` does not do any further deposits, rewards or calls the `getReward` function, as she is comfortable with her position
- the pool sees no new increase in funds for a long period of time, example 60 days
- at one point, `bob` sees this as an opportunity and invests so much in the pool that `alice`'s stake becomes 15% of the pool, at day 61
- `alice` wishes to get her rewards as day 62 and, logically one should expect 60 days worth of 30% pool ownership plus 2 days of 15% pool ownership.
- instead `alice` gets 62 days worth of 15% pool ownership


## Tools Used

Manual analysis

## Recommended Mitigation Steps

This is an issue by design, needing a redesign to save system states and calculate the rewards a user would be entitled to based on the system state history. Technically:
- at each deposit and withdrawal save the snapshot of entire system `pool.totalPoints` 
- also save the total points for each recipient. The output of:
```Solidity
    for (uint256 i; i < _stakerS1Position[_recipient].length; ) {
        uint256 citizenId = _stakerS1Position[_recipient][i];
        StakedS1Citizen memory s1Citizen = stakedS1[_recipient][citizenId];
        unchecked {
            points += s1Citizen.points;
            i++;
        }
    }
```
- when calculating rewards, cross reference what user had staked, to what was the system total point at that time and correlated it with the reward windows.

User can mitigate this issue to some extent by frequently calling the `getReward` function. This workaround incurs high gas costs over a long period of time and would only partial resolve the issue at the expense of the user.


# Quality Assurance (grade-b)

## Overview
During the audit, 2 non-critical and 2 refactoring issues were found.

### Non-critical Issues

Total: 2 instances over 1 issues

|#|Issue|Instances|
|-|:-|:-:|
| [NC-01]| Enum AssetType range validations are incorect | 2 |
| [NC-02]| No support for Uniswap V3 LP staking when BYTESv1 uses a V3 pool | 1 |

### Refactoring Issues

Total: 5 instances over 2 issues

|#|Issue|Instances|
|-|:-|:-:|
| [R-01]| Enum AssetType range validations are redundant | 2 |
| [R-02]| Use constants for values with special meaning | 3 |

#


## Non-critical Issues (2)
#

### [NC-01] Enum AssetType range validations are incorect
##### Description

`AssetType` enum is defined as:
https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L266-L271
```Solidity
    enum AssetType {
        S1_CITIZEN, // uint8 value: 0
        S2_CITIZEN, // uint8 value: 1
        BYTES,      // uint8 value: 2
        LP          // uint8 value: 3
    }
```
There are several places in code, where this is an input for external functions and it is checked, incorrectly, to be in the Enum value range:
```Solidity
        if (uint8(_assetType) > 4) {
            revert InvalidAssetType(uint256(_assetType));
        }
```

In Solidity, Enums start from 0: https://docs.soliditylang.org/en/v0.8.19/types.html#enums
The check `uint8(_assetType) > 4)` should be modified to `uint8(_assetType) > 3)` as it allowes the unmapped value 4 to be considered valid.

This does not impact code as there are other checks that invalidate this issue (also see `[R-01]`)

##### Instances (2)

https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L1196-L1207

https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L1659-L1670

##### Recommendation

Modify `uint8(_assetType) > 4)` to `uint8(_assetType) > 3)` or simply remove the check (as suggested in `[R-01]`)


### [NC-02] No support for Uniswap V3 LP staking when BYTESv1 uses a V3 pool
##### Description

The `BYTESv2` staking contract is designed to use Uniswap V2 compatible LP tokens only and doest not support V3 LP staking (ERC721 LP tokens).
Normaly, this would not be mentioned as an issue, but the current `BYTESv1` liquidity is set in a Uniswap V3 LP pool. 
These types of features are usually kept as they are at protocol level.

##### Instances (1)

https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol

##### Recommendation

Reconsider if downgrading to a V2 LP staking is what it is best for the protocol in terms of consistency.
Document the decision.

#
## Refactoring Issues (2)
#

### [R-01] Enum AssetType range validations are redundant
##### Description

`AssetType` enum is defined as:
https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L266-L271
```Solidity
    enum AssetType {
        S1_CITIZEN, // uint8 value: 0
        S2_CITIZEN, // uint8 value: 1
        BYTES,      // uint8 value: 2
        LP          // uint8 value: 3
    }
```
There are several places in code, where this is an input for external functions and it is checked (incorrectly) to be in a specific range.
Example:
https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L1204-L1207
```Solidity
        // Validate that the asset being staked is of a valid type.
        if (uint8(_assetType) > 4) {
            revert InvalidAssetType(uint256(_assetType));
        }
```
As the value is passed as an argument in these cases

https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L1196-L1197
```Solidity
    function stake (
        AssetType _assetType,
        // ...
```

the Solidity compiler already validates that any passed argument is a valid Enum value by doing an explicit cast. If it is not, it will revert via a Panic error:

https://docs.soliditylang.org/en/v0.8.19/types.html#enums

> The explicit conversion from integer checks at runtime that the value lies inside the range of the enum and causes a Panic error otherwise.

A contradictory observation is that, according to best practices

https://docs.soliditylang.org/en/v0.8.19/control-structures.html#panic-via-assert-and-error-via-require

> Properly functioning code should never create a Panic, not even on invalid external input.

_Creating a panic or not on invalid code is outside of the scope of this issue, it is left for the developer to decide._

##### Instances (2)

https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L1196-L1207

https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L1659-L1670

In this second case, the `uint8(_assetType) > 4)` is redundant.

##### Recommendation

Remove redundant check

#

### [R-02] Use constants for values with special meaning
##### Description

Use constants for values with special meaning when used in code

##### Instances (3)

https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L950
String value may be set as a const variable

https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L1022
https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L946

Consider using a const for "DEFAULT_VAULT_VAULE_MULTIPLIER = 100"

##### Recommendation

Save values as a const variables and use them as so.

# Downgraded issues

## BYTES bonus staking points mechanism rounds bonus to 0, silently, if less then 2 BYTES are staked

## Impact

When staking BYTES for bonus points, due to a rounding error, the minimum amount required to actually increase the bonus points is 2 BYTES. 
If less then this is provided, the stake succeeds but does not increase the bonus poinsts. User is also unawear that he has gained nothing while having his staked tokens locked until the minimum withdrawl window for his staked citizens is reached.

For transparency, the impact at current price levels is neglijabel (current price for BYTESv1 is \$3.1) and would only impact users that would stake less then 6$ worth of `BYTESc1` (for comparrison), practially no one.

But the issue becomes more apparent and sever as price increase:
- `BYTESv1`, at its ATH, was around 200 USD ([Uniswap V3: BYTES](https://dexscreener.com/ethereum/0x3782a3425cd093d5cd0c5b684be72641e199029c))
- `BYTESv2` will be the main token of the entire *Neo Tokyo (NT)* ecosystem and has good change at surpassing this value
- for a `BYTESv2` token value of 200 USD this would translate in anybody staking less then 400$ worth will have their funds locked up without any actual bonus
- user is also not aware of this, as the staking succeeds and the emited event `Stake` only contains the amount staked, not the bonus increased
    - a user can, however, determine this if he retrieved the `stakedS1[msg.sender][citizenId].points/stakedS2[msg.sender][citizenId].points` value before and after staking, and then compares them to see the increase. This is not something a user will do in a normal execution flow 

### Detailed description

After a user has staked either an S1 or S2, then they can also stake BYTES in association to the staked citizen(s). 
Staking BYTES starts from the external `stake` function and continues with the `_stakeBytes` function.

https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L1196
https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L1044-L1054

After some input validation on provided `amount`, `citizenId` and `seasonId`, the bonus points allocated for the stake is calculated 

(for S1) https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L1077-L1080

(for S2) https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L1098-L1101

in the same way:
```Solidity
    uint256 bonusPoints = (amount * 100 / _BYTES_PER_POINT);
    citizenStatus.stakedBytes += amount;
    citizenStatus.points += bonusPoints;
    pool.totalPoints += bonusPoints;
```

where `_BYTES_PER_POINT` is a constant declared as: 

https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L202-L203
```Solidity
    /// The number of BYTES needed to get one point in BYTES staking calculations.
    uint256 constant private _BYTES_PER_POINT = 200 * 1e18;
```

`BYTESv2` will have 18 decimals and passed `amount`, in the above flow, is in WEI, such that the calculation can be reduced to:

```
bonusPoints = full_amount * 1e18 * 100 / (200 * 1e18)
            = full_amount / 2
            => rounds to 0 if full_amount is less then 2 (BYTES)
```

It can also be observed that even if `bonusPoints` is 0, the staked token amount is increased (which is good, the staked BYTES are not lost).
```Solidity
    citizenStatus.stakedBytes += amount;
```
and an event, signaling a succesful stake, is emited containing the following information:

https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L1109-L1115

```Solidity
    event Stake (
        address indexed staker, // The address of the caller who staked an `asset`.
        address indexed asset,  // The address of the asset being staked.
        uint256 timelockOption, // Data encoding the parameters surrounding the timelock options used, a standing in this case
        uint256 amountOrTokenId // The amount of `asset` staked
    );
```

Staked BYTES can only be retrieved when a timelock period ends for the associated staked citizen NFT and the `withdrawl` function is succesfully called.

## Proof of Concept

POC can be added to `NeoTokyoStaker.test.js`, before test on line 561

https://github.com/code-423n4/2023-03-neotokyo/blob/main/test/NeoTokyoStaker.test.js#L561

```Solidity
        it('rounding error leads to no reward points in BYTES staking', async function () {

            // Bob stakes his S1 Citizen.
            await NTStaking.connect(bob.signer).stake(
                ASSETS.S1_CITIZEN.id,
                TIMELOCK_OPTION_IDS['30'],
                citizenId1,
                0,
                0
            );

            // Activate the ability to stake BYTES into Citizens.
            await NTStaking.connect(owner.signer).configurePools(
                [
                    {
                        assetType: ASSETS.BYTES.id,
                        daoTax: 0,
                        rewardWindows: [
                            {
                                startTime: 0,
                                reward: 0
                            }
                        ]
                    }
                ]
            );

            // Bob stakes less then 2 BYTES into his S1 Citizen.
            const lessThenMinimumRequiredAmountToStake = ethers.utils.parseEther('1.999');
            const stakedStatusBefore = await NTStaking.stakedS1(bob.address, citizenId1);
            await NTStaking.connect(bob.signer).stake(
                    ASSETS.BYTES.id,
                    TIMELOCK_OPTION_IDS['30'],
                    lessThenMinimumRequiredAmountToStake,
                    citizenId1,
                    1
                );
            const stakedStatusAfter = await NTStaking.stakedS1(bob.address, citizenId1);

            // No change in bonus points after the stake
            stakedStatusBefore.points.should.be.equal(stakedStatusAfter.points);
            // But a change in stakedBytes (this is good)
            stakedStatusAfter.stakedBytes.should.be.equal(stakedStatusBefore.stakedBytes.add(lessThenMinimumRequiredAmountToStake));

            // if Bob stakes the minimum required
            await NTStaking.connect(bob.signer).stake(
                    ASSETS.BYTES.id,
                    TIMELOCK_OPTION_IDS['30'],
                    ethers.utils.parseEther('2'), // 2 BYTES minimum stake to overcome rounding error
                    citizenId1,
                    1
                );
            const stakedStatusAfterMinimum = await NTStaking.stakedS1(bob.address, citizenId1);

            // then we see an increase in points bases
            stakedStatusAfterMinimum.points.should.be.equal(stakedStatusBefore.points.add(1));
        });
```

## Tools Used

Manual analysis

## Recommended Mitigation Steps

2 sugested solutions:
- increase the precision of the bonus points (similar to how `shares` are guarded by `_PRECISION` constant)
- check if `bonusPoints` is 0 after each stake and raise an error in that case