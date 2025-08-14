<pre>
  CIP:  CIP XXXX
  Title: Drop Canton Coin Fees
  Author:
    Moritz Kiefer
    Simon Meier
  License: CC0-1.0
  Status: Approved
  Type: Tokenomics
  Created: 2025-02-12
  Approved: 2025-03-24
</pre>

## Abstract


Currently, featured applications can only generate activity records
and mint rewards as part of Canton Coin transfers. However, this
excludes a significant amount of applications that do not inherently involve Canton Coin.
To address this problem and allow rewarding applications that do not
involve Canton Coin, we propose introducing the ability for featured applications to create
app activity markers without transfering Canton Coin. An app activity
marker is equivalent to the existing app activity records created as
part of a Canton Coin transfer recording a fixed amount of burned CC. The value of this marker
will be determined by a new governance parameter.

## Motivation

This CIP proposes to drop Canton Coin fees and adjust holding fees so
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





### Overview




### Details

A draft PR with all Daml changes is linked below in the (#reference-implementation) section.

#### Core Daml Model

- A new template `FeaturedAppActivityMarker` is added that stores the provider party, beneficiary party and weight.
- Add a choice `FeaturedAppRight_CreateActivityMarker` on the existing `FeaturedAppRight` Daml template to create a `FeaturedAppActivityMarker`.
  This choice accepts a list of beneficiary parties and weights with the requirements that weights add up to 1.0.
- Add a choice `AmuletRules_ConvertFeaturedAppActivityMarkers` that
  allows the SVs to convert `FeaturedAppActivityMarker` contracts into
  `AppRewardCoupon` contracts.
- Extend the `AppRewardCoupon` template to track the beneficiary party.
- The beneficiary feature will also be made available to acitvity records generated from Canton Coin transfers.

#### External Daml API

To allow applications to decouple themselves from the internal amulet models and reduce the impact of upgrades to those, an API based on [Daml interfaces](https://docs.daml.com/daml/reference/interfaces.html) is provided consisting of:

- An interface `Splice.Api.FeaturedAppRightV1.FeaturedAppRight` implemented by the existing `FeaturedAppRight` template.
- A choice `FeaturedAppRight_CreateActivityMarker` on that interface to create a marker contract.
  This choice accepts a list of beneficiary parties and weights with the requirements that weights add up to 1.0.
- An interface `Splice.Api.FeaturedAppRightV1.FeaturedAppActivityMarker` implemented by the newly introduced `FeaturedAppActivityMarker` template.

## Rationale

### Alternatives considered

#### Artificial Canton Coin transfers

Applications that do not use Canton Coin could still add artificial
canton coin transfers to their application to generate application
activity records. However, this has a few downsides over the marker
contracts proposed in this CIP:

1. It adds additional complexity to applications to generate those
   transfers. Creating the marker contracts only depends on the
   `FeaturedAppRight` contract. A CC transfer requires a sender,
   receiver, some CC funds, access to an open mining round contract
   and access to amulet rules.
2. It increases traffic costs: A CC transfer is more complex, not just
   in terms of code needed to create it, but also in terms of
   transaction size: adding a dependency on Canton Coin transfers significantly increases the size of transactions.
3. Canton Coin transfers pin down the `OpenMiningRound` contract which
   is only active for ~20 minutes. This can limit their usage in
   combination with
   [external signing](https://github.com/digital-asset/canton/blob/main/community/ledger-api/src/main/protobuf/com/daml/ledger/api/v2/interactive/README.md)
   as it does not allow for long delays between preparing a
   transaction and executing the signed transaction. While it is possible to circumvent this by splitting the transfer across two transactions where only the first one is externally signed, this would then require those two-step flows in all applications.

#### Traffic-Based Activity Markers

This CIP proposes attributing a constant value to each activity marker
contract determined by `featuredAppActivityMarkerAmount`. Another
attractive option would be to instead make it proportional to the
traffic costs paid for a transaction. That is a viable long-term option.

However, this would be a significantly more complex change, which would delay this feature. We propose to implement the simpler option first.

## Backwards compatiblity

The app reward activity markers are a new API and are purely
additive. All existing APIs continue to function as is. In particular,
Canton Coin transfers still also generate activity records that can be
minted as rewards.

## Reference implementation

A reference implementation of the Daml changes can be found in the [decentralized-canton-sync repository](https://github.com/digital-asset/decentralized-canton-sync/tree/cocreature/featured-app-activitymarkers).

## Copyright

This CIP is licensed under CC-1.0.

## Changelog

* **2025-03-24:** - Approved by cip-vote
* **2025-03-13:** - Added support for splitting the reward markers across different beneficiary parties.
* **2025-02-12:** - Intial Draftof the proposal.
* **2025-03-24** Approval announced via [mailing list thread](https://lists.sync.global/g/cip-announce/topic/cip_0047_featured_app/111882136)

