<pre>
  CIP: ?
  Layer: TODO
  Title: Remove Free Traffic
  Author:
    Simon Meier
  Discussions-To: TODO
  Status: Draft
  Type: Tokenomics
  Created: 2026-10-28
  License: CC0-1.0
</pre>

# CIP ?: Remove Free Traffic

## Abstract

TODO: ~200 word summary. Cover:

- what free traffic is today
- what this CIP removes
- how the previously free operations are paid for instead

## Specification

### Overview

TODO

### Replacing Free Traffic for Bootstrapping Workarounds

TODO: for each category, specify who pays, how, and how much.

- ACS commitments: TODO
- Time-proofs: TODO
  - Synchronizer fees paid by SVs: TODO
- Topology transactions: TODO

### Configuration Changes

TODO

### Roll-Out Plan

TODO

## Motivation

Free traffic should be removed to stop subsidy farming, align incentives, and make charges predictable.

- **Stop farming the subsidy.** Free traffic rewards participants for consuming it rather than for useful activity. Removing it aligns incentives and avoids misincentives.
- **It does not scale.** Free traffic does not grow with the network, but its cost does. Its cost therefore scales with network size and must eventually be zero.
- **It was a workaround.** Free traffic was introduced to solve bootstrapping issues for:
  - ACS commitments
  - time-proofs (and the related synchronizer fees paid by SVs)
  - topology transactions
- **Predictability.** Participants should know how much they will be charged.

## Rationale

TODO

- Alternatives considered: TODO

## Backwards compatibility

TODO

- Impact on validators: TODO
- Impact on SVs: TODO
- Impact on apps and wallets: TODO

## Reference Implementation

TODO

## Copyright

This CIP is licensed under CC0-1.0: [Creative Commons CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)

## Changelog

- 2026-10-28: draft published
