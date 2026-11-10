# Vendor Compliance Analytics (PostgreSQL)

SQL project on contract-labour vendor compliance: statutory documents (PF, ESIC, PT, LWF), headcount and vendor billing across 25 vendors, 23 locations and 57 vendor sites (Sep 2025 to Aug 2026).

**Note:** the dataset is synthetic (dummy) data created for practice. It contains no real company data.

## What this project covers
- 8-table star schema (6 dimensions, 2 facts) in PostgreSQL
- 50 business questions, from basics to window functions, views, a function and a trigger
- Every solution has the query, sample output and a one-line insight

## Folder structure
| Folder / file | Contents |
|---|---|
| `schema.sql` | Table definitions |
| `data/` | 8 CSV files |
| `sql/solutions.sql` | All 50 solutions |
| `docs/` | PDF with questions, solutions and insights; question list |

## How to run
1. Create a database and run `schema.sql`
2. Load the CSVs 
3. Run `sql/solutions.sql`

## Key findings
- Overall compliance is 85.3% and flat all year (82.5% to 87.6%)
- ESIC ECR (78.2%), PT Challan (79.5%) and PF ECR (79.7%) are the weakest documents
- 12 of 25 vendors are below average; 8 of them are below 63%
- 70 site-document combinations were missed in 4 or more months
- Top 3 vendors account for 43.5% of total payout (Rs 146.58 crore)
- One vendor billed 8.0% instead of the contracted 7.5% on two sites (Rs 4,03,787 over-billed)
- 30 of 65 "PF challan awaited" remarks were stale

## Skills shown
JOINs, CTEs, subqueries, CASE WHEN, window functions (RANK, ROW_NUMBER, LAG, NTILE, running totals, moving averages), PERCENTILE_CONT, views, PL/pgSQL function, trigger, data validation

Full list of questions: [docs/SQL_Business_Questions.md](docs/SQL_Business_Questions.md)
