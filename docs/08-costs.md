# Cost Analysis: Self-Hosting vs. Streaming

Is building and running your own media server actually cheaper than simply
paying for streaming subscriptions? This chapter documents the real costs of
this build and compares them against the streaming services that would be
needed to replace it.

The reference scenario is a household that wants **up to five people streaming
simultaneously**. All figures are in **Swiss Francs (CHF)**.

!!! note "Pricing date"
    Streaming prices were last verified in **July 2026** using current Swiss
    retail prices. Providers change their plans and prices frequently, so treat
    these numbers as a snapshot rather than a fixed truth.

## Self-Hosted Media Server

### One-Time Costs

| Item | Cost (CHF) |
| --- | ---: |
| 2× Seagate IronWolf 12 TB (April 2025) | 492.00 |
| Seagate IronWolf 4 TB (August 2024) | 99.00 |
| DIY media server build (Jan 2026) | 577.60 |
| Plex Pass (Lifetime, December 2024 + trial monthly subs from months before) | 108.63 |
| English indexer | 40.00 |
| **Total** | **1 317.23** |

### Recurring Monthly Costs

| Item | Cost (CHF) |
| --- | ---: |
| German indexer (CHF 15/year) | 1.25 |
| Usenet access | 5.78 |
| VPS | 3.25 |
| Electricity (≈ 60 W → 31 kWh × CHF 0.33/kWh) ¹ | 10.23 |
| **Total** | **20.51** |

¹ This has been measured over a period of more than a month. During this period the server had few idle time and the measurement can be taken as a worst case estimate.

## Streaming Alternative

To match the content and the five-simultaneous-stream requirement, the
following subscriptions would be needed. Netflix and Disney+ only reach five
concurrent streams by adding a paid **extra member** on top of their four-stream
Premium plans; the remaining services top out at three to four streams.

| Service | Plan (for up to 5 simultaneous streams) | Monthly (CHF) |
| --- | --- | ---: |
| Netflix | Premium (4 streams) + 1 extra member | 30.80 |
| Disney+ | Premium (4 streams) + 1 extra member | 28.80 |
| Prime Video | Standard (3 streams) | 9.99 |
| Sky Show | Premium (4 streams) | 17.90 ² |
| Paramount+ | Premium (4 streams) | 17.90 |
| **Total** | | **105.39** |

² Sky Show Premium drops by CHF 4.00 to **CHF 13.90/month** after the first six
months, bringing the total down to **CHF 101.39/month** thereafter.

## Break-Even Analysis

The one-time investment of CHF 1 317.23 pays for itself once the money *not*
spent on streaming exceeds the server's own running costs. How long that takes
depends on how many streaming services the server actually replaces.

| Replaced streaming services | Break-even |
| --- | ---: |
| Netflix only | ≈ 10.7 years |
| Netflix + Disney+ | ≈ 2.8 years |
| All five services | ≈ 1.3 years |

In other words: if the server merely replaces a single Netflix subscription,
the savings are marginal. But as a replacement for the full stack of services
required to satisfy five simultaneous viewers, it pays for itself in a little
over a year.

## Effective Monthly Cost by Runtime

Amortising the one-time investment over the server's lifetime gives its true
effective monthly cost (one-time cost spread across the runtime, plus the
recurring monthly costs). The longer it runs, the cheaper it gets:

| Runtime | Effective monthly cost (CHF) |
| --- | ---: |
| 2 years | 75.39 |
| 3 years | 57.10 |
| 5 years | 42.46 |
| 7 years | 36.19 |

Even at a pessimistic two-year lifespan, the effective monthly cost stays below
the CHF 105.39 streaming bill; over five years it drops to well under half.

## Additional Comment

We have to mention here that prices of components and the Plex subscription have changed. The component prices have to be accepted, however the Plex subscription is not needed since there is [Jellyfin](https://jellyfin.org/) as an alternative. But what we need to keep in mind is that if you built a server with a good CPU and enough fast Memory there are way more than 5 simultaneous and 4K streams possible as well, these features would all have to be paid additionally when using streaming services. The main invest is your time but once the server is running and automated there is not much for you to do.