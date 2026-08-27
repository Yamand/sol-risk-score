# SOL Risk Score

A daily composite 0–1 risk score for SOL, built entirely from Binance public
OHLCV data (no API key, no paid on-chain data). Static site + GitHub Actions.

This is a direct port of [btc-risk-score](https://github.com/Yamand/btc-risk-score) —
same four components, same weights, same zone table, same repo layout.
**No Telegram alert in this version** (dropped by request) — just the score
generator and the static dashboard. Read **"Porting caveats"** below before
you trust it the way you trust the BTC version; the methodology transfers
mechanically, but part of it is on weaker theoretical footing for SOL —
more so than for XRP, if you've also got that version running.

**Live idea:** `0` = cheap, accumulate harder. `1` = expensive, reduce buys /
start distributing once holdings clear your $500 sell-tier threshold.

## How the score is built

Four components, each normalized to 0–1 via **expanding historical percentile
rank** (today's raw value ranked against every prior day back to the start of
data) — this means there are no hardcoded "cheap" / "expensive" thresholds to
maintain; the scale self-calibrates as more history accumulates.

| Component | Weight | What it captures |
|---|---|---|
| Log-regression band position | 35% | Price vs. long-term log-log growth curve, refit each run |
| 200-day MA multiple | 25% | Price stretch vs. long-term trend (price ÷ 200d MA) |
| RSI-14 (daily) | 20% | Short-term overbought/oversold |
| Volatility-adjusted momentum | 20% | 30d return ÷ 30d realized volatility |

Composite = weighted sum of the four, then smoothed with a **3-day EMA**
before being used for zone lookup.

## Porting caveats — read before trusting this like the BTC score

**1. The log-regression component (35% of the score) is on the weakest ground
of all three assets in this series.** The BTC version fits `log(price)`
against `log(days since genesis block)` — a power-law model that's
reasonably well documented for BTC, because BTC has a fixed, disinflationary
issuance schedule and 15+ years of comparatively continuous adoption. This
script uses the **same math** but against
`log(days since Solana's Mainnet Beta genesis block, 2020-03-16)`. SOL's
history doesn't share BTC's assumptions, and arguably fits worse than XRP's
does:
- **Short history.** SOL has roughly 4–5 years of price data vs. BTC's 15+.
  A log-log regression needs a lot of history to average out short-term
  noise into a stable "trend"; SOL's fit will keep shifting meaningfully
  every time a new market regime shows up, for longer than either of the
  other two versions.
- **Issuance is still inflationary and stepping down**, not fixed like BTC's
  halving schedule — the supply-side assumption behind "long-term power-law
  growth" doesn't map cleanly.
- **One dominant, non-recurring shock dwarfs the rest of the series**: SOL
  fell roughly 95% peak-to-trough after the November 2022 FTX collapse
  (Solana Labs' close ties to FTX/Alameda made SOL one of the hardest-hit
  large-cap tokens), then recovered sharply through 2023–24. A single
  log-log line fit across "pre-collapse," "collapse," and "recovery" is
  averaging over what were really three different regimes, not one smooth
  curve — the "fair value band" this produces is a much rougher
  approximation than the same fit is for BTC, and rougher than for XRP too.

  Practically: expect this component to be noisier and slower to stabilize
  than the BTC or XRP versions. Weight it with real skepticism, especially
  in the first year or two of accumulated history, and especially around the
  Nov 2022 date if you backfill through it.

**2. Zone boundaries (0.10 / 0.20 / 0.25 / 0.35 / 0.60 / 0.70 / 0.80) are
carried over unchanged, not re-tuned on SOL's own history.** SOL is more
volatile than both BTC and XRP, so expect these to fire more often / less
discriminatingly than on either other dashboard.

**3. Binance's SOLUSDT history starts a few months after mainnet launch**,
so — like the XRP version — the log-regression fit and expanding percentile
ranks will be less stable in year one than BTC's own (already-flagged) early
history caveat. Run a full backfill and check the printed start date.

**4. Prices are formatted to 4 decimal places below $10, 2 above** — SOL has
traded from single digits up to several hundred dollars, so a fixed
precision either flattens the low end or looks odd at the high end.
`sol_risk_score.py` and the dashboard both use the same threshold.

None of this means the score is wrong — it's the same well-reasoned
methodology, applied to an asset it wasn't originally validated against, and
with less history to validate against than either BTC or XRP have. Use it as
one more input, not as a standalone signal.

## Repo structure

```
sol_risk_score.py                    # fetch + compute + write data/
data/sol_risk_history.json           # generated — one scored row per day
data/sol_prices_raw.json             # generated — full raw close-price cache
index.html                           # static site, reads data/ directly, Chart.js
.github/workflows/daily-update.yml   # cron job, runs sol_risk_score.py --update daily
```

No `*_risk_alert.py` file in this repo — the BTC and XRP versions include a
Telegram daily-summary script; this one was left out on request. The daily
workflow only fetches, recomputes, and commits the JSON — no alert step.

## Setup

1. Push this repo to GitHub, enable **GitHub Pages** (Settings → Pages →
   Deploy from branch → `main` / root).
2. Run a full backfill once, locally, so both `data/sol_risk_history.json`
   and `data/sol_prices_raw.json` exist before the site goes live — see
   "Local run" below.
3. Commit **both** files in `data/` — the workflow commits both on every run
   too, since `sol_prices_raw.json` has to persist across runs for
   `--update` to work correctly.
4. The daily workflow (`daily-update.yml`) runs automatically at 00:15 UTC,
   fetches the last 400 days, merges with the cached full history,
   recomputes, and commits both updated JSON files. GitHub Pages redeploys
   automatically on push.

### Local run

```bash
pip install pandas numpy requests
python sol_risk_score.py            # full history backfill (first run)
python sol_risk_score.py --update   # fast daily run — same math, fewer requests
python -m http.server 8000          # then open localhost:8000
```

## Notes

- No API key required — Binance's `/api/v3/klines` endpoint is public.
- The score is descriptive, not a signal to auto-trade on.
- Not financial advice.
