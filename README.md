# Query-Plan Experiment Notebook

![Python](https://img.shields.io/badge/Python-3.10+-blue)
![SQL](https://img.shields.io/badge/SQL-SQLite-lightblue)
![Platform](https://img.shields.io/badge/Run%20on-Google%20Colab-orange)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen)
![Cost](https://img.shields.io/badge/Cost-Free-success)

A repeatable experiment for SQL indexes: same fake data, same queries, five index candidates, and a report with query plans, repeated timings, a noise floor, result checks and write cost.

## Table of Contents
1. [Problem](#problem)
2. [What I Built](#what-i-built)
3. [Results](#results)
4. [Noise Floor](#noise-floor)
5. [Write Cost](#write-cost)
6. [Acceptance Tests](#acceptance-tests)
7. [Tech Stack](#tech-stack)
8. [How to Run](#how-to-run)
9. [Limitations](#limitations)
10. [About Me](#about-me)

## Problem
A developer adds an index because a query "feels slow", without knowing the whole workload or what the index costs on every write. This project tests that idea properly: several candidates, the same queries, results that must be identical, and a measurement of how noisy the timings are before any speed-up is called real.

## What I Built
- **Fake `customers` (5,000) and `orders` (200,000) tables**, fixed seed, skewed on purpose: customer 3 has 7,344 orders and customer 4000 has 5. Status is 85% delivered, 8% shipped, 5% pending, 2% cancelled
- **6 queries**: Q1a heavy customer filter, Q1b light customer filter, Q2 pending orders with pagination, Q3 revenue by region (join + group by), Q4 count of delivered orders, and Q5 a deliberately terrible self-join
- **5 candidates** (each is a copy of the same base database plus one index set): `none`, `idx_customer` (customer_id), `idx_status_created` (status, created_at), `idx_status_only` (status), `idx_created_cust` (created_at, customer_id)
- For every candidate and query: the **query plan**, a **warm-up run** (kept separate) and **9 timed runs** where the candidates take turns in a rotating order, and a **hash of the result rows**
- A **noise floor** measured from candidates that share the same plan
- **Write cost** test (insert 20,000 rows, update 5,000 rows), 5 repeats, plus the extra index size
- A saved report (`query_plan_report.json`) with versions, row counts, plans, timings and conditions

## Results
Median time in milliseconds (9 timed runs):

| Query | none | idx_customer | idx_status_created | idx_status_only | idx_created_cust |
|---|---|---|---|---|---|
| Q1a heavy customer | 10.26 | **3.54** | 9.38 | 9.20 | 5.90 |
| Q1b light customer | 8.62 | **0.02** | 8.59 | 8.57 | 0.69 |
| Q2 pending, page 6 | 10.71 | 10.63 | **0.07** | 4.99 | 10.81 |
| Q3 revenue by region | **109.32** | 318.37 | 180.11 | 124.85 | 185.54 |
| Q4 delivered count | 17.98 | 18.34 | **11.26** | **11.10** | 18.22 |
| Q5 bad self-join | 500.10 | 500.11 | 500.11 | 500.10 | 500.06 |

A change is only called real when **the plan changed AND the change is bigger than the noise floor**. What the experiment found:

- **Q1 (customer filter):** `idx_customer` is the clear win. 431x faster for the light customer (8.62 ms to 0.02 ms) and 2.9x faster for the heavy one, which still has to return thousands of rows
- **Q2 (pagination):** `idx_status_created` is 153x faster (10.71 ms to 0.07 ms) because the index already holds `pending` rows in date order. `idx_status_only` is only 2.1x faster since it still needs a temporary sort
- **Q3 (join + group by) got slower with every index that changed its plan**: 2.9x slower with `idx_customer`, 1.6x with `idx_status_created`, 1.7x with `idx_created_cust`. With `idx_customer` SQLite flipped the join order (`SCAN c` then `SEARCH o`). A different plan is not always a better plan
- **Q4 (delivered count):** 1.6x faster with a status index, but only because SQLite counts from a smaller covering index. 85% of the rows are `delivered`, so it still reads most entries
- **Q5:** timed out in **15 of 15 runs** (0.5 second limit). Timeouts stay in the table and are never removed

Query plans for Q2 (example):
```
none                 SCAN orders | USE TEMP B-TREE FOR ORDER BY
idx_status_created   SEARCH orders USING INDEX idx_orders_status_created (status=?)
idx_status_only      SEARCH orders USING INDEX idx_orders_status (status=?) | USE TEMP B-TREE FOR ORDER BY
```

## Noise Floor
If two candidates have the **same plan** for a query, any time difference between them is noise. In the final run the biggest spread among same-plan candidates was **x1.14**, so the noise floor was set to the minimum of **x1.3**.

Why this exists: in my first run, the same plan produced very different times (`Q4` with `idx_customer` took 91 ms against 18 ms, with an identical `SCAN orders` plan) and some indexed inserts even looked faster than no index. Running the candidates in rotating order, adding warm-up runs and adding the noise floor made the results consistent with the plans.

## Write Cost
5 repeats on a fresh copy each time. A change counts only if the min to max ranges do not overlap and the median moved by more than the noise floor.

| Candidate | Insert 20k (median ms) | vs none | Update 5k (median ms) | vs none | Extra size |
|---|---|---|---|---|---|
| none | 19.5 | - | 1.4 | - | 0 MB |
| idx_customer | 32.0 | unclear (inside noise) | 1.4 | unclear (inside noise) | 1.95 MB |
| idx_status_created | 54.1 | slower x2.8 | 13.5 | slower x9.4 | 5.50 MB |
| idx_status_only | 33.8 | slower x1.7 | 6.7 | slower x4.7 | 3.37 MB |
| idx_created_cust | 41.1 | slower x2.1 | 1.4 | unclear (inside noise) | 4.06 MB |

The status indexes slow updates the most, because the updated column (`status`) is inside the index. The `idx_customer` index is cheap to keep: its write cost is inside the noise here.

## Acceptance Tests
| Test | Result |
|---|---|
| Every candidate returns identical results (same row hash) for every query that finished | PASS |
| The report records versions, row counts, plans, conditions and timings | PASS |
| The timed-out query (Q5) is recorded, 15 timeouts in 15 runs | PASS |

## Tech Stack
| Area | Tools |
|---|---|
| Language | Python |
| Database | SQLite (built into Python, SQLite 3.45.1) |
| Data | pandas, numpy |
| Charts | matplotlib |
| Environment | Google Colab |

## How to Run
1. Open `Query_Plan_Experiment_Notebook.ipynb` in Google Colab
2. Click **Runtime > Run all** (takes about 1 to 2 minutes)
3. No dataset or API key is needed, all data is generated inside the notebook (seed 42)

Or locally: `pip install -r requirements.txt` and open the notebook in Jupyter.

## Limitations
- Timings come from one machine and one in-memory SQLite database. Read them as relative, not absolute
- SQLite is not PostgreSQL. A real PostgreSQL run with `EXPLAIN (ANALYZE)` and its row estimates is not done yet
- The data is fake. Real data can have different skew, and the results would change with it
- There is no automatic regression gate yet that flags a plan change between runs

## About Me
**Akshat Kesharwani** - Data Analyst / Data Scientist (fresher)

- Portfolio: https://akshatkesharwani-info.github.io/
- GitHub: https://github.com/akshatkesharwani-info

Idea source: "Top 50 Projects to Build in 2026" (Yasir Arafat Sharfi). The build, data and code here are my own.
