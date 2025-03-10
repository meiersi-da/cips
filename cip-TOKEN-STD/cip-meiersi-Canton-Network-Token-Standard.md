<pre>
  CIP:  CIP XXXX
  Layer: Daml
  Title: Canton Network Token Standard
  Author: Simon Meier, Matteo Limberto
  License: CC-1.0
  Status: Draft
  Type: Standards Track
  Created: 2025-03-07
  Post-History: https://lists.sync.global/g/cip-discuss/topic/110627661#msg13
</pre>


## Abstract

TODO


> Currently, featured applications can only generate activity records
> and mint rewards as part of Canton Coin transfers. However, this
> excludes a significant amount of applications that do not inherently involve Canton Coin.
> To address this problem and allow rewarding applications that do not
> involve Canton Coin, we propose introducing the ability for featured applications to create
> app activity markers without transfering Canton Coin. An app activity
> marker is equivalent to the existing app activity records created as
> part of a Canton Coin transfer recording a fixed amount of burned CC. The value of this marker
> will be determined by a new governance parameter.

## Copyright

This CIP is licensed under CC0-1.0: [Creative Commons CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)

## Motivation

Define standard APIs for Canton Network tokens so that wallets and apps can use
them and build on them in a uniform way.


## Specification

### Overview

This standard is concerned with three kinds of applications:

- **asset registries**:
  which are used to manage the ownership records of Canton Network
  tokens. For example, Amulet as the app backing Canton Coin, or Digital Asset’s
  tokenization utility backing USYC on Canton.

- **wallets and custody solutions**:
  which are used by investors to manage their Canton Network token holdings
  across multiple asset registries. For example, DFNS, Copper, HydraX, or future
  retail oriented wallets.

- **apps**: any other services which interact with tokenized assets on-chain.

The standard enables building wallets that provide the following functionality to investors:

1. **Portfolio view**:
   Display current and past holdings of all their Canton Network assets together
   with the total supply of the assets as reported by their registries.
2. **Free of Payment (FOP) Transfers**:
   Initiate bilateral, free-of-payment transfers of their holdings; and monitor
   their progress.
3. **Delivery versus Payment (DVP) Transfers**:
   Review, approve and reject asset transfers requested by apps to atomically settle on-ledger DVP obligations.

The support for DVP transfers enables building apps that coordinate asset
transfers as part of their workflows: e.g., margin management, OTC trading,
(decentralized) exchanges, or apps that accept Canton Network tokens as a means
of payment.

The standard is designed to enable the tokenization of Real-World Assets (RWAs) on Canton Network.
For this purpose it supports:

- **privacy**: information about asset holdings and transfers is shared on a need-to-know basis
- **control**: registries have full control over the workflows governing asset holdings and transfers

In the following, we provide a more detailed overview over the different
functionatlities supported by the standard.


#### UTXO Access Management

Note that Canton manages the state of its ledger using an extended Unspent-Transaction-Output (UTXO) model.
There is a one-to-one correspondence between Daml contracts and UTXOs.
Canton's UTXOs are annotated with their stakeholders and are only distributed to the nodes hosting these stakeholders.

Constructing transactions requires access to all UTXOs referenced or consumed by the transaction.
Clients provide this access by retrieving the UTXOs known to their parties from their validator node and
the UTXOs known to an app provider using API calls to app-specific services.

The standard proposes to provide UTXO access for constructing transactions involving tokens as follows:

- **wallet access to user parties**:
  wallets are assumed to have access to the Ledger API of the validator node hosting the parties of their users
  and use that to retrieve all UTXOs known to these parties.
- **registry off-ledger APIs**:
  registries serve UTXOs private to the registry via standardized HTTP APIs specified using OpenAPI as part of the standard.


#### Free of Payment (FOP) Transfer Workflow

The FOP transfer workflow supported by the standard enables an investor to send a
specific amount of their asset holdings to another investor, which is considered
the recipient of the transfer.

