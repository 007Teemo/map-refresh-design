# Adaptive Google map Refresh Prioritization

*A two-tier, demand-aware system for deciding which parts of a live map get updated, and how often.*

**Author:** Dinesh Panwar · dpanwar0088@gmail.com

---

## The Problem

A global map is too large to keep uniformly fresh. Re-scanning imagery, re-verifying roads, and re-checking points of interest are expensive operations, and applying them everywhere at the same frequency wastes resources on areas that rarely change (deserts, low-traffic rural roads) while under-serving areas that change constantly (new developments, dense urban cores).

The real question isn't *"how do we keep the whole map current"* — it's: **given a fixed refresh budget, which regions should be checked more often, and which category of information actually needs that treatment?**

This isn't presented as an unsolved problem — commercial map providers already separate live conditions from base map data in some form. This is a personal design exercise: independently reasoning through the architecture, stress-testing it, and correcting it as flaws turned up.

## Core Idea: Split by Data Type First

Map data isn't one thing. It splits into two categories with very different cost profiles:

| | Static (roads, buildings, POIs) | Dynamic (traffic, weather, open/closed) |
|---|---|---|
| **Cost to verify** | Expensive — imagery, field data, records | Cheap — sensor feeds, live APIs |
| **How often it actually changes** | Rarely | Constantly |
| **Treatment** | Slow baseline sweep, guarantees a staleness ceiling | Needs to be near-live, but *only where it matters* |

## Two-Tier Architecture

**System 1 — Baseline Sweep (the floor)**
Every region on Earth is checked on a slow, low-cost cycle — satellite/imagery diffing, permit records, crowd-sourced edits — regardless of activity. This guarantees nothing goes permanently stale, even in areas nobody happens to be using right now.

**System 2 — Active Zone Refresh (the accelerator)**
Scoped to dynamic data only. Wherever there are active users right now, live data gets prioritized around them — but the mechanism differs by data type, because a one-size-fits-all zone turned out to be the wrong model:

- **Traffic** — a speed-scaled circle, biased toward a longer lookahead (~8 min), since it's genuinely about what's ahead on the route. Radius scales with current speed (`x = clamp(speed × lookahead, min, max)`), because a flat radius under-covers highway driving and over-covers city driving.
- **Open/closed status** — *query-triggered*, not proximity-based. Searching "best restaurants near B" from city A should return live status for places at B, not wherever the user currently is. This replaced an earlier, weaker version of the design that used a fixed radius around the user — which fails exactly this case.
- **Weather** — pulled out of the zone system entirely. It's a coarse regional field that doesn't get more accurate per-user, so it's served as a simple always-on regional feed.

**Design decision worth calling out:** zones are deliberately *symmetric*, not biased toward direction of travel. A direction-biased zone would give better lookahead on predictable routes, but breaks the moment a user takes a U-turn or an unplanned exit — a prediction-free symmetric circle has no failure mode there. Since dynamic data is cheap and fast to refresh, the lost lookahead is an acceptable tradeoff for that robustness.

**Overlapping zones are merged** before dispatching a refresh job — 10,000 people on the same highway shouldn't trigger 10,000 separate jobs.

## What Changed Through Iteration

This design went through several real revisions, not just polish:

1. Started as a static historical "hotspot" score (users × change frequency) — found to have a gap: a route between a hot zone and a cold zone leaves the traveler exposed to stale data mid-journey.
2. Moved to a live, presence-triggered active zone instead of a historical score.
3. Initially considered biasing zones toward direction of travel for lookahead — rejected once the U-turn failure mode was identified.
4. Initially gave open/closed status a small fixed radius — rejected once it became clear that misses the actual use case (searching for a place at your *destination*, not your current location). Replaced with query-triggered fetching.

## Status

This is a personal design exercise, not a proposal submitted to any company — written to practice systems thinking and documented here as a portfolio artifact.

---

*Full design doc with cost model, parameter derivations, and edge case analysis available on request.*
