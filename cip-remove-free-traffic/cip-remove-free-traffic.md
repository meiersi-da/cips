<pre>
  CIP: ?
  Title: Remove Free Traffic
  Author: Simon Meier
  Discussions-To: TODO
  Status: Draft
  Type: Tokenomics
  Created: 2026-10-28
  License: CC0-1.0
</pre>

# Remove Free Traffic

**Open items:**

- TODO: confirm that traffic accounting handles a 7-day `burstWindow` and a `burstAmount` of 0.
- TODO: measure the current traffic usage of ACS commitments and time-proofs from metrics.
- TODO: change table to also show individual subsidies

## Abstract

At the time of writing, every validator receives free base-rate traffic:
a 200 KB burst that refills every 20 minutes.
Across the current network of about 800 validators, this amounts to a subsidy of up to $0.7M per day.

This CIP proposes to remove this subsidy to improve tokenomics.
The removal is phased over four months, so that the ecosystem has time to adjust to the change.


## Specification

Change `decentralizedSynchronizer.fees.baseRateTrafficLimits` in `AmuletConfig`
in three steps, each enacted by a separate SV vote
as specified in [Config Votes](#config-votes):

| When | `burstAmount` | `burstWindow` | Network-wide free traffic | Subsidy per day | Reduction vs. today |
|---|---|---|---|---|---|
| Today | 200 KB | 20 min | 11.52 GB/day | $691,200 | — |
| +7 days | 200 KB | 6 hours | 640 MB/day | $38,400 | −94.4% (18×) |
| +1 month | 200 KB | 7 days | 22.9 MB/day | $1,371 | −99.8% (504×) |
| +4 months | 0 KB | 7 days | 0 MB/day | $0 | −100% |

Times are measured from CIP acceptance.
Figures assume 800 validators using their full free allowance at $60/MB.

### Config Votes

Each step is enacted by a separate vote,
proposed using the "*Set Amulet Rules Configuration*" vote proposal form.
All other config values remain unchanged.
The votes are created one week ahead of their effectivity date.

**Step 1: +7 days after acceptance**

- Set `decentralizedSynchronizer.fees.baseRateTrafficLimits.burstWindow`
  from `1200000000` (20 min) to `21600000000` (6 hours).
- Set the effective date to 7 days after CIP acceptance.

**Step 2: +1 month after acceptance**

- Set `decentralizedSynchronizer.fees.baseRateTrafficLimits.burstWindow`
  from `21600000000` (6 hours) to `604800000000` (7 days).
- Set the effective date to 1 month after CIP acceptance.

**Step 3: +4 months after acceptance**

- Set `decentralizedSynchronizer.fees.baseRateTrafficLimits.burstAmount`
  from `200000` to `0`.
- Set the effective date to 4 months after CIP acceptance.

## Motivation

Free traffic harms tokenomics in two ways:

- **Farming.**
  Free traffic is used to submit transactions whose only purpose is to earn app rewards,
  which converts the traffic subsidy into minted CC.
  This is contrary to the goal of app rewards: to incentivize useful activity.
- **Lost revenue.**
  Transactions paid with free traffic burn no CC.
  At full usage and 800 validators, the network forgoes up to $0.7M of burn per day.

Both effects grow linearly with the number of validators and are thus exacerbated as the network scales.


## Rationale

### Bootstrapping Problem Solved

Free traffic was introduced to solve a bootstrapping problem:
buying traffic requires CC, and using CC requires traffic.
This problem is now solved.

Validator operators can bootstrap the traffic balance of a validator in two ways:

1. By purchasing traffic using one of their other validators that they operate.
2. Through token standard wallets using a [token standard transfer to the special traffic purchase party](https://docs.canton.network/global-synchronizer/deployment/synchronizer-traffic#buying-traffic-with-a-token-standard-transfer).

Given the wide availability of token standard wallets,
we can reasonably assume that every validator operator can
bootstrap their first validator through a token standard wallet.
This assumptions also aligns well with the DoS protection proposed in CIP-0128,
where every new validator must actually have a sufficient traffic balance to even connect to the network.

### Phased Rollout

Lowering the refill rate rather than the burst amount
keeps the current recovery path operational during the transition:
200 KB is enough for a validator with no traffic left to buy more.
The phases are chosen for the following rationales:

- **Cut farming:**
  7-days after acceptance 94% of the subsidy is removed,
  while validators can still recover or bootstrap their free traffic balance within 6 hours.
- **Call to action:**
  1 month after acceptance,
  the refill rate is reduced to take 7 days to refill their free traffic balance, signaling the upcoming removal and
  urging operators to set up their traffic recovery strategy.
  A backstop is kept for validators that are used infrequently: they can recover in 7 days without any action.
- **Remove free traffic:**
  4 months after acceptance, free traffic is completely removed.

### Validator Operators Pay for Background Operations

TODO: write this

Validators pay for all traffic they send once free traffic is removed, in particular:

- **ACS commitments and time-proofs.**
  Both are sent at regular intervals.
  ACS commitments are sent to each counter-participant,
  so their cost grows with network size.
- **Transfer preapprovals.**
  [CIP-0119](../cip-0119/cip-0119.md) allows creating and renewing preapprovals
  using only traffic, including free traffic.
  After this CIP, this requires purchased traffic.

### Alternatives Considered

- **Remove free traffic immediately.**
  Rejected, as it gives the ecosystem no time to adjust.
- **Lower the burst amount instead of the refill rate.**
  Rejected, as validators with no traffic left could no longer buy more.
- **Keep a small permanent free traffic allocation.**
  Rejected, as it would undermine the goal of removing free traffic entirely.

## Backwards compatibility

This CIP requires no code or Daml changes.

Validators that do not buy traffic will be unable to submit transactions
once free traffic is removed.
We recommend validator operators to:

1. Enable the validator app's [top-up automation](https://docs.canton.network/global-synchronizer/deployment/synchronizer-traffic#buying-traffic-with-the-validator-app’s-top-up-automation) and maintain a sufficient CC balance on the validator to buy traffic when needed.
2. Keep CC for recovery in a wallet hosted on a different validator,
   so that they can buy traffic when their own validator has run out of traffic.

## Reference Implementation

Not required; the change is enacted via SV votes on `AmuletConfig`.

## Copyright

This CIP is licensed under CC0-1.0: [Creative Commons CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)

## Changelog

- 2026-10-28: draft published
