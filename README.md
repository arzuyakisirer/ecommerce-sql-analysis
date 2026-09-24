# ecommerce-sql-analysis
# E-Commerce Ads & Landing Page Performance Analysis

##  Project Objective
The objective of this project is to integrate raw digital marketing spend data from multiple channels (Google and Facebook Ads) with GA4 user interaction data. By doing so, this analysis aims to evaluate advertising budget efficiency, track key performance metrics, and understand user conversion behavior across the web platform.

##  Technologies & Dataset
- **Tools:** Google BigQuery, DBeaver, PostgreSQL / SQL
- **Dataset:** Daily Facebook & Google Ads spend reports and GA4 event-level data models.

---

##  SQL Queries & Business Insights

###   Multi-Channel Ad Spend Consolidation & Statistical Analysis
**Business Question:** What is the daily distribution of marketing budgets across Facebook and Google? On which specific dates did anomalous or peak (maximum/minimum) budget expenditures occur?

**Analytical Approach:** Utilized a `UNION ALL` structure to consolidate disparate data streams from both ad platforms into a unified staging model. Applied aggregations (`AVG`, `MAX`, `MIN`) grouped by date and media source to calculate granular budget metrics and cast the metrics to `numeric` for high precision.

```sql
WITH combined_ads AS (
    -- Fetching Facebook Ads data
    SELECT 
        ad_date,
        'Facebook' AS media_source,
        spend
    FROM facebook_ads_basic_daily
    
    UNION ALL
    
    -- Fetching Google Ads data
    SELECT 
        ad_date,
        'Google' AS media_source,
        spend
    FROM google_ads_basic_daily
)
SELECT 
    ad_date,
    media_source,
    AVG(spend)::numeric AS avg_spend,
    MAX(spend)::numeric AS max_spend,
    MIN(spend)::numeric AS min_spend
FROM combined_ads
GROUP BY ad_date, media_source
ORDER BY ad_date, media_source;
```

**Key Findings:** 
* Google and Facebook ad expenditures were successfully normalized into a single time-series framework, streamlining cross-channel budget auditing.
* Identified peak spending periods (`MAX spend`), providing business stakeholders visibility into high-impact promotional campaign dates or underlying spending anomalies.


###  Top 5 Highest Peak ROMI (Return on Marketing Investment) Analysis
**Business Question:** On which specific dates did our marketing spend yield the highest financial returns? What were the top 5 days where marketing efficiency peaked?

**Analytical Approach:** Combined daily spend and conversion value data streams using a unified common table expression (`WITH daily_metrics`). Calculated the aggregated ROMI metric by dividing total conversion value by total spend. Handled potential data anomalies (such as zero-spend days causing division errors) cleanly by implementing `COALESCE` and `NULLIF` functions, and filtered out non-spending days using a `HAVING` clause.

```sql
WITH daily_metrics AS (
    SELECT ad_date, spend, value FROM facebook_ads_basic_daily
    UNION ALL
    SELECT ad_date, spend, value FROM google_ads_basic_daily
)
SELECT 
    ad_date,
    (1.00 * COALESCE(SUM(value), 0) / NULLIF(SUM(spend), 0))::numeric AS romi
FROM daily_metrics
GROUP BY ad_date
HAVING SUM(spend) > 0
ORDER BY romi DESC, ad_date DESC
LIMIT 5;
```

**Key Findings:** 
* Extracted the top 5 highest-performing dates based on campaign profitability, allowing teams to cross-reference these peaks with seasonal discounts or product launches.
* The query structure robustly prevents mathematical crashes, showing production-ready SQL development standards.

###  Peak Performing Weekly Campaign Identification
**Business Question:** Which specific marketing campaign yielded the single highest aggregated revenue value within a single calendar week? 

**Analytical Approach:** Standardized and aggregated daily revenue metrics into regular weekly periods using the `DATE_TRUNC` function. For Facebook Ads data, a relational `JOIN` was implemented with the `facebook_campaign` dimension table to map granular campaign identifiers to human-readable campaign names. The combined Google and Facebook weekly dataset was structured within a CTE, aggregated, and filtered to isolate the single highest-performing campaign execution instance.

