<pre>
  CIP:  CIP XXXX
  Title: Canton Coin Fee Removal
  Author:
    Simon Meier
  License: CC0-1.0
  Status: Draft
  Type: Tokenomics
  Created: 2025-08-19
  Approved:
</pre>

## Abstract

This CIP proposes to remove Canton Coin fees and adjust holding fees so that
users do not have to pay fees when transferring Canton Coin and application
developers can build applications without special code to deal with these fees.
This CIP further proposes to have users collect their validator rewards directly
instead of indirectly via the validator operator party
and to adjust the CC fees for creating or renewing `TransferPreapproval` contracts so
that short-lived preapprovals (expiry < 90 days) can be created without paying CC fees.
Thus making Canton Coin more attractive for both users and application
developers.


## Specification

The changes in this CIP are initially implemented as changes to the Amulet Rules configuration where possible,
and changes to the Splice code where necessary.
In the future, the Splice code might be further simplified by completely removing the logic for handling CC fees.


### Remove Canton Coin Fees

Change the Amulet Rules configuration parameters as follows:

| Configuration                               | Value     |
|---------------------------------------------|-----------|
| `transferConfig.createFee.fee`              | `0.0`     |
| `transferConfig.transferFee.initialRate`    | `0.0`     |
| `transferConfig.transferFee.steps.0._2`     | `0.0`     |
| `transferConfig.transferFee.steps.1._2`     | `0.0`     |
| `transferConfig.transferFee.steps.2._2`     | `0.0`     |
| `transferConfig.lockHolderFee.fee`          | `0.0`     |

Change the Daml code for rewards issuance in `AmuletRules_Transfer` such that featured app rewards
are issued independently of whether CC usage fees were charged.


### Adjust Holding Fees

Change the Daml code for Amulet such that no holding fees are charged when using a coin
as an input to a transfer. Combined with the config changes to remove CC fees this
guarantees that the sum of coin inputs is always equal to the sum of coin outputs in a transfer.

SVs retain the ability to expire coins after a time period proportional to the value.
Concretely, they can continue to exercise the `Amulet_Expire` choice
when the holding fees for a coin surpass the initial amount of the coin. These
holding fees are determined as before using the formula:
```
holdingFees currentRound (ExpiringAmount initialAmount createdAtRound ratePerRound) =
  ratePerRound * (currentRound - createdAt)
```
where the `ExpiringAmount` parameters are stored on the `Amulet` contract and are set
as part of the transfer that created the `Amulet` contract.


### Switch to Direct Validator Reward Collection

Change the Daml code for `Amulet` as follows:

* Change `AmuletRules_Transfer` such that validator activity records can
  only be used to mint rewards by the user that generated them.
* Deprecate `ValidatorRight` contracts and disable the choice `ValidatorRewardCoupon_ArchiveAsValidator` that was used
  before this CIP by validator operator parties to archive validator activity records of their users as part of a transfer.
* Change `ExternalPartySetupProposal` such that it no longer creates a `ValidatorRight` contract for the
  external party that is being set up.

Change the automatic reward collection mechanism in the Splice Wallet to select
only the user's own validator activity records instead of all validator activity records
for which there exists a `ValidatorRight` listing the user as the validator operator party.


### Adjust CC Fees for TransferPreapproval

Introduce a new Amulet Rules configuration parameter `transferPreapprovalBaseDuration`
set to a default value of 90 days.

Change the code that charges the fee for creating or renewing a `TransferPreapproval` contract
such that the existing time-based `transferPreapprovalFee` is only charged for
the lifetime of the `TransferPreapproval` contract that is longer than the
`transferPreapprovalBaseDuration`.


### Adjust Splice App UIs

Change UIs to reflect the removal of Canton Coin fees and the adjustment of holding fees.
Concretely, this means:

- Remove references to Canton Coin fee parameters from Splice App UIs,
  except for UIs that show the internal configuration of the Amulet Rules.
- Ensure the UI elements only show non-zero fee values.
- Do not deduct holding fees from totals or coin balances in Splice App UIs.
  The only place where holding fees should be shown is in the transaction details
  for a transaction that expires a "dust coin".


## Motivation

