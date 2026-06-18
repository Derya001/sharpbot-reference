# sharpbot

A Python-based live arbitrage betting bot built as a freelance project and portfolio reference. Connects to the BetBurger Live API for surebet signals and automatically places bets on Pinnacle (PS3838) and Orbit Exchange simultaneously via the BetInAsia BLACK API.

This is not a proof-of-concept. The bot handles real money, real accounts, and real time pressure — arbitrage windows can close in under 60 seconds.

---

## What It Does

Arbitrage betting exploits odds discrepancies between bookmakers: by placing opposing bets on the same event at two different platforms, a guaranteed profit is locked in regardless of the outcome.

The bot automates the entire pipeline:

1. Polls the BetBurger Live API for surebet signals
2. Filters by sport (7 supported) and ROI (1.0%–6.0%)
3. Calculates optimal stake distribution for an 80 EUR total bet, correcting for Orbit's 3% commission
4. Places both bets simultaneously via parallel async API calls
5. Manages global exposure (max 1000 EUR active at any time)
6. Enforces one bet per event
7. Auto-releases exposure after 15 minutes if no settlement signal arrives
8. Logs all activity with structured output

---

## Stack

| Layer | Technology |
|---|---|
| Language | Python 3.11+ |
| HTTP / Async | aiohttp + asyncio |
| Data Validation | Pydantic |
| Config | YAML |
| Logging | structlog |
| OS Target | Ubuntu Linux (VPS) |

---

## Supported Platforms

| Platform | Role | Integration |
|---|---|---|
| Pinnacle (PS3838) | Sharp bookmaker | BetInAsia BLACK API |
| Orbit Exchange | Betting exchange (Betfair Asia) | BetInAsia Live Orbit API |
| BetBurger | Live surebet scanner | REST API (polling) |

---

## Repository Structure

```
sharpbot/
├── main.py                  # Entry point (--dry-run / --test flags)
├── config/
│   └── config.yaml          # User-editable settings (gitignored)
├── src/
│   ├── config.py            # Config loader
│   ├── models.py            # Pydantic data models
│   ├── calculator.py        # Arbitrage calculator with commission correction
│   ├── risk_manager.py      # Exposure tracking and safety checks
│   ├── betburger.py         # BetBurger Live API client
│   ├── betinasia.py         # BetInAsia BLACK API client (Pinnacle + Orbit)
│   └── logger.py            # structlog setup
└── tests/
    └── test_calculator.py   # Calculator unit tests
```

---

## Project Status

| Component | Status |
|---|---|
| Project structure | ✅ Complete |
| Pydantic data models | ✅ Complete |
| Arbitrage calculator (with Orbit commission) | ✅ Complete |
| Risk manager (exposure, dedup, auto-release) | ✅ Complete |
| structlog logger | ✅ Complete |
| BetBurger Live API client | ✅ Complete |
| BetInAsia BLACK API client (Pinnacle + Orbit) | ✅ Complete |
| Unit tests (9/9) | ✅ Complete |
| dry-run mode | ✅ Complete |
| BetBurger Live API token | ⏳ Awaiting client |
| BetInAsia BLACK API key | ⏳ Awaiting client |
| Live API integration test | ⏳ Pending API credentials |
| VPS deployment | ⏳ Pending |

---

## Unit Test Results

Latest test run (2026-06-18):

| Test | Result |
|---|---|
| Profitable surebet → TradeOrder returned | ✅ |
| Unprofitable surebet → None | ✅ |
| ROI > 6% → None (suspicious, skipped) | ✅ |
| Sport filter — unsupported sport → None | ✅ |
| Missing Pinnacle outcome → None | ✅ |
| Orbit commission correctly applied to effective odds | ✅ |
| Risk manager — first bet approved | ✅ |
| Risk manager — duplicate event rejected | ✅ |
| Risk manager — exposure limit enforced | ✅ |

---

## Configuration

All settings live in `config/config.yaml` — editable with any text editor, no development environment needed. The file is gitignored; copy from `config/config.yaml.example` to get started.

```yaml
betburger:
  api_token: "YOUR_BETBURGER_LIVE_API_TOKEN"
  polling_interval: 1.0
  per_page: 30

betinasia:
  api_key: "YOUR_BETINASIA_API_KEY"
  api_url: "https://betinasia.com"

betting:
  total_bankroll: 4000.0      # EUR
  match_stake: 80.0           # EUR per bet
  max_global_exposure: 1000.0 # EUR (25% safety limit)
  orbit_commission: 0.03      # 3% on winning bets
  min_roi: 1.0                # % — below this: skip
  max_roi: 6.0                # % — above this: likely data error, skip
  exposure_timeout: 900       # seconds before auto-release

allowed_sports:
  - soccer
  - tennis
  - basketball
  - table tennis
  - esports
  - volleyball
  - badminton
```

---

## Running the Bot

```bash
pip install -r requirements.txt

python main.py --test      # smoke test — calculator + risk manager, no API needed
python main.py --dry-run   # full run, logs everything, places no real bets
python main.py             # live mode
pytest tests/              # unit tests
```

---

## Why This Project

Built as a freelance delivery for a client running a manual arbitrage operation. The goal: eliminate human reaction time as the bottleneck in a time-critical, multi-platform workflow.

Key engineering decisions:
- Pure API architecture — no browser automation, no DOM scraping
- asyncio.gather() for simultaneous Pinnacle + Orbit bet placement, minimizing the window between the two legs
- Orbit commission factored into the arbitrage calculation before any decision is made — a bet that looks profitable on raw odds may not be after the 3% cut
- Hard ROI ceiling (6%) to filter out stale or erroneous scanner signals
- Event-level deduplication to prevent double-exposure on the same match
- Automatic exposure release after 15 minutes for unresolved live events

---

## Notes

- Client credentials are excluded via `.gitignore` — `config/config.yaml` is never committed
- Balance management (deposits/withdrawals) is out of scope
- VPS setup (Ubuntu, Frankfurt/London region) is the client's responsibility
- BetBurger Live API subscription and BetInAsia BLACK API access are the client's responsibility
