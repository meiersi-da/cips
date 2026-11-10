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
- TODO: validate the exact sizes of the ACS commitments and their cost during the reduced free traffic phase

## Abstract

At the time of writing, every validator receives free base-rate traffic:
a 200 KB burst that refills every 20 minutes.
Across the current network of about active 800 validators, this amounts to a subsidy of up to $0.7M per day.

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

The rate limit is set such that every validator can submit one ACS
commitment per commitment window per active validator on the network,
where a validator counts as active if it has not fallen behind the sequencer pruning window
(30 days, at the time of writing).

### Phased Removal of Free Traffic

Change `decentralizedSynchronizer.fees.baseRateTrafficLimits` in `AmuletConfig`
in two steps, each enacted by a separate SV vote
as specified in [Config Votes](#config-votes):

| When | `burstAmount` | `burstWindow` | Free traffic per validator | Free traffic network-wide | Reduction vs. today |
|---|---|---|---|---|---|
| Today | 200 KB | 20 min | 14.4 MB/day ($864) | 11.52 GB/day ($691,200) | — |
| +7 days | 200 KB | 200 min | 1.44 MB/day ($86.40) | 1.152 GB/day ($69,120) | −90% (10×) |
| +4 months | 0 KB | 200 min | 0 | 0 | −100% |

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

### Free ACS Commitments

Validators send ACS commitments at regular intervals (every 30 minutes) without any user action,
one to each counter-participant.
An single ACS commitment for a single counter-participant costs about 360 bytes (TODO: traffic cost),
which amounts to 360 bytes * 48 = 17,280 bytes, i.e., about $1 per day per counter-participant.

Requiring validators to pay for ACS commitments would impose a non-trivial cost on them, especially for well-connected nodes with many counter-participants.
Given that ACS commitments are required by the protocol, we propose to treat them like confirmation responses, i.e., as free and rate-limited traffic.

### Phased Rollout

Lowering the refill rate rather than the burst amount
keeps the current recovery path operational during the transition:
200 KB is enough for a validator with no traffic left to buy more.
The rollout has the two phases described below.

#### Phase 1: Cut Farming

7 days after acceptance, the refill window is extended from 20 minutes to 200 minutes,
so that each validator gets 10× less free traffic per day.
This removes 90% of the subsidy,
while validators can still recover or bootstrap their traffic balance within 200 minutes.

The 200-minute refill covers about 1.44 MB/day,
which pays for the ACS commitments to about 80 counter-participants
(at 17,280 bytes per counter-participant per day, see [Free ACS Commitments](#free-acs-commitments)).
Validators with more counter-participants already pay for part of their ACS commitments in this phase.

#### Phase 2: Remove Free Traffic

4 months after acceptance, free traffic is completely removed.
This gives time to implement and roll out free ACS commitments,
and for validator operators to set up their traffic recovery strategy.
The removal is contingent on free ACS commitments being enabled.

#### Impact on Validators Using the Most Free Traffic

The validators on MainNet that use the most free traffic
are run by organizations that also receive app rewards.
They are partially compensated for the loss of free traffic
by the increase in their app rewards,
as less farmed activity competes with theirs for app rewards.

### Validator Operators Pay for Transfer Preapprovals

[CIP-0119](../cip-0119/cip-0119.md) allows creating and renewing preapprovals
using only traffic, including free traffic.
After this CIP, this requires purchased traffic.
This is intended:
validator operators are best placed to decide which preapprovals to provide
value and how long they should remain valid.

### Alternatives Considered

- **Remove free traffic immediately.**
  Rejected, as it gives the ecosystem no time to adjust.
- **Lower the burst amount instead of the refill rate.**
  Rejected, as validators with no traffic left could no longer buy more.
- **Have validators pay for ACS commitments.**
  Rejected, as their cost grows with network size
  and would make running a well-connected validator expensive
  for traffic that is required by the protocol.
- **Keep a small permanent free traffic allocation.**
  Rejected, as any free allocation remains a target for farming
  and it would still not scale to the actual cost of ACS commitments.

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

The removal of free traffic is enacted via SV votes on `AmuletConfig`.

The change to make ACS commitments free and rate-limited is implemented in Canton and does not require any Daml changes.

## Copyright

This CIP is licensed under CC0-1.0: [Creative Commons CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)

## Changelog

- 2026-10-28: draft published

## Appendix

Query
