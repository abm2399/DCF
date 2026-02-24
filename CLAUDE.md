# CLAUDE.md

## Project Overview

**DCF** is a Python command-line tool for computing Discounted Cash Flow (DCF) valuations of publicly traded companies. It fetches financial statements from the [financialmodelingprep.com](https://financialmodelingprep.com) API, performs a 2-stage DCF model (explicit forecast period + terminal value via perpetuity growth), and optionally visualizes historical intrinsic value versus actual share price.

Author: Hugh Alessi

## Repository Structure

```
DCF/
├── main.py                    # CLI entry point — argparse setup and orchestration
├── modeling/
│   ├── data.py                # API client — fetches financial statements and stock prices from FMP
│   └── dcf.py                 # Core DCF engine — FCF forecasting, enterprise/equity value calculation
├── visualization/
│   ├── plot.py                # matplotlib/seaborn charts — historical DCF vs. share price plots
│   └── printouts.py           # Terminal pretty-printing of DCF results
├── imgs/                      # Saved chart output (PNG files)
├── README.md                  # User-facing documentation
└── .gitignore                 # Ignores __pycache__, .vscode, _config.yml
```

There are no `__init__.py` files; modules are imported via `from modeling.data import *` style wildcard imports.

## Key Modules

### `main.py`
- Entry point. Parses CLI arguments and dispatches to either single-DCF or sensitivity-analysis mode.
- `main(args)`: Routes based on whether step_increase (`--s`) is specified. If steps > 0 and a variable is given, runs `run_setup()` for sensitivity analysis. Otherwise computes a single historical DCF.
- `run_setup(args, variable)`: Iterates through step increments, mutating growth rate parameters and collecting DCFs for each step.
- Known bug at `main.py:32`: The `or` conditions in the variable-matching chain (e.g., `args.v == 'eg' or 'earnings_growth_rate'`) always evaluate to `True` due to Python truthiness of non-empty strings. The first branch (`'eg'`) is always taken regardless of `args.v`.

### `modeling/dcf.py`
- `DCF(...)`: Computes a single-point DCF valuation. Returns `{'date', 'enterprise_value', 'equity_value', 'share_price'}`.
- `historical_DCF(...)`: Wraps `DCF()` to iterate over multiple historical periods (annual or quarterly).
- `enterprise_value(...)`: Forecasts unlevered free cash flows for `period` years, discounts by WACC, adds terminal value via perpetuity growth model.
- `equity_value(...)`: Converts enterprise value to equity value (subtracts debt, adds cash) and derives per-share price.
- `ulFCF(...)`: Unlevered Free Cash Flow to Firm formula.
- `get_discount_rate()`: Stub — returns hardcoded 0.1. Dynamic WACC calculation is a planned feature.

### `modeling/data.py`
- API client for financialmodelingprep.com (v3 API).
- Functions: `get_income_statement()`, `get_balance_statement()`, `get_cashflow_statement()`, `get_EV_statement()`, `get_stock_price()`, `get_batch_stock_prices()`, `get_historical_share_prices()`.
- Uses `urllib.request.urlopen` (no third-party HTTP library).
- `get_api_url()` constructs endpoint URLs with period and API key parameters.

### `visualization/plot.py`
- `visualize_bulk_historicals(dcfs, ticker, condition, apikey)`: Plots multiple DCF scenarios (sensitivity analysis) against historical stock price. Saves to `imgs/`.
- `visualize()` and `visualize_historicals()`: Stubs / not fully implemented.
- Uses matplotlib and seaborn with `sns.set_context('paper')`.

### `visualization/printouts.py`
- `prettyprint(dcfs, years)`: Formats DCF results for terminal output when `years <= 1` (no chart).

## Dependencies

No `requirements.txt` exists. Install manually:

```
pip install matplotlib urllib3 seaborn
```

Standard library modules used: `argparse`, `os`, `json`, `traceback`, `decimal`, `sys`, `urllib.request`.

## Running the Project

### API Key

Requires a free API key from financialmodelingprep.com. Provide via:
- `--apikey <key>` CLI argument, or
- `APIKEY` environment variable

### Basic Usage

```bash
# Single current DCF for AAPL with defaults
python main.py --t AAPL --apikey <key>

# Historical DCFs with sensitivity analysis
python main.py --t AAPL --i annual --y 3 --eg .15 --steps 2 --s 0.1 --v eg --apikey <key>
```

### CLI Arguments

| Argument | Description | Default |
|---|---|---|
| `--p` | Years to forecast FCF | 5 |
| `--t` | Ticker symbol | AAPL |
| `--y` | Years of historical DCFs | 1 |
| `--i` | Interval: `annual` or `quarter` | annual |
| `--s` | Step increase for sensitivity analysis | 0 |
| `--steps` | Number of sensitivity steps | 5 |
| `--v` | Variable to step: `eg`, `cg`, `pg`, `discount` | None |
| `--d` | Discount rate (WACC) | 0.1 |
| `--eg` | Earnings growth rate | 0.05 |
| `--cg` | CapEx growth rate | 0.045 |
| `--pg` | Perpetual growth rate | 0.05 |
| `--apikey` | FMP API key | `$APIKEY` env var |

## Code Conventions

- **Python version**: No version pinned; code uses f-strings (3.6+).
- **Import style**: Wildcard imports (`from module import *`) throughout.
- **No tests**: There is no test suite or test framework configured.
- **No linter/formatter**: No configuration for flake8, black, mypy, etc.
- **No `__init__.py`**: Packages lack init files; imports rely on direct module paths.
- **Argument abbreviations**: CLI args use short names (`--eg`, `--cg`, `--pg`) that double as internal variable keys.
- **Print-based logging**: All output uses `print()` — no logging framework.
- **Error handling**: Broad `except Exception` with `traceback.format_exc()` printed to stdout.

## Known Issues and TODOs

These are documented in the source code:

1. **`main.py:32`** — Variable matching uses `or` with string literals incorrectly (always true). All sensitivity analysis defaults to `'eg'` regardless of input.
2. **`dcf.py:116`** — `get_discount_rate()` is a stub returning 0.1. Dynamic WACC calculation is planned.
3. **`dcf.py:176`** — Change in working capital decay rate (`cwc * 0.7`) is hardcoded without justification.
4. **`plot.py:32`** — `visualize()` returns `NotImplementedError` (as a value, not raised).
5. **`main.py:92`** — `multiple_tickers()` is fully commented out and returns `NotImplementedError` (not raised).
6. **No `requirements.txt`** or `setup.py` / `pyproject.toml` for dependency management.
7. Terminal value uses perpetuity growth model only; EBITDA multiples method is planned but not implemented.

## Development Notes

- Chart output is saved to `imgs/` as PNG files named `{ticker}_{variable}.png`.
- The project has no build step — run `main.py` directly.
- Financial data depends on the FMP API; rate limits and data availability may affect results.
- The `enterprise_value()` function prompts for user input via `input()` if EBIT is missing from the statement, which can block non-interactive execution.