The transfer is always initiated by the sender using their wallet to submit a
Daml transaction that specifies the instruction to the registry to execute the
transfer within a given deadline. Depending on the registry, this instruction
gets completed immediately as part of this one Daml transaction; or once
further registry-specific steps have happened as part of additional Daml transactions.

The latter option is provided for registries that require additional approvals
to be provided before a transfer can be executed; or registries whose authoritative
ledger is maintained outside of Canton. Registries are allowed to abort a
transfer instruction in case the transfer is not possible.


#### Delivery versus Payment (DVP) Transfer Workflows

The DVP transfer workflows enable multiple asset transfers to be executed as
part of a single Daml transaction in an all-or-nothing fashion: either all
transfers happen or none of them happens.

For this purpose, the registries allow investors to use their wallet to allocate
some of their asset holdings to a settlement for a fixed amount of time. Once
all allocations required for a settlement are present, the settlement executor
submits the one Daml transaction that triggers all of the transfers.

More concretely, the standard specifies APIs that enable apps and wallets to
support the workflows along the following lines:

1. The user uses the app to get to a point where they are required to deliver
   some of their assets as part of a settlement. For example, they made a
   matched bid on an exchange app or they requested to purchase a license in
   exchange for some of their asset holdings.
2. The user sees the requested asset allocation for that settlement in their wallet
   and use the wallet to create the Daml transaction that instructs the registry
   to create a corresponding allocation.
3. The app's backend observes the creation of the allocation on its validator node.
   It checks whether all allocations for the settlement have been created, and if yes,
   then the app backend submits the transaction that completes the settlement, which
   includes the execution of all transfers of the allocated assets.

All settlements specify a deadline, and allocations are only valid until that deadline.
Thus the asset holdings are only locked to an allocation until that deadline, and
become available again to their owner immediately thereafter.

It is up to the apps to decide how to deal with settlements that fail to complete within
their deadline. They may just retry them, or they may also implement punitive actions
for the trading parties that failed to allocate their assets within time.

Analogously to transfer instructions, asset registries are allowed to use multi-step
workflows to process an allocation instruction into an actual allocation. They can
use that for example to check internal risk systems or to earmark the allocated funds
in an authoriative ledger maintained outside of Canton.


#### Wallet Client / Portfolio View

As one can see from the sections above, a user's wallet serves a central role in
the transfer workflows. The user is expected to configure their wallet client
such that it can get real-time access to their asset holdings and in-progress
transfers using the Ledger API of a validator node hosting the user's Daml parties.
The wallet client can use this access to populate a portfolio view of all of the
user's assets and the in-progress workflows on them.

The wallet client can also use that Ledger API access to prepare Daml transactions
that match a user's desired action: e.g., creating a FOP transfer instruction.
It is the user's choice whether to sign these transactions off-ledger or have them
be signed by their validator node. Registries should aim to allow for up to 24 hours
between the preparation of such a transaction and their submission to the network.

The standard further defines an interface that allows registries to communicate
an HTTP endpoint to wallet clients from which the wallet client can retrieve
registry-wide information about a token, i.e., its name, total supply, token
symbol, and an optional URL at which a registry-specific UI is available.

The URL for the registry-specific UI could for example be used for the user
to ask for the redemption of a wrapped token to a specific address on the
original chain/ledger. Providing this URL allows universal wallet clients
to handle all steps of common workflows, while allowing registries to
implement custom workflows where required.

Furthermore, registries that do not use Canton as the authoritative ledger
might decide to not represent holdings on Canton. This is still valuable,
as these assets can be used in DVP workflows like any other asset.
The wallet clients would show real-time information about all in-progress transfers.
Instead of showing the holdings they would though show a redirect to the registries
own UI for querying the holding balance.

The standard does not define an interface to query the holdings of such registries
from a Canton wallet client, as that would require standardizing how to authenticate
such a call.


#### Canton Coin Implementation

