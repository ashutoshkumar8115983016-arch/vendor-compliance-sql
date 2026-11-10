
--Vendor Compliance Analytics

--Q.1 How many vendors, locations and vendor sites do we have? How many sites are currently active vs closed?

SELECT (SELECT COUNT(*) FROM dim_vendor)   AS total_vendors,
       (SELECT COUNT(*) FROM dim_location) AS total_locations,
       COUNT(*)                            AS total_sites,
       COUNT(*) FILTER (WHERE is_active = 1) AS active_sites,
       COUNT(*) FILTER (WHERE is_active = 0) AS closed_sites
FROM dim_vendor_site;

--Q2. How many vendors are there in each vendor type (Manpower, Security, NAPS)?

SELECT
	vendor_type,
	COUNT(vendor_id)
FROM dim_vendor
GROUP BY 1
ORDER BY 1

--Q3. How many locations are in each region and state? Which state has the most locations?

SELECT 
	region,
	state,
	COUNT(location_id)
FROM dim_location
GROUP BY 1,2
ORDER BY 1,3 DESC


--Q4. List all vendors with a contract service charge of 9% or more, highest first.

SELECT vendor_name,contract_service_charge_pct
FROM dim_vendor
WHERE contract_service_charge_pct >= 9
ORDER BY 2 DESC

--Q5. Which sites were terminated? Show vendor, location, start month and end month.

SELECT s.site_id, v.vendor_name, l.location_name,
       ds.month_name AS start_month, de.month_name AS end_month
FROM dim_vendor_site s
JOIN dim_vendor   v  ON v.vendor_id   = s.vendor_id
JOIN dim_location l  ON l.location_id = s.location_id
JOIN dim_date     ds ON ds.date_key   = s.start_date_key
LEFT JOIN dim_date de ON de.date_key  = s.end_date_key
WHERE s.is_active = 0;

--Q6. What is the total headcount deployed in the latest month (Aug 2026)?

SELECT SUM(current_headcount) AS total_headcount
FROM fact_vendor_monthly
WHERE date_key = (SELECT MAX(date_key) FROM fact_vendor_monthly);

--Q7. What is the total payout over the 12 months, and the average monthly payout (in Rs crore)?

SELECT SUM(total_amount_paid)                                   AS total_paid,
       ROUND(SUM(total_amount_paid) / 10000000.0, 2)            AS total_crore,
       ROUND(SUM(total_amount_paid) / COUNT(DISTINCT date_key) / 10000000.0, 2) AS avg_monthly_crore
FROM fact_vendor_monthly;

--Q8. How many site-months fall under each mail status (Sent, Reminder-1, Reminder-2, Not Sent)?
SELECT mail_status,
       COUNT(*) AS site_months,
       ROUND(100.0 * COUNT(*) / (SELECT COUNT(*) FROM fact_vendor_monthly), 1) AS pct
FROM fact_vendor_monthly
GROUP BY mail_status
ORDER BY site_months DESC

--Q9. What are the most frequent open-point remarks? Show the top 3 (ignore blanks).

SELECT
	Open_points_remarks,
	COUNT(Open_points_remarks)
FROM fact_vendor_monthly
WHERE Open_points_remarks IS NOT NULL
GROUP BY 1
HAVING COUNT(Open_points_remarks) >1
ORDER BY 2 DESC
LIMIT 3

--Q10. Which locations do not have PT or LWF applicable?

SELECT location_name, state, pt_applicable, lwf_applicable
FROM dim_location
WHERE pt_applicable = 0 OR lwf_applicable = 0
ORDER BY state, location_name;

--Q11. How many compliance documents are Received, Not Received and N/A?

SELECT st.status_name, COUNT(*) AS documents
FROM fact_compliance_doc d
JOIN dim_status st ON st.status_id = d.status_id
GROUP BY st.status_name
ORDER BY documents DESC;

--Q12. Show month-wise total headcount and total payout (Sep'25 to Aug'26).

SELECT dd.month_name,
       SUM(f.current_headcount)  AS total_headcount,
       SUM(f.total_amount_paid)  AS total_paid
FROM fact_vendor_monthly f
JOIN dim_date dd ON dd.date_key = f.date_key
GROUP BY dd.month_name, dd.month_sort
ORDER BY dd.month_sort;

--Q13. Top 10 vendors by total payout and each vendor's % share of overall payout.

SELECT v.vendor_name, v.vendor_type,
       SUM(f.total_amount_paid) AS total_paid,
       ROUND(100.0 * SUM(f.total_amount_paid)
             / (SELECT SUM(total_amount_paid) FROM fact_vendor_monthly), 2) AS pct_share
FROM fact_vendor_monthly f
JOIN dim_vendor_site s ON s.site_id = f.site_id
JOIN dim_vendor v      ON v.vendor_id = s.vendor_id
GROUP BY v.vendor_name, v.vendor_type
ORDER BY total_paid DESC
LIMIT 10;