```sql
WITH weekly_campaign_metrics AS (
    -- Facebook Campaigns (Fetching names from relational table)
    SELECT 
        DATE_TRUNC('week', f.ad_date)::date AS week_start,
        c.campaign_name,
        SUM(f.value) AS weekly_value
    FROM facebook_ads_basic_daily f
    JOIN facebook_campaign c ON f.campaign_id = c.campaign_id
    GROUP BY 1, 2

    UNION ALL
    -- Google Campaigns
    SELECT 
        DATE_TRUNC('week', g.ad_date)::date AS week_start,
        g.campaign_name,
        SUM(g.value) AS weekly_value
    FROM google_ads_basic_daily g
    GROUP BY 1, 2
)
SELECT 
    week_start,
    campaign_name,
    SUM(weekly_value)::numeric AS weekly_value
FROM weekly_campaign_metrics
GROUP BY week_start, campaign_name
ORDER BY weekly_value DESC
LIMIT 1;
```

**Key Findings:** 
* Demonstrates advanced ability to handle relational schema structures and manage data granularity transformations (converting daily logs to normalized weekly business cycles).
* Effectively isolated the highest-yielding revenue spike event, allowing strategic business teams to replicate identical campaign settings for future marketing operations.


###   MoM (Month-over-Month) Campaign Reach Growth Analysis
**Business Question:** Which marketing campaign achieved the highest absolute growth in unique user reach compared to its previous month?

**Analytical Approach:** Standardized daily advertising logs into monthly periods using the `TO_CHAR` function. Constructed a unified subquery to combine Facebook and Google reach data, applying a relational `JOIN` for the Facebook schema. Implemented an advanced Window Function (`LAG() OVER (PARTITION BY... ORDER BY...)`) inside a second CTE (`growth_calculation`) to calculate the exact absolute month-over-month reach growth for each individual campaign. Finally, filtered out null values from the initial baseline months and sorted the results to extract the single highest growth spike.

```sql
WITH monthly_campaign_reach AS (
    SELECT 
        TO_CHAR(ad_date, 'YYYY-MM') AS ad_month,
        campaign_name,
        SUM(reach) AS monthly_reach
    FROM (
        SELECT ad_date, c.campaign_name, reach 
        FROM facebook_ads_basic_daily f
        JOIN facebook_campaign c ON f.campaign_id = c.campaign_id
        UNION ALL
        SELECT ad_date, campaign_name, reach FROM google_ads_basic_daily
    ) combined
    GROUP BY 1, 2
),
growth_calculation AS (
    SELECT 
        ad_month,
        campaign_name,
        monthly_reach,
        monthly_reach - LAG(monthly_reach) OVER(PARTITION BY campaign_name ORDER BY ad_month) AS monthly_growth
    FROM monthly_campaign_reach
)
SELECT 
    ad_month,
    campaign_name,
    monthly_reach::numeric,
    monthly_growth::numeric
FROM growth_calculation
WHERE monthly_growth IS NOT NULL
ORDER BY monthly_growth DESC
LIMIT 1;
```

**Key Findings:** 
* Demonstrates advanced proficiency in complex SQL architectures, multi-layered CTEs, and cohort-style tracking via Window Functions (`LAG`).
* Pinpointed the exact campaign and month that generated the most significant velocity in audience acquisition, helping marketing teams understand which creative assets scaled most effectively.



### Continuous Ad-Set Delivery Sequence Analysis (Gaps & Islands Problem)
**Business Question:** Which specific ad set achieved the longest consecutive run (streak) of active daily delivery with non-zero impressions, and what were its exact start and end dates?

**Analytical Approach:** Implemented a highly sophisticated Advanced SQL design pattern known as the **"Gaps and Islands"** methodology. First, isolated distinct daily active intervals where `impressions > 0`. Second, leveraged chronological ranking using `ROW_NUMBER() OVER (PARTITION BY adset_name ORDER BY ad_date)` combined with date subtraction mechanics to project continuous records into matching transactional `date_group` clusters. Finally, aggregated these clusters via `MIN`, `MAX`, and date-differencing logic to isolate the single longest uninterrupted active marketing lifecycle instance.

