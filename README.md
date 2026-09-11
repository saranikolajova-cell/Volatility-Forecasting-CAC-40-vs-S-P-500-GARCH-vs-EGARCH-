# Volatility Forecasting: CAC 40 vs S&P 500 (GARCH vs EGARCH)

**Question:** can a EGARCH model, which allows volatility to react asymmetrically to
positive and negative shocks, forecast next-day market volatility better than a
standard symmetric GARCH model?

## Summary

- **Data:** daily closing prices for the CAC 40 (`^FCHI`) and S&P 500 (`^GSPC`),
  2015–present, via Yahoo Finance.
- **Models:** GARCH(1,1) and EGARCH(1,1), Student-*t* innovations, fit on log returns.
- **Evaluation:** rolling-window, one-step-ahead out-of-sample forecasts, scored with
  QLIKE (the standard loss function for variance forecast evaluation) and MSE against
  realized (squared-return) volatility.

## Results

Run on 2015-01-05 to 2026-09 (rolling backtest window = 1000 days, horizon = 250 days):

| Index | AIC (GARCH / EGARCH) | BIC (GARCH / EGARCH) | Backtest QLIKE (GARCH / EGARCH) | Backtest MSE (GARCH / EGARCH) | Better fit |
|---|---|---|---|---|---|
| CAC 40 | 8239.43 / **8238.16** | 8269.45 / **8268.18** | 1.6026 / **1.5972** | 3.1582 / **3.0840** | **EGARCH** (wins on all 4) |
| S&P 500 | **7380.94** / 7395.90 | **7410.87** / 7425.83 | **1.6064** / 1.6189 | **1.3468** / 1.3546 | **GARCH** (wins on all 4) |

### Interpretation

EGARCH outperforms GARCH on **every** metric — in-sample fit (AIC, BIC) and genuine
out-of-sample backtest accuracy (QLIKE, MSE) — for the **CAC 40**. Since EGARCH's only
structural difference from GARCH is that it lets volatility respond asymmetrically to
positive vs. negative shocks, this consistent win is evidence of a **leverage effect**:
CAC 40 volatility appears to react more strongly to negative news than to positive news
of the same size. The margin is modest (≈0.3% QLIKE, ≈2.3% MSE improvement) but the fact
that it points the same direction across four independent metrics makes it more credible
than any single number in isolation.

For the **S&P 500**, the result flips: GARCH beats EGARCH on all four metrics, with a
noticeably larger AIC/BIC gap (~15 points) than the CAC 40 case. Over this sample,
adding the asymmetry term doesn't earn its keep — the symmetric model describes S&P 500
volatility at least as well, and generalizes better out-of-sample.

Taken together, **the CAC 40 shows more evidence of asymmetric, bad-news-driven
volatility than the S&P 500 does over this period** — negative shocks appear to move CAC
40 volatility more than comparable shocks move the S&P 500. That's a statement about
*volatility dynamics* (how each index reacts to shocks), not a claim that the S&P 500 is
unconditionally "safer" — the backtest MSE scale itself is lower for the S&P 500
(~1.35 vs ~3.1), which partly reflects the CAC 40 simply carrying higher baseline
volatility over the sample, independent of the asymmetry question. Framed carefully: the
CAC 40 looks more exposed to negative-shock-driven volatility spikes than the S&P 500,
based on how well each model captures its dynamics.

*(Caveat: this is one 2015–2026 sample, one backtest window, and no significance test
on the QLIKE gap — see "Possible extensions" below for the natural next step, a
Diebold-Mariano test, before treating this as a strong claim rather than a
suggestive pattern.)*

## Why this matters

Volatility forecasting underpins risk management, derivatives pricing, and portfolio
sizing. Equity markets typically show a **leverage effect** — volatility rises more
after negative shocks than after positive ones of the same size — which symmetric
GARCH cannot capture but EGARCH can. This project tests whether that asymmetry is
detectable and whether it translates into better forecasts, not just a better in-sample
fit.

## Method

1. **Data & returns** — download adjusted close prices, compute log returns
   (`compute_log_returns`).
2. **Full-sample fit** — fit GARCH(1,1) and EGARCH(1,1) on the whole return series,
   compare AIC/BIC.
3. **Rolling backtest** — re-estimate both models on a rolling window (default: 1000
   trading days ≈ 4 years) and produce a genuine one-step-ahead out-of-sample forecast
   at each step, for the last `--horizon` days (default: 250 ≈ 1 year). This avoids the
   look-ahead bias of evaluating a model on the same data it was fit on.
4. **Scoring** — compare forecast variance to realized variance (squared return, the
   standard low-frequency proxy) using QLIKE and MSE.

## Repository structure

```
volatility-forecast/
├── forecast.py          # main pipeline (data, models, backtest, plots)
├── requirements.txt
├── data/                 # cached price CSVs (created on first run)
├── results/               # summary tables + plots (created on run)
└── README.md
```

## Usage

```bash
pip install -r requirements.txt

# Both indices, default settings
python forecast.py

# One index, custom window/horizon
python forecast.py --tickers ^FCHI --start 2012-01-01 --window 1000 --horizon 500
```

Outputs land in `results/`:
- `<ticker>_returns.png` — daily return series (visualizes volatility clustering)
- `<ticker>_volatility.png` — realized vs. GARCH vs. EGARCH forecast volatility
- `<ticker>_summary.csv` — AIC/BIC and backtest QLIKE/MSE for both models

## Possible extensions

- Add a GJR-GARCH model as a second asymmetric benchmark
- Extend to a multi-step-ahead forecast horizon (5-day, 20-day)
- Add a Diebold-Mariano test for statistical significance of the QLIKE difference
- Apply the same pipeline to interest rate series (e.g. Euribor, OAT yields) instead
  of equity indices

## Notes

- Returns are scaled ×100 before model fitting (standard convention — keeps the
  optimizer numerically well-behaved).
- The rolling backtest refits the model at every step, which is the correct approach
  for genuine walk-forward validation but is slower than a single in-sample fit; reduce
  `--horizon` for a quicker run while testing.