--Q14. Region-wise headcount and payout for Aug 2026.

SELECT l.region,
       SUM(f.current_headcount) AS headcount,
       SUM(f.total_amount_paid) AS total_paid
FROM fact_vendor_monthly f
JOIN dim_vendor_site s ON s.site_id = f.site_id
JOIN dim_location    l ON l.location_id = s.location_id
WHERE f.date_key = 20260801
GROUP BY l.region
ORDER BY headcount DESC;

--Q15. State-wise number of active sites and total headcount in the latest month.

SELECT l.state,
       COUNT(*) AS active_sites,
       SUM(f.current_headcount) AS headcount
FROM fact_vendor_monthly f
JOIN dim_vendor_site s ON s.site_id = f.site_id
JOIN dim_location    l ON l.location_id = s.location_id
WHERE f.date_key = (SELECT MAX(date_key) FROM fact_vendor_monthly)
  AND f.vendor_status = 'Active'
GROUP BY l.state
ORDER BY headcount DESC;

--Q16. What is the overall compliance %?

SELECT COUNT(*) FILTER (WHERE st.is_received = 1) AS received,
       COUNT(*) AS in_compliance_base,
       ROUND(100.0 * COUNT(*) FILTER (WHERE st.is_received = 1) / COUNT(*), 2) AS compliance_pct
FROM fact_compliance_doc d
JOIN dim_status st ON st.status_id = d.status_id
WHERE st.counts_in_compliance_base = 1;

--Q17. Which of the 9 document types has the lowest compliance %? Rank them.

SELECT dt.doc_name,
       ROUND(100.0 * COUNT(*) FILTER (WHERE st.is_received = 1) / COUNT(*), 1) AS compliance_pct
FROM fact_compliance_doc d
JOIN dim_doc_type dt ON dt.doc_type_id = d.doc_type_id
JOIN dim_status   st ON st.status_id   = d.status_id
WHERE st.counts_in_compliance_base = 1
GROUP BY dt.doc_name
ORDER BY compliance_pct;

--Q18. Compliance % by document category (PF, ESIC, PT, LWF, Salary, Records).

SELECT dt.doc_category,
       COUNT(*) AS docs_in_base,
       ROUND(100.0 * COUNT(*) FILTER (WHERE st.is_received = 1) / COUNT(*), 1) AS compliance_pct
FROM fact_compliance_doc d
JOIN dim_doc_type dt ON dt.doc_type_id = d.doc_type_id
JOIN dim_status   st ON st.status_id   = d.status_id
WHERE st.counts_in_compliance_base = 1
GROUP BY dt.doc_category
ORDER BY compliance_pct;

--Q19. Which vendors have a compliance % below the overall average?

WITH base AS (
    SELECT v.vendor_name, st.is_received
    FROM fact_compliance_doc d
    JOIN dim_status      st ON st.status_id = d.status_id
    JOIN dim_vendor_site s  ON s.site_id    = d.site_id
    JOIN dim_vendor      v  ON v.vendor_id  = s.vendor_id
    WHERE st.counts_in_compliance_base = 1
)
SELECT vendor_name,
       ROUND(100.0 * SUM(is_received) / COUNT(*), 1) AS compliance_pct
FROM base
GROUP BY vendor_name
HAVING 100.0 * SUM(is_received) / COUNT(*) < (SELECT 100.0 * SUM(is_received) / COUNT(*) FROM base)
ORDER BY compliance_pct;

--Q20. Month-wise compliance % trend. Which month was the worst?

SELECT dd.month_name,
       ROUND(100.0 * COUNT(*) FILTER (WHERE st.is_received = 1) / COUNT(*), 1) AS compliance_pct
FROM fact_compliance_doc d
JOIN dim_status st ON st.status_id = d.status_id
JOIN dim_date   dd ON dd.date_key  = d.date_key
WHERE st.counts_in_compliance_base = 1
GROUP BY dd.month_name, dd.month_sort
ORDER BY dd.month_sort;

--Q21. Which vendors needed Reminder-2 more than twice?

SELECT v.vendor_name, COUNT(*) AS reminder2_count
FROM fact_vendor_monthly f
JOIN dim_vendor_site s ON s.site_id = f.site_id
JOIN dim_vendor v      ON v.vendor_id = s.vendor_id
WHERE f.mail_status = 'Reminder-2'
GROUP BY v.vendor_name
HAVING COUNT(*) > 2
ORDER BY reminder2_count DESC;

--Q22. Compare vendor types: average headcount per site-month, total payout and compliance %.

