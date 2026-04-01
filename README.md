# 5-Minute BTC Polymarket Trading Bot

**Rust trading bot for Polymarket prediction markets.** It automates hedging and position management on **BTC 5-minute** binary (Up/Down) markets: lock profit when the combined cost per pair is favorable, and adjust exposure when the opposite side is rising and PnL is skewed.

[![Rust](https://img.shields.io/badge/rust-1.70%2B-orange.svg)](https://www.rust-lang.org/)
[![Polymarket](https://img.shields.io/badge/Polymarket-CLOB-blue)](https://polymarket.com)

**Repository:** [github.com/Poly-Tutor/5min-btc-polymarket-trading-bot](https://github.com/Poly-Tutor/5min-btc-polymarket-trading-bot)

---

## Quick start (first time)

Follow these steps in order. You do **not** need Polymarket API keys to try **simulation** mode.

| Step | What to do |
|------|-------------|
| 1 | **Install Rust** from [rustup.rs](https://rustup.rs/). Open a new terminal and run `rustc --version` — you should see 1.70 or newer. |
| 2 | **Clone this repo** (see [Installation](#installation)) and `cd` into the project folder. |
| 3 | **Build:** `cargo build --release` (first build may take several minutes). |
| 4 | **Run in simulation:** `cargo run --release -- --simulation` — no real orders; output goes to the console and `history.toml`. |
| 5 | **Read [logic.md](logic.md)** to understand lock, expansion, and when the bot trades vs. sits flat. |
| 6 | **Live trading:** create `config.json` from `config.example.json`, add your Polymarket credentials, then run with `--production --config config.json` (see [Configuration](#configuration) and [Usage](#usage)). |

**Windows (PowerShell):** use `Copy-Item config.example.json config.json` instead of `cp`. Use the same `cargo` commands as below.

**Problems?** Check [Requirements](#requirements), confirm your Rust toolchain is current, and open an [issue](https://github.com/Poly-Tutor/5min-btc-polymarket-trading-bot/issues) with your OS and the exact error text.

---

## How we profit

We buy **Up** and **Down** so that **cost per pair (Up + Down) < $1**. Whichever side wins pays out $1 per share, so we can lock in profit regardless of outcome. The bot only adds a side when the resulting average cost per pair stays under your cap (for example `0.99`).

---

## Features

- **5-minute BTC only:** Trades **5m** binary Up/Down markets for **BTC** on [Polymarket](https://polymarket.com).
- **Lock rule:** Buys the opposite side only when **cost per pair** stays under your cap (e.g. Up_avg + Down_ask ≤ 0.99), so you lock in edge.
- **Expansion rule:** When you cannot lock (cost would exceed max) but the other side is **rising** and its outcome PnL is worse, the bot buys that side to improve exposure (see [logic.md](logic.md)).
- **Ride the winner:** When one side is clearly winning (trend UpRising/DownRising), the bot adds to that side to grow “PnL if that side wins.”
- **PnL rebalance:** If one outcome’s PnL is negative and you are not strongly trending the other way, it buys the weak side (within cost limits).
- **Flat = no trade:** When the short-term trend is **Flat** (no clear move), the bot only locks if conditions allow; otherwise it does not open new risk (Example 6 in [logic.md](logic.md)).
- **Simulation mode:** Run with `--simulation` (default) to log trades without sending orders.
- **Market resolution:** Checks for closed markets, computes actual PnL, and logs “Total actual PnL (all time).”
- **Timestamped logs:** Price feed and history lines include `[YYYY-MM-DDTHH:MM:SS]` for easier debugging and backtesting.

---

## Strategy overview

The bot keeps **positions** per market (Up shares, Down shares, average prices). Each tick it:

1. Updates **trend** from the last 4–5 price points (UpRising, DownRising, Flat, UpFalling, DownFalling).
2. **Lock:** If adding the underweight side keeps cost per pair ≤ `cost_per_pair_max` → buy that side (lock).
3. **Expansion:** If you *cannot* lock (cost would exceed max) but the other side is **rising** and “PnL if that side wins” is worse → buy that side (new leg / rebalance).
4. **Ride winner:** If trend is UpRising or DownRising (and not Flat) → buy the rising side (within cost and buy limits).
5. **PnL rebalance:** If one outcome’s PnL is negative and trend is not strongly the other way → buy the weak side (within limits).
6. **Trend fallback:** DownFalling → can buy Up; UpFalling → can buy Down (no buy on Flat except lock).

Detailed examples (no position, only Up, only Down, have both, Flat, market close) are in **[logic.md](logic.md)**.

---

## Requirements

- **Rust** 1.70 or newer — install via [rustup](https://rustup.rs/) and verify with `rustc --version`.
- **For live trading:** Polymarket API credentials (API key, secret, passphrase, and optionally private key / proxy wallet as required by your setup). Simulation mode does not send orders; you still get realistic logging.

---

## Installation

**Linux / macOS (bash):**

```bash
git clone https://github.com/Poly-Tutor/5min-btc-polymarket-trading-bot.git
cd 5min-btc-polymarket-trading-bot
cargo build --release
```

**Windows (PowerShell):**

```powershell
git clone https://github.com/Poly-Tutor/5min-btc-polymarket-trading-bot.git
cd 5min-btc-polymarket-trading-bot
cargo build --release
```

If `git` or `cargo` is not recognized, install [Git for Windows](https://git-scm.com/download/win) and Rust from [rustup.rs](https://rustup.rs/), then restart the terminal.

---

## Configuration

1. Copy the example file to `config.json`:

   ```bash
   cp config.example.json config.json
   ```

   On **Windows PowerShell:**

   ```powershell
   Copy-Item config.example.json config.json
   ```

2. Edit `config.json`: set `polymarket` fields (API key, secret, passphrase, and wallet fields if you use live trading). Use the same structure as [config.example.json](config.example.json).

**Security:** `config.json` is gitignored. Do not commit API keys or private keys.

### Trading options (`config.json` → `trading`)

| Option | Description | Example |
|--------|-------------|---------|
| `markets` | Assets to trade (this bot: BTC only) | `["btc"]` |
| `timeframes` | Periods (this bot: 5m only) | `["5m"]` |
| `cost_per_pair_max` | Max cost per pair when locking | `0.99` |
| `min_side_price` | Do not buy below this ask | `0.05` |
| `max_side_price` | Do not buy above this ask | `0.99` |
| `cooldown_seconds` | Min seconds between buys | `0` |
| `cooldown_seconds_1h` | Min seconds between buys (1h; N/A for 5m-only) | `45` |
| `shares` | Override size per order; `null` = default (BTC 5m = 24) | `null` |
| `size_reduce_after_secs` | Start reducing size in last N seconds | `300` |
| `market_closure_check_interval_seconds` | How often to check for resolved markets | `20` |

---

## Usage

**Simulation (recommended first — no real orders):**

```bash
cargo run -- --simulation
# or
cargo run --release -- --simulation
```

**Live trading (sends FAK orders to Polymarket):**

```bash
cargo run --release -- --production --config config.json
```

**Redeem winnings for a resolved market:**

```bash
cargo run --release -- --redeem --condition-id <CONDITION_ID>
```

Logs go to `history.toml` (and your configured log target). Example price line:

```text
[2026-02-04T23:17:23] BTC 5m Up Token BID:$0.52 ASK:$0.53 Down Token BID:$0.47 ASK:$0.48 remaining time:4m 12s
```

---

## Project structure

```text
.
├── Cargo.toml
├── config.json          # Your config (gitignored); use config.example.json as template
├── logic.md             # Strategy examples (lock, expansion, ride winner, flat)
├── history.toml         # Append-only trade/price log (gitignored in default .gitignore)
├── docs/
│   └── TARGET_STRATEGY_ANALYSIS.md
├── src/
│   ├── main.rs          # CLI, config load, monitor + trader spawn
│   ├── config.rs        # Config and defaults
│   ├── api.rs           # Polymarket Gamma + CLOB API
│   ├── monitor.rs       # Price feed, snapshot, timestamps
│   ├── trader.rs        # Lock/expansion/ride-winner/PnL logic
│   ├── models.rs        # API and market data types
│   └── bin/
│       └── analyze_target_history.rs  # Analyze btc-5m.toml history
└── README.md
```

---

## Disclaimer

This bot is for **educational and research purposes**. Trading prediction markets involves risk of loss. Past behavior in simulation or backtests does not guarantee future results. Use at your own risk. The authors are not responsible for any financial loss.

---

## Community

- **Issues & questions:** [GitHub Issues](https://github.com/Poly-Tutor/5min-btc-polymarket-trading-bot/issues)
- **Telegram:** [@gabagool21](https://t.me/gabagool21)

---

## Keywords (for search)

Polymarket trading bot, Polymarket bot, Polymarket automation, crypto prediction market bot, BTC prediction market, Polymarket CLOB, Polymarket API, prediction market hedging, binary market bot, Rust Polymarket, Polymarket 5m, Polymarket arbitrage, Polymarket hedging bot, 5-minute BTC.
