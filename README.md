# N-CORE

A quant trading terminal I designed, built and deployed myself. It scans a 298-ticker universe, backtests and walk-forward validates strategy families per asset, and delivers live signals to a mobile web app and a Telegram bot.

**Author:** Daron Nyarko · Built May 2026 to August 2026, with Claude Code

**Status:** Previously hosted on Railway; currently runs locally.

<p>
  <img src="docs/screenshots/hub.png" width="230" alt="HUB screen: catalyst alert, S&P 500 chart, pinned assets with playbook status">
  <img src="docs/screenshots/bias.png" width="230" alt="Bias screen: gold state, z-score, bands, entry, invalidation and target levels">
  <img src="docs/screenshots/telegram.png" width="230" alt="Telegram bot posting a live playbook setup with chart, stop and target">
</p>
<p><em>Left to right: HUB, Bias, and the Telegram signal bot.</em></p>

## Results

| Metric | Value |
|---|---|
| Strategy families | 9 |
| Ticker universe | 298 |
| Backtests run | about 2,400 |
| Asset-strategy pairs that passed validation | 43 |
| Live signals | about 6.5 a week |

Most candidates were rejected. That was the point: about 2,400 backtests produced 43 pairs that passed validation.

## What it does

- **Bias and levels.** For each asset, the engine reports the current state, trend, entry level, invalidation and target, so I can see at a glance whether a setup is live.
- **Strategy lab.** Every strategy family is backtested and walk-forward validated per asset. Only pairs that pass validation get assigned to the playbook.
- **Screener.** Scans the full universe for candidates that match a validated setup.
- **Positions and journal.** Tracks open positions, real P&L and closed trades, with a "shield" that flags positions at risk ahead of earnings.
- **AI layer.** Claude reads market news for sentiment, answers plain-English questions about a setup, and writes market memos.
- **Alerts.** Push notifications in the app and a Telegram bot (`/bias sofi`, `/levels gold`, `/status`) that answers from engine math with no AI cost.

## How it's built

```
api/        FastAPI backend: 30+ JSON endpoints and the Telegram webhook
  main.py       routes
  analytics.py  strategy lab, walk-forward validation, playbook
  feed.py       market data feed (Yahoo Finance) with caching and rate-limit backoff
  screener.py   universe scan
  ai.py         Claude API calls for sentiment, Q&A and memos
  store.py      positions, watchlist and playbook persistence
  db.py         Supabase connection
web/        React + Vite mobile PWA (installable, push notifications, Supabase auth)
quant_core.py   strategy definitions and backtest engine
signal_engine.py  signal generation
db/         Supabase SQL schema and migrations
docs/       build briefs and screenshots
app.py      the original Streamlit dashboard (v1)
```

**Stack:** Python, FastAPI, Pandas, NumPy, React 19, Vite, lightweight-charts, Supabase (Postgres + auth), Claude API, Telegram Bot API, Railway.

## How I built it with Claude Code

I built N-CORE in phases. For each phase I wrote a brief that set the scope, the API contract between backend and frontend, what not to build yet, and a definition of done. Then I used Claude Code to implement it and reviewed the result before moving on. The briefs are in [`docs/claude-code-briefs/`](docs/claude-code-briefs/).

| Phase | What shipped |
|---|---|
| 1 | Quant core API and Telegram command bot |
| 2 | Mobile PWA with five tabs: HUB, Bias, Chart, News, Positions |
| 3–4 | AI tier (sentiment, Ask), alpha optimizer, trade validation, strategy lab |
| 5–6 | Automation, notifications, position entry, agent worker |
| 7–9 | Real P&L, journal, Supabase persistence, screener, signal ledger, earnings shield |
| 10 | Three new strategy families (Catalyst Momentum, Gap Reversal, Trend Rider) and a 30-day lock that makes assigned pairs re-earn their slot |
| 11 | Pooled universe validation, grading each family on the combined out-of-sample trades of every watchlist asset |
| 12 | Universe Lab, a resumable Saturday batch job that scans all 298 tickers (about 2,400 backtests) |
| 13–14 | Ninth strategy family, quiet mode, feed hardening, cluster-risk banner |

## How I validated it

**Walk-forward testing.** Parameters were tuned on the first 70% of each asset's history, and the verdict came only from the untouched last 30%. To pass, a pair needed at least 10 out-of-sample trades, a 55% or higher out-of-sample win rate, a profit factor of 1.3 or better, positive expectancy, and results consistent with the in-sample period. If nothing passed, the lab returned `NOTHING_VALIDATED`. After 30 days, assigned pairs had to re-defend their slot every week.

**Costs and fills.** Entries filled at the next bar's open, and a stop that price gapped through filled at the open, not at the stop price. Commissions, spreads, and slippage were not modeled. When I tested the signals as options contracts, theta decay, bid/ask spreads, and IV changes made many of them unprofitable, which led to a later rebuild.

**Overfitting.** Pass thresholds were set before testing and never changed. Every strategy family needed an economic reason to exist before I tested it. Pooled validation was stricter than testing one asset at a time, because a family that only works on one ticker fails when pooled. With this many tests, some of the 43 pairs likely passed by luck, so a live signal ledger graded every signal going forward.

## Run it locally

Backend:

```
pip install -r requirements.txt
# .env: ANTHROPIC_API_KEY, plus Supabase and Telegram keys if you want those features
uvicorn api.main:app --reload --port 8000
```

Frontend:

```
cd web
cp .env.example .env   # set VITE_API_URL to http://localhost:8000
npm install
npm run dev
```

Original Streamlit dashboard: `streamlit run app.py`

## Where it started

N-CORE began as a Streamlit dashboard for gold. It pulled live prices, scored news sentiment, and had Claude write an investment memo on each load. It also pulled high-impact economic events from Forex Factory to flag macro risk. Its mean-reversion engine used a 20 EMA baseline, ±2 standard deviation bands, a 200 EMA trend filter and a 2.5% stop. That logic became the first strategy family in N-CORE. The earlier static analysis is in [Gold-Market-Analysis](https://github.com/TechnicallyDaron/Gold-Market-Analysis).

*Not financial advice. This is a personal research tool.*
