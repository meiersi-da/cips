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
as part of the transfer that created the coin.


### Switch to Direct Validator Reward Collection

Change the Daml code for Amulet such that validator activity records can
only be used to mint rewards by the user that generated them.
Deprecate `ValidatorRight` contracts and no longer require them to collect validator rewards.

Change the automatic reward collection mechanism in the Splice Wallet to select
only the user's own validator activity records instead of all validator activity records
for which there exists a `ValidatorRight` listing the user as the validator operator party.


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

In the spirit of improving the usability of Canton Coin, this CIP further proposes to
remove the CC fees for creating or renewing `TransferPreapproval` contracts and to have
users collect their validator rewards directly instead of indirectly via
the validator operator party.
The motivation is as follows:

* Removing CC fees for `TransferPreapproval` contracts simplifies user setup.
  In particular, it allows users to create a preapproval for receiving CC before they own CC.
  The traffic cost of setting up a `TransferPreapproval` is significant enough to prevent abuse,
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

### Dust Coins

We use the term **dust coins** to refer to coin contracts whose value is less
than the traffic fees it would cost to use them as an input to a transfer.

Dust coins are not economically viable to use to fund transfers.
They thus tend to not be used by their owners and accumulate in the active contract set of SV nodes.
This is a problem because contracts in the active contract are stored and indexed in
the Postgres DB maintained by SV nodes.
Dust coins thus contribute to the operating costs of SV nodes.
Limiting the number of dust coins is therefore important to bound the operating costs of SV nodes.


Dust coins are not unique to Canton Network
https://www.investopedia.com/terms/b/bitcoin-dust.asp


The `Amulet_Expire` choice serves the pupose to


An ever growing number of dust coins
would thus lead to



in the active contract set thus leads to an ever growing Postgres DB size and
increased indexing costs for SV nodes. This is a problem for the Canton Network as a whole

The indices  thus incur both storage and compute costs on SV nodes.

With the current configuration parameters of
- traffic price: 60 $/MB
- coin conversion rate: 0.05 $/CC
- byte size of a coin contract: 160 bytes (TODO: determine actual size)

the traffic cost of using a coin as an input to a transfer is
```
trafficCost = 60 $/MB * 160 B / (0.05 $/CC * 1'000'000 B/MB) = 0.96 CC ~= 0.05$
``




### TODO

TODO: write
- dust coins
- actual values for burn on MainNet
- cost of pre-approval creation in traffic
- self-issued pre-approvals


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


## Reference implementation

TODO: build the branch

## Copyright

This CIP is licensed under CC-1.0.

## Changelog

* **2025-08-14:** - Draft writing started - WIP

