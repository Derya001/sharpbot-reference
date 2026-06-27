# sharpbot

A Python-based live arbitrage betting bot built as a freelance project and portfolio reference. Connects to the BetBurger Live API for surebet signals and automatically places bets on Pinnacle (PS3838) and Orbit Exchange (Betfair Asia) via Playwright browser automation on the BetInAsia Black (MollyBet) platform.

This is not a proof-of-concept. The bot handles real money, real accounts, and real time pressure — arbitrage windows can close in under 60 seconds.

---

## What It Does

Arbitrage betting exploits odds discrepancies between bookmakers: by placing opposing bets on the same event at two different platforms, a guaranteed profit is locked in regardless of the outcome.

The bot automates the entire pipeline:

1. Polls the BetBurger Live API for live surebet signals
2. Filters by sport (7 supported) and ROI (1.0%–6.0%)
3. Calculates optimal stake distribution for an 80 EUR total bet, correcting for Orbit's 3% commission
4. Opens the BetInAsia Black Arb calc tab and fills in the Pinnacle stake — Orbit (bf) stake is auto-calculated by MollyBet
5. Submits both legs simultaneously with a single Place button click
6. Manages global exposure (max 1000 EUR active at any time)
7. Enforces one bet per event
8. Hard stops on login failure — exactly 1 login attempt, no retries
9. Auto-releases exposure after 15 minutes if no settlement signal arrives
10. Automatically refreshes BetBurger API token on startup via headless login
11. Logs all activity with structured output

---

## Architecture Note

The original design used a pure REST API approach (BetInAsia BLACK API for Pinnacle + Orbit). During development, BetInAsia discontinued the BLACK API and Orbit Exchange confirmed they have no public API. The execution layer was redesigned to use Playwright browser automation on the BetInAsia Black (MollyBet) web interface, which handles both Pinnacle (pin88) and Betfair/Orbit (bf) legs on a single betslip.

The calculator, risk manager, and BetBurger integration are unchanged.

---

## Stack

| Layer | Technology |
|---|---|
| Language | Python 3.11+ |
| Browser Automation | Playwright (headless Chromium) |
| HTTP / Async | aiohttp + asyncio |
| Data Validation | Pydantic |
| Config | YAML |
| Logging | structlog |
| OS Target | Ubuntu Linux (VPS) |

---

## Supported Platforms

| Platform | Role | Integration |
|---|---|---|
| Pinnacle (PS3838) | Sharp bookmaker | BetInAsia Black (MollyBet) — pin88 field |
| Orbit Exchange | Betting exchange (Betfair Asia) | BetInAsia Black (MollyBet) — bf field (auto-calculated) |
| BetBurger | Live surebet scanner | REST API — POST /api/v1/arbs/bot_pro_search |

---

## Repository Structure

```
sharpbot/
├── main.py                      # Entry point (--dry-run / --test / --betburger-login flags)
├── config/
│   ├── config.yaml              # User-editable settings (gitignored)
│   └── config.yaml.example      # Template to copy from
├── src/
│   ├── config.py                # Config loader
│   ├── models.py                # Pydantic data models
│   ├── calculator.py            # Arbitrage calculator with commission correction
│   ├── risk_manager.py          # Exposure tracking and safety checks
│   ├── betburger.py             # BetBurger Live API client
│   ├── betburger_auth.py        # BetBurger auto-login and token refresh
│   ├── betinasia_playwright.py  # Playwright execution layer (BetInAsia Black)
│   └── logger.py                # structlog setup
└── tests/
    └── test_calculator.py       # Calculator unit tests
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
| BetBurger auto-auth (headless token refresh) | ✅ Complete (2026-06-27) |
| Playwright execution layer (BetInAsia Black) | ✅ Complete |
| DOM validation with live account | ✅ Complete (2026-06-25) |
| Hard stop on login failure (1 attempt only) | ✅ Complete |
| Unit tests (9/9) | ✅ Complete |
| dry-run mode | ✅ Complete |
| BetBurger Live API token | ✅ Active |
| End-to-end dry-run (BetBurger + BetInAsia) | ✅ Complete (2026-06-26) |
| VPS deployment | ⏳ Pending |
| End-to-end live test on VPS | ⏳ Pending |

---

## Unit Test Results

Latest test run (2026-06-27):

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

## DOM Validation Results (2026-06-25)

Validated against live BetInAsia Black (MollyBet) interface:

| Element | Selector | Source |
|---|---|---|
| Username input | `[data-testid="6ea20084"]` | Login page HTML |
| Password input | `[data-testid="38fdcdb7"]` | Login page HTML |
| Login button | `[data-testid="753534ba"]` | Login page HTML |
| Trade nav | `[data-testid="3382650e"]` | /live page HTML |
| Trade table | `#tradeTable` | JS bundle |
| Pinnacle stake input | `.price-input[0]` | JS bundle |
| Orbit stake (auto) | Auto-calculated by MollyBet after 0.5s | Verified live |
| Submit button | `button:has-text('place')` | JS bundle |