WITH ops AS (
    SELECT v.vendor_type,
           COUNT(DISTINCT s.site_id)         AS sites,
           ROUND(AVG(f.current_headcount),1) AS avg_headcount,
           SUM(f.total_amount_paid)          AS total_paid
    FROM fact_vendor_monthly f
    JOIN dim_vendor_site s ON s.site_id = f.site_id
    JOIN dim_vendor      v ON v.vendor_id = s.vendor_id
    GROUP BY v.vendor_type
),
comp AS (
    SELECT v.vendor_type,
           ROUND(100.0 * COUNT(*) FILTER (WHERE st.is_received = 1) / COUNT(*), 1) AS compliance_pct
    FROM fact_compliance_doc d
    JOIN dim_status      st ON st.status_id = d.status_id
    JOIN dim_vendor_site s  ON s.site_id    = d.site_id
    JOIN dim_vendor      v  ON v.vendor_id  = s.vendor_id
    WHERE st.counts_in_compliance_base = 1
    GROUP BY v.vendor_type
)
SELECT o.vendor_type, o.sites, o.avg_headcount, o.total_paid, c.compliance_pct
FROM ops o
JOIN comp c ON c.vendor_type = o.vendor_type
ORDER BY o.total_paid DESC;

--Q23. List site-months where mail was 'Not Sent' or the vendor was not Active, with remarks.

SELECT dd.month_name, v.vendor_name, l.location_name,
       f.mail_status, f.vendor_status, f.open_points_remarks
FROM fact_vendor_monthly f
JOIN dim_vendor_site s ON s.site_id     = f.site_id
JOIN dim_vendor      v ON v.vendor_id   = s.vendor_id
JOIN dim_location    l ON l.location_id = s.location_id
JOIN dim_date       dd ON dd.date_key   = f.date_key
WHERE f.mail_status = 'Not Sent' OR f.vendor_status <> 'Active'
ORDER BY f.date_key, v.vendor_name;

--Q24. Bucket active sites by size in the latest month (Small <25, Medium 25-100, Largem >100). Count sites and payout per bucket.

SELECT CASE WHEN current_headcount < 25   THEN '1. Small (<25)'
            WHEN current_headcount <= 100  THEN '2. Medium (25-100)'
            ELSE                                '3. Large (>100)' END AS site_size,
       COUNT(*)                 AS sites,
       SUM(total_amount_paid)   AS payout
FROM fact_vendor_monthly
WHERE date_key = (SELECT MAX(date_key) FROM fact_vendor_monthly)
  AND vendor_status = 'Active'
GROUP BY 1
ORDER BY 1;

--Q25. Total service charge and GST per vendor. What is the effective service charge as a % of CTC?

SELECT v.vendor_name,
       SUM(f.monthly_ctc_amount)    AS total_ctc,
       SUM(f.service_charge_amount) AS service_charge,
       SUM(f.gst_on_service_charge) AS gst,
       ROUND(100.0 * SUM(f.service_charge_amount)
             / NULLIF(SUM(f.monthly_ctc_amount), 0), 2) AS effective_sc_pct
FROM fact_vendor_monthly f
JOIN dim_vendor_site s ON s.site_id = f.site_id
JOIN dim_vendor v      ON v.vendor_id = s.vendor_id
GROUP BY v.vendor_name
ORDER BY service_charge DESC
LIMIT 10;

--Q26. Which active sites grew headcount by more than 20% between their first and last month?

WITH ranked AS (
    SELECT site_id, current_headcount,
           ROW_NUMBER() OVER (PARTITION BY site_id ORDER BY date_key)      AS rn_first,
           ROW_NUMBER() OVER (PARTITION BY site_id ORDER BY date_key DESC) AS rn_last
    FROM fact_vendor_monthly
    WHERE vendor_status = 'Active'
)
SELECT f1.site_id, v.vendor_name,
       f1.current_headcount AS first_headcount,
       l1.current_headcount AS last_headcount,
       ROUND(100.0 * (l1.current_headcount - f1.current_headcount) / f1.current_headcount, 1) AS growth_pct
FROM ranked f1
JOIN ranked l1 ON l1.site_id = f1.site_id AND l1.rn_last = 1
JOIN dim_vendor_site s ON s.site_id = f1.site_id
JOIN dim_vendor v      ON v.vendor_id = s.vendor_id
WHERE f1.rn_first = 1
  AND f1.current_headcount > 0
  AND 100.0 * (l1.current_headcount - f1.current_headcount) / f1.current_headcount > 20
ORDER BY growth_pct DESC;

--Q27. Which vendors operate in more than 3 locations?

SELECT v.vendor_name, COUNT(DISTINCT s.location_id) AS locations
FROM dim_vendor_site s
JOIN dim_vendor v ON v.vendor_id = s.vendor_id
GROUP BY v.vendor_name
HAVING COUNT(DISTINCT s.location_id) > 3
ORDER BY locations DESC;

--Q28. Which locations currently have no active vendor site?

SELECT l.location_id, l.location_name, l.state
FROM dim_location l
LEFT JOIN dim_vendor_site s
       ON s.location_id = l.location_id AND s.is_active = 1
