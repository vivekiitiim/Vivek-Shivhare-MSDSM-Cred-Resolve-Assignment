# Collections Recovery Forensics: Is the "+11% MoM" real?

**Answer: no.** The +11% is March (31 days) compared with February (28 days) on a raw, duplicate-inflated monthly sum.
Per day, on deduplicated payments and a fixed portfolio, recovery moved **+0.3%** that month and has been flat from
January to July 2026 (trend −0.4%/month, 95% CI −1.1% to +0.3%). A second reporting trap, a status-based "active
accounts" denominator that shrinks 29%, would show +42% "improvement" on the same cash.
**Recommendation:** put the ₹10 Cr into **better borrower targeting**, released in stages behind a 10% randomised
holdout. Base case is 1.7× ROI over 3 years with break-even at month 24, and the most we can lose is ₹4.5 Cr.

| Deliverable | Where |
|---|---|
| 1. SQL repository | [`sql/`](sql): `00_reference` → `01_staging` → `02_clean` → `03_golden` → `04_metrics` → `05_forensics` → `06_analysis` → `07_dq_tests` |
| 2. Analysis notebook | [`notebooks/analysis.ipynb`](notebooks/analysis.ipynb) (executed, with outputs). Source: `notebooks/analysis_source.py` |
| 3. Golden dataset + pipeline | [`outputs/golden/`](outputs/golden) (Parquet + CSV.gz), rebuilt by [`pipeline/run_pipeline.py`](pipeline/run_pipeline.py) |
| 4. Data quality report | [`reports/data_quality_report.md`](reports/data_quality_report.md) / `.pdf` |
| 5. Executive dashboard | [`dashboard/executive_dashboard.html`](dashboard/executive_dashboard.html) (one screen, self-contained) and `.png` |
| 6. Executive memo (2 pages) | [`reports/executive_memo.pdf`](reports/executive_memo.pdf) (also `.md`, `.docx`) |
| 7. Architecture diagram | [`architecture/architecture.png`](architecture/architecture.png) / `.svg` and [`docs/production_design.md`](docs/production_design.md) |
| Metric definitions | [`docs/metric_definitions.md`](docs/metric_definitions.md) |

![dashboard](dashboard/executive_dashboard.png)

## Reproduce

Requires Python 3.10+.

```bash
pip install -r requirements.txt
python pipeline/run_pipeline.py      # raw zip → DuckDB warehouse → golden + metrics + DQ tests (~10 s)
python pipeline/build_notebook.py    # builds and executes notebooks/analysis.ipynb, writes outputs/figures
python pipeline/build_dashboard.py   # writes dashboard/executive_dashboard.html
```

The raw data ships as `data/raw/collections_30k_dataset.zip` and is extracted automatically. The pipeline is
deterministic: the same input gives the same output, the source files are fingerprinted with SHA-256 in
`meta_raw_files`, and every rule's impact is logged in `dq_log`. If any **BLOCKER** test fails, the pipeline exits
with status 1.

## How the analysis is built

```
raw_*  (verbatim CSV + _row ingestion order + file SHA-256)
  └─ stg_*  typed, upper-cased codes, every timestamp → IST (source tz kept)
      └─ cln_*  dedup (payment_id, call_id, event_id), entity resolution (borrower latest version,
               agent_id master, vendor → 5 vendors), disposition harmonisation, session merging   → dq_log
          └─ gld_*  dim_account, dim_agent, dim_month, fct_payment (with attribution columns),
                   fct_touch, fct_ptp (payment-verified kept), account_status_month,
                   account_month panel (30,000 accounts × 8 months, nobody dropped)
              └─ mtr_*  reported-replica vs independent KPIs; fx_* forensics A–G; an_* drivers
```

## Key findings (tags: Fact / Strong evidence / Correlation / Hypothesis)

1. **Fact.** The data covers 1 Jan – 8 Aug 2026 (7.3 months, not 12). August is partial.
2. **Fact.** The reported +11% equals the raw Mar/Feb SUCCESS sum (+10.99%). Per day it's +0.29%.
3. **Strong evidence.** Recovery is flat. Daily trend p = 0.31, no structural break at any month boundary, residuals
   behave like white noise.
4. **Fact.** 500 duplicate payment rows (₹2.59 Cr, ~2%) inflate the level but not the trend. 7,216 shared payment
   references are cross-account collisions, not retries.
5. **Fact.** A status-based denominator falls from 26.8k to 18.9k accounts. 26–37% of ₹ comes from "exited" accounts.
   The fake improvement is +42% from January to July.
6. **Fact.** Last-campaign attribution credits campaigns that had already ended 79% of the time. The share of ₹
   "attributed" ranges from 4% to 85% depending on the look-back window.
7. **Fact.** PTP kept is 25% in the CRM against 4% verified by payment. PTP and PROMISE_TO_PAY are both used in all
   three concurrent code versions.
8. **Fact / Hypothesis.** 54% of calls fall outside 08:00–19:00 IST in every timezone label. Either the timestamps
   aren't dial times or there's a compliance issue.
9. **Fact.** Cross-system keys are broken: dispositions↔calls 0%, event borrower IDs agree with the master in <0.01%
   of rows, and each agent ID carries about 30 names.
10. **Strong evidence.** No mix shift, no Simpson's paradox, 0 of 46 segments changed (FDR-adjusted), agent spread
    matches chance.
11. **Strong evidence (given parallel trends).** v3 targeting had no detectable effect: DiD ≈ 0, flat pre-trend, null
    placebo, and PSM agrees.
12. **Correlation (null).** No channel shows an incremental effect, and the data can't rank the investment options.
    Hence targeting plus measurement, gated by a holdout.

## Repository layout

```
sql/            production SQL (DuckDB dialect; dbt-portable)
pipeline/       run_pipeline.py, build_notebook.py, build_dashboard.py
notebooks/      analysis.ipynb (executed) + analysis_source.py
data/raw/       original dataset zip (extracted/ is git-ignored)
outputs/        golden/ (Parquet), tables/ (metrics, forensics, DQ as CSV), figures/, raw_to_golden.csv
reports/        executive memo, data quality report
dashboard/      one-screen executive dashboard
architecture/   production architecture diagram
docs/           metric definitions, production design (Part 5)
```

## Assumptions and limitations

* Tables without a timezone are treated as IST. Fixed UTC offsets are used.
* Unit costs are market assumptions, because no cost data was provided (`ref_cost_assumptions`).
* DPD, risk and status come from overwritten snapshots. DPD at the time of each event can't be recovered.
* There is no client, language or dialler-mode field. Language is inferred from state, and IVR, predictive and AI
  voice can't be separated from other calls.
* The "targeting change" is operationalised as v3 against legacy campaigns from 1 Apr. Strategy versions ran side by
  side rather than switching cleanly.
* Investment returns are assumption-driven scenarios with a capped downside, not forecasts.
