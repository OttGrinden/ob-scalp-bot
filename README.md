# OB-Scalp Bot

Paper/virtual scalping bot based on Smart Money Concepts (SMC).

**Strategy:** 4H Order Block detection + 1m pin bar confirmation → virtual entry with TP 0.15% / SL ~0.1–0.2%.

**Backtest result (2yr, BTC/ETH/SOL/BNB/XRP):** ~74% Win Rate, BE = 40%.

> No real orders are placed. This is a paper/simulation system only.

---

## Requirements

- Node.js >= 18
- Python >= 3.10
- No external API keys required (uses Binance public endpoint for 1m bars)

---

## Directory Structure

```
ob-scalp-bot/
├── scripts/
│   ├── smc_detector.py           # SMC core logic (4H sweep + score G1–G7)
│   ├── smc_scalp_detector.py     # Live signal detector (Binance 1m bars)
│   ├── backtest_smc.py           # Backtest engine (fetches from Binance)
│   ├── backtest_ob_scalp_2yr.py  # 2-year validation
│   ├── backtest_ob_scalp_4yr.py  # 4-year validation
│   ├── backtest_scalp_grid.py    # Grid search over TP/SL/hold configs
│   └── execution/
│       ├── scalp_open.mjs        # Open scalp position in virtual portfolio
│       ├── paper_broker.mjs      # Paper order lifecycle
│       ├── audit_log.mjs         # Append-only audit log
│       ├── schemas.mjs           # Order state machine schemas
│       ├── order_state.mjs       # State transition logic
│       ├── realism_config.mjs    # Optional slippage/fee simulation
│       ├── fill_model.mjs        # Fill cost calculation
│       └── portfolio/
│           └── portfolio_state.mjs  # Virtual EUR portfolio state
├── trading/
│   ├── cache/                    # OHLCV + heatmap cache (populated at runtime)
│   └── execution/                # Orders, audit logs, portfolio state
└── results/                      # Backtest output (auto-created)
```

---

## Usage

### 1. Run the signal detector (live)

```bash
python3 scripts/smc_scalp_detector.py
```

Emits NDJSON signals when a Score ≥ 9 OB-Scalp setup + 1m pin bar is confirmed.

**Requires** OHLCV daily/4h cache files in `trading/cache/`. These are written by an external OHLCV fetcher (e.g. via Binance REST). Expected filenames:
- `trading/cache/ohlcv_daily_BTC.json`
- `trading/cache/ohlcv_4h_BTC.json`
- `trading/cache/liq_heatmap.json` (for price lookup in scalp_open.mjs)

### 2. Open a virtual position

```bash
node scripts/execution/scalp_open.mjs \
  --symbol BTC \
  --score 9 \
  --sl-pct 0.22 \
  --tp-pct 0.15 \
  --reason "ob_scalp sweep62499.9 pin62510.3 tp0.15pct"
```

Add `--dry-run` to preview without writing.

### 3. Run backtests

```bash
python3 scripts/backtest_ob_scalp_2yr.py   # 2-year validation
python3 scripts/backtest_ob_scalp_4yr.py   # 4-year validation
python3 scripts/backtest_scalp_grid.py     # Grid search
```

Results are saved to `./results/`.

---

## Portfolio Config

Virtual portfolio starts at **1000 EUR** (`INITIAL_CAPITAL` in `portfolio_state.mjs`).

Key parameters in `scalp_open.mjs`:
| Parameter | Default | Description |
|-----------|---------|-------------|
| `SCALP_LEVERAGE` | 15x | Futures leverage (Leverage-Tabelle: SL ≤3% → max 15x) |
| `SCALP_NOTIONAL_PCT` | 3% | Margin per trade (% of NAV) |
| `SCALP_MAX_HOLD_HOURS` | 2h | Max hold before thesis_expired close |
| `MAX_SCALP_POSITIONS` | 2 | Max concurrent scalp slots |

---

## Scoring (SMC G1–G7)

| Gate | Criterion | Points |
|------|-----------|--------|
| G1 | BOS confirmed in trade direction | +2 |
| G2 | Price above/below EMA200 | +1 |
| G3 | IDM (Inducement) swept | +2 |
| G4 | Order Block drawn | +2 |
| G5 | Price returns to OB | +1 |
| G6 | Candlestick confirmation | +1 |
| G7 | R/R ≥ 1:2 | +1 |

Score ≥ 9 required for scalp entry.

---

## License

MIT
