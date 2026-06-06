# Wealth Dashboard

Personal finance dashboard built with Python and Streamlit. Reads PDF bank and investment statements from a local folder, computes net worth over time, and tracks investment performance.

## Features

- **Net Worth tracking** — monthly balance time-series across all accounts
- **Multi-account support** — banks, brokerages, crypto, pension funds, and liabilities (mortgage, credit cards)
- **Performance metrics** — HPR, TWR, and annualised Sharpe ratio
- **Multi-currency** — USD-denominated accounts converted to MXN via a configurable rate
- **PDF parsing** — table extraction with regex fallback; manual `balances.csv` override per account

## Tech Stack

- [Streamlit](https://streamlit.io) — UI
- [pdfplumber](https://github.com/jsvine/pdfplumber) — PDF extraction
- [Pandas](https://pandas.pydata.org) / [NumPy](https://numpy.org) — data processing
- [Plotly](https://plotly.com/python) — charts
- [PyYAML](https://pyyaml.org) — configuration

## Setup

```bash
pip install -r requirements.txt
```

Configure `config.yaml` with your `statements_root` path and account mappings.

```bash
# Generate synthetic data for testing
python scripts/generate_sample_data.py

# Run the app
streamlit run app.py
```

## Configuration

All accounts are declared in `config.yaml`. No sensitive data is stored in this repo — statements and balances live in a local folder outside version control.

```yaml
statements_root: ~/Documents/finances/Accounts

usd_mxn: 17.15        # static rate for v1; live rate via yfinance in v2
risk_free_rate: 0.1125  # annual CETES rate

accounts:
  my-bank:    { type: asset,     currency: MXN }
  my-broker:  { type: investment, currency: USD }
  my-card:    { type: liability,  currency: MXN }
```

## Project Status

MVP in planning — implementation in progress.
