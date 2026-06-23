# Discrete-time cloglog for incrementality at 10M-user scale

*April 2026*

Incrementality measurement in digital advertising is a causal-inference problem wearing a regression trenchcoat. The nominal model (click → conversion) is censored, the treatment assignment is not random, the outcome is time-to-event rather than binary, and the scale is tens of millions of users. Standard last-touch attribution is a story told about this problem, not a measurement of it.

The apparatus that works — I've run it across 33.6M unique users, 7.3M conversions, 7 discrete time periods spanning ~6 months — is discrete-time hazard modeling with complementary log-log link, propensity-score-balanced doubly-robust estimation, and Rosenbaum sensitivity bounds on top.

Here's what each piece does and why the stack is the right shape.

---

## Why discrete time (not Cox PH)

Cox proportional hazards is the default when people hear "survival." It assumes continuous time, proportional hazards, and the event indicator is all you observe. Digital marketing data violates every one of these:

- **Time is not continuous.** You observe user-period snapshots — daily, weekly, biweekly. Pretending otherwise biases the partial likelihood.
- **PH is frequently violated.** Ad effectiveness decays with exposure cumulative count; the hazard ratio is not constant over time. A covariate × time interaction test shows clear time-varying effects.
- **The panel is huge.** 33.6M users × 7 periods = 235M person-period rows. Cox's baseline-hazard estimation doesn't scale to this cleanly.

Discrete-time hazard with complementary log-log link handles all three:

```
log(-log(1 - h(t | X))) = α_t + β · X
```

- `α_t` is a per-period intercept, non-parametric baseline hazard — no PH assumption, no continuous-time lie.
- cloglog (vs logit) is the correct link under discrete-time grouping of an underlying continuous-time Cox process.
- The model fits as binomial GLM with cloglog link on person-period data. L2-penalize the β coefficients, tune penalty via k-fold cross-validation. Done.

## Covariate specification

Three classes of covariates:

**Period dummies (`α_t`)** — 6 dummies for 7 periods, first period as reference. These are the non-parametric baseline hazard.

**Cumulative exposure bucketed** — not continuous. Bucket at 0 / 1–3 / 4–7 / 8+ exposures. Continuous treatment variables in hazard models invite functional-form misspecification; bucketing is coarser but more defensible.

**Behavioral cohort** — `buyer_lifecycle` with 5 levels (new-category-entrant / light / moderate / heavy / win-back). This is the dominant predictor. At 33.6M scale, the heavy-buyer cohort runs ≈ 51× the baseline hazard of the new-entrant cohort. Any incrementality model that ignores this interaction will severely overstate effects on new entrants.

**Interactions** — exposure × lifecycle if you want to know *which* cohorts respond. Exposure × time if you want to see decay. Both matter for budget allocation.

---

## Nonparametric diagnostics

Before you ship a cloglog β̂, check that the model isn't lying about its baseline hazard:

**Kaplan-Meier with Greenwood variance.** Compute the survival function S(t) = Π(1 - h_t) non-parametrically. Compare against the cloglog-implied S(t). If they disagree materially, something's wrong with the parametric baseline. At 33.6M scale, compute Greenwood variance in log-space to avoid numerical issues.

**Nelson-Aalen cumulative hazard.** Complementary to KM; often more stable in the tail. Use as a sanity check on the shape of the hazard trajectory.

**Covariate × time interaction.** The PH test used here: add a `covariate × time` term and test its coefficient. A significant interaction means that covariate's hazard ratio is changing over time — the cloglog spec with time-invariant β for that covariate is mis-specified — so keep the interaction in the fit. (Schoenfeld residual plots against time are the classical equivalent diagnostic.)

**Log-log parallelism plots.** Visual PH check. Plot `log(-log(S(t)))` for each subgroup. Parallel curves = PH holds for that stratification. Non-parallel = stratify the model.

---

## The causal layer

The cloglog model gives you *associational* hazard ratios. Treated users who see more ads convert faster — but those users aren't random. They might be users already more likely to convert (selection into treatment). That's correlation; incrementality is causal.

### Propensity-score doubly-robust estimation