WHERE s.site_id IS NULL;

--Q29. Rank vendors by total payout within each vendor type.
WITH vp AS (
    SELECT v.vendor_type, v.vendor_name, SUM(f.total_amount_paid) AS total_paid
    FROM fact_vendor_monthly f
    JOIN dim_vendor_site s ON s.site_id = f.site_id
    JOIN dim_vendor      v ON v.vendor_id = s.vendor_id
    GROUP BY v.vendor_type, v.vendor_name
)
SELECT vendor_type, vendor_name, total_paid,
       RANK() OVER (PARTITION BY vendor_type ORDER BY total_paid DESC) AS rank_in_type
FROM vp
ORDER BY vendor_type, rank_in_type;

--Q30. Top 3 sites by headcount in each region (latest month).

WITH latest AS (
    SELECT l.region, v.vendor_name, l.location_name, f.current_headcount,
           ROW_NUMBER() OVER (PARTITION BY l.region
                              ORDER BY f.current_headcount DESC) AS rn
    FROM fact_vendor_monthly f
    JOIN dim_vendor_site s ON s.site_id     = f.site_id
    JOIN dim_vendor      v ON v.vendor_id   = s.vendor_id
    JOIN dim_location    l ON l.location_id = s.location_id
    WHERE f.date_key = (SELECT MAX(date_key) FROM fact_vendor_monthly)
)
SELECT region, rn AS rank, vendor_name, location_name, current_headcount
FROM latest
WHERE rn <= 3
ORDER BY region, rn;

--Q31. Compute month-on-month headcount change per site with LAG and flag drops above 10%. Also check LAG against the previous_headcount column.
WITH hc AS (
    SELECT f.site_id, dd.month_name, f.date_key,
           f.current_headcount, f.previous_headcount,
           LAG(f.current_headcount) OVER (PARTITION BY f.site_id
                                          ORDER BY f.date_key) AS lag_headcount
    FROM fact_vendor_monthly f
    JOIN dim_date dd ON dd.date_key = f.date_key
)
SELECT site_id, month_name, lag_headcount, current_headcount,
       ROUND(100.0 * (current_headcount - lag_headcount) / lag_headcount, 1) AS mom_pct
FROM hc
WHERE lag_headcount > 0 AND current_headcount > 0
  AND 100.0 * (current_headcount - lag_headcount) / lag_headcount < -10
ORDER BY mom_pct;

WITH hc AS (
    SELECT previous_headcount,
           LAG(current_headcount) OVER (PARTITION BY site_id ORDER BY date_key) AS lag_headcount
    FROM fact_vendor_monthly
)
SELECT COUNT(*) AS mismatches
FROM hc
WHERE lag_headcount IS NOT NULL AND lag_headcount <> previous_headcount;

--Q32.Month-on-month % growth in total payout.

WITH m AS (
    SELECT dd.month_sort, dd.month_name, SUM(f.total_amount_paid) AS total_paid
    FROM fact_vendor_monthly f
    JOIN dim_date dd ON dd.date_key = f.date_key
    GROUP BY dd.month_sort, dd.month_name
),
x AS (
    SELECT month_sort, month_name, total_paid,
           LAG(total_paid) OVER (ORDER BY month_sort) AS prev_paid
    FROM m
)
SELECT month_name, total_paid, prev_paid,
       ROUND(100.0 * (total_paid - prev_paid) / prev_paid, 2) AS mom_growth_pct
FROM x
ORDER BY month_sort;

--Q33. Cumulative payout over the months, resetting each fiscal year.
WITH m AS (
    SELECT dd.fiscal_year, dd.month_sort, dd.month_name,
           SUM(f.total_amount_paid) AS monthly_paid
    FROM fact_vendor_monthly f
    JOIN dim_date dd ON dd.date_key = f.date_key
    GROUP BY dd.fiscal_year, dd.month_sort, dd.month_name
)
SELECT fiscal_year, month_name, monthly_paid,
       SUM(monthly_paid) OVER (PARTITION BY fiscal_year
                               ORDER BY month_sort) AS cumulative_paid
FROM m
ORDER BY month_sort;

--Q34. 3-month moving average of total headcount.

