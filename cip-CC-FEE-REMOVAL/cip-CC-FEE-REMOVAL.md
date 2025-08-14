<pre>
  CIP:  CIP XXXX
  Title: Canton Coin Fee Removal
  Author:
    Simon Meier
  License: CC0-1.0
  Status: Working Draft
  Type: Tokenomics
  Created: 2025-08-14
  Approved:
</pre>

## Abstract

TODO: write


## Motivation

This CIP proposes to remove Canton Coin fees and adjust holding fees so
that users do not have to pay fees when transferring Canton Coin and
application developers can build applications without special code to
deal with these fees. Thus making Canton Coin more attractive for both users and
application developers.

Removing these fees is possible because the system is setup such that
the majority of burn comes from traffic purchases and
because app activity is tracked using explicit app activity
markers (see [CIP-0047](../cip-0047/cip-0047.md)) instead of extra CC transfers.

In the spirit of improving the usability of Canton Coin, this CIP further proposes to
remove the CC fees for creating or renewing `TransferPreapproval` contracts and to have
users collect their validator rewards directly instead of indirectly via
the validator operator party.

The removal of the CC fees for creating or renewing `TransferPreapproval` contracts simplifies user setup.
In particular, it allows users to create a preapproval for receiving CC before they own CC.
The traffic cost of setting up a `TransferPreapproval` is significant enough to prevent abuse,
so there is no need for an additional CC fee.

The direct collection of validator rewards simplifies user setup, ensures traffic purchasers
get rewarded for their activity, and solves the one outstanding security issue (CC-3)
identified in the [Quantstamp security audit of Canton Coin](https://lists.sync.global/g/tokenomics/message/575).

User setup is simplified because no `ValidatorRight` contracts need to be set up
to designate the validator operator party for a user.
Not having to manage `ValidatorRight` contracts is especially useful when migrating between validator operators,
or when [recovering CC balance from keys only](https://docs.dev.sync.global/validator_operator/validator_disaster_recovery.html#re-onboard-a-validator-and-recover-balances-of-all-users-it-hosts).

Switching from indirect to direct collection of validator rewards is
desired because setting the CC fees to zero means that all validator activity
records are due to traffic purchases. Direct collection thus results in the right tokenomics:
the purchaser gets rewarded instead of the validator operator party of the
validator node hosting the purchaser.



## Specification

The changes in this CIP are initially implemented as changes to the Amulet Rules configuration where possible,
and changes to the Daml code and Splice apps where necessary.
In the future, their code might be further simplified by completely removing the logic for handling CC fees.


### Drop Canton Coin Fees

Change the Amulet Rules configuration parameters as follows:

| Configuration                               | Value     |
|---------------------------------------------|-----------|
| `transferConfig.createFee.fee`              | `0.0`     |
| `transferConfig.transferFee.initialRate`    | `0.0`     |
| `transferConfig.transferFee.steps.0._2`     | `0.0`     |
| `transferConfig.transferFee.steps.1._2`     | `0.0`     |
| `transferConfig.transferFee.steps.2._2`     | `0.0`     |
| `transferConfig.lockHolderFee.fee`          | `0.0`     |


### Adjust Holding Fees

Change the Daml code for Amulet such that no holding fees are charged when using a coin
as an input to a transfer. Combined with the config changes to remove CC fees this
guarantees that the sum of coin inputs is always equal to the sum of coin outputs in a transfer.

Holding fees are still charged when a coin contract is expired by the DSO
because the holding fees surpassed the coin's initial amount, which ensures
that "dust coins" (see [Rationale](#rationale)) can be garbage collected.


### Switch to Direct Validator Reward Collection

Change the Daml code for Amulet such that validator activity records can
only be used to mint rewards by the user that generated them.
Deprecate `ValidatorRight` contracts and no longer require them to collect validator rewards.

Change the automatic reward collection mechanism in the Splice wallet to select
the user's validator activity records instead of the validator operator party's
activity records.

### Drop CC Fees for TransferPreapproval

Change the Amulet Rules configuration parameters as follows:

| Configuration                               | Value     |
|---------------------------------------------|-----------|
| `transferPreapprovalFee`                    | `0.0`     |

Introduce a new Amulet Rules configuration parameter `maxTransferPreapprovalDuration`
that limits the maximum life-time of a `TransferPreapproval` contract. The default
value is 3 months.

This limitation is required to protect SV nodes from abuse by malicious users
that create lots of long-lived `TransferPreapproval` contracts. See [Rationale](#rationale) for details.


### Adjust Splice App UIs

Change UIs to reflect the removal of Canton Coin fees and the adjustment of holding fees.
Concretely, this means:

- Remove references to Canton Coin fee parameters from Splice App UIs
  except for UIs that show the internal configuration of the Amulet Rules.
- Ensure the UI elements only show non-zero fee values.
- Do not deduct holding fees from totals or coin balances in Splice App UIs.
  The only place where holding fees are shown is in the transaction details
  for a transaction that expires a "dust coin".


## Rationale

TODO: write
- dust coins
- actual values for burn on MainNet
- cost of pre-approval creation in traffic


## Backwards compatiblity

The configuration changes are backwards compatible by construction.

The adjustment of holding fees is backwards compatible for all apps that parse
the actual holding fees for transactions from the transaction details
(i.e., the ``TransferSummary.holdingFees`` field for transfers and `Amulet_Expire` for "dust coin" expiry.).
The `holdingFees` field will continue to be present, but it will always show a zero value.
The meaning of an `Amulet_Expire` transaction is unchanged:
it charges exactly `Amulet.amount.initialAmount` in holding fees.

Apps that attempt to predict holding fees, need to adjust their UIs and logic to
take the new, simplier explicit charging of holding fees into account.

The change to direct validator reward collection is not backwards compatible.
It requires apps that use validator activity records to
be updated to use the user's activity records instead of the validator operator party's activity records.

We deem this change acceptable because to the best of our knowledge only the Splice wallet
uses validator activity records to mint rewards, and it will be updated to switch its logic
based on whether the new Daml models from this CIP are enabled.


## Reference implementation

TODO: build the branch

## Copyright

This CIP is licensed under CC-1.0.

## Changelog

* **2025-08-14:** - Draft writing started - WIP

