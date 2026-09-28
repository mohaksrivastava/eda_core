# edacore

A deterministic Python library for exploratory data analysis: assumption
checks, hypothesis tests, effect sizes, multiplicity correction, power
analysis, and visualisation, over `pandas` DataFrames.

Every function is pure (no input mutation), registered with metadata and a
code template, and returns a typed `pydantic` result rather than a bare
tuple or dict. Every statistical result is checked against a reference
value computed in R.

> **Snapshot of the deterministic core of edacopilot as of 2026-09-28.**
> Active development continues in
> [edacopilot](https://github.com/mohaksrivastava/edacopilot); this
> repository is not kept in sync.

## What's included

| Module | Purpose |
|---|---|
| `edacore.contracts` | Shared `pydantic` result types (`TestResult`, `PostHocResult`, `AssumptionCheck`, `DatasetProfile`, ...) |
| `edacore.registry` | The `@register` decorator, `FunctionSpec`, lookup by name/stage/tag, JSON-schema export |
| `edacore.codegen` | Renders a registered call back to runnable Python source from its recorded parameters |
| `edacore.profiling` | Dataset profiling: semantic type inference, structure detection (cross-sectional / time series / panel), duplicate and PII detection, leakage candidates |
| `edacore.assumptions` | Assumption checks: normality, variance, sphericity, linearity, monotonicity, expected counts, and more |
| `edacore.stattests.*` | Hypothesis tests, one file per family: `one_sample`, `two_sample`, `k_independent`, `k_related`, `categorical`, `correlation`, `factorial`, `posthoc` |
| `edacore.effect_sizes` | Effect sizes and confidence intervals paired with the tests above |
| `edacore.multiplicity` | Multiple-comparison correction (Holm, Benjamini-Hochberg, ...) |
| `edacore.power` | Power and required-sample-size calculations |
| `edacore.viz` | Static `matplotlib` plotting functions returning a bare `Figure` (no `pyplot`, so nothing leaks into a host kernel) |
| `edacore.validity_notes` | Deterministic interpretation guards attached to a result (e.g. "statistically significant but negligible effect size") |

Not included in this snapshot: missingness diagnostics/imputation,
outlier detection, data transforms, time series, and text analysis are
edacopilot milestones that had not landed in `edacore` as of the date
above.

## Correctness: verified against R

Every hypothesis test, assumption check, and effect size is checked
against the equivalent R computation (`t.test`, `wilcox.test`,
`kruskal.test`, `aov`/`TukeyHSD`, `chisq.test`, `fisher.test`,
`car::leveneTest`, `effectsize::*`, and more), to a tolerance of `1e-6`
(documented, wider tolerances for resampling methods with fixed seeds).
`docs/m3_effect_size_audit.md` records, function by function, which
tests return an effect size with a CI and which are exempt and why; a
test in the suite fails if the code and that document drift apart.

Reference values live as JSON in `tests/fixtures/r_reference/`, generated
by `scripts/generate_r_fixtures.R`. To regenerate them:

```bash
Rscript scripts/generate_r_fixtures.R
```

then diff the output directory against the committed fixtures before
trusting any new Python test written against them — a script that
silently computes against the wrong R object is a correctness bug that
only a full fixture diff catches, not just a non-zero exit code.

## Install

```bash
pip install "git+https://github.com/mohaksrivastava/eda_core.git"
```

For development:

```bash
git clone https://github.com/mohaksrivastava/eda_core.git
cd eda_core
pip install -e ".[dev]"
pytest -m "not slow"
```

## Usage

```python
import pandas as pd
from edacore.stattests.two_sample import welch_t

df = pd.DataFrame(
    {
        "value": [5.1, 4.9, 6.2, 5.8, 7.0, 6.5, 6.8, 7.2, 6.9, 7.5],
        "group": ["a"] * 5 + ["b"] * 5,
    }
)

result = welch_t(df, group_col="group", value_col="value")

print(result.estimand)  # "difference in means (independent groups, unequal variance)"
print(result.statistic, result.p_value)
print(result.estimate, result.ci)  # mean difference and its CI
print(result.effect_size_name, result.effect_size)  # Hedges' g(av)
```

Every registered function also renders back to plain, runnable code from
its own parameters:

```python
from edacore.registry import registry

params = {"group_col": "group", "value_col": "value", "ci": 0.95, "nan_policy": "omit"}
print(registry.to_code("welch_t", params, df_var="df"))
# edacore.stattests.two_sample.welch_t(df, group_col='group', value_col='value', ci=0.95, nan_policy='omit')
```

## License

Apache-2.0. See `LICENSE`.