WITH m AS (
    SELECT dd.month_sort, dd.month_name, SUM(f.current_headcount) AS total_headcount
    FROM fact_vendor_monthly f
    JOIN dim_date dd ON dd.date_key = f.date_key
    GROUP BY dd.month_sort, dd.month_name
)
SELECT month_name, total_headcount,
       ROUND(AVG(total_headcount) OVER (ORDER BY month_sort
             ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 1) AS moving_avg_3m
FROM m
ORDER BY month_sort;

--Q35. Compare each vendor's compliance % in Sep-Feb vs Mar-Aug. Who improved most and who declined most?

WITH c AS (
    SELECT v.vendor_name,
           100.0 * COUNT(*) FILTER (WHERE st.is_received = 1 AND d.date_key <= 20260201)
                 / NULLIF(COUNT(*) FILTER (WHERE d.date_key <= 20260201), 0) AS h1_pct,
           100.0 * COUNT(*) FILTER (WHERE st.is_received = 1 AND d.date_key >= 20260301)
                 / NULLIF(COUNT(*) FILTER (WHERE d.date_key >= 20260301), 0) AS h2_pct
    FROM fact_compliance_doc d
    JOIN dim_status      st ON st.status_id = d.status_id
    JOIN dim_vendor_site s  ON s.site_id    = d.site_id
    JOIN dim_vendor      v  ON v.vendor_id  = s.vendor_id
    WHERE st.counts_in_compliance_base = 1
    GROUP BY v.vendor_name
)
SELECT vendor_name,
       ROUND(h1_pct, 1) AS sep_feb_pct,
       ROUND(h2_pct, 1) AS mar_aug_pct,
       ROUND(h2_pct - h1_pct, 1) AS change_pts
FROM c
WHERE h1_pct IS NOT NULL AND h2_pct IS NOT NULL
ORDER BY change_pts DESC;

--Q36. Find sites where PF Challan was 'Not Received' in two consecutive months.

WITH pf AS (
    SELECT d.site_id, d.date_key, dd.month_name, d.status_id,
           LAG(d.status_id) OVER (PARTITION BY d.site_id ORDER BY d.date_key) AS prev_status
    FROM fact_compliance_doc d
    JOIN dim_date dd ON dd.date_key = d.date_key
    WHERE d.doc_type_id = 1
)
SELECT pf.site_id, v.vendor_name, l.location_name, pf.month_name
FROM pf
JOIN dim_vendor_site s ON s.site_id     = pf.site_id
JOIN dim_vendor      v ON v.vendor_id   = s.vendor_id
JOIN dim_location    l ON l.location_id = s.location_id
WHERE pf.status_id = 2 AND pf.prev_status = 2
ORDER BY pf.site_id, pf.date_key;

--Q37. Repeat defaulters: which site + document combinations were 'Not Received' in 4 or more months?

SELECT d.site_id, v.vendor_name, dt.doc_name, COUNT(*) AS months_not_received
FROM fact_compliance_doc d
JOIN dim_doc_type    dt ON dt.doc_type_id = d.doc_type_id
JOIN dim_vendor_site s  ON s.site_id      = d.site_id
JOIN dim_vendor      v  ON v.vendor_id    = s.vendor_id
WHERE d.status_id = 2
GROUP BY d.site_id, v.vendor_name, dt.doc_name
HAVING COUNT(*) >= 4
ORDER BY months_not_received DESC, d.site_id
LIMIT 10;

--Q38. Average turnaround days per document type (PF Challan, ESIC Challan, Salary UTR).
SELECT dt.doc_name,
       COUNT(*) AS docs,
       ROUND(AVG(d.doc_date - dd.month_start), 1) AS avg_days,
       MIN(d.doc_date - dd.month_start)AS min_days,
       MAX(d.doc_date - dd.month_start)AS max_days
FROM fact_compliance_doc d
JOIN dim_doc_type dt ON dt.doc_type_id = d.doc_type_id
JOIN dim_date     dd ON dd.date_key    = d.date_key
WHERE d.doc_date IS NOT NULL
GROUP BY dt.doc_name
ORDER BY avg_days DESC;

--Q39. Which 5 vendors are slowest at submitting PF Challan? Show median turnaround, not just average.

SELECT v.vendor_name,
       COUNT(*) AS docs,
       ROUND(AVG(d.doc_date - dd.month_start), 1) AS avg_days,
       ROUND((PERCENTILE_CONT(0.5) WITHIN GROUP
             (ORDER BY (d.doc_date - dd.month_start)))::numeric, 1) AS median_days
FROM fact_compliance_doc d
JOIN dim_date        dd ON dd.date_key = d.date_key
JOIN dim_vendor_site s  ON s.site_id   = d.site_id
JOIN dim_vendor      v  ON v.vendor_id = s.vendor_id
WHERE d.doc_type_id = 1 AND d.doc_date IS NOT NULL
GROUP BY v.vendor_name
ORDER BY median_days DESC, avg_days DESC
LIMIT 5;

--Q40. Fiscal-quarter-wise payout and compliance %.

WITH pay AS (
    SELECT date_key, SUM(total_amount_paid) AS paid
    FROM fact_vendor_monthly
    GROUP BY date_key
),
comp AS (
    SELECT d.date_key,
           COUNT(*) FILTER (WHERE st.is_received = 1) AS received,
           COUNT(*)                                    AS in_base
    FROM fact_compliance_doc d
    JOIN dim_status st ON st.status_id = d.status_id
    WHERE st.counts_in_compliance_base = 1
    GROUP BY d.date_key
)
SELECT dd.fiscal_year, dd.fiscal_quarter,
       SUM(p.paid) AS total_paid,
       ROUND(100.0 * SUM(c.received) / SUM(c.in_base), 2) AS compliance_pct
FROM dim_date dd
JOIN pay  p ON p.date_key = dd.date_key
JOIN comp c ON c.date_key = dd.date_key
GROUP BY dd.fiscal_year, dd.fiscal_quarter
ORDER BY MIN(dd.month_sort);

--Q41. Build a vendor x month compliance % matrix (pivot).
SELECT v.vendor_name,
  ROUND(100*AVG(st.is_received) FILTER (WHERE d.date_key = 20250901), 0) AS sep_25,
  ROUND(100*AVG(st.is_received) FILTER (WHERE d.date_key = 20251001), 0) AS oct_25,
  ROUND(100*AVG(st.is_received) FILTER (WHERE d.date_key = 20251101), 0) AS nov_25,
  ROUND(100*AVG(st.is_received) FILTER (WHERE d.date_key = 20251201), 0) AS dec_25,
  ROUND(100*AVG(st.is_received) FILTER (WHERE d.date_key = 20260101), 0) AS jan_26,
  ROUND(100*AVG(st.is_received) FILTER (WHERE d.date_key = 20260201), 0) AS feb_26,
  ROUND(100*AVG(st.is_received) FILTER (WHERE d.date_key = 20260301), 0) AS mar_26,
  ROUND(100*AVG(st.is_received) FILTER (WHERE d.date_key = 20260401), 0) AS apr_26,
  ROUND(100*AVG(st.is_received) FILTER (WHERE d.date_key = 20260501), 0) AS may_26,
  ROUND(100*AVG(st.is_received) FILTER (WHERE d.date_key = 20260601), 0) AS jun_26,
  ROUND(100*AVG(st.is_received) FILTER (WHERE d.date_key = 20260701), 0) AS jul_26,
  ROUND(100*AVG(st.is_received) FILTER (WHERE d.date_key = 20260801), 0) AS aug_26
FROM fact_compliance_doc d
JOIN dim_status      st ON st.status_id = d.status_id
JOIN dim_vendor_site s  ON s.site_id    = d.site_id
JOIN dim_vendor      v  ON v.vendor_id  = s.vendor_id
WHERE st.counts_in_compliance_base = 1
GROUP BY v.vendor_name
ORDER BY v.vendor_name;

--Q42. Cost per head (CTC / headcount) by site. Show the 5 costliest sites within each vendor type and how far each is above its type average.
WITH site_cost AS (
    SELECT f.site_id, v.vendor_type, l.location_name,
           ROUND(SUM(f.monthly_ctc_amount)::numeric
                 / NULLIF(SUM(f.current_headcount), 0), 0) AS ctc_per_head
    FROM fact_vendor_monthly f
    JOIN dim_vendor_site s ON s.site_id     = f.site_id
    JOIN dim_vendor      v ON v.vendor_id   = s.vendor_id
    JOIN dim_location    l ON l.location_id = s.location_id
    WHERE f.current_headcount > 0
    GROUP BY f.site_id, v.vendor_type, l.location_name
),
x AS (
    SELECT *,
           AVG(ctc_per_head) OVER (PARTITION BY vendor_type) AS type_avg,
           ROW_NUMBER() OVER (PARTITION BY vendor_type
                              ORDER BY ctc_per_head DESC)    AS rn
    FROM site_cost
)
SELECT vendor_type, site_id, location_name, ctc_per_head,
       ROUND(type_avg, 0) AS type_avg,
       ROUND(100.0 * (ctc_per_head - type_avg) / type_avg, 1) AS pct_above_avg
FROM x
WHERE rn <= 5
ORDER BY vendor_type, rn;

--Q43. Billing audit: find rows where the charged service charge % differs from the contract %. How much was over/under-billed?
SELECT dd.month_name, f.site_id, v.vendor_name,
       v.contract_service_charge_pct AS contract_pct,
       f.service_charge_pct          AS charged_pct,
       f.service_charge_amount       AS charged_amount,
       ROUND(f.monthly_ctc_amount * v.contract_service_charge_pct / 100) AS expected_amount,
       f.service_charge_amount
         - ROUND(f.monthly_ctc_amount * v.contract_service_charge_pct / 100) AS over_under_billed
FROM fact_vendor_monthly f
JOIN dim_vendor_site s ON s.site_id   = f.site_id
JOIN dim_vendor      v ON v.vendor_id = s.vendor_id
JOIN dim_date       dd ON dd.date_key = f.date_key
WHERE f.service_charge_pct <> v.contract_service_charge_pct
ORDER BY f.date_key, f.site_id;

--Q44. Recalculate service charge, 18% GST and total payout. Flag any row where stored values do not match.

SELECT
  COUNT(*) FILTER (WHERE ABS(service_charge_amount
        - ROUND(monthly_ctc_amount * service_charge_pct / 100)) > 1) AS service_charge_mismatch,
  COUNT(*) FILTER (WHERE ABS(gst_on_service_charge
        - ROUND(service_charge_amount * 0.18)) > 1)                  AS gst_mismatch,
  COUNT(*) FILTER (WHERE ABS(total_amount_paid
        - (monthly_ctc_amount + service_charge_amount
           + gst_on_service_charge)) > 1)                            AS total_mismatch
FROM fact_vendor_monthly;

--Q45. Cross-check remarks vs documents: sites with remark 'PF challan awaited' but PF Challan marked Received.

SELECT dd.month_name, f.site_id, v.vendor_name,
       f.open_points_remarks, st.status_name AS pf_challan_status
FROM fact_vendor_monthly f
JOIN fact_compliance_doc d  ON d.site_id = f.site_id
                           AND d.date_key = f.date_key
                           AND d.doc_type_id = 1
JOIN dim_status      st ON st.status_id = d.status_id
JOIN dim_vendor_site s  ON s.site_id    = f.site_id
JOIN dim_vendor      v  ON v.vendor_id  = s.vendor_id
JOIN dim_date       dd  ON dd.date_key  = f.date_key
WHERE f.open_points_remarks LIKE 'PF challan awaited%'
  AND st.is_received = 1
ORDER BY f.date_key, f.site_id
LIMIT 10;

SELECT COUNT(*) AS inconsistent_rows
FROM fact_vendor_monthly f
JOIN fact_compliance_doc d ON d.site_id = f.site_id
                          AND d.date_key = f.date_key
                          AND d.doc_type_id = 1
JOIN dim_status st ON st.status_id = d.status_id
WHERE f.open_points_remarks LIKE 'PF challan awaited%'
  AND st.is_received = 1;

--Q46. Create a view vw_site_monthly_compliance with site, vendor, location, month, received count, not-received count and compliance %.

CREATE OR REPLACE VIEW vw_site_monthly_compliance AS
SELECT d.site_id, v.vendor_name, l.location_name,
       dd.date_key, dd.month_name, dd.month_sort,
       COUNT(*) FILTER (WHERE st.is_received = 1)                 AS docs_received,
       COUNT(*) FILTER (WHERE st.is_received = 0
                          AND st.counts_in_compliance_base = 1)   AS docs_not_received,
       ROUND(100.0 * COUNT(*) FILTER (WHERE st.is_received = 1)
             / NULLIF(COUNT(*) FILTER (WHERE st.counts_in_compliance_base = 1), 0), 1) AS compliance_pct
FROM fact_compliance_doc d
JOIN dim_status      st ON st.status_id   = d.status_id
JOIN dim_vendor_site s  ON s.site_id      = d.site_id
JOIN dim_vendor      v  ON v.vendor_id    = s.vendor_id
JOIN dim_location    l  ON l.location_id  = s.location_id
JOIN dim_date        dd ON dd.date_key    = d.date_key
GROUP BY d.site_id, v.vendor_name, l.location_name,
         dd.date_key, dd.month_name, dd.month_sort;
SELECT site_id, vendor_name, location_name, month_name, docs_received, docs_not_received, compliance_pct
FROM vw_site_monthly_compliance
WHERE compliance_pct IS NOT NULL
ORDER BY compliance_pct, month_sort
LIMIT 5;

--Q47. Create a view vw_vendor_scorecard: average compliance %, reminders, averageturnaround days, total payout and a risk label (High / Medium / Low).

CREATE OR REPLACE VIEW vw_vendor_scorecard AS
WITH comp AS (
    SELECT s.vendor_id,
           ROUND(100.0 * COUNT(*) FILTER (WHERE st.is_received = 1) / COUNT(*), 1) AS compliance_pct,
           ROUND(AVG(d.doc_date - dd.month_start), 1)                              AS avg_turnaround_days
    FROM fact_compliance_doc d
    JOIN dim_status      st ON st.status_id = d.status_id
    JOIN dim_vendor_site s  ON s.site_id    = d.site_id
    JOIN dim_date        dd ON dd.date_key  = d.date_key
    WHERE st.counts_in_compliance_base = 1
    GROUP BY s.vendor_id
),
ops AS (
    SELECT s.vendor_id,
           COUNT(*) AS site_months,
           COUNT(*) FILTER (WHERE f.mail_status IN ('Reminder-1', 'Reminder-2')) AS reminders,
           SUM(f.total_amount_paid) AS total_paid
    FROM fact_vendor_monthly f
    JOIN dim_vendor_site s ON s.site_id = f.site_id
    GROUP BY s.vendor_id
)
SELECT v.vendor_id, v.vendor_name, v.vendor_type,
       c.compliance_pct, c.avg_turnaround_days,
       o.reminders,
       ROUND(100.0 * o.reminders / o.site_months, 1) AS reminder_rate_pct,
       o.total_paid,
       CASE WHEN c.compliance_pct < 75 THEN 'High'
            WHEN c.compliance_pct < 90 THEN 'Medium'
            ELSE 'Low' END AS risk_label
FROM dim_vendor v
JOIN comp c ON c.vendor_id = v.vendor_id
JOIN ops  o ON o.vendor_id = v.vendor_id;
SELECT risk_label, COUNT(*) AS vendors, SUM(total_paid) AS total_paid
FROM vw_vendor_scorecard
GROUP BY risk_label
ORDER BY CASE risk_label WHEN 'High' THEN 1 WHEN 'Medium' THEN 2 ELSE 3 END;


--Q48. Build a vendor risk score (50% compliance gap, 30% reminder rate, 20% turnaround) and split vendors into quartiles.

WITH x AS (
    SELECT vendor_name, compliance_pct, reminder_rate_pct, avg_turnaround_days,
           100 - compliance_pct AS compliance_gap
    FROM vw_vendor_scorecard
),
n AS (
    SELECT *,
      (compliance_gap - MIN(compliance_gap) OVER ())
        / NULLIF(MAX(compliance_gap) OVER () - MIN(compliance_gap) OVER (), 0)         AS gap_n,
      (reminder_rate_pct - MIN(reminder_rate_pct) OVER ())
        / NULLIF(MAX(reminder_rate_pct) OVER () - MIN(reminder_rate_pct) OVER (), 0)   AS rem_n,
      (avg_turnaround_days - MIN(avg_turnaround_days) OVER ())
        / NULLIF(MAX(avg_turnaround_days) OVER () - MIN(avg_turnaround_days) OVER (), 0) AS ta_n
    FROM x
),
scored AS (
    SELECT *, ROUND(100 * (0.5 * gap_n + 0.3 * rem_n + 0.2 * ta_n), 1) AS risk_score
    FROM n
)
SELECT vendor_name, compliance_pct, reminder_rate_pct, avg_turnaround_days, risk_score,
       NTILE(4) OVER (ORDER BY risk_score DESC) AS risk_quartile   -- 1 = highest risk
FROM scored
ORDER BY risk_score DESC;

--Q49. Write a stored function fn_vendor_compliance(vendor_id, from_date_key, to_date_key) that returns compliance %.

CREATE OR REPLACE FUNCTION fn_vendor_compliance(
    p_vendor_id VARCHAR, p_from INT, p_to INT)
RETURNS NUMERIC
LANGUAGE plpgsql
AS $$
DECLARE
    v_pct NUMERIC;
BEGIN
    SELECT ROUND(100.0 * COUNT(*) FILTER (WHERE st.is_received = 1)
                 / NULLIF(COUNT(*), 0), 2)
    INTO v_pct
    FROM fact_compliance_doc d
    JOIN dim_status      st ON st.status_id = d.status_id
    JOIN dim_vendor_site s  ON s.site_id    = d.site_id
    WHERE s.vendor_id = p_vendor_id
      AND d.date_key BETWEEN p_from AND p_to
      AND st.counts_in_compliance_base = 1;
    RETURN v_pct;
END;
$$;
SELECT fn_vendor_compliance('V001', 20250901, 20260801) AS v001_full_year,
       fn_vendor_compliance('V001', 20260301, 20260801) AS v001_mar_aug;

--Q50. Write an audit trigger that logs old and new total_amount_paid into an audit table whenever fact_vendor_monthly is updated.

CREATE TABLE IF NOT EXISTS audit_vendor_monthly (
    audit_id    SERIAL PRIMARY KEY,
    fact_id     INT,
    old_total   BIGINT,
    new_total   BIGINT,
    changed_by  TEXT      DEFAULT CURRENT_USER,
    changed_at  TIMESTAMP DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION trg_log_payout_change()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF NEW.total_amount_paid IS DISTINCT FROM OLD.total_amount_paid THEN
        INSERT INTO audit_vendor_monthly (fact_id, old_total, new_total)
        VALUES (OLD.fact_id, OLD.total_amount_paid, NEW.total_amount_paid);
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_vendor_monthly_audit
AFTER UPDATE ON fact_vendor_monthly
FOR EACH ROW
EXECUTE FUNCTION trg_log_payout_change();

BEGIN;
UPDATE fact_vendor_monthly SET total_amount_paid = total_amount_paid + 1000 WHERE fact_id = 1;
SELECT fact_id, old_total, new_total FROM audit_vendor_monthly;
ROLLBACK;


SELECT * FROM dim_date
SELECT * FROM dim_doc_type
SELECT * FROM dim_location
SELECT * FROM dim_status
SELECT * FROM dim_vendor
SELECT * FROM dim_vendor_site
SELECT * FROM fact_compliance_doc
SELECT * FROM fact_vendor_monthly