Key finding: MollyBet automatically calculates the Orbit (bf) LAY stake — only the Pinnacle (pin88) stake needs to be filled in manually.

---

## BetBurger API

- Endpoint: `POST https://rest-api-lv.betburger.com/api/v1/arbs/bot_pro_search`
- Auth: `access_token` form body field (alphanumeric token from betburger.com/profile/api)
- Token refresh: handled automatically via `--betburger-login` flag — bot logs into BetBurger headlessly, extracts the current token, and updates `config.yaml`

---

## Configuration

All settings live in `config/config.yaml` — editable with any text editor, no development environment needed. Copy from `config/config.yaml.example` to get started.

```yaml
betburger:
  api_token: "YOUR_BETBURGER_API_TOKEN"
  username: "YOUR_BETBURGER_EMAIL"
  password: "YOUR_BETBURGER_PASSWORD"
  polling_interval: 1.0
  per_page: 30

betinasia:
  username: "YOUR_BETINASIA_USERNAME"
  password: "YOUR_BETINASIA_PASSWORD"
  base_url: "https://black.betinasia.com"
  headless: true
  odds_tolerance: 0.02

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
playwright install chromium

python main.py --betburger-login  # first run only: headless BetBurger login, token refresh
python main.py --test             # smoke test — calculator + risk manager, no login needed
python main.py --dry-run          # full run, logs everything, places no real bets
python main.py                    # live mode
pytest tests/                     # unit tests
```

---

## Why This Project

Built as a freelance delivery for a client running a manual arbitrage operation. The goal: eliminate human reaction time as the bottleneck in a time-critical, multi-platform workflow.

Key engineering decisions:
- Playwright browser automation with persistent session — login once, reuse session across betting cycles
- Exactly 1 login attempt allowed: failed login triggers immediate hard stop to prevent account lockout from retry loops
- BetBurger token auto-refresh via headless login — session persists until BetBurger invalidates it, no manual token copying needed
- MollyBet Arb calc tab flow: single betslip handles both Pinnacle and Orbit legs, submitted with one click
- Orbit commission factored into the arbitrage calculation before any decision is made
- Hard ROI ceiling (6%) to filter out stale or erroneous scanner signals
- Event-level deduplication to prevent double-exposure on the same match
- Automatic exposure release after 15 minutes for unresolved live events

---

## Notes

- Client credentials are excluded via `.gitignore` — `config/config.yaml` and `config/.betburger_session.json` are never committed
- Balance management (deposits/withdrawals) is out of scope
- VPS setup (Ubuntu 24.04 LTS, Frankfurt region) is the client's responsibility
- BetBurger Live API subscription (API Arbs Live 2) is the client's responsibility
- BetInAsia BLACK API was discontinued during development — execution layer redesigned to use Playwright on BetInAsia Black (MollyBet) web interface
