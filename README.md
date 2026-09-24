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
