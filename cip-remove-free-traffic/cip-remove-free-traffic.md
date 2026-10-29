<pre>
  CIP: ?
  Layer: Canton, Daml
  Title: Remove Free Traffic
  Author:
    Simon Meier
  Discussions-To: TODO
  Status: Draft
  Type: Tokenomics
  Created: 2026-10-28
  License: CC0-1.0
</pre>

# Remove Free Traffic

## Abstract

This CIP proposes the removal of free traffic in the Canton network.
The current free traffic configuration grants each validator up to 200,000 bytes of free traffic per 20-minute window,
which amounts to subsidizing each validator's traffic cost up to $864 per day.

This subsidy solved an important bootstrapping problem for validator operators:
purchasing traffic requires holding CC which in turn require operating a validator
with a positive traffic balance.
Given that nowadays CC wallets are widely available and
traffic can be purchased with a token standard transfer to `splice_traffic-manager::1220000000000000000000000000000000000000000000000000000000000000abcd` this cyclic bootstrapping problem no longer exists.


a simple transfer to
the `splice_tra

was important during the bootstrapping phase of the network when
it was difficult for validator operators to purchase traffic for their validators.
This difficult is no longer present since in



This subsidy is

The removal is motivated by


This subsidy is neither sustainable with the growing number of validators
nor is it required anymore.


At 800 validators this amounts to a

to subsidizing up to $0.7M per day (11.52 GB/day), assuming full usage.



Free traffic currently allows

validators to consume a certain amount of network traffic without incurring costs.
While this was initially introduced to facilitate bootstrapping and ease of use,
it has led to subsidy farming and misaligned incentives.
By removing free traffic,
all network participants will pay for the traffic they consume,
ensuring fair compensation for validators
and predictable costs for all parties.

TODO: ~200 word summary. Cover:

- what free traffic is today
- what this CIP removes
- how the previously free operations are paid for instead
- validator operators can buy traffic from any token standard wallet (see [Buying Traffic](#buying-traffic))

## Specification

### Overview

TODO

### Buying Traffic

Validator operators can buy traffic for their validator
(or any other synchronizer member)
from any token standard wallet,
without validator app automation,
by making a CC transfer:

- Receiver: the traffic purchase party (TODO: confirm final party ID)
- Amount: the CC to spend on traffic
- Memo: the `splice.lfdecentralizedtrust.org/reason` metadata value, of the form
  `splice/memo:member-id=<member>&synchronizer-id=<synchronizer>&migration-id=<int>`
  (optional: `forfeit-validator-reward-coupon=true|false`)

The purchased traffic must be at least `minTopupAmount` bytes
(currently `200000`, i.e. 0.2 MB, about $12 at $60/MB).
See the [traffic documentation](https://github.com/canton-network/cf-docs/pull/1880/changes)
for the exact format,
price computation,
and failure modes.

TODO: link the final docs page once merged.

### Replacing Free Traffic for Bootstrapping Workarounds

TODO: for each category, specify who pays, how, and how much.

- ACS commitments: TODO
- Time-proofs: TODO
  - Synchronizer fees paid by SVs: TODO
- Topology transactions: TODO

### Configuration Changes

Remove the free base-rate traffic by setting in `AmuletConfig`:

- `decentralizedSynchronizer.fees.baseRateTrafficLimits.burstAmount` from `200000` to `0`

Leave `burstWindow` (`1200000000μs`),
`extraTrafficPrice`,
`readVsWriteScalingFactor`,
and `minTopupAmount` unchanged.

Validators then rely on [buying traffic](#buying-traffic)
(top-up automation or a token standard transfer with a memo).

TODO: confirm the traffic accounting accepts `burstAmount = 0`,
and decide whether to phase the reduction in steps.

### Roll-Out Plan

TODO

## Motivation

Free traffic should be removed to stop subsidy farming,
align incentives,
and make charges predictable.

- **Stop farming the subsidy.**
  Free traffic rewards participants for consuming it rather than for useful activity.
  Removing it aligns incentives and avoids misincentives.
- **It does not scale.**
  Free traffic does not grow with the network,
  but its cost does.
  Its cost therefore scales with network size
  and must eventually be zero.
  - Current base rate: 200,000 bytes per 20 minutes,
    i.e. 14.4 MB/day per validator,
    worth $864/day at $60/MB.
  - With ~800 validators this is up to ~$0.7M/day (11.52 GB/day),
    assuming full usage.
- **It was a workaround.**
  Free traffic was introduced to solve bootstrapping issues for:
  - ACS commitments
  - time-proofs (and the related synchronizer fees paid by SVs)
  - topology transactions
- **Predictability.**
  Participants should know how much they will be charged.

## Rationale

TODO

- Alternatives considered: TODO

## Backwards compatibility

TODO

- Impact on validators: TODO;
  operators can buy traffic from any token standard wallet via a memo transfer,
  see [Buying Traffic](#buying-traffic)
- Impact on SVs: TODO
- Impact on apps and wallets: TODO

## Reference Implementation

TODO

## Copyright

This CIP is licensed under CC0-1.0: [Creative Commons CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)

## Changelog

- 2026-10-28: draft published