```sql
WITH active_adsets AS (
    SELECT DISTINCT ad_date, adset_name 
    FROM (
        SELECT f.ad_date, a.adset_name, f.impressions 
        FROM facebook_ads_basic_daily f
        JOIN facebook_adset a ON f.adset_id = a.adset_id
        UNION ALL
        SELECT ad_date, adset_name, impressions FROM google_ads_basic_daily
    ) combined
    WHERE impressions > 0
),
ordered_groups AS (
    SELECT 
        ad_date,
        adset_name,
        ad_date - ROW_NUMBER() OVER(PARTITION BY adset_name ORDER BY ad_date)::int AS date_group
    FROM active_adsets
),
streak_intervals AS (
    SELECT 
        adset_name,
        MIN(ad_date) AS streak_start,
        MAX(ad_date) AS streak_end,
        (MAX(ad_date) - MIN(ad_date) + 1) AS streak_length
    FROM ordered_groups
    GROUP BY adset_name, date_group
)
SELECT 
    adset_name,
    streak_start,
    streak_end,
    streak_length
FROM streak_intervals
ORDER BY streak_length DESC
LIMIT 1;
```
**Key Findings:**
* Solved a highly complex structural data challenge (Gaps & Islands), which demonstrates exceptional logical query architecture and advanced production SQL competencies.
* Uncovered key stability patterns across marketing operational streams, pinpointing which creative ad sets maintain consistent audience engagement without performance degradation or budget suspension.



---

## GA4 BigQuery Data Modeling for Business Intelligence (BI)

### Event-Level E-Commerce Data Transformation & Pipeline Preparation
**Business Question:** How can we restructure and clean raw, unstructured Google Analytics 4 (GA4) nested event logs from 2021 into a high-performance, session-level data model optimized for BI dashboard visualization?

**Analytical Approach:** Developed a robust BigQuery SQL script to filter and transform raw e-commerce events. Converted the raw, nested `event_timestamp` into a human-readable, standardized date-time format using `TIMESTAMP_MICROS`. Extracted the nested session identifiers cleanly via `UNNEST(event_params)`. Implemented strict filters to isolate the core user conversion path from the public GA4 e-commerce sample dataset.

```sql
SELECT
  TIMESTAMP_MICROS(event_timestamp) AS event_timestamp,
  user_pseudo_id,
  event_name,
  (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS session_id,
  geo.country AS country,
  device.category AS device_category,
  traffic_source.source AS source,
  traffic_source.medium AS medium,
  traffic_source.name AS campaign
FROM
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_20210131`
WHERE
  event_name IN (
    'session_start', 
    'view_item', 
    'add_to_cart', 
    'begin_checkout', 
    'add_shipping_info', 
    'add_payment_info', 
    'purchase'
  )
