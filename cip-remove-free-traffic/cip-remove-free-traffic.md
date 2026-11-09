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

- TODO: confirm that traffic accounting handles a `burstAmount` of 0.
- TODO: specify how the sequencer identifies ACS commitments and the rate limit applied to them.

## Abstract

At the time of writing, every validator receives free base-rate traffic:
a 200 KB burst that refills every 20 minutes.
Across the current network of about 800 validators, this amounts to a subsidy of up to $0.7M per day.

This CIP proposes to remove this subsidy to improve tokenomics.
The removal is phased over four months, so that the ecosystem has time to adjust to the change.
In exchange, ACS commitments become free but rate-limited,
analogous to confirmation responses in [CIP-0104](../cip-0104/cip-0104.md),
so that validators do not pay for this protocol-mandated background traffic.


## Specification

### Free Rate-Limited ACS Commitments

The Canton Protocol is changed so that ACS commitments become free:

1. Change the sequencer to not charge traffic for ACS commitments.
2. Rate limit ACS commitments per validator node to prevent abuse.

As in [CIP-0104](../cip-0104/cip-0104.md#heuristic-implementation),
the change is enabled by a dynamic synchronizer flag
that the SV apps set once all SV nodes have upgraded their sequencers.

### Phased Removal of Free Traffic

Change `decentralizedSynchronizer.fees.baseRateTrafficLimits` in `AmuletConfig`
in two steps, each enacted by a separate SV vote
as specified in [Config Votes](#config-votes):

| When | `burstAmount` | `burstWindow` | Free traffic per validator | Free traffic network-wide | Reduction vs. today |
|---|---|---|---|---|---|
| Today | 200 KB | 20 min | 14.4 MB/day ($864) | 11.52 GB/day ($691,200) | — |
| +7 days | 200 KB | 6 hours | 800 KB/day ($48) | 640 MB/day ($38,400) | −94.4% (18×) |
| +4 months | 0 KB | 6 hours | 0 | 0 | −100% |

Times are measured from CIP acceptance.
Dollar amounts are the subsidy per day.
We assume 800 validators using their full free allowance at $60/MB.

### Config Votes

Each step is enacted by a separate vote,
proposed using the "*Set Amulet Rules Configuration*" vote proposal form.
All other config values remain unchanged.
The votes are created one week ahead of their effective date.

**Step 1: +7 days after acceptance**

- Set `decentralizedSynchronizer.fees.baseRateTrafficLimits.burstWindow`
  from `1200000000` (20 min) to `21600000000` (6 hours).
- Set the effective date to 7 days after CIP acceptance.

**Step 2: +4 months after acceptance**

This vote is only created once [free rate-limited ACS commitments](#free-rate-limited-acs-commitments)
are enabled on MainNet.

- Set `decentralizedSynchronizer.fees.baseRateTrafficLimits.burstAmount`
  from `200000` to `0`.
- Set the effective date to 4 months after CIP acceptance,
  or one week after the vote is created, whichever is later.

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

1. By buying traffic from another validator that they operate.
2. By using a token standard wallet to make a [token standard transfer to the traffic purchase party](https://docs.canton.network/global-synchronizer/deployment/synchronizer-traffic#buying-traffic-with-a-token-standard-transfer).

Given the wide availability of token standard wallets,
we can reasonably assume that every validator operator can
bootstrap their first validator through a token standard wallet.
This assumption also aligns with the DoS protection proposed in [CIP-0128](https://github.com/canton-foundation/cips/pull/274),
where a new validator must have a sufficient traffic balance to connect to the network.

### Phased Rollout

Lowering the refill rate rather than the burst amount
keeps the current recovery path operational during the transition:
200 KB is enough for a validator with no traffic left to buy more.
The phases are chosen as follows:

- **Cut farming:**
  7 days after acceptance, 94% of the subsidy is removed,
  while validators can still recover or bootstrap their traffic balance within 6 hours.
- **Remove free traffic:**
  4 months after acceptance, free traffic is completely removed.
  This gives time to implement and roll out free ACS commitments,
  and for validator operators to set up their traffic recovery strategy.
  The removal is contingent on free ACS commitments being enabled.
  Until then, validators with more than about 15 counter-participants
  already pay for part of their ACS commitments,
  as the 6-hour refill covers only about 800 KB/day.

### Free ACS Commitments

Validators send ACS commitments at regular intervals without any user action,
one to each counter-participant.
Their cost thus grows with network size:
an SV connected to all 800 validators sends about 43.2 MB/day of commitments,
which would cost $2,592/day,
or about $3.24/day per counter-participant.
Today, free traffic covers these costs for most validators.

ACS commitments are required by the protocol, just like confirmation responses.
We therefore make them free and rate-limited,
following the same reasoning as [CIP-0104](../cip-0104/cip-0104.md) for confirmation responses.

Time-proofs, the other periodic background message, are no longer sent with the newest Canton protocol version.

### Validator Operators Pay for Transfer Preapprovals

[CIP-0119](../cip-0119/cip-0119.md) allows creating and renewing preapprovals
using only traffic, including free traffic.
After this CIP, this requires purchased traffic.
This is intended:
validator operators are best placed to decide
whether running their validator is worth its cost.

### Alternatives Considered

- **Remove free traffic immediately.**
  Rejected, as it gives the ecosystem no time to adjust.
- **Lower the burst amount instead of the refill rate.**
  Rejected, as validators with no traffic left could no longer buy more.
- **Keep a small permanent free traffic allocation.**
  Rejected, as any free allocation remains a target for farming
  and its cost grows with the number of validators.
- **Have validators pay for ACS commitments.**
  Rejected, as their cost grows with network size
  and would make running a well-connected validator expensive
  for traffic that is required by the protocol.

## Backwards compatibility

This CIP requires a Canton change to make ACS commitments free,
but no Daml changes.

Validators that do not buy traffic will be unable to submit transactions
once free traffic is removed.
We recommend that validator operators:

1. Enable the validator app's [top-up automation](https://docs.canton.network/global-synchronizer/deployment/synchronizer-traffic#buying-traffic-with-the-validator-app’s-top-up-automation)
   and maintain a sufficient CC balance on the validator to buy traffic when needed.
2. Keep CC for recovery in a wallet hosted on a different validator,
   so that they can buy traffic when their own validator has run out of traffic.

## Reference Implementation

TODO: link the Canton change for free rate-limited ACS commitments.
The removal of free traffic is enacted via SV votes on `AmuletConfig`.

## Copyright

This CIP is licensed under CC0-1.0: [Creative Commons CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)

## Changelog

- 2026-10-28: draft published

## Appendix

Query
