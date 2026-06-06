# Wealth Management Dashboard — MVP Plan

## Overview

Personal finance dashboard in Python/Streamlit that reads PDF bank and investment
statements from a local folder, computes net worth over time, and tracks investment
performance (HPR, TWR, Sharpe ratio).

---

## Project Structure

```
wealth-dashboard/
├── app.py                        # Streamlit entry point (multi-tab layout)
├── requirements.txt
├── .gitignore
├── config.yaml                   # statements_root, account mappings, usd_mxn rate
├── src/
│   ├── __init__.py
│   ├── pdf_parser.py             # PDF text extraction + closing balance parsing
│   ├── accounts.py               # Account model, asset/liability classification
│   ├── performance.py            # HPR, TWR, Sharpe ratio
│   └── charts.py                 # Plotly figure builders
├── scripts/
│   └── generate_sample_data.py   # Creates synthetic balances.csv for demo/testing
└── tests/
    └── test_calculations.py      # Unit tests for HPR, TWR, Sharpe
```

---

## Data Source

Statements live at a configurable path set in `config.yaml` under `statements_root`.
Current location: `~/Documents/finances/Accounts/` (configurable via `statements_root` in `config.yaml`)
This will move to the repo's working folder in a future step — update `statements_root` then.

```
Accounts/
  Banks/
    banamex-tdd/
    bbva-tdd/
    hsbc-tdd-6178/
    hsbc-tdc-2now/
    santander-tdd/
    santander-tdc-aeromex/
    santander-tdc-zero/
    banregio-tdc-mas/
    Banorte/
    Openbank/
  Investments/
    cetesdirecto/
    gbm/
    prestadero/
    ibkr/                  ← USD-denominated, converted via usd_mxn in config
    bitso/
    allianz-ppr/
    profuturo/             ← AFORE/pension, ~3 statements/year (every 4 months)
    odessa-msci/
  Liabilities/
    hsbc-hipoteca/         ← mortgage, balance subtracts from net worth
```

**Rule:** the dashboard reads files in-place. Never move or reorganize the Accounts folder.

---

## Data Flow

```
Accounts/{subfolder}/
  YYYY-MM_*.pdf            ← one file per statement period
  balances.csv             ← optional manual override (takes precedence over PDF)
        │
        ▼
  pdf_parser.py
  StatementLoader.load_all()
  ─ pdfplumber extracts raw text
  ─ table extraction first, regex fallback second
  ─ returns List[Statement(account, date, balance)]
        │
        ▼
  accounts.py
  AccountManager
  ─ classifies accounts from config.yaml explicit mappings
  ─ converts IBKR balances: MXN = balance_usd × usd_mxn
  ─ builds monthly balance time-series per account
  ─ net_worth(month) = Σ assets − Σ liabilities
        │
        ▼
  performance.py
  PerformanceCalculator
  ─ HPR  = (EV − BV − CF) / (BV + CF)
  ─ TWR  = Π(1 + sub_period_HPR) − 1
  ─ Sharpe = (mean_r − rf) / σ_r   [annualised monthly returns]
        │
        ▼
  charts.py → app.py → Streamlit tabs
```

---

## Account Configuration (`config.yaml`)

All accounts are explicitly mapped — no keyword guessing. Example:

```yaml
statements_root: ~/Documents/finances/Accounts

usd_mxn: 17.15   # static rate for v1; live rate via yfinance in v2

risk_free_rate: 0.1125   # annual, approximate CETES rate

accounts:
  # Banks — Assets
  banamex-tdd:         { type: asset,      currency: MXN }
  bbva-tdd:            { type: asset,      currency: MXN }
  hsbc-tdd-6178:       { type: asset,      currency: MXN }
  hsbc-tdc-2now:       { type: liability,  currency: MXN }
  santander-tdd:       { type: asset,      currency: MXN }
  santander-tdc-aeromex: { type: liability, currency: MXN }
  santander-tdc-zero:  { type: liability,  currency: MXN }
  banregio-tdc-mas:    { type: liability,  currency: MXN }
  Banorte:             { type: asset,      currency: MXN }
  Openbank:            { type: asset,      currency: MXN }

  # Investments — Assets
  cetesdirecto:        { type: investment, currency: MXN }
  gbm:                 { type: investment, currency: MXN }
  prestadero:          { type: investment, currency: MXN }
  ibkr:                { type: investment, currency: USD }  # converted to MXN
  bitso:               { type: investment, currency: MXN }
  allianz-ppr:         { type: investment, currency: MXN }
  profuturo:           { type: investment, currency: MXN }  # ~3 statements/year
  odessa-msci:         { type: investment, currency: MXN }

  # Liabilities
  hsbc-hipoteca:       { type: liability,  currency: MXN }
```

