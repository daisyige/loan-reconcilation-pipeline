# Loan Portfolio Reconciliation & Investor Reporting Pipeline

An automated pipeline that takes a lender's raw loan and transaction data, **reconciles
expected vs. actual repayments**, flags data-quality issues, computes portfolio KPIs, and
generates a formatted **Excel investor pack** — all from a single command.

It replaces a recurring manual task (reconcile in Excel, recompute metrics, assemble a
report) that typically takes a couple of hours each week with a run that finishes in under
a second.

---

## What it does

```
raw CSVs ──▶ reconcile ──▶ compute KPIs ──▶ build investor pack
(3 files)    (exceptions)   (PAR, etc.)      (formatted .xlsx + charts)
```

1. **Reconciliation** — matches every repayment against the loan it belongs to and the
   amount that was scheduled, then flags four kinds of exception:
   *orphan transactions* (no matching loan), *duplicate payments*, *overpayments*, and
   *underpayments*. Output: `output/discrepancy_report.csv`.
2. **KPIs** — collection rate, outstanding principal, PAR30 / PAR90, default rate, a
   vintage (cohort) collection curve, and exposure by sector.
3. **Investor pack** — a multi-tab `.xlsx` (`output/investor_pack.xlsx`) with a Summary
   dashboard, the full exception list, vintage analysis, and sector breakdown. The Summary
   KPIs are **live Excel formulas** referencing the detail tab, so the pack recalculates if
   the data changes.

## How to run

```bash
pip install -r requirements.txt
python src/generate_data.py      # creates synthetic raw data in data/
python src/run_pipeline.py       # reconciles, computes KPIs, writes the pack
```

The synthetic generator plants a known set of data-quality issues
(`data/_planted_issues.csv`) so you can confirm the reconciliation engine catches them.

## Using real data (LendingClub)

To run the pipeline on a **real loan book** instead of synthetic loans, use the
LendingClub loader. It downloads the public LendingClub dataset from Kaggle, maps its
accepted-loans file into the same schema (real principal, term, interest rate,
installment, issue date, purpose), and reuses the repayment simulator to build a
transaction feed on top — so everything downstream runs unchanged.

```bash
pip install kagglehub
python src/load_lending_club.py   # instead of generate_data.py
python src/run_pipeline.py
```

Requires Kaggle credentials (a `kaggle.json` in `~/.kaggle`, or `KAGGLE_USERNAME` /
`KAGGLE_KEY` environment variables). The loader sets the report date to a few months
after the newest loan, so recent vintages are mid-life and older ones have matured.

**Honesty note:** LendingClub is US consumer P2P data, historical (2007–2018) and
anonymised. It is a public research dataset — never present it as a specific employer's
live data. With this loader the loan book is real and the repayment transactions are
simulated on top; the README and code say so plainly.

## Methodology notes

- **Repayment schedule** — flat simple-interest installments (common in SME/fintech
  lending): installment = principal x (1 + rate x term/12) / term.
- **Days past due (DPD)** — the gap between the report date and the oldest scheduled
  installment a loan hasn't yet covered.
- **PAR30 / PAR90** — outstanding balance of loans more than 30 / 90 days past due, as a
  share of total outstanding balance.
- **Collection rate** — total collected / total expected-to-date.

## Project layout

```
src/
  generate_data.py   synthetic loan book, disbursements & repayments (with planted issues)
  reconcile.py       expected-vs-actual matching + exception detection
  metrics.py         DPD, PAR, collection rate, vintage curve, sector view
  build_pack.py      formatted Excel investor pack (openpyxl, with charts)
  run_pipeline.py    orchestrator — one command runs the whole flow
data/                generated raw inputs land here
output/              discrepancy_report.csv + investor_pack.xlsx land here
```

## Tech

Python, pandas, openpyxl. No external data; everything is synthetic and reproducible
(seeded), so the repo is safe to share publicly.

---

*Note: data is synthetic and generated locally for demonstration. The methodology mirrors
standard portfolio-monitoring practice but is simplified for clarity.*
