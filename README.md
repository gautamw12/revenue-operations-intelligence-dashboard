# Event Revenue Reporting Dashboard

**Portfolio demonstration built on synthetic data.** The company (Harbourline Events), its markets, events, dates, prices, discount codes and results are fictional. They were generated for this demo and are not real business results. The architecture, calculation logic and methodology mirror a production revenue-reporting model I built for a multi-country B2B events business.

## What's in the package

| File | What it is |
|---|---|
| `Harbourline_Revenue_Dashboard_DEMO.html` | Interactive executive dashboard, one self-contained page |
| `Harbourline_Revenue_Dashboard_DEMO.xlsx` | Editable Excel version: formula-driven dashboard, dropdown filters, native charts, targets and FX input sheets |
| `powerbi/*.csv` | Star-schema dataset ready for Power BI |
| `README.md` | This file |

## The business problem

A conference organiser runs recurring summits in several countries. Each edition sells tickets over about 40 weeks, in local currency, with a mix of paid, free (FOC), sponsor and speaker tickets, early-bird pricing, discount codes and split Day 1 / Day 2 passes. Leadership wants to know three things at any point in the cycle: where revenue stands, whether each event is ahead of or behind last year at the **same point in its sales cycle**, and where it is likely to land.

## Dashboard features

**KPI strip:** ticket revenue, total tickets, paid tickets, free tickets (FOC, sponsor and speaker), average paid ticket value, like-for-like YoY growth, revenue target achievement and remaining revenue to target. Each card shows the change against the previous year.

**Analysis:**
- Cumulative pacing curve (WK 40 → WK 0) for the current year, previous year, a dashed forecast projection and the target line
- Weekly sales, current vs previous year
- Revenue by country and by event, with the YoY change in tooltips
- Revenue contribution by event
- Paid vs free ticket mix by event
- Daily ticket sales with 7-day averages, previous year aligned by sales-cycle day
- Revenue by ticket tier, and paid tickets by discount type

**Event performance table:** sortable, with ahead/behind-PY flags, top and weakest performer, targets, % of target and forecast vs target.

**Forecast and velocity:** forecast cards for editions still on sale (three methods, median and range), plus 7- and 28-day velocity against the same days of the previous cycle and the weekly run-rate required to hit target.

**Targets editor:** edit targets, prefill them from previous-year outturn plus a set %, and copy them as CSV.

**Data quality and checks:** line-treatment reconciliation (cancelled, test, VAT/fee, out-of-window) and calculation checks.

**Filters:** reporting year, country, event, ticket category, reporting period presets, custom order-date range, an **as-of date** (backtesting) and base or local currency. Every KPI, chart and table recalculates from the same ticket-line table.

## Methodology

- **Grain.** One row per ticket line. Revenue is the line value net of separate VAT, booking-fee and donation lines.
- **Edition sales window.** From the Monday 40 weeks before event week to the Sunday of event week. Lines outside the window are excluded.
- **Ticket categories.** Sponsor (type contains "Sponsor"), Speaker (contains "Speaker"), Free/FOC (value 0) and Paid (value above 0).
- **No double counting.** Every ticket line is counted once. Combined two-day passes are attributed to both days only in a day-attendance view, with revenue split 50/50, and are never summed twice.
- **Like-for-like YoY.** Previous-year sales are aligned by day of the sales cycle rather than calendar date, so day *n* of last year's window is compared with day *n* of this year's. Date-range and as-of filters map onto the previous-year cycle in the same way. Growth includes only events that have a prior edition.
- **Constant currency.** Cross-country totals convert at one rate per currency for all years, so growth excludes FX movements. A local-currency view is available when the selection shares one currency.
- **Forecast.** Applies only to editions still on sale, and is always labelled and never added to actuals:
  1. Pace ratio: actual × PY outturn ÷ PY at the same day
  2. PY remaining added: actual + (PY outturn − PY at the same day)
  3. Average sales curve: actual ÷ average share of final sales reached by that day across prior editions

  The headline figure is the median of the three and the range is min–max. A method is skipped when history is too thin to be stable.
- **Backtesting.** Setting the as-of date earlier hides later sales. For editions that have since closed, the dashboard shows the forecast next to the actual outturn and the forecast error.

## Power BI data model

```
dim_date ──┐
dim_edition ──< fact_ticket_lines >── dim_ticket_category
dim_programme ──< dim_edition          fx_rates (by currency)   targets (by edition)
```

Relationships: `fact[order_date] → dim_date[date]`, `fact[edition_id] → dim_edition[edition_id]`, `dim_edition[programme_id] → dim_programme[programme_id]`, `fact[category] → dim_ticket_category[category]` and `targets[edition_id] → dim_edition[edition_id]`. The `lfl_date_next_edition` column carries the like-for-like mapping, so previous-year measures can be filtered by the current year's date slicer through an inactive relationship to `dim_date`.

Starter DAX:

```DAX
Revenue := SUM ( fact_ticket_lines[revenue_base] )
Paid Tickets := CALCULATE ( SUM ( fact_ticket_lines[tickets] ), fact_ticket_lines[category] = "Paid" )
Avg Paid Ticket Value := DIVIDE ( [Revenue], [Paid Tickets] )
Revenue PY LfL :=
    VAR cy = SELECTEDVALUE ( dim_edition[edition_year] )
    RETURN CALCULATE ( [Revenue],
        REMOVEFILTERS ( dim_date ), dim_edition[edition_year] = cy - 1,
        USERELATIONSHIP ( fact_ticket_lines[lfl_date_next_edition], dim_date[date] ) )
YoY % := DIVIDE ( [Revenue] - [Revenue PY LfL], [Revenue PY LfL] )
Target Achievement % := DIVIDE ( [Revenue], SUMX ( targets, targets[revenue_target_local] / RELATED ( dim_edition[fx_per_base] ) ) )
```

## Technical implementation

- **Pipeline (Python/pandas):** re-implements the source workbook's classification rules (event mapping table first, then text-pattern fallback; skip rules for test events, cancellations and tax lines; Day 1 / Day 2 split logic). It writes the star schema, the dashboard payload and the Excel workbook from one model.
- **Validation:** in the production version, the re-implementation matched the source workbook row by row and cell by cell, with zero mismatches.
- **HTML dashboard:** vanilla JavaScript and Chart.js. All aggregation runs client-side from a compact columnar payload. Light and dark themes, responsive down to phone width, keyboard-accessible controls, and a data-table view of the pacing chart.
- **Excel:** SUMIFS over named ranges on a fact table, data-validation dropdowns, criteria helpers that handle "All" with wildcards, a MINIFS week cursor and openpyxl-generated native charts. It recalculates with zero formula errors.
- **Synthetic data generation:** ticket lines were generated from a fictional price list with randomised volumes, category mix, discount usage and shifted dates. They were sampled from smoothed, randomised sales-curve shapes so the pacing behaves realistically without reproducing any real figures.