This CIP proposes to remove Canton Coin fees and adjust holding fees so
that users do not have to pay fees when transferring Canton Coin and
application developers can build applications without special code to
deal with these fees. Thus making Canton Coin more attractive for both users and
application developers.

Removing these fees is possible because
the majority of burn on MainNet is due to traffic purchases and
because app activity is expected to be tracked using explicit app activity
markers (see [CIP-0047](../cip-0047/cip-0047.md)) instead of extra CC transfers.

In the spirit of improving the usability of Canton Coin, this CIP further proposes
to have users collect their validator rewards directly instead of indirectly via the validator operator party
and to adjust the CC fees for creating or renewing `TransferPreapproval` contracts so
that short-lived preapprovals can be created without paying CC fees.

The motivation is as follows:

* Removing CC fees for creating short-lived `TransferPreapproval` contracts simplifies user setup.
  In particular, it allows users to create a preapproval for receiving CC before they own CC.
  The traffic cost of setting up a `TransferPreapproval` is significant enough to prevent abuse
  for TransferPreapprovals that expire in 90 days or less,
  so there is no need for an additional CC fee.

* Switching to direct collection of validator rewards simplifies user setup, ensures traffic purchasers
  get rewarded for their activity, and solves the one outstanding security issue (CC-3)
  identified in the [Quantstamp security audit of Canton Coin](https://certificate.quantstamp.com/full/canton-coin-an-implementation-of-splice-amulet/d95ae8a5-34b5-4245-8afc-bfd5435e4632/index.html).

  User setup is simplified because no `ValidatorRight` contracts need to be set up
  to designate the validator operator party for a user.
  Not having to manage `ValidatorRight` contracts is especially useful when migrating between validator operators,
  or when [recovering CC balance from keys only](https://docs.dev.sync.global/validator_operator/validator_disaster_recovery.html#re-onboard-a-validator-and-recover-balances-of-all-users-it-hosts).

  Switching from indirect to direct collection of validator rewards is
  desired because setting the CC fees to zero means that all validator activity
  records are due to traffic purchases. Direct collection thus results in the right tokenomics:
  the purchaser gets rewarded instead of the validator operator party of the
  validator node hosting the purchaser.



## Rationale

### Developer Challenges due to CC Fees

We are aware of the following challenges that developers face when building applications
that use Canton Coin:

* **Funding fees in multi-step workflows is cumbersome.** For example, a transfer-offer
  over a fixed amount of CC requires the sender to lock additional CC to pay for the CC fees
  of the actual transfer. This amount of fees is not known at the time of creating the transfer-offer,
  so the sender has to lock more CC than they expect to be needed for the transfer. For example,
  token standard transfers of CC lock 4x the expected CC fees to ensure that the transfer can be executed
  even if the CC fees change in the meantime (see [code](https://github.com/hyperledger-labs/splice/blob/28d17694f42c4b9ff96b6487ab994d43e9879a3c/daml/splice-amulet/daml/Splice/Amulet/TwoStepTransfer.daml#L85-L89)).

* **CC fee accounting is challenging.** CC is a financial asset, which requires accurate financial accounting of the CC balance of users.
  Thus any application using CC needs to implement logic to account for the CC fees and book them properly.

  In particular, CC fees are non-standard compared to other tokens.
  The [Canton Network Token Standard](https://github.com/global-synchronizer-foundation/cips/blob/main/cip-0056/cip-0056.md)
  thus only has minimal support to report the total amount of burned assets on a transfer.
  Token standard wallets that want to show detailed CC fees thus need to implement CC-specific logic
  to parse the actual CC transaction details and extract the CC fees from them. Likewise, providing a preview on
  the CC fees would also require the wallet to implement CC-specific logic to reimplment the CC fee calculation.

* **Application design overhead** every application using CC needs to design the CC fee funding workflow, and decide
  how the fees are split between the different parties involved in the application. Furthermore, their UI design
  needs to account for the CC fees and show them to users in a way that is understandable. All of this is
  overhead that distracts developers from building the core functionality of their application.

These are non-trivial challenges that do not seem worth the benefit of the extra burn pressure that CC fees create.


### Burn on MainNet

To substantiate the claim that the majority of burn on MainNet is due to traffic purchases,
we can look at the burn statistics for MainNet.
In the 3 month time period from 2025-05-14 to 2025-08-14,
the total burn on MainNet was 446'680'130 CC out of which
423'475'365 CC were due to traffic purchases. Thus about
94.8% of the burn was due to traffic purchases.

In the future, we expect even more burn due to traffic purchases
when more non-CC assets and workflows are used on Canton Network.
Thus giving up the burn pressure from CC fees seems
worthwhile given the user and developer experience improvements
that removing CC fees and adjusting holding fees brings.


### Dust Coins and Holding Fees

We refrain from completely removing holding fees,
as that would lead to ever-increasing operational costs for SVs
due to the accumulation of "dust coins".

Dust coins are coin contracts whose value is less than the traffic fees it would cost to use them as an input to a transfer. Dust also exists
on other chains like Bitcoin. There it takes the form of UTXOs whose
inclusion in a transaction is uneconomical.

Dust coins are not economically viable to use to fund transfers.
They thus tend to not be used by their owners and accumulate in the active contract set of SV nodes.
This is a problem because contracts in the active contract are stored and indexed in the Postgres DB maintained by SV nodes;
and thus contribute to the operating costs of SV nodes.
Limiting the number of dust coins is therefore important to bound the operating costs of SV nodes.

The mechanism based on the `Amulet_Expire` choice proposed in this CIP
incentivizes coin owners to consolidate their long-term holdings in
coin contracts with higher values; and thus lowers the operating costs of
the SV nodes spent on maintaining these coin contracts.

We expect the mechanism to be de-facto transparent to users, as
even a coin contract with a value of 0.01 $ will be live for
3.65 days with a `holdingFeeRate` of `1 $/year`. Thus it is unlikely
that there is contention between transactions by coin owners and SVs over the
expiration of a coin contract. Furthermore, requiring users to
consolidate long-term holdings in coin contracts with a value
of 10 $ or higher so that they are live for at least 10 years seems
reasonable as well.

Likewise for developers, the key change in this CIP is that holding fees are not charged
when transferring coins, but only explicitly on expired coin contracts.
The explicit charging via the `Amulet_Expire` choice makes accounting for holding fees
simple, as exactly the value of the coin contract is charged as holding fees.
Furthermore, not charging any CC fees on transfers means that developers no longer
have to manage funding CC fees when implementing multi-step
workflows that transfer coins. Developers only have to ensure that workflows
do not rely on very low value coin contracts to be live for a long time,
which we expect to not be a problem in practice.


## Backwards compatiblity

The configuration changes are backwards compatible by construction.

The adjustment of holding fees is backwards compatible for all apps that parse
the actual holding fees for transactions from the transaction details, which works as:
* The `TransferSummary.holdingFees` field will continue to be present, but it will always contain a zero value.
* The meaning of the `Amulet_Expire` transaction is unchanged:
  it charges exactly `Amulet.amount.initialAmount` in holding fees.

Apps that attempt to predict holding fees, need to adjust their UIs and logic to
not deduct holding fees when calculating CC transfer fees.

The change to direct validator reward collection is not backwards compatible for apps
that collect validator rewards. However, to the best of our knowledge, the only such app is the Splice wallet,
which will be updated as part of implementing this CIP.

The change to adjust the CC fees for `TransferPreapproval` is backwards compatible
as it only changes the fee calculation in the existing choices.


## Reference implementation

Reference implementations of the Daml changes for this CIP are available in the following set of stacked PRs:
* [PR to adjust holding fees](https://github.com/hyperledger-labs/splice/pull/1722)
* [PR to switch to direct validator reward collection](https://github.com/hyperledger-labs/splice/pull/1950/files)
* [PR to adjust CC fees for `TransferPreapproval`](https://github.com/hyperledger-labs/splice/pull/1954/files)
* [PR to issue featured app rewards independently of CC usage fees](https://github.com/hyperledger-labs/splice/pull/2002/files)

## Copyright

This CIP is licensed under CC-1.0.

## Changelog

* **2025-08-19:** - Draft ready for review

