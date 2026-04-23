# Duan smearing vs Jensen's inequality for log-scale forecasts

*April 2026*

If you fit a regression on `log(y)` and then exponentiate the predictions to get back to the original scale, your forecasts are biased low. That's a fact of Jensen's inequality: for a convex function `f`, `E[f(X)] ≥ f(E[X])`. When you exponentiate the mean of `log(y)` you're not getting the mean of `y` — you're getting the geometric mean, which is always ≤ the arithmetic mean.

People know this. Most production forecasting code handles it by applying a Jensen correction:

```
ŷ = exp(μ̂ + σ̂² / 2)
```

where `σ̂²` is the residual variance of the log-scale model. Simple, closed-form, and — for well-behaved data — close to right.

For noisy, heteroscedastic panel data it can be off by double digits.

## The failure mode

Jensen's correction is exact only under the assumption that the log-scale residuals are Gaussian. Real data isn't Gaussian. Three failure modes in particular:

1. **Heteroscedastic residuals.** If error variance grows with the predicted value (common in retail demand, ad response, attribution outcomes), a single `σ̂²` over-corrects in low-variance regions and under-corrects in high-variance regions.
2. **Fat tails.** Lognormal assumption falls apart under heavy right tails. The true `E[exp(ε)]` exceeds `exp(σ²/2)` by a factor that grows with the kurtosis.
3. **Horizon-dependent bias.** Forecast residuals at horizon 1 vs horizon 12 have different distributions. A single global correction averages over them.

## The fix: Duan smearing (1983)

Naihua Duan proposed the *smearing estimator* ([JASA 1983](https://www.jstor.org/stable/2288126)): don't assume a distribution for the log-scale residuals; estimate the bias correction non-parametrically from the residuals themselves.

Given a log-scale model with residuals `ε̂_i` on the training set, the smearing factor is:

```
φ̂ = (1/n) Σ exp(ε̂_i)
```

You then take your log-scale prediction, exponentiate, and multiply by φ̂:

```
ŷ = exp(μ̂) · φ̂
```

No distributional assumption. `φ̂` is whatever the data says the mean of `exp(ε)` is.

For Gaussian residuals `φ̂ ≈ exp(σ²/2)` — Duan reduces to Jensen. For heteroscedastic or fat-tailed residuals, Duan differs substantially and is (empirically) less biased.

## Horizon-specific smearing

If you're forecasting at multiple horizons (1, 2, 3, …, 12 months out), residual distributions differ by horizon. The correction does too. Compute smearing factors per horizon from an expanding-window backtest:

```python
for h in horizons:
    residuals_h = []
    for split in expanding_window_splits:
        model.fit(train[split])
        pred_h  = model.predict(h, test[split])
        obs_h   = test[split].at_horizon(h)
        residuals_h.extend(log(obs_h) - log(pred_h))
    phi_h = np.mean(np.exp(residuals_h))
    smearing_factor_by_horizon[h] = phi_h
```

At prediction time, look up φ̂ by horizon and apply. Linearly interpolate beyond the longest backtested horizon if needed. Cap at a ceiling (we use 5.0) so pathological tails don't blow up forecasts.

## Tier-specific smearing for panel ML

If the model is a panel ML regressor (RF / XGBoost / SVM) over heterogeneous panels — brands, channels, geographies — residual distributions vary by panel tier. A tier-1 panel (24+ months of history, dense data) has tighter residuals than a tier-6 panel (14 months, sparse). Global φ̂ over-corrects some and under-corrects others.

The fix: compute φ̂ per tier from the OOB residuals (random forest) or from the backtest residuals, Winsorize at 5% to stop outliers from dominating the mean, cap at 5.0 for sanity.

```python
def tier_smearing(oob_residuals_by_tier, winsor=0.05, cap=5.0):
    out = {}
    for tier, residuals in oob_residuals_by_tier.items():
        r = np.array(residuals)
        lo, hi = np.quantile(r, [winsor, 1 - winsor])
        r = np.clip(r, lo, hi)
        out[tier] = min(cap, float(np.mean(np.exp(r))))
    return out
```

## When *not* to use Duan

- **Low-data regimes.** If you have < 30 residuals to average over, Duan's variance gets bad and Jensen might be more stable. Rule of thumb: Duan needs the same residual sample size you'd need to estimate a mean reliably.
- **You know the distribution.** If you've verified residuals are Gaussian (rare in practice), Jensen is exact and marginally cheaper.

## What changed when I switched

Replaced Jensen's correction with horizon-specific Duan smearing across a production forecasting system (700+ paths × 4 divisions × 1.2M observations):

- Aggregate forecast bias fell substantially on heteroscedastic panels.
- The tier-6 panels (sparse, long-tail) stopped systematically under-forecasting at long horizons.
- Cap at 5.0 × mattered — two panels had kurtosis bad enough to produce `φ̂ > 5` before clipping, which would have produced nonsense forecasts.

## Moving-block bootstrap on top

Related gotcha: if you're bootstrapping residuals to produce prediction intervals, do *not* use IID bootstrap on time-series residuals. It ignores autocorrelation. Use moving-block bootstrap with block length chosen by the rule `L ≈ n^(1/3)`. Combined with Duan smearing on the point estimate, interval coverage gets substantially closer to nominal.

## References

- Duan, N. (1983). "Smearing estimate: a nonparametric retransformation method." *Journal of the American Statistical Association*, 78(383), 605–610.
- Manning, W.G., Mullahy, J. (2001). "Estimating log models: to transform or not to transform?" *Journal of Health Economics*, 20(4), 461–494 — discusses the choice between log-linear + Duan and GLM alternatives.
- Politis, D.N., Romano, J.P. (1994). "The stationary bootstrap." *JASA*, 89(428), 1303–1313 — for moving-block bootstrap variants.
