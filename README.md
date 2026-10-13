# Gold Market Behaviour Analysis

**SQL + Python analysis of gold futures (XAU/USD proxy): intraday session behaviour, macro-event volatility, dollar and real-yield links, anomaly detection and walk-forward volatility forecasting.**

Traders often rely on rules of thumb about gold: "London sets the trend", "the breakout of the Asian range works", "gold just follows real yields". This project tests those beliefs on 16 years of daily data and 2 years of hourly data. Every claim is checked against an explicit benchmark (a null model, a naive baseline or an earlier period) and reported with a confidence interval, and every model is validated walk-forward so it never sees the future.

---

## Key results

| Question | Answer |
|---|---|
| When does gold move? | New York hours produce **~72%** of the daily range; the 08:00–10:00 NY window (US data releases) has the largest hourly bars. |
| Is London special? | No. London holds **10.7%** of daily highs, exactly what a random walk with gold's volatility profile predicts (10.7%). Asia makes the day's low more often than chance (**46% vs 40%**, p = 0.009). |
| Does the London breakout of the Asian range work? | **No evidence of an edge.** Buying upside breaks earned +11.6 bps/trade before adjustment (p = 0.09); after removing gold's uptrend it falls to **+4.4 bps, 95% CI [−9, +17]**. |
| How much do news days move gold? | Daily range vs a normal day: **NFP 1.41x** [1.30–1.53], **FOMC 1.31x** [1.19–1.50], CPI 1.10x [0.98–1.21]. In the release hour itself: FOMC **4.0x**, NFP 2.2x, CPI 1.6x. |
| Does gold still follow real yields and the dollar? | Day to day, yes: correlation with real-yield changes is **−0.28** before and after 2022 (no significant change). Weekly, the link **weakened significantly** (−0.45 → −0.26, p = 0.004). In levels it broke: since Jan 2022 real yields rose from −0.97% to 2.91% while gold rose **+134%**. |
| Which days were abnormal? | Isolation Forest flags 41 days (top 1%), stable across random seeds (Jaccard 0.82). Top hits are known shocks: Brexit (24 Jun 2016), the April 2013 crash, the US downgrade (8 Aug 2011), March 2020, the April 2025 tariff announcement and the −10.8% day on 30 Jan 2026. |
| Can tomorrow's volatility be predicted? | Partly. Logistic regression reaches **0.61 holdout AUC** and is well calibrated (Brier 0.242 vs 0.250 baseline). On the 10% of days it flags, **71%** were high-volatility vs 49% overall. It does **not** significantly beat a naive or HAR benchmark. |
| Can tomorrow's direction be predicted? | No. AUC 0.50–0.52, a coin flip. |

---

## Approach

| # | Section | Methods |
|---|---|---|
| 0 | Data pipeline & quality | yfinance + US Treasury downloads, cached CSVs, SQLite database with indexes, assertion-based data-quality checks |
| 1 | Intraday sessions | SQL window functions (`ROW_NUMBER`, `NTILE`), Monte Carlo random-walk null model (arcsine law), chi-square test |
| 2 | London breakout backtest | SQL view with realistic stop fills, drift adjustment, bootstrap CIs, Holm correction, cost sensitivity |
| 3 | Macro event study | Range normalised to a 20-day baseline, Mann-Whitney U with Holm correction, bootstrap CIs, release-hour analysis |
| 4 | Dollar & real yields | Correlation of changes at daily/weekly/monthly horizons, OLS with HAC (Newey-West) errors, Fisher z-test across eras |
| 5 | Anomaly detection | Isolation Forest, contamination and seed stability, per-day driver attribution, contract-roll artefact tagging |
| 6 | Volatility forecasting | Logistic regression vs XGBoost vs HAR vs naive; walk-forward CV with tuning on training years only; block-bootstrap AUC CIs; calibration; economic (position-sizing) test |

### Modelling choices worth noting
- **No look-ahead:** features use only data known at today's close (plus the pre-scheduled event calendar); an assertion checks every feature date precedes its target date.
- **Honest benchmarks:** HAR (Corsi, 2009) is the standard volatility-forecasting baseline; a model has to beat it, not just a naive guess.
- **Uncertainty first:** AUC differences are tested with a paired block bootstrap. Logistic regression (0.610) vs naive (0.607) is reported as *no significant difference*, not a win.
- **Explainability:** the simpler logistic model is preferred over XGBoost at equal performance. Strongest drivers (odds ratio per 1 SD): today's relative range 1.44, NFP tomorrow 1.26, VIX 1.14, FOMC tomorrow 1.14.

---

## Data

| Source | Series | Coverage |
|---|---|---|
| Yahoo Finance | Gold futures `GC=F` (daily) | 2010 – present |
| Yahoo Finance | Gold futures `GC=F` (hourly) | last ~2 years (Yahoo limit) |
| Yahoo Finance | US Dollar Index `DX-Y.NYB`, VIX `^VIX` | 2010 – present |
| US Treasury | 10-year real yield (TIPS curve) | 2010 – present |
| Federal Reserve / BLS | FOMC, CPI and NFP release dates | 2010 – 2026 |

Database tables: `gold_daily`, `gold_hourly`, `macro_daily`, `real_yield`, `events`; views: `asian_breakout`, `daily_metrics`. Sessions are in New York time: Asia 18:00–03:00, London 03:00–08:00, New York 08:00–17:00.

---

## How to run

**Google Colab (easiest):** open `gold_market_analysis.ipynb` in Colab and choose **Runtime → Run all**. Data downloads automatically on the first run (~2 minutes).

**Locally:**
```bash
git clone https://github.com/<your-username>/gold-market-behaviour-analysis.git
cd gold-market-behaviour-analysis
pip install -r requirements.txt
jupyter notebook gold_market_analysis.ipynb
```

`requirements.txt`
```
yfinance
pandas
numpy
scipy
statsmodels
scikit-learn
xgboost
matplotlib
requests
```

Downloads are cached in `data/`; delete that folder to refresh with the latest prices. Results will shift slightly as new data arrives.

---

## Repository structure
```
gold-market-behaviour-analysis/
├── gold_market_analysis.ipynb   # full analysis (SQL + Python)
├── README.md
├── requirements.txt
└── data/                        # created on first run (cached downloads, gold.db)
```

---

## Limitations
- **Futures, not spot:** `GC=F` is a continuous futures contract. It tracks spot closely but carries a small cost of carry and can show roll gaps (checked and tagged, not removed).
- **Short intraday history:** hourly data covers ~2 years, all inside the 2024–26 bull market, so Sections 1–2 describe one regime.
- **Data gaps:** Yahoo has 149 stale daily bars (high = low), which are dropped; about 30 event dates fall on those days and are excluded from the event study.
- **Position sizing:** halving exposure on flagged days cut annualised volatility from 22.1% to 20.1%, but did not reduce maximum drawdown. The forecast helps with day-to-day risk, not tail protection.

