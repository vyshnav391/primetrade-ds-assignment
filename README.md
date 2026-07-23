# Trader Performance vs. Bitcoin Market Sentiment

Analysis of how trader performance on **Hyperliquid** relates to the
**Bitcoin Fear & Greed Index**, built for the Primetrade.ai data science
hiring task.

## Objective

Explore the relationship between market sentiment (Fear/Greed) and trader
behavior/performance, surface hidden patterns, and translate them into
insights that could inform a trading strategy.

## Data

| File | Description |
|---|---|
| `data/fear_greed_index.csv` | Daily Bitcoin Fear & Greed Index, `2018-02-01` → `2025-05-02` (`value`, `classification`). |
| `data/historical_data.csv` | ~211k Hyperliquid trade fills, `2023-05-01` → `2025-05-01`, across 32 accounts and 246 symbols (execution price, size, side, PnL, fees, etc.). |

The two datasets are joined on calendar date (`Timestamp IST` truncated to
date, matched against the index's `date` column). 211,218 of 211,224 trade
rows (99.997%) find a matching sentiment day.

## Repo structure

```
.
├── data/                       # raw input CSVs
├── src/analysis.py             # end-to-end analysis script
├── notebooks/exploration.ipynb # exploratory notebook (same analysis, cell-by-cell)
├── outputs/
│   ├── sentiment_summary.csv        # core metrics per sentiment bucket
│   ├── account_level_avg_pnl.csv    # per-account avg PnL x sentiment (checks it isn't a few whales)
│   ├── trade_size_distribution.csv  # trade-size distribution by sentiment
│   ├── correlation.txt              # daily PnL vs FG-index correlation
│   └── figures/                     # charts referenced below
├── requirements.txt
└── README.md
```

## How to run

```bash
pip install -r requirements.txt
python src/analysis.py
```

This regenerates every file under `outputs/`.

## Methodology

1. Load trades and the Fear & Greed Index; parse timestamps and derive a
   join key (`date`).
2. Merge trades with same-day sentiment classification.
3. A trade row is treated as **closed** (used for win-rate) when
   `Closed PnL != 0`; this excludes pure position-opening fills that have
   no realized PnL yet.
4. Aggregate by sentiment bucket: trade count, total/avg/median closed
   PnL, average trade size, total volume, average fee, win rate.
5. Cross-check at the account level so the sentiment effect isn't an
   artifact of one or two large accounts.
6. Build a daily time series (7-day rolling average PnL) against the raw
   index value and compute a Pearson correlation as a sanity check on the
   bucketed result.

## Key findings

**1. Performance is not simply "better sentiment, better PnL."**
Average closed PnL per trade by sentiment bucket:

| Sentiment | Trades | Avg PnL (USD) | Win rate |
|---|---:|---:|---:|
| Extreme Fear | 21,400 | 34.54 | 76.2% |
| Fear | 61,837 | 54.29 | **87.3%** |
| Neutral | 37,686 | 34.31 | 82.4% |
| Greed | 50,303 | 42.74 | 76.9% |
| Extreme Greed | 39,992 | **67.89** | **89.2%** |

The relationship is **U-shaped, not linear**: the best average PnL and win
rates cluster at the emotional extremes (Fear and Extreme Greed), while
Neutral and plain Greed are comparatively weaker for these accounts. This
suggests the traders in this dataset do relatively well buying into
capitulation-style fear and riding strong greed-driven trends, but are less
efficient in choppier, less decisive sentiment regimes.

![Average PnL by sentiment](outputs/figures/avg_pnl_by_sentiment.png)
![Win rate by sentiment](outputs/figures/win_rate_by_sentiment.png)

**2. Fear days see disproportionately high trading volume.**
`Fear` accounts for the single largest share of both trade count (61.8k
trades) and total traded volume (~$483M) despite not being the most
"positive" sentiment label — i.e., these accounts are most active exactly
when the broader market is nervous.

![Volume by sentiment](outputs/figures/volume_by_sentiment.png)

**3. Fee efficiency improves with confidence.**
Average fee per trade drops from ~$1.50 in Fear to ~$0.68 in Extreme
Greed, consistent with fewer, larger, more decisive trades once sentiment
is strongly positive (smaller trades / more churn tends to relatively
increase fee drag).

**4. The day-to-day correlation is weak.**
Pearson correlation between daily total closed PnL and the raw Fear &
Greed index value is **-0.08** — i.e., essentially no linear relationship
at the daily aggregate level. This confirms the bucketed pattern above is
a **regime effect** (how traders behave at each sentiment *label*) rather
than a simple linear dependency on the index score itself, and reinforces
that any resulting strategy should key off sentiment *state* transitions,
not the raw index level.

![Daily PnL vs FG index](outputs/figures/daily_pnl_vs_fg_index.png)

## Suggested strategy implications

- **Lean into extremes, be cautious in the middle.** These accounts'
  historical edge is strongest in Extreme Greed and Fear; sizing could be
  reduced in Neutral/Greed regimes where win rate and avg PnL are lower.
- **Fear days deserve tighter risk controls, not less activity** — traders
  are already most active here; the priority is protecting the elevated
  win rate rather than trading more.
- **Don't rely on the raw index score for timing** — because the daily
  correlation is near zero, a discrete sentiment-*regime* signal
  (Extreme Fear / Fear / Neutral / Greed / Extreme Greed) is a more useful
  input than treating the index as a continuous predictor.

## Caveats

- Dataset covers 32 accounts only — findings describe *these* traders'
  behavior, not the broader market.
- No explicit leverage column exists in the raw export; trade size (USD)
  is used as an aggressiveness proxy instead of leverage.
- Correlation ≠ causation: sentiment could be reactive to price moves that
  also drive PnL, rather than causally driving trader behavior.

## Requirements

See `requirements.txt`. Analysis uses `pandas`, `matplotlib`, and `numpy`
only — no exotic dependencies.