1. Fit a propensity model: `P(treated | X)` where X is pre-treatment covariates. Use any flexible binary classifier.
2. Cross-fit the propensity scores using 5-fold splitting — train on 4 folds, predict on the held-out fold, rotate. This is the critical step that avoids over-fitting-induced bias in the treatment-effect estimate.
3. Compute inverse-probability weights (IPW): `w_i = 1 / P̂(T_i | X_i)`.
4. Separately, fit an outcome model: `Ê[Y | T, X]`.
5. Combine via the AIPW (Augmented IPW, "doubly robust") estimator:

```
τ̂_DR = (1/n) Σ [ Ê[Y(1) | X_i] − Ê[Y(0) | X_i]
                 + T_i · (Y_i − Ê[Y(1) | X_i]) / P̂(T | X_i)
                 − (1 − T_i) · (Y_i − Ê[Y(0) | X_i]) / (1 − P̂(T | X_i)) ]
```

The doubly-robust property: `τ̂_DR` is consistent if *either* the propensity model OR the outcome model is correctly specified. You don't have to be right about both. Most real-world models are wrong in different ways; DR gets you consistency if you're right in at least one place.

### Trimming for positivity

Positivity: every unit must have `P(treated | X)` strictly between 0 and 1. Propensity scores near 0 or 1 produce explosive IPW weights that dominate the estimate.

[Crump et al. (2009)](https://doi.org/10.1093/biomet/asn055) trimming: drop units with `P̂ < 0.01` or `P̂ > 0.99`. This changes the estimand (you're now estimating the effect on the trimmed population, not the full one) but that's an honest tradeoff against the alternative (infinite-variance estimator).

### Balance diagnostics post-matching

After cross-fitted weighting or matching, check that your covariates are balanced across treated and control. Standardized mean difference per covariate should be < 0.1. If any covariate is imbalanced, the weighting hasn't worked and the DR estimator isn't doing its job.

### Rosenbaum sensitivity bounds

The scary assumption of every observational causal estimator is "no unmeasured confounders." You can never verify this. What you *can* do is ask: how bad would an unmeasured confounder have to be to flip my result?

[Rosenbaum's sensitivity analysis](https://www.jstor.org/stable/2286054): compute the smallest Γ such that an unmeasured confounder with prevalence-odds-ratio Γ (conditional on observed X) would null out your effect. Report Γ alongside the estimate. Γ = 1.2 means a weak unmeasured confounder could explain your effect; Γ = 3.0 means a very strong one would be required.

---

## At 33.6M scale — what you actually run into

**Numerical stability.** KM's Greenwood variance `Σ d_t / (n_t · (n_t - d_t))` can overflow. Compute in log-space.

**Memory.** Don't materialize the 235M-row person-period matrix. Stream by period.

**CV folding.** 5-fold CV is fine for the propensity model. Don't use leave-one-out at this scale.

**Interactions blow up.** Cumulative exposure × buyer_lifecycle is 5 bucket × 5 cohort = 25 columns of interaction dummies. Keep the penalizer on.

**Minimum matched pairs.** After trimming + balance checks, enforce `n_matched ≥ 100` per stratum. Below that the estimator is too noisy to report.

---

## What you get

At 33.6M users: concordance index 0.97 on the cloglog model itself. Strongest predictor is `buyer_lifecycle` — heavy buyers have ≈ 51× baseline hazard vs new entrants. Ad exposure produces HR 1.10–1.13 for light/moderate buyers, HR ≈ 0.98 (null) for new buyers, and HR ≈ 1.13 for heavy buyers. Period interaction grows from HR 0.99 in period 1 to ≈ 7.0 in period 7, i.e. exposure effects compound over time.

ROAS-optimized campaigns outperform DPVR at 2.4× per dollar on incrementality, but DPVR captures 74% of the budget — the allocation is misaligned with the causal evidence. That's the finding that matters; the hazard ratios are the machinery that makes it defensible.

## References

- Allison, P.D. (1982). "Discrete-time methods for the analysis of event histories." *Sociological Methodology*, 13.
- Crump, R.K., Hotz, V.J., Imbens, G.W., Mitnik, O.A. (2009). "Dealing with limited overlap in estimation of average treatment effects." *Biometrika*, 96(1).
- Hernán, M.A., Robins, J.M. (2020). *Causal Inference: What If*. — the canonical text on AIPW + DR.
- Rosenbaum, P.R. (2002). *Observational Studies* (2nd ed.).
