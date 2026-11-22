# Trenchwatch Data Library · Dataset sheet

**Provider:** EMERGE (Barcelona, Spain) · **Product:** Trenchwatch · **Version:** 1.0, November 2026
**Contact:** info@trenchwatch.dev · [trenchwatch.dev](https://trenchwatch.dev)

---

## In one line

A daily, delayed record of new token launches on major Solana launchpads, with launch-time signals that show when early buying comes from linked address groups rather than organic demand, and the on-chain evidence behind each signal.

## What problem it answers

In a new token's first seconds, organic demand and coordinated, non-organic buying look the same in price and volume. The library lets research, risk and data teams study launch activity with that distinction already made, across every launch on the covered launchpads, not a sample.

## Products

| Product | What it is | Delivery |
| --- | --- | --- |
| **Daily library (T+2)** | Every launch and launch-time signal, delivered with a 48-hour delay | Daily file, subscription |
| **Closed historical months** | Complete, final months, delivered after each month closes | One-off purchase per month or bundle |

Real-time alerts are a separate product (the Trenchwatch licence) and are **not** part of the library.

## Coverage

| | |
| --- | --- |
| Chain | Solana |
| Venues | 5 major Solana launchpads (full list available on request) |
| History | Continuous since October 2026, growing every day |
| Scale | 1,127,107 launches read and 9,506,316 early purchases read (as of 17 November 2026) |
| Update frequency | Daily |
| Timestamps | UTC, with on-chain slot and transaction signature |

## Tables

Field names are indicative and may change before the first delivery. Every row can be traced back to public Solana transactions.

**1. `launches`**: one row per new token
- launch time (UTC), slot, launchpad
- token mint, name and symbol
- creator address and, when matched, its address-group ID (`TW-xxxx`)
- launch-time aggregates: number of early purchases, unique early buyers, SOL volume
- share of early buying that comes from linked address groups
- evidence level and risk label (see below)

**2. `signals`**: one row per launch-time signal
- detection time (UTC) and latency from launch
- token mint, address-group ID, signal category
- evidence level
- transaction signatures that support the signal

**3. `address_groups`**: one row per address group
- stable ID (`TW-xxxx`), first seen and last seen
- number of linked addresses and tokens launched
- evidence level
- linked addresses (licensed files only)

**4. `coverage_daily`**: one row per launchpad and day
- launches, unique creators, early purchases, unique buyers, SOL volume

## Evidence levels and risk labels

| Evidence level | Meaning |
| --- | --- |
| RELATED | Linked to an address group already seen in repeat launches |
| COORDINATED | Linked addresses acting together at launch |
| SUSPECTED | Several independent signals point the same way |
| CONFIRMED | Reviewed and approved by a person; never assigned automatically |

API risk labels: HIGH, ELEVATED, LOW, UNKNOWN.

## Formats and delivery

- CSV or Parquet; JSON for the signals table if preferred
- Delivery by SFTP, cloud bucket (S3 or GCS) or a data marketplace
- Sample files from closed days available on request, after a short conversation about the intended use

## Data sources and quality

- **Public data only.** Every input comes from the public Solana ledger. The library contains EMERGE's own derived and aggregated analysis, not raw data resold from any provider.
- **Coverage notes.** During traffic peaks a small share of launches on some venues may not be read in full; these gaps are recorded in `coverage_daily` rather than hidden.
- **Human review.** No address group reaches CONFIRMED without a person approving it.
- **Stable IDs.** `TW-xxxx` IDs do not change once assigned, so files from different days can be joined.

## Principles and limits

- **Addresses, not people.** The library describes addresses and on-chain patterns, never individuals.
- **Not financial advice.** Signals are probabilistic observations, not price predictions or trading recommendations.
- **The method stays private.** The library delivers results and evidence; the detection method is proprietary.

## Typical uses

- Research on launch dynamics and early demand on Solana
- Risk and listing due diligence on new tokens
- Labelling address groups in analytics products
- Backtesting surveillance or risk models on complete, timestamped launch data

## Commercial terms

Licensing terms are shared privately, on request.

---

© 2026 EMERGE. All rights reserved.