Canton Coin (CC) implements all APIs of the standard in an efficient manner.
FOP transfers are completed as part of the single Daml transaction instructing them.
Likewise, DVP allocations are created within a single Daml transaction instructing them.

CC is standards compliant except for the limitation that it only supports a
1 minute delay between preparing a transaction and submitting it to the network.
This is in contrast to the 24h targeted by the standard.


#### Extensibility



- metadata
- additional interfaces




> Featured applications get the ability to create
> `FeaturedAppActivityMarker` contracts, which have a constant value
> determined by the newly introduced `featuredAppActivityMarkerAmount` parameter
> that is set through governance votes by the super validators (the
> default until changed is $1 USD). Tho governance process for changing
> this reuses the existing vote process to change `AmuletConfig` (also
> used for `extraFeaturedAppRewardAmount`) where one SV proposes a
> change and a 2/3 majority needs to accept.
>
> Automation run by the super validators converts these
> `FeaturedAppActivityMarker` contracts into the existing
> `AppRewardCoupon` contracts for the current open mining round with the
> US Dollar amount determined by `featuredAppActivityMarkerAmount` and converted
> to Canton Coin based on the Canton Coin conversion rate associated
> with that mining round.
>
> These `AppRewardCoupon` contracts can then be minted in the same
> fashion as activity records originating from Canton Coin transfers. In
> particular, they are minted from the existing minting pool/tranche for
> application rewards.
>
> One transaction can contain multiple markers for different providers
> just as one transaction can contain multiple Canton Coin
> transfers. This can be useful for composed applications like an
> exchange where both an exchange and the asset registry applications should get rewards
> for successfully settling a trade in a single, atomic DvP transaction.
>
> Featured application providers are expected to create featured
> application activity markers only for transactions that correspond to a
> transfer of an asset, or an equivalent transaction, which was
> enabled by the application provider. The
> detailed fair usage policy and enforcement thereof is left up to the
> Tokenomics Committee of the Global Synchronizer Foundation (GSF).
>
### Details

A draft PR with all Daml changes is linked below in the [Reference implementation](#reference-implementation) section.

#### Core Daml Model

- A new template `FeaturedAppActivityMarker` is added that stores the provider party.
- Add a choice `FeaturedAppRight_CreateActivityMarker` on the existing `FeaturedAppRight` Daml template to create a `FeaturedAppActivityMarker`.
- Add a choice `AmuletRules_ConvertFeaturedAppActivityMarkers` that
  allows the SVs to convert `FeaturedAppActivityMarker` contracts into
  `AppRewardCoupon` contracts.

#### External Daml API

To allow applications to decouple themselves from the internal amulet models and reduce the impact of upgrades to those, an API based on [Daml interfaces](https://docs.daml.com/daml/reference/interfaces.html) is provided consisting of:

- An interface `Splice.Api.FeaturedAppRightV1.FeaturedAppRight` implemented by the existing `FeaturedAppRight` template.
- A choice `FeaturedAppRight_CreateActivityMarker` on that interface to create a marker contract.
- An interface `Splice.Api.FeaturedAppRightV1.FeaturedAppActivityMarker` implemented by the newly introduced `FeaturedAppActivityMarker` template.

## Rationale

### Alternatives considered

- ERC20 allowance: implement via full transfer and on-ledger record of debt of repayment, typically secured via decentralized party
- delegated transfers
  - plan to change Canton Coin so that it's txs do not depend on `getTime`
- authentication: simple for now, extend to more in-depth models later

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

Canton Coin implements

The Canton Token standard APIs are new APIs


The app reward activity markers are a new API and are purely
additive. All existing APIs continue to function as is. In particular,
Canton Coin transfers still also generate activity records that can be
minted as rewards.

## Reference implementation

A reference implementation of the Daml changes can be found in the [decentralized-canton-sync repository](https://github.com/digital-asset/decentralized-canton-sync/tree/cocreature/featured-app-activitymarkers).

## Copyright

This CIP is licensed under CC-1.0.

## Changelog

2025-02-12 - Intial Draft
