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
- TODO: determine the exact upper bound on the ACS commitment message size (or change the rate limit formulation),
  and validate the resulting cost during Phase 1.
- TODO: quantify the amount of farming going on.
- TODO: check the traffic cost of validator license updates;
  if material, consider keeping a small free allocation for them instead of setting `burstAmount` to 0.
- TODO: quantify whether the increase in app rewards from cutting farming outweighs the interim ACS commitment cost
  for validators with more than about 80 counter-participants during Phase 1.

## Abstract

At the time of writing, every validator receives free base-rate traffic:
a 200 KB burst that refills every 20 minutes.
Across about 800 active validators, this is a subsidy of up to $0.7M per day.
It is farmed for app rewards, burns no CC,
and grows with every validator, rewarding running validators just to collect it.

Its original purpose, bootstrapping a validator's traffic balance,
is now served by token standard wallets.
This CIP removes the subsidy in two steps, giving the ecosystem time to adjust:
90% is removed 7 days after acceptance, and the rest after four months.
In exchange, ACS commitments become free but rate-limited
(like confirmation responses in [CIP-0104](../cip-0104/cip-0104.md)),
as they are protocol-mandated background traffic.

## Specification

### Free Rate-Limited ACS Commitments

The Canton Protocol is changed so that ACS commitments become free:

1. The sequencer does not charge traffic for ACS commitments.
2. The sequencer rate-limits ACS commitment bytes per validator to prevent abuse.

The rate limit per commitment interval is the upper bound on the size of one ACS commitment message
(about 360 bytes on MainNet) times the number of active validators,
i.e., enough for one ACS commitment to every active validator, but no more.
A validator is active if it has not fallen behind the sequencer pruning window
(30 days at the time of writing).

### Phased Removal of Free Traffic