LIMIT 1000;
```

**Key Findings & Schema Modeled:**
* **Schema Flattening:** Successfully transformed unstructured, nested GA4 JSON columns into a flat, relational table, significantly reducing query computational costs for BI tool connectors (like Tableau/Power BI).
* **Funnel Readiness:** Mapped the exact end-to-end user behavioral pipeline (`session_start` ➡️ `view_item` ➡️ `add_to_cart` ➡️ `begin_checkout` ➡️ `add_shipping_info` ➡️ `add_payment_info` ➡️ `purchase`), laying the groundwork for precise conversion funnel drop-off analysis.




---

###  Traffic Channel Conversion Funnel Analysis
**Business Question:** What are the baseline conversion rates from initial user sessions to high-intent actions (`add_to_cart`, `begin_checkout`, `purchase`) across different marketing traffic acquisition channels?

**Analytical Approach:** Built an advanced aggregated funnel model inside BigQuery. To handle data quality edge cases where multiple users might share identical session IDs, implemented a unique composite key combining `user_pseudo_id` and `session_id` via `CONCAT`. Normalized null traffic fields using `COALESCE` to match industry standards `(direct / none / not set)`. Utilized conditional aggregation combined with `SAFE_DIVIDE` to calculate accurate performance metrics without causing mathematical execution crashes.

```sql
WITH prepared_data AS (
  SELECT
    DATE(TIMESTAMP_MICROS(event_timestamp)) AS event_date,
    event_name,
    user_pseudo_id,
    (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS session_id,
    traffic_source.source AS source,
    traffic_source.medium AS medium,
    traffic_source.name AS campaign
  FROM
    `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_20210131`
  WHERE
    event_name IN ('session_start', 'add_to_cart', 'begin_checkout', 'purchase')
)
SELECT
  event_date,
  COALESCE(source, '(direct)') AS source,
  COALESCE(medium, '(none)') AS medium,
  COALESCE(campaign, '(not set)') AS campaign,
  COUNT(DISTINCT CONCAT(user_pseudo_id, CAST(session_id AS STRING))) AS user_sessions_count,
  ROUND(SAFE_DIVIDE(
    COUNT(DISTINCT CASE WHEN event_name = 'add_to_cart' THEN CONCAT(user_pseudo_id, CAST(session_id AS STRING)) END),
    COUNT(DISTINCT CONCAT(user_pseudo_id, CAST(session_id AS STRING)))
  ) * 100, 2) AS visit_to_cart,
  ROUND(SAFE_DIVIDE(
    COUNT(DISTINCT CASE WHEN event_name = 'begin_checkout' THEN CONCAT(user_pseudo_id, CAST(session_id AS STRING)) END),
    COUNT(DISTINCT CONCAT(user_pseudo_id, CAST(session_id AS STRING)))
  ) * 100, 2) AS visit_to_checkout,
  ROUND(SAFE_DIVIDE(
    COUNT(DISTINCT CASE WHEN event_name = 'purchase' THEN CONCAT(user_pseudo_id, CAST(session_id AS STRING)) END),
    COUNT(DISTINCT CONCAT(user_pseudo_id, CAST(session_id AS STRING)))
  ) * 100, 2) AS visit_to_purchase
FROM
  prepared_data
WHERE
  session_id IS NOT NULL
GROUP BY
  event_date,
  source,
  medium,
  campaign
ORDER BY
  user_sessions_count DESC
LIMIT 500;
```

**Key Findings & Business Impact:**
* **Funnel Optimization:** Enabled granular calculation of marketing funnel velocity (e.g., Session-to-Cart Conversion, Session-to-Purchase Conversion), helping performance marketers pinpoint exactly where user drop-off occurs.
* **ROI Attribution:** Allowed direct cross-channel evaluation to determine which source/medium combinations bring high-volume traffic versus high-converting traffic.


---

###  Landing Page Conversion Performance Analysis
**Business Question:** Which initial landing pages (entry URLs) generate the highest volume of traffic, and which ones achieve the best final purchase conversion rates?

**Analytical Approach:** Developed an advanced attribution model using multiple Common Table Expressions (CTEs) and relational `LEFT JOIN` logic. Extracted clean URL paths by applying regular expressions (`REGEXP_EXTRACT`) on nested `page_location` parameters during `session_start` events. Queried wildcard historical tables (`events_2020*`) to process the 2020 dataset efficiently. Integrated session data with purchase data via a multi-key join (`user_pseudo_id` and `session_id`) and calculated standard session-to-purchase conversion rates (CR) using `SAFE_DIVIDE`.

```sql
WITH session_landing_pages AS (
  SELECT
    user_pseudo_id,
    (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS session_id,
    REGEXP_EXTRACT(
      (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'page_location'),
      r'https?://[^/]+(/[^?#]*)'
    ) AS landing_page_path
  FROM
    `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_2020*`
  WHERE
    event_name = 'session_start'
),
session_purchases AS (
  SELECT
    user_pseudo_id,
    (SELECT value.int_value FROM UNNEST(event_params) WHERE key = 'ga_session_id') AS session_id,
    COUNT(*) AS purchase_count
  FROM
    `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_2020*`
  WHERE
    event_name = 'purchase'
  GROUP BY
    user_pseudo_id,
    session_id
)
SELECT
  COALESCE(lp.landing_page_path, '/') AS landing_page_path,
  COUNT(DISTINCT CONCAT(lp.user_pseudo_id, CAST(lp.session_id AS STRING))) AS user_sessions_count,
  COALESCE(SUM(p.purchase_count), 0) AS total_purchases,
  ROUND(
    SAFE_DIVIDE(
      COUNT(DISTINCT CASE WHEN p.purchase_count > 0 THEN CONCAT(lp.user_pseudo_id, CAST(lp.session_id AS STRING)) END),
      COUNT(DISTINCT CONCAT(lp.user_pseudo_id, CAST(lp.session_id AS STRING)))
    ) * 100, 2
  ) AS session_to_purchase_conversion_rate
FROM
  session_landing_pages lp
LEFT JOIN
  session_purchases p 
  ON lp.user_pseudo_id = p.user_pseudo_id 
  AND lp.session_id = p.session_id
WHERE
  lp.session_id IS NOT NULL
GROUP BY
  landing_page_path
ORDER BY
  user_sessions_count DESC
LIMIT 500;
```

**Key Findings & Business Impact:**
* **UX & Content Optimization:** Identified the exact high-traffic entry points that underperform in conversion, signaling a need for better user experience (UX) or copy optimization on those specific landing pages.
* **Data Cleansing:** Implemented precise string parsing (`REGEXP_EXTRACT`) to strip parameters and queries from URLs, ensuring that page performance data is aggregated accurately without fragmentation.