---

## PDF Parsing Strategy (`pdf_parser.py`)

Reference: `step2/statement_aggregator.py` (530-line prototype) — review for useful
regex patterns and institution-specific logic. Do not copy blindly; rewrite cleanly.
Also port extraction patterns from `step2/account_config.json`.

Priority order per PDF:
1. `balances.csv` in the account folder — takes precedence, enables manual override
2. `pdfplumber.extract_tables()` — works for most modern PDFs
3. Regex on raw text — fallback:
   - Closing balance: `r'(?:ending|closing|current|saldo)\s+balance[:\s]+\$?([\d,]+\.\d{2})'`

**Scope for v1:** extract closing balance only. No individual transaction parsing.

**Irregular cadence** (e.g. Profuturo every 4 months) is handled naturally — the
time-series will have sparse data points and charts will render gaps.

---

## Finance Math (`performance.py`)

Reference: `Finances/Cycles/NetWorth.py` — proven TWR, PnL, equity curve formulas.
Review and port the core logic.

```python
# HPR (single period)
def hpr(begin_value, end_value, cash_flow=0):
    return (end_value - begin_value - cash_flow) / (begin_value + cash_flow)

# TWR (chain-linked sub-periods)
def twr(sub_period_returns):
    return prod(1 + r for r in sub_period_returns) - 1

# Sharpe (annualised, monthly returns series)
def sharpe(monthly_returns, risk_free_annual):
    rf_monthly = (1 + risk_free_annual) ** (1/12) - 1
    excess = monthly_returns - rf_monthly
    return (excess.mean() / excess.std()) * sqrt(12)
```

---

## Streamlit App (`app.py`) — 3 Tabs

| Tab | Contents |
|-----|----------|
| **Net Worth** | KPI cards (current NW, MoM change), line chart of NW over time |
| **Accounts** | Stacked bar chart by account type + sortable balance table |
| **Performance** | HPR / TWR / Sharpe metric cards, rolling Sharpe line chart, investment account selector |

Sidebar: `statements_root` path override, date range slider, account multi-select filter.

All amounts displayed in **MXN**.

---

## Excluded from MVP (future sessions)

- Transactions tab (individual debit/credit line items per account)
- Live FX rate (IBKR USD→MXN uses static rate in config)
- Benchmark comparison (S&P500, IPC Mexico, BTC, CETES via `update_benchmarks.py`)
- `buro-credito` credit report

---

## Files to Create (in order)

1. `requirements.txt` — streamlit, pdfplumber, pandas, numpy, plotly, pyyaml
2. `.gitignore` — Python standard + exclude Accounts/ path + any local data/
3. `config.yaml` — full account mappings as above
4. `src/__init__.py`
5. `src/pdf_parser.py` — `Statement` dataclass, `StatementLoader`
6. `src/accounts.py` — `AccountManager` with net worth time-series
7. `src/performance.py` — `PerformanceCalculator`
8. `src/charts.py` — Plotly figure builders
9. `app.py` — Streamlit layout (3 tabs)
10. `scripts/generate_sample_data.py` — synthetic `balances.csv` files for all accounts
11. `tests/test_calculations.py` — unit tests for HPR, TWR, Sharpe
12. `README.md` — setup, config instructions, how to run

---

## Verification

```bash
pip install -r requirements.txt
python scripts/generate_sample_data.py
streamlit run app.py
# → http://localhost:8501
# Check: Net Worth tab shows line chart, Performance tab shows Sharpe ratio

python -m pytest tests/ -v
```

---

## Security

- `Accounts/` path and any folder containing real financial data must never be committed.
- `.gitignore` must cover the `statements_root` path and any `data/` folder with real balances.
- `config.yaml` itself does not contain sensitive data (only paths and rates) — safe to commit.