Change `decentralizedSynchronizer.fees.baseRateTrafficLimits` in `AmuletConfig`
in two steps, each enacted by a [separate SV vote](#config-votes):

| When | `burstAmount` | `burstWindow` | Free traffic per validator | Free traffic network-wide | Reduction vs. today |
|---|---|---|---|---|---|
| Today | 200 KB | 20 min | 14.4 MB/day ($864) | 11.52 GB/day ($691,200) | — |
| +7 days | 200 KB | 200 min | 1.44 MB/day ($86.40) | 1.152 GB/day ($69,120) | −90% (10×) |
| +4 months | 0 KB | 200 min | 0 | 0 | −100% |

Times are from CIP acceptance.
Dollar amounts are daily subsidies, assuming 800 validators use their full free allowance at $60/MB.

### Config Votes

Both votes use the "*Set Amulet Rules Configuration*" vote proposal form,
leave all other config values unchanged,
and are created one week ahead of their effective date.

**Step 1: +7 days after acceptance**

- Set `decentralizedSynchronizer.fees.baseRateTrafficLimits.burstWindow`
  from `1200000000` (20 min) to `12000000000` (200 min).
- Set the effective date to 7 days after CIP acceptance.

**Step 2: +4 months after acceptance**

This vote is only created once [free rate-limited ACS commitments](#free-rate-limited-acs-commitments)
are enabled on MainNet.

- Set `decentralizedSynchronizer.fees.baseRateTrafficLimits.burstAmount`
  from `200000` to `0`.
- Set the effective date to 4 months after CIP acceptance,
  or one week after the vote is created, whichever is later.

## Motivation

Free traffic harms tokenomics in three ways:

- **Farming.**
  Free traffic is used for transactions whose only purpose is to earn app rewards,
  converting the subsidy into minted CC.
  This dilutes the rewards of apps with real utility,
  contrary to the goal of app rewards: to incentivize useful activity.
- **Unpriced resource usage.**
  Free traffic consumes sequencer capacity but burns no CC:
  up to $0.7M per day at full usage.
- **Sybil incentive.**
  Every validator gets the same allowance,
  so running additional validators just to collect it pays off.

Meanwhile, its original justification, bootstrapping new validators,
[no longer holds](#bootstrapping-problem-solved).

## Rationale

### Bootstrapping Problem Solved

Free traffic was introduced to solve a bootstrapping problem:
buying traffic requires CC, and using CC requires traffic.
This problem is now solved.

Validator operators can bootstrap the traffic balance of a validator in two ways:

1. By buying traffic from another validator that they operate.
2. By using a token standard wallet to make a [token standard transfer to the traffic purchase party](https://docs.canton.network/global-synchronizer/deployment/synchronizer-traffic#buying-traffic-with-a-token-standard-transfer).

Token standard wallets are widely available,
so every operator can bootstrap their first validator this way.
This matches the DoS protection proposed in [CIP-0128](https://github.com/canton-foundation/cips/pull/274),
which requires new validators to have sufficient traffic to connect to the network.

### Free ACS Commitments

Validators send ACS commitments every 30 minutes without user action,
one to each counter-participant.
At about 360 bytes each, this is 360 bytes × 48 = 17,280 bytes,
or about $1, per day per counter-participant.

Paying for them would be costly for well-connected validators,
although the protocol requires them.
We therefore treat them like confirmation responses: free and rate-limited.

### Phased Rollout

Lowering the refill rate rather than the burst amount
keeps the recovery path working during the transition:
200 KB is enough for a validator without traffic to buy more.

#### Phase 1: Cut Farming

Phase 1 sends a clear signal: farming free traffic stops paying off now, not in four months.
7 days after acceptance, the refill window grows from 20 to 200 minutes.
This cuts 90% of the subsidy,
while validators can still recover or bootstrap within 200 minutes.

The remaining 1.44 MB/day covers the ACS commitments to about 80 counter-participants
(see [Free ACS Commitments](#free-acs-commitments)).
Validators with more counter-participants pay for the rest until free ACS commitments are live.
These are typically app validators, which also gain the most from less farming.

#### Phase 2: Remove Free Traffic

Four months after acceptance, free traffic is removed completely,
but not before free ACS commitments are live on MainNet.
This gives time to roll out free ACS commitments
and for validator operators to set up their traffic recovery.

### Validator Operators Pay for Transfer Preapprovals

[CIP-0119](../cip-0119/cip-0119.md) allows creating and renewing preapprovals
using only traffic, including free traffic.
After this CIP, this requires purchased traffic.
This is intended:
validator operators are best placed to decide which preapprovals are worth paying for, and for how long.

### Alternatives Considered

- **Remove free traffic immediately.**
  Rejected, as it gives the ecosystem no time to adjust.
- **Wait for free ACS commitments before reducing free traffic.**
  Rejected, as farming would continue for months,
  while the well-connected validators bearing the interim ACS commitment cost
  are expected to gain more than they pay.
- **Lower the burst amount instead of the refill rate.**
  Rejected, as validators need the burst to absorb traffic spikes during onboarding or traffic recovery.
- **Have validators pay for ACS commitments.**
  Rejected, as their cost grows with network size,
  making well-connected validators expensive to run for protocol-required traffic.
- **Keep a small permanent free traffic allocation.**
  Rejected, as any free allocation remains a target for farming.

## Backwards Compatibility

Once free traffic is removed, validators that do not buy traffic cannot submit transactions.
We recommend that validator operators:

1. Enable the validator app's [top-up automation](https://docs.canton.network/global-synchronizer/deployment/synchronizer-traffic#buying-traffic-with-the-validator-app’s-top-up-automation)
   and keep enough CC on the validator to buy traffic.
2. Keep CC for recovery in a wallet on a different validator,
   to buy traffic when their own validator has none left.

## Reference Implementation

The removal of free traffic is enacted via SV votes on `AmuletConfig`.
Free rate-limited ACS commitments are implemented in Canton.
No Daml changes are required.

## Copyright

This CIP is licensed under CC0-1.0: [Creative Commons CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)

## Changelog

- 2026-10-28: draft published
