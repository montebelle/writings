# Five gates for a GeoLift study

*April 2026*

Most synthetic-control studies fail silently. They produce a lift number. They don't tell you whether the synthetic control was actually a valid comparison. The number gets reported; decisions get made; nobody audits the design.

The fix is to make design validity a *binary decision* before any result is computed. Five gates. All five must pass. If any fails, the study is rejected and the team picks new markets or a different method.

Here's the framework I've run over [GeoLift](https://github.com/facebookincubator/GeoLift) studies for enterprise-scale ad-effectiveness work. The gate thresholds are tuned to real-world DMA-week panel data; adjust for your setting.

---

## The five gates

### Gate 1 — Effective donors ≥ 3

Synthetic control weights a pool of untreated markets to approximate the treated market's counterfactual. If one donor dominates (weight = 0.8), the synthetic control *is* that donor, and your "study" is a matched-pair comparison with all the selection bias that implies.

Compute the effective number of donors:

```
N_eff = 1 / Σᵢ wᵢ²
```

This is Herfindahl's inverse — it collapses to the *effective* count of donors contributing meaningfully to the synthetic control. Threshold: **N_eff ≥ 3**.

Why 3: with < 3 effective donors you're essentially matched-pair or matched-triple design. The synthetic control's value (variance reduction across many donors) has evaporated. Published GeoLift guidance cites this threshold; my reading agrees.

### Gate 2 — Top single donor weight ≤ 25%

Complementary to Gate 1. Even with `N_eff ≥ 3`, one donor might carry 60% of the weight and two others split the remaining 40%. The single-donor cap prevents this.

Threshold: **max(w) ≤ 0.25**.

Heuristic, not proven — but empirically, above 0.25 the synthetic control's fit quality is fragile to perturbations of that one donor. Holding it at a quarter keeps the mixture from degenerating.

### Gate 3 — Donor-to-treatment volume ratio ≥ 1.5×

If your treated markets have total volume X, the donor pool should sum to ≥ 1.5 X. Less than that and you're asking a too-small donor pool to reconstruct an outcome trajectory it cannot cover.

This is a convex-hull argument: synthetic control is a convex combination of donors. The treated trajectory has to lie within the hull of donor trajectories to be reconstructable. If donors collectively can't outspan the treated volume, the hull is too small and the pre-period fit degrades.

Threshold: **sum(donor_volume) / sum(treated_volume) ≥ 1.5**.

### Gate 4 — In-time placebo |abs_lift_in_zero| ≤ 2%

Fit your synthetic control on the pre-period, then compute what the model *says* the lift was during the pre-period itself. Should be near zero. If it isn't, the model is capturing spurious signal and your treated-period lift estimate is untrustworthy.

GeoLift exposes this as `abs_lift_in_zero`. Threshold: **|abs_lift_in_zero| ≤ 0.02** (2%).

Stricter than the GeoLift default. 2% is roughly the noise floor we tolerate for a campaign measurement claim; anything above that and the study is disqualified.

### Gate 5 — Pre-fit L2 imbalance ≤ 0.25

Pre-period fit quality, scaled. GeoLift reports `AvgScaledL2Imbalance` — the average L2 distance between synthetic and treated trajectories in the pre-period, normalized by treated-outcome magnitude.

Threshold: **AvgScaledL2Imbalance ≤ 0.25**. GeoLift's own guidance.

Above 0.25 and the synthetic isn't tracking the treated market's history well enough to trust the counterfactual projection.

---

## All five or nothing

The gates are binary and conjunctive. Pass 4 of 5 and the study is still rejected. Teams hate this at first — they want to ship the number — and then they get used to it, because the failure cases are real.

Example from a real run: gates 1–4 all passed cleanly on the original donor pool. Gate 5 failed at L2 = 0.31. The team would have shipped a 6.4% lift estimate that nobody could defend if the pre-fit imbalance had been checked. We ran `MultiCellMarketSelection` with an expanded donor pool; L2 came down to 0.121; all gates passed; the re-run lift was 4.9%. The 1.5-point difference was the cost of not auditing design quality upstream.

## Power aggregation over lookback windows

Orthogonal to the gates but related: don't run power analysis on a single pre-period length. A design that has 80% power at 10 weeks lookback can have 55% at 6 weeks. Run `MultiCellMarketSelection` with aggregation over 11 lookback windows (≈ 1/10 of the pre-treatment period rule) and report the power curve, not a point estimate.

## Bias-aware fork

Base GeoLift from Meta is good. For the gate-heavy workflow above I use a bias-aware fork that adds the five gates as hard pre-conditions. Studies that fail any gate return `status: rejected` with the failing gate named, rather than a lift number.

This seems obvious. It is not the default in most analytics teams.

## When to relax the gates

- **Emergency measurement.** If a CMO needs a lift estimate by Friday and the gates fail, relax to a smaller study with tighter constraints rather than reporting a bad number from a bad design.
- **Exploratory / hypothesis-generating work.** Gates can be weaker (e.g., N_eff ≥ 2, L2 ≤ 0.35) when the output is "should we invest in a bigger study" rather than "what's the lift." Label clearly.

## What the gates don't catch

- **Treatment interference across geographies** — if treated DMAs' marketing affects untreated ones (spillover), synthetic control is biased regardless of fit quality.
- **Selection of the treated markets themselves** — if the team picked markets where they expected the campaign to work, the estimated effect is not the ATE on a random market.
- **Time-varying confounders** that affect treated but not donor markets during the treatment period.

These need separate design discipline (pre-registration, randomization over eligible markets, placebo periods) — the five gates handle fit-quality validity, not causal identification at a deeper level.

## References

- Abadie, A., Diamond, A., Hainmueller, J. (2010). "Synthetic control methods for comparative case studies." *JASA*, 105(490).
- Meta / Facebook Incubator. "GeoLift: An end-to-end geo-experiment framework." https://github.com/facebookincubator/GeoLift
- Ben-Michael, E., Feller, A., Rothstein, J. (2021). "The augmented synthetic control method." *JASA*, 116(536).
- Arkhangelsky, D. et al. (2021). "Synthetic difference-in-differences." *American Economic Review*, 111(12).
