# writings

Technical writing — methodology, architecture, and case studies. Each post stands alone.

## Contents

| Post | Topic |
|---|---|
| [Duan smearing vs Jensen's inequality](posts/2026-04-duan-smearing.md) | Why log-scale forecasts need retransformation bias correction, and why horizon-specific Duan smearing (1983) beat Jensen's inequality in production. |
| [Five gates for a GeoLift study](posts/2026-04-geolift-five-gates.md) | A statistical-validity framework that makes synthetic-control design a binary decision before any result is computed. |
| [Discrete-time cloglog for incrementality at 10M-user scale](posts/2026-04-cloglog-incrementality.md) | Why discrete-time hazard models beat Cox PH for digital-marketing incrementality, and the full apparatus (KM + Nelson-Aalen + covariate-time-interaction PH test + IPW + AIPW + Rosenbaum + Crump). |
| [Deterministic pipelines beat routing agents](posts/2026-04-deterministic-pipelines.md) | Why an orchestrated 5-stage pipeline with LLM reasoning *inside* stages outperformed every router-based MAS design I tried, citing MAST 2025, Agentless, and blackboard-pattern papers. |

## License

MIT on the prose — see [`LICENSE`](LICENSE).
