# IMC Prosperity 4 Backtester

[![Build Status](https://github.com/nabayansaha/imc-prosperity-4-backtester/workflows/Build/badge.svg)](https://github.com/nabayansaha/imc-prosperity-4-backtester/actions/workflows/build.yml)
[![PyPI Downloads](https://static.pepy.tech/personalized-badge/prosperity4btest?period=total&units=INTERNATIONAL_SYSTEM&left_color=BLACK&right_color=GREEN&left_text=downloads)](https://pepy.tech/projects/prosperity4btest)

This repository contains a backtester for [IMC Prosperity 4](https://prosperity.imc.com/) algorithms, based on [jmerle's backtester for Prosperity 3](https://github.com/jmerle/imc-prosperity-3-backtester).

## Usage

Basic usage:
```sh
# Install the latest version of the backtester
$ pip install -U prosperity4btest

# Run the backtester on an algorithm using all data from round 0
$ prosperity4btest <path to algorithm file> 0
```

Run `pip install -U prosperity4btest` again when you want to update the backtester to the latest version.

The PyPI distribution and CLI are named `prosperity4btest`. The Python package directory is still `prosperity4bt` (e.g. bundled data lives under `prosperity4bt/resources`).

Some more usage examples:
```sh
# Backtest on all days from round 1
$ prosperity4btest example/starter.py 1

# Backtest on round 1 day 0
$ prosperity4btest example/starter.py 1-0

# Backtest on round 1 day -1 and round 1 day 0
$ prosperity4btest example/starter.py 1--1 1-0

# Backtest on all days from rounds 1 and 2
$ prosperity4btest example/starter.py 1 2

# You get the idea

# Merge profit and loss across days
$ prosperity4btest example/starter.py 1 --merge-pnl

# Automatically open the result in the visualizer when done
# Assumes your algorithm logs in the visualizer's expected format
$ prosperity4btest example/starter.py 1 --vis

# Write algorithm output to custom file
$ prosperity4btest example/starter.py 1 --out example.log

# Skip saving the output log to a file
$ prosperity4btest example/starter.py 1 --no-out

# Round 2: include fee-aware PnL assumption for Market Access Fee
# (accepted = subtract Trader.bid() once from round 2 PnL)
$ prosperity4btest sample.py 2 --round2-access accepted

# Backtest on custom data
# Requires the value passed to `--data` to be a path to a directory that is similar in structure to https://github.com/nabayansaha/imc-prosperity-4-backtester/tree/master/prosperity4bt/resources
$ prosperity4btest example/starter.py 1 --data prosperity4bt/resources

# Print trader's output to stdout while running
# This may be helpful when debugging a broken trader
$ prosperity4btest example/starter.py 1 --print
```

## Round 2 Additions

### Market Access Fee (`Trader.bid`)

For round 2, if your algorithm exposes:

```python
class Trader:
    def bid(self):
        return 15
```

the backtester now reads this value and reports fee-aware round 2 summary numbers.

- `bid()` is sanitized to a non-negative integer (invalid/negative => `0`).
- The backtester cannot know acceptance locally (officially top 50% of bids), so use:
  - `--round2-access unknown` (default): no deduction, informational only.
  - `--round2-access accepted`: subtract bid once from round 2 PnL.
  - `--round2-access rejected`: no deduction from round 2 PnL.

### Manual Challenge Calculator (`invest` command)

Use the built-in calculator for the "Invest & Expand" challenge:

```sh
# Uses default speed multiplier = 0.5
prosperity4btest invest --research 55 --scale 35 --speed 10

# Rank-based speed multiplier (rank 4 out of 20)
prosperity4btest invest --research 55 --scale 35 --speed 10 --speed-rank 4 --players 20

# Or directly specify assumed speed multiplier
prosperity4btest invest --research 55 --scale 35 --speed 10 --speed-multiplier 0.7
```

Formula used:

- `research(x) = 200_000 * ln(1 + x) / ln(101)`
- `scale(x) = 7 * x / 100`
- `gross_pnl = research * scale * speed_multiplier`
- `net_pnl = gross_pnl - budget_used` with budget cap `50,000` XIRECs and total allocation <= 100%

## Order Matching

Orders placed by `Trader.run` at a given timestamp are matched against the order depths and market trades of that timestamp's state. Order depths take priority, if an order can be filled completely using volume in the relevant order depth, market trades are not considered. If not, the backtester matches your order against the timestamp's market trades. In this case the backtester assumes that for each trade, the buyer and the seller of the trade are willing to trade with you instead at the trade's price and volume. Market trades are matched at the price of your orders, e.g. if you place a sell order for €9 and there is a market trade for €10, the sell order is matched at €9 (even though there is a buyer willing to pay €10, this appears to be consistent with what the official Prosperity environment does).

Matching orders against market trades can be configured through the `--match-trades` option:
- `--match-trades all` (default): match market trades with prices equal to or worse than your quotes.
- `--match-trades worse`: match market trades with prices worse than your quotes, inspired by [team Linear Utility's Prosperity 2 write-up](https://github.com/ericcccsliu/imc-prosperity-2).
- `--match-trades none`: do not match market trades against orders.

Limits are enforced before orders are matched to order depths. If for a product your position would exceed the limit, assuming all your orders would get filled, all your orders for that product get canceled. During matching, each fill is also clamped so your position never goes beyond the configured limit (even if multiple price levels would otherwise fill more).

### Position limits

Known Prosperity 4 products are defined in `prosperity4bt/data.py` (`LIMITS`).

- Round 0 products: `EMERALDS` = **80**, `TOMATOES` = **80**
- Round 1 products: `ASH_COATED_OSMIUM` = **80**, `INTARIAN_PEPPER_ROOT` = **80**

Any product not listed there uses a default of **50** until you add it to `LIMITS`.

Override limits from the CLI without editing code:

```sh
prosperity4btest sample.py 0 --limit EMERALDS:80 --limit TOMATOES:80
```

## Data Files

Data files for Prosperity 4 rounds will be added as they become available.

Round 0:
Tomatoes: 80
Emeralds: 80

Conversions are not supported.

## Environment Variables

During backtests two environment variables are set for the trader to know the round and day it's being backtested on. The environment variable named `PROSPERITY4BT_ROUND` contains the round number and `PROSPERITY4BT_DAY` contains the day number. Note that these environment variables do not exist in the official submission environment, so make sure the code you submit doesn't require them to be defined.

## Development

Follow these steps if you want to make changes to the backtester:
1. Install [uv](https://docs.astral.sh/uv/).
2. Clone (or fork and clone) this repository.
3. Open a terminal in your clone of the repository.
4. Create a venv with `uv venv` and activate it.
5. Run `uv sync`.
6. Any changes you make are now automatically taken into account the next time you run `prosperity4btest` inside the venv.

