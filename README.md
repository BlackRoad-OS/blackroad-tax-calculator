# BlackRoad Tax Calculator

> **Production-grade US Federal 2024 income tax engine** — bracket logic, effective & marginal rates, withholding estimates, quarterly payments, FICA, LTCG, self-employment tax, and Stripe-ready billing hooks.

Part of the [BlackRoad OS](https://github.com/BlackRoad-OS) platform.

[![Tests](https://img.shields.io/badge/tests-15%20functions-brightgreen)](#testing)
[![Python](https://img.shields.io/badge/python-3.9%2B-blue)](#installation)
[![License](https://img.shields.io/badge/license-Proprietary-red)](#license)
[![IRS](https://img.shields.io/badge/data-IRS%20Rev.%20Proc.%202023--34-informational)](#data-sources)

---

## Table of Contents

1. [Overview](#overview)
2. [Features](#features)
3. [Installation](#installation)
   - [Python / pip](#python--pip)
   - [npm wrapper](#npm-wrapper)
4. [Quick Start](#quick-start)
5. [CLI Reference](#cli-reference)
6. [Python API Reference](#python-api-reference)
   - [TaxCalculatorService](#taxcalculatorservice)
   - [calculate\_tax](#calculate_tax)
   - [get\_effective\_rate](#get_effective_rate)
   - [tax\_brackets](#tax_brackets)
   - [withholding\_estimate](#withholding_estimate)
   - [quarterly\_estimate](#quarterly_estimate)
   - [export\_brackets\_csv](#export_brackets_csv)
   - [Data Classes](#data-classes)
   - [Enumerations](#enumerations)
7. [Stripe Integration](#stripe-integration)
8. [2024 Tax Data](#2024-tax-data)
9. [Architecture](#architecture)
10. [Testing](#testing)
11. [Data Sources](#data-sources)
12. [License](#license)

---

## Overview

**BlackRoad Tax Calculator** is the authoritative tax computation engine powering the BlackRoad OS financial platform. It provides accurate, auditable US federal income tax calculations against all 2024 IRS brackets — including ordinary income, long-term capital gains, self-employment, and qualified dividends — with per-paycheck withholding projections and safe-harbor quarterly estimates.

It ships as a standalone Python library and CLI, with a thin npm wrapper for JavaScript environments. Payment processing and subscription billing are handled via [Stripe](#stripe-integration).

---

## Features

| Feature | Description |
|---|---|
| **2024 Brackets** | All 7 federal brackets for Single, MFJ, MFS, HoH — sourced from IRS Rev. Proc. 2023-34 |
| **Effective & Marginal Rates** | `get_effective_rate()` and full bracket breakdown via `calculate_tax()` |
| **Income Categories** | Ordinary income, short-term & long-term capital gains, self-employment, qualified dividends |
| **Withholding Estimates** | Monthly / bi-weekly / weekly per-paycheck projections including FICA (SS + Medicare) |
| **Quarterly Estimates** | Q1–Q4 due amounts with safe-harbor 110% prior-year rule |
| **Standard Deductions** | Auto-applied by filing status; itemized deduction override supported |
| **Self-Employment Tax** | Full 15.3% SE tax with 50% above-the-line deduction |
| **LTCG / Qualified Dividends** | Separate 0% / 15% / 20% bracket tables |
| **SQLite Persistence** | Audit log of every calculation stored locally |
| **CSV Export** | Bracket tables exportable as CSV for reporting |
| **Stripe-Ready** | Structured calculation results map directly to Stripe `metadata` and `invoice` line items |

---

## Installation

### Python / pip

```bash
pip install blackroad-tax-calculator
```

Or install directly from source:

```bash
git clone https://github.com/BlackRoad-OS/blackroad-tax-calculator.git
cd blackroad-tax-calculator
pip install -e .
```

**Requirements:** Python 3.9+, no third-party runtime dependencies for core tax calculations.

> **Optional:** The [Stripe integration examples](#stripe-integration) require the `stripe` Python package (`pip install stripe`). Stripe is not required for any core calculation functionality.

### npm wrapper

For JavaScript / TypeScript projects that need to shell out to the calculator:

```bash
npm install @blackroad-os/tax-calculator
```

The npm package bundles the Python CLI and exposes a Promise-based API:

```js
const { calculateTax } = require('@blackroad-os/tax-calculator');

const result = await calculateTax({ income: 75000, status: 'single' });
console.log(result.totalTax);   // e.g. 10294.00
console.log(result.effectiveRate); // e.g. 0.1372
```

---

## Quick Start

```python
from decimal import Decimal
from tax_calculator import TaxCalculatorService, TaxDatabase, FilingStatus, TaxCategory
from pathlib import Path

svc = TaxCalculatorService(TaxDatabase(Path("/tmp/tax.db")))

# --- Basic federal tax ---
calc = svc.calculate_tax(Decimal("75000"), "US", filing_status=FilingStatus.SINGLE)
print(f"Total tax:      ${calc.total_tax:,.2f}")
print(f"Effective rate: {calc.effective_rate:.2%}")
print(f"Marginal rate:  {calc.marginal_rate:.0%}")
print(f"Net income:     ${calc.net_income:,.2f}")

# --- Withholding per paycheck ---
est = svc.withholding_estimate(Decimal("75000"), FilingStatus.SINGLE)
print(f"Bi-weekly withholding: ${est.per_paycheck_biweekly:,.2f}")

# --- Q2 quarterly estimate ---
q = svc.quarterly_estimate(Decimal("37500"), Decimal("2000"), quarter=2)
print(f"Q2 amount due:  ${q.amount_due:,.2f}  (due {q.due_date})")
```

---

## CLI Reference

```
Usage: tax-calc <command> [options]
```

| Command | Description | Example |
|---|---|---|
| `calculate` | Full bracket-by-bracket breakdown | `tax-calc calculate 75000 --status single` |
| `rate` | Effective rate only | `tax-calc rate 150000` |
| `brackets` | Print bracket table (or `--csv`) | `tax-calc brackets --status married_jointly` |
| `withholding` | Per-paycheck withholding estimate | `tax-calc withholding 90000 --status single` |
| `quarterly` | Quarterly estimated payment | `tax-calc quarterly 45000 2500 --quarter 2` |

**Global options**

| Flag | Default | Description |
|---|---|---|
| `--db <path>` | `~/.blackroad/tax_calculator.db` | SQLite database path |

**`calculate` options**

| Flag | Values | Default |
|---|---|---|
| `--status` | `single` \| `married_jointly` \| `married_separately` \| `head_of_household` | `single` |
| `--year` | integer | `2024` |
| `--category` | `ordinary_income` \| `capital_gains_short` \| `capital_gains_long` \| `self_employment` \| `qualified_dividends` | `ordinary_income` |
| `--itemized` | dollar amount | *(use standard deduction)* |

**Sample output**

```
=======================================================
  US FEDERAL TAX CALCULATION 2024
=======================================================
  Gross Income:       $    75,000.00
  Std Deduction:      $    14,600.00
  Taxable Income:     $    60,400.00
───────────────────────────────────────────────────────
     10%  $         0–$    11,600    tax=$  1,160.00
     12%  $    11,600–$    47,150    tax=$  4,266.00
     22%  $    47,150–$    60,400    tax=$  2,915.00
───────────────────────────────────────────────────────
  Total Tax:          $     8,341.00
  Effective Rate:           11.12%
  Marginal Rate:            22%
  Net Income:         $    66,659.00
```

---

## Python API Reference

### TaxCalculatorService

```python
class TaxCalculatorService:
    def __init__(self, db: Optional[TaxDatabase] = None)
```

Central computation engine. Pass a `TaxDatabase` instance to persist calculations, or omit for the default path (`~/.blackroad/tax_calculator.db`).

---

### calculate\_tax

```python
def calculate_tax(
    income: Decimal,
    jurisdiction: str,                          # "US" (only supported value)
    year: int = 2024,
    filing_status: FilingStatus = FilingStatus.SINGLE,
    category: TaxCategory = TaxCategory.ORDINARY_INCOME,
    itemized_deductions: Optional[Decimal] = None,
) -> TaxCalculation
```

Performs a full bracket-by-bracket federal income tax calculation.

**Returns** a [`TaxCalculation`](#taxcalculation) dataclass containing:

| Field | Type | Description |
|---|---|---|
| `gross_income` | `Decimal` | Input income |
| `taxable_income` | `Decimal` | After deductions |
| `standard_deduction` | `Decimal` | Applied deduction |
| `total_tax` | `Decimal` | Total federal tax owed |
| `effective_rate` | `Decimal` | `total_tax / gross_income` |
| `marginal_rate` | `Decimal` | Rate of the highest bracket reached |
| `net_income` | `Decimal` | `gross_income - total_tax` (property) |
| `brackets_applied` | `List[TaxBracket]` | Per-bracket breakdown |

**Raises** `ValueError` for negative income or unsupported jurisdiction.

---

### get\_effective\_rate

```python
def get_effective_rate(
    income: Decimal,
    jurisdiction: str,
    filing_status: FilingStatus = FilingStatus.SINGLE,
    year: int = 2024,
) -> Decimal
```

Convenience wrapper — returns the effective rate as a decimal (e.g. `0.1372` for 13.72%).

---

### tax\_brackets

```python
def tax_brackets(
    jurisdiction: str,
    filing_status: FilingStatus = FilingStatus.SINGLE,
    year: int = 2024,
    category: TaxCategory = TaxCategory.ORDINARY_INCOME,
) -> List[TaxRule]
```

Returns the list of [`TaxRule`](#taxrule) objects for the given parameters. Useful for display, audit, or pre-flight UI rendering.

---

### withholding\_estimate

```python
def withholding_estimate(
    salary: Decimal,
    filing_status: FilingStatus = FilingStatus.SINGLE,
    allowances: int = 1,
    year: int = 2024,
) -> WithholdingEstimate
```

Calculates per-paycheck withholding including FICA (Social Security + Medicare + Additional Medicare on high earners).

**Returns** a [`WithholdingEstimate`](#withholdingestimate) with:

| Field | Description |
|---|---|
| `annual_tax` | Federal income tax for the year |
| `fica_social_security` | Annual SS contribution (capped at wage base $168,600) |
| `fica_medicare` | Annual Medicare contribution (+ 0.9% above $200k) |
| `total_annual_withholding` | `annual_tax + fica_ss + fica_medicare` |
| `per_paycheck_monthly` | `total / 12` |
| `per_paycheck_biweekly` | `total / 26` |
| `per_paycheck_weekly` | `total / 52` |

---

### quarterly\_estimate

```python
def quarterly_estimate(
    ytd_income: Decimal,
    ytd_paid: Decimal,
    quarter: int,                               # 1–4
    filing_status: FilingStatus = FilingStatus.SINGLE,
    year: int = 2024,
) -> QuarterlyEstimate
```

Projects the estimated quarterly tax payment using annualized YTD income and the safe-harbor 110% rule.

**Returns** a [`QuarterlyEstimate`](#quarterlyestimate) including `amount_due` and `due_date`.

**Raises** `ValueError` if `quarter` is not 1–4.

---

### export\_brackets\_csv

```python
def export_brackets_csv(
    filing_status: FilingStatus = FilingStatus.SINGLE,
) -> str
```

Returns the 2024 ordinary income brackets for the given filing status as a CSV string. Columns: `Rate`, `Income From`, `Income To`, `Tax On Lower Brackets`.

---

### Data Classes

#### TaxCalculation

```python
@dataclass
class TaxCalculation:
    gross_income: Decimal
    jurisdiction: str
    filing_status: FilingStatus
    year: int
    category: TaxCategory
    brackets_applied: List[TaxBracket]
    total_tax: Decimal
    effective_rate: Decimal
    marginal_rate: Decimal
    standard_deduction: Decimal
    taxable_income: Decimal
    calculated_at: datetime          # UTC timestamp

    @property
    def net_income(self) -> Decimal  # gross_income - total_tax
```

#### TaxBracket

```python
@dataclass
class TaxBracket:
    rate: Decimal
    min_income: Decimal
    max_income: Optional[Decimal]    # None = unbounded
    income_in_bracket: Decimal
    tax_owed: Decimal
```

#### WithholdingEstimate

```python
@dataclass
class WithholdingEstimate:
    gross_salary: Decimal
    filing_status: FilingStatus
    annual_tax: Decimal
    per_paycheck_monthly: Decimal
    per_paycheck_biweekly: Decimal
    per_paycheck_weekly: Decimal
    fica_social_security: Decimal
    fica_medicare: Decimal
    total_annual_withholding: Decimal
```

#### QuarterlyEstimate

```python
@dataclass
class QuarterlyEstimate:
    ytd_income: Decimal
    ytd_paid: Decimal
    quarter: int
    estimated_annual_income: Decimal
    estimated_annual_tax: Decimal
    amount_due: Decimal
    due_date: str                    # e.g. "April 15"
```

#### TaxRule

```python
@dataclass
class TaxRule:
    jurisdiction: str
    rate: Decimal
    category: TaxCategory
    min_income: Decimal
    max_income: Optional[Decimal]
    filing_status: FilingStatus
    year: int
```

---

### Enumerations

#### FilingStatus

| Value | Constant |
|---|---|
| `"single"` | `FilingStatus.SINGLE` |
| `"married_jointly"` | `FilingStatus.MARRIED_JOINTLY` |
| `"married_separately"` | `FilingStatus.MARRIED_SEPARATELY` |
| `"head_of_household"` | `FilingStatus.HEAD_OF_HOUSEHOLD` |

#### TaxCategory

| Value | Constant | Notes |
|---|---|---|
| `"ordinary_income"` | `TaxCategory.ORDINARY_INCOME` | Default; uses standard brackets |
| `"capital_gains_short"` | `TaxCategory.CAPITAL_GAINS_SHORT` | Taxed as ordinary income |
| `"capital_gains_long"` | `TaxCategory.CAPITAL_GAINS_LONG` | Separate 0%/15%/20% brackets |
| `"self_employment"` | `TaxCategory.SELF_EMPLOYMENT` | Adds 15.3% SE tax; deducts 50% |
| `"qualified_dividends"` | `TaxCategory.QUALIFIED_DIVIDENDS` | Uses LTCG brackets |

---

## Stripe Integration

BlackRoad Tax Calculator is designed to feed directly into Stripe billing workflows.

### Attaching tax results to a Stripe PaymentIntent

```python
import stripe
from decimal import Decimal
from tax_calculator import TaxCalculatorService, TaxDatabase, FilingStatus
from pathlib import Path

stripe.api_key = "sk_live_..."   # or sk_test_... for sandbox

svc = TaxCalculatorService(TaxDatabase(Path("/tmp/tax.db")))
calc = svc.calculate_tax(Decimal("75000"), "US", filing_status=FilingStatus.SINGLE)

# Attach calculation results as Stripe metadata
intent = stripe.PaymentIntent.create(
    amount=int(calc.total_tax * 100),   # Stripe uses cents
    currency="usd",
    metadata={
        "gross_income":    str(calc.gross_income),
        "taxable_income":  str(calc.taxable_income),
        "total_tax":       str(calc.total_tax),
        "effective_rate":  str(calc.effective_rate),
        "marginal_rate":   str(calc.marginal_rate),
        "filing_status":   calc.filing_status.value,
        "tax_year":        str(calc.year),
        "calculated_at":   calc.calculated_at.isoformat(),
    },
)
```

### Creating an Invoice line item per bracket

```python
invoice = stripe.Invoice.create(customer=customer_id)

for bracket in calc.brackets_applied:
    stripe.InvoiceItem.create(
        customer=customer_id,
        invoice=invoice.id,
        amount=int(bracket.tax_owed * 100),
        currency="usd",
        description=(
            f"Federal income tax {bracket.rate:.0%} bracket "
            f"(${bracket.min_income:,.0f}–"
            f"{'unlimited' if bracket.max_income is None else f'${bracket.max_income:,.0f}'})"
        ),
    )

stripe.Invoice.finalize_invoice(invoice.id)
```

### Quarterly estimated payment subscription

```python
from tax_calculator import QuarterlyEstimate

q: QuarterlyEstimate = svc.quarterly_estimate(
    ytd_income=Decimal("50000"),
    ytd_paid=Decimal("3000"),
    quarter=2,
)

stripe.PaymentIntent.create(
    amount=int(q.amount_due * 100),
    currency="usd",
    metadata={
        "type":             "quarterly_estimated_tax",
        "quarter":          str(q.quarter),
        "due_date":         q.due_date,
        "ytd_income":       str(q.ytd_income),
        "estimated_annual": str(q.estimated_annual_income),
    },
)
```

> **Note:** Always use Stripe test mode (`sk_test_...`) during development. Never commit live API keys to source control.

---

## 2024 Tax Data

### Ordinary Income Brackets

| Rate | Single | Married Filing Jointly | Married Separately | Head of Household |
|---|---|---|---|---|
| 10% | $0 – $11,600 | $0 – $23,200 | $0 – $11,600 | $0 – $16,550 |
| 12% | $11,600 – $47,150 | $23,200 – $94,300 | $11,600 – $47,150 | $16,550 – $63,100 |
| 22% | $47,150 – $100,525 | $94,300 – $201,050 | $47,150 – $100,525 | $63,100 – $100,500 |
| 24% | $100,525 – $191,950 | $201,050 – $383,900 | $100,525 – $191,950 | $100,500 – $191,950 |
| 32% | $191,950 – $243,725 | $383,900 – $487,450 | $191,950 – $243,725 | $191,950 – $243,700 |
| 35% | $243,725 – $609,350 | $487,450 – $731,200 | $243,725 – $365,600 | $243,700 – $609,350 |
| 37% | $609,350+ | $731,200+ | $365,600+ | $609,350+ |

### Standard Deductions

| Filing Status | 2024 Standard Deduction |
|---|---|
| Single | $14,600 |
| Married Filing Jointly | $29,200 |
| Married Filing Separately | $14,600 |
| Head of Household | $21,900 |

### Long-Term Capital Gains Brackets (2024)

| Rate | Single | Married Filing Jointly |
|---|---|---|
| 0% | $0 – $47,025 | $0 – $94,050 |
| 15% | $47,025 – $518,900 | $94,050 – $583,750 |
| 20% | $518,900+ | $583,750+ |

### FICA Rates (2024)

| Tax | Rate | Notes |
|---|---|---|
| Social Security | 6.2% | Employee share; wage base capped at $168,600 |
| Medicare | 1.45% | Employee share; no wage cap |
| Additional Medicare | 0.9% | On wages above $200,000 (single) |
| Self-Employment | 15.3% | Both halves; 50% is deductible |

---

## Architecture

```
blackroad-tax-calculator/
├── src/
│   └── tax_calculator.py      # Core engine (~610 lines)
│       ├── FilingStatus       # Enum: single, married_jointly, married_separately, head_of_household
│       ├── TaxCategory        # Enum: ordinary_income, capital_gains_*, self_employment, qualified_dividends
│       ├── TaxRule            # Dataclass: a single bracket rule
│       ├── TaxBracket         # Dataclass: bracket result with income and tax applied
│       ├── TaxCalculation     # Dataclass: full calculation result
│       ├── WithholdingEstimate# Dataclass: per-paycheck withholding breakdown
│       ├── QuarterlyEstimate  # Dataclass: quarterly payment projection
│       ├── TaxDatabase        # SQLite persistence (audit log)
│       └── TaxCalculatorService # Primary API
├── tests/
│   └── test_tax_calculator.py # 15 test functions
└── README.md
```

**Design principles:**
- **`Decimal` throughout** — all monetary arithmetic uses `decimal.Decimal` with `ROUND_HALF_UP` to avoid floating-point drift.
- **Immutable results** — every calculation returns a new frozen dataclass; no mutation after construction.
- **Audit log** — every `calculate_tax` call can be persisted to SQLite for compliance.
- **No runtime dependencies** — stdlib only (`decimal`, `dataclasses`, `sqlite3`, `csv`, `argparse`).

---

## Testing

Run the full test suite:

```bash
pip install pytest
pytest tests/ -v
```

Run a single test:

```bash
pytest tests/test_tax_calculator.py::test_calculate_tax_two_brackets -v
```

### End-to-End (e2e) tests

The test suite covers the following scenarios end-to-end:

| Test | What it validates |
|---|---|
| `test_calculate_tax_zero_income` | $0 income → $0 tax, 0% rate |
| `test_calculate_tax_below_standard_deduction` | Income < std deduction → $0 taxable income |
| `test_calculate_tax_single_bracket_10pct` | Single bracket math at 10% |
| `test_calculate_tax_two_brackets` | Two-bracket spillover ($60k single) |
| `test_calculate_tax_married_jointly` | MFJ effective rate sanity check |
| `test_effective_rate_increases_with_income` | Monotone rate progression |
| `test_tax_brackets_returns_all_brackets` | All 7 brackets present for single |
| `test_unsupported_jurisdiction_raises` | Invalid jurisdiction → `ValueError` |
| `test_withholding_estimate_biweekly` | Bi-weekly withholding math + FICA |
| `test_fica_ss_wage_base_cap` | SS wage base cap for high earners |
| `test_quarterly_estimate_q1` | Q1 estimate + due date |
| `test_quarterly_estimate_no_underpayment` | Overpaid → $0 due |
| `test_quarterly_estimate_invalid_quarter` | Quarter 5 → `ValueError` |
| `test_ltcg_rates_lower_than_ordinary` | LTCG tax ≤ ordinary income tax |
| `test_export_brackets_csv` | CSV export contains correct rates |

---

## Data Sources

- **IRS Rev. Proc. 2023-34** — 2024 tax bracket thresholds and standard deductions
- **IRS Publication 15** — FICA rates and Social Security wage base ($168,600 for 2024)
- **IRS Form 1040-ES** — Quarterly estimated tax safe-harbor rules

---

## License

Proprietary — © BlackRoad OS, Inc. All rights reserved.

See [LICENSE](./LICENSE) for full terms.
