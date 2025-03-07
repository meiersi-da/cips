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



Currently, featured applications can only generate activity records
and mint rewards as part of Canton Coin transfers. However, this
excludes a significant amount of applications that do not inherently involve Canton Coin.
To address this problem and allow rewarding applications that do not
involve Canton Coin, we propose introducing the ability for featured applications to create
app activity markers without transfering Canton Coin. An app activity
marker is equivalent to the existing app activity records created as
part of a Canton Coin transfer recording a fixed amount of burned CC. The value of this marker
will be determined by a new governance parameter.

## Copyright

This CIP is licensed under CC0-1.0: [Creative Commons CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)

## Motivation

Define standard APIs for Canton Network tokens so that wallets and apps can use
them and build on them in a uniform way.


## Specification

### Overview

This standard is concerned with three kinds of applications:

- **asset registries**:
  which are used by registrars to manage the ownership records of Canton Network
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

- **privacy**: asset holdings and transfers are by shared on a need-to-know basis
- **control**: registries have full control over the workflows governing asset holdings and transfers

#### UTXO Access Management

Canton manages the state of its ledger using an extended Unspent-Transaction-Output (UTXO) model.
There is a one-to-one correspondence between Daml contracts and UTXOs.
Canton's UTXOs are annotated with their stakeholders and are only distributed to the nodes hosting these stakeholders.

Constructing transactions requires access to all UTXOs referenced or consumed by the transaction.
Clients provide this access by retrieving the UTXOs known to their parties from their validator node and
the UTXOs known to an app provider using API calls to app-specific services.

The standard assumes that UTXO access is provided via:

- **wallet access to user parties**:
  wallets have access to the Ledger API of the validator node hosting the parties of their users
  and use that to retrieve all UTXOs known to these parties.
- **registry off-ledger APIs**:
  registries serve UTXOs private to the registry operator via standardized HTTP APIs specified using OpenAPI.


#### Free of Payment (FOP) Transfer Workflows

The FOP workflow used by the standard is a straightforward linear flow:

- investor specifies transfer recipient, amount, input holdings, deadline for execution
- wallet client retrieves required UTXOs from registry
- wallet client initiates transfer
- depending on registry
  - specified amount of holdings gets transferred to the recipient immediately
  - transfer instruction gets created
-

Considerations:

- opt-in for receiving token
- abortion of transfer instruction



#### Delivery versus Payment (DVP) Transfer Workflows





#### Wallet Client / Portfolio View



- wallet as the central interaction gateway





#### Canton Coin Implementation

- implements all APIs
- limitation: 1 min delay between prepare and submit


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
