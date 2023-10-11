# Overview

Original contest issue and links: 

|#|Issue|Original link|
|-|:-|:-:|
| [M&#x2011;01] | Auction can be force started and first token force minted by calling `settle()` before the auction was launched | [#83](https://github.com/sherlock-audit/2023-02-fair-funding-judging/issues/83) |
| [M&#x2011;02] | Migration logic is implemented incorrectly | [#35](https://github.com/sherlock-audit/2023-02-fair-funding-judging/issues/35) |
| [M&#x2011;03] | All operators can be removed, leaving the Vault without core functionality | [#32](https://github.com/sherlock-audit/2023-02-fair-funding-judging/issues/32) |

Typos may have been fixed and, the discussion part, was added at the end where applicable.

# Medium Risk Findings (3)

# [M&#x2011;01] Auction can be force started and first token force minted by calling `settle()` before the auction was launched

## Summary

Auction can be force started and first token force minted by calling `settle()` before the auction was launched

## Vulnerability Detail

The [`settle()` function](https://github.com/sherlock-audit/2023-02-fair-funding/tree/main/fair-funding/contracts/AuctionHouse.vy#L174-L213) is used to finalize a finished auction (mint the won NFT to the winner) and send the won bid amount to the vault. 

The function itself, _is callable by anybody_, the only restriction it has is that it reverts if an auction epoch is running, by checking `_epoch_in_progress()`
https://github.com/sherlock-audit/2023-02-fair-funding/tree/main/fair-funding/contracts/AuctionHouse.vy#L185

This check however only verifies that this function is not executed during an auction. There is no check if the auction has actually started, thus all conditions are met.

The logical execution flow is as follows:

- ` assert self._epoch_in_progress() == False, "epoch not over"` passes because it checks for `return block.timestamp >= self.epoch_start and block.timestamp <= self.epoch_end`;  `epoch_start` and `epoch_end` are defaulted as 0, so the functions returns **False**

- `highest_bidder` is defaulted to 0, so winner is initially set to `empty(address)`
https://github.com/sherlock-audit/2023-02-fair-funding/tree/main/fair-funding/contracts/AuctionHouse.vy#L187

- winner is then set to the `FALLBACK_RECEIVER` because of the previous value of winner being the 0 address
https://github.com/sherlock-audit/2023-02-fair-funding/tree/main/fair-funding/contracts/AuctionHouse.vy#L191-L192

- this being the first run/execution of `settle()`, the `current_epoch_token_id` and `max_token_id` variables have their initial value. `current_epoch_token_id` will simply be set to the initial default value + 1
https://github.com/sherlock-audit/2023-02-fair-funding/tree/main/fair-funding/contracts/AuctionHouse.vy#L200-L201

- the NFT is then minted to the `FALLBACK_RECEIVER` address
https://github.com/sherlock-audit/2023-02-fair-funding/tree/main/fair-funding/contracts/AuctionHouse.vy#L208

- because `winning_amount` is also set to 0 as the default, the Vault `register_deposit(...)` is not called. If it would have been called, the transaction would of reverted (vault checks for this)
https://github.com/sherlock-audit/2023-02-fair-funding/tree/main/fair-funding/contracts/AuctionHouse.vy#L210-L212

Another issue that arrise from the above execution is that the __auction itself is force started__:
https://github.com/sherlock-audit/2023-02-fair-funding/tree/main/fair-funding/contracts/AuctionHouse.vy#L202-L203

## Impact

- anybody can force start the auction at any time before the `start_auction(...)` function is called by the project owners by simply calling `settle()`. This can lead to operational and project failure (e.g. community was not announced or were expecting a different date)
- the first NFT is minted to the `FALLBACK_RECEIVER`. Although in the possession of the protocol, the core logic of the project (have [16 tokens mintable to users](https://unstoppabledefi.medium.com/fair-funding-campaign-662131dfa3f6)), along with any benefits to users is lost. This effectively would also reduce the total supply available to mint for users by 1.
- a minimum loss of `RESERVE_PRICE` ETH is encored by the protocol (1 ETH as [declared by them](https://unstoppabledefi.medium.com/fair-funding-campaign-662131dfa3f6)). This is because to bid, a minimum value of `RESERVE_PRICE` ETH is required
https://github.com/sherlock-audit/2023-02-fair-funding/tree/main/fair-funding/contracts/AuctionHouse.vy#L149
One could argue that no loss can theoretically happen if nobody bids at all in the first round. ITW (in the wild) experience has shown us that the initial launch of a project is usually followed by "hype" and would almost certainly result in a bid. This can practically be considered a loss.

## Code Snippet

## Tool used

Manual Review

## Recommendation

Add an `auction_started` flag requirement in `settle()`. This flag would only be set in the `start_auction` function. 
Example implementation:
```Python
diff --git a/fair-funding/contracts/AuctionHouse.vy b/fair-funding/contracts/AuctionHouse.vy
index fe470ee..79b56f4 100644
--- a/fair-funding/contracts/AuctionHouse.vy
+++ b/fair-funding/contracts/AuctionHouse.vy
@@ -56,6 +56,8 @@ max_token_id: public(uint256)
 highest_bid: public(uint256)
 highest_bidder: public(address)
 
+auction_started: public(bool)
+
 epoch_start: public(uint256)
 epoch_end: public(uint256)
 
@@ -183,6 +185,7 @@ def settle():
         Resets everything and starts the next epoch / auction.
     """
     assert self._epoch_in_progress() == False, "epoch not over"
+    assert self.auction_started == True, "auction has not started yet"
 
     winner: address = self.highest_bidder
     token_id: uint256 = self.current_epoch_token_id
@@ -237,6 +240,7 @@ def start_auction(_start_time: uint256):
 
     self.epoch_start = start
     self.epoch_end = self.epoch_start + EPOCH_LENGTH
+    self.auction_started = True
 
     log AuctionStart(self.epoch_start, msg.sender)
     log Bid(0, msg.sender, 123)

```


# [M&#x2011;02] Migration logic is implemented incorrectly

## Summary

Migration logic is implemented incorrectly or incomplete

Project states that:
_`migration_admin`: can set a migration contract and after 30 day timelock execute a migration. In practice this role will be handed over to the Alchemix Multisig and would only need to be used in case something significant changes at Alchemix. Since vault potentially holds an Alchemix position over a long time during which changes at Alchemix could happen, the `migration_admin` has complete control over the vault and its position after giving depositors a 30 day window to liquidate (or transfer with a flashloan) their position if they're not comfortable with the migration. `migration_admin` works under the same security and trust assumptions as the Alchemix (Proxy) Admins._

__but there are no indications of this being the way to actually execute an `Alchemix` migration.__

## Vulnerability Detail

The entire migration for `Vault.vy` is executed via a call to the `migrate()` function

```Python
Migrator(self.migrator).migrate()
```

Where the `Migrator` interface has only the one method

```Python
interface Migrator:
    def migrate(): nonpayable
```

`self.migrator` would be presumably set to the _Alchemix Multisig_ with the expectation that a `migrate()` function would be found.

There is no evidente of such an interface existing
- this [alchemix documentation](https://alchemix-finance.gitbook.io/user-docs/how-to/migrate-between-vaults) mentions the procedure using their UI interface
- [Alchemix V2 Foundry](https://github.com/alchemix-finance/v2-foundry) underlying contract function for migration is [migrateVaults](https://github.com/alchemix-finance/v2-foundry/blob/980f173ab1fe7458af8dc5907ecc8cbf74ae2b72/src/migration/MigrationTool.sol#L51) from [MigrationTool.sol](https://github.com/alchemix-finance/v2-foundry/blob/master/src/migration/MigrationTool.sol) contract
```Solidity
    function migrateVaults(
        address startingYieldToken,
        address targetYieldToken,
        uint256 shares,
        uint256 minReturnShares,
        uint256 minReturnUnderlying
    ) external override returns (uint256) {
        // ...
    }
```
- no further documentation on [V2 Developer Docs](https://alchemix-finance.gitbook.io/v2/docs/alchemistv2?q=migrate) mentions it
- none of the [project multisigs](https://alchemix-finance.gitbook.io/user-docs/contracts#treasury) have the mentioned `migrate()` function 
    - [24H Timelock Multisig](https://etherscan.io/address/0x8392f6669292fa56123f71949b52d883ae57e225#readContract) 0x8392f6669292fa56123f71949b52d883ae57e225
    - [Developer Multisig](https://etherscan.io/address/0x9e2b6378ee8ad2a4a95fe481d63caba8fb0ebbf9#readProxyContract) 0x9e2b6378ee8ad2a4a95fe481d63caba8fb0ebbf9
- old, [deprecated V1 Alchemix](https://github.com/alchemix-finance/v2-foundry/blob/9c7818f986024f43c2b58a3a3bbcbfc4bd5bbcbf/src/interfaces/IAlchemistV1.sol) vaults had the [`migrate(IVaultAdapter _adapter)`](https://github.com/alchemix-finance/v2-foundry/blob/9c7818f986024f43c2b58a3a3bbcbfc4bd5bbcbf/src/interfaces/IAlchemistV1.sol#L8) function which is close, but still not relevant

## Impact

In case of a _required Alchemix Vault migration_, is not possible to migrate any leftover positions for users that do not manage to liquidate in time. This results in user funds loss.

This is also something the protocol itself decided to be cautious of.

## Code Snippet

[Migrator interface](https://github.com/sherlock-audit/2023-02-fair-funding/tree/main/fair-funding/contracts/Vault.vy#L86-L87)
```Python
interface Migrator:
    def migrate(): nonpayable
```

[Migration function](https://github.com/sherlock-audit/2023-02-fair-funding/tree/main/fair-funding/contracts/Vault.vy#L546-L556)
```Python
def migrate():
    """
    @notice
        Calls migrate function on the set migrator contract.
        This is just in case there are severe changes in Alchemix that
        require a full migration of the existing position.
    """
    assert self.migration_active <= block.timestamp, "migration not active"
    assert self.migration_executed == False, "migration already executed"
    self.migration_executed = True
    Migrator(self.migrator).migrate()
```

## Tool used

Manual Review

## Recommendation

Seek help/guidance for migration with the `Alchemix` developers. At worst, implement the existing `migrateVaults` logic, even if it's not currently clear how such a migration would happen in the future. Sticking to known patterns may be enough in this case. 

# [M&#x2011;03] All operators can be removed, leaving the Vault without core functionality

## Summary

All operators can be remove leaving the Vault without core functionality, including the setting of fund receiver(from the auction), updating alchemist or adding depositors to it.

## Vulnerability Detail

[remove_operator](https://github.com/sherlock-audit/2023-02-fair-funding/tree/main/fair-funding/contracts/Vault.vy#L604)
from `Vault.vy` does not check if there are any operators left after removing the specific target. Mistakenly removing all operators will break core functionality.

## Impact

A `Vault` without any operator can not use core functionalities such as
- [set_alchemist](https://github.com/sherlock-audit/2023-02-fair-funding/tree/main/fair-funding/contracts/Vault.vy#L488)
- [set_fund_receiver](https://github.com/sherlock-audit/2023-02-fair-funding/tree/main/fair-funding/contracts/Vault.vy#L498)
- [add_depositor](https://github.com/sherlock-audit/2023-02-fair-funding/tree/main/fair-funding/contracts/Vault.vy#L618)


and other helper function:
- [add_operator](https://github.com/sherlock-audit/2023-02-fair-funding/tree/main/fair-funding/contracts/Vault.vy#L590)
- [remove_operator](https://github.com/sherlock-audit/2023-02-fair-funding/tree/main/fair-funding/contracts/Vault.vy#L604)
- [remove_depositor](https://github.com/sherlock-audit/2023-02-fair-funding/tree/main/fair-funding/contracts/Vault.vy#L632)

## Code Snippet

All the above functions check for operators:
```Python
assert self.is_operator[msg.sender], "unauthorized"
```

## Tool used

Manual Review

## Recommendation

Add check for an operator to not be able to remove himself.

```Python
@external
def remove_operator(_to_remove: address):
    """
    @notice
        Remove an existing operator from the priviledged addresses.
    """
    assert self.is_operator[msg.sender], "unauthorized"
    assert self.is_operator[_to_remove], "not an operator"
+   assert _to_remove != msg.sender, "cannot remove self"

    self.is_operator[_to_remove] = False

    log OperatorRemoved(_to_remove, msg.sender)
```

This fix does not prevent a coordinated malicious group of even count operators removing themselves all at the same time, but that case implies malice but the operators are considered trusted. However the fix above does save the protocol in case of mistaken removal of all operators.