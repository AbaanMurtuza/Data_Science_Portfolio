/*
=======================================================================================
E-COMMERCE CONVERSION & REVENUE ANALYSIS: UNIFIED MASTER REPORT
=======================================================================================
Project Goal: To analyze a 30-day window of user event data to identify checkout 
friction, evaluate marketing channel performance, and calculate overall revenue health.

In this script, we utilize chained Common Table Expressions (CTEs) to calculate 
global averages, user journey times, and channel-specific metrics independently. 
We then JOIN these CTEs into a single master output to compare individual channel 
performance against global baselines—without relying on any external tables.
=======================================================================================
*/

CREATE DATABASE IF NOT EXISTS project02;
USE project02;

/*
---------------------------------------------------------------------------------------
CTE 1: GLOBAL METRICS
---------------------------------------------------------------------------------------
First, we calculate the grand totals for the entire business over the 30-day window.
We pivot the event types into columns using CASE WHEN to capture total visitors, 
cart additions, purchasers, and overall revenue. We will cross-join this later to 
compare individual marketing channels against these global baselines.
---------------------------------------------------------------------------------------
*/
WITH global_metrics AS (
    SELECT
        COUNT(DISTINCT CASE WHEN event_type = 'page_view' THEN user_id END) AS global_views,
        COUNT(DISTINCT CASE WHEN event_type = 'add_to_cart' THEN user_id END) AS global_carts,
        COUNT(DISTINCT CASE WHEN event_type = 'purchase' THEN user_id END) AS global_purchasers,
        SUM(CASE WHEN event_type = 'purchase' THEN amount END) AS global_revenue,
        SUM(CASE WHEN event_type = 'purchase' THEN 1 END) AS global_orders
    FROM user_events
    WHERE event_date BETWEEN DATE_SUB('2026-02-09', INTERVAL 30 DAY) AND '2026-02-09'
),

/*
---------------------------------------------------------------------------------------
CTE 2: USER JOURNEY TIMESTAMPS
---------------------------------------------------------------------------------------
To measure friction, we need to know how long users take to check out. This CTE 
isolates only users who successfully purchased and extracts the exact timestamps 
for their first view, first cart addition, and purchase.
---------------------------------------------------------------------------------------
*/
user_journey_timestamps AS (
    SELECT
        user_id,
        MIN(CASE WHEN event_type = 'page_view' THEN event_date END) AS view_time,
        MIN(CASE WHEN event_type = 'add_to_cart' THEN event_date END) AS cart_time,
        MIN(CASE WHEN event_type = 'purchase' THEN event_date END) AS purchase_time
    FROM user_events
    WHERE event_date BETWEEN DATE_SUB('2026-02-09', INTERVAL 30 DAY) AND '2026-02-09'
    GROUP BY user_id
    HAVING MIN(CASE WHEN event_type = 'purchase' THEN event_date END) IS NOT NULL
),

/*
---------------------------------------------------------------------------------------
CTE 3: GLOBAL FRICTION AVERAGES
---------------------------------------------------------------------------------------
Here, we calculate the average time (in minutes) spent between milestones based on 
the timestamps extracted in the previous CTE. We aggregate this into a single row.
---------------------------------------------------------------------------------------
*/
global_friction AS (
    SELECT
        AVG(TIMESTAMPDIFF(MINUTE, view_time, cart_time)) AS avg_view_to_cart_min,
        AVG(TIMESTAMPDIFF(MINUTE, cart_time, purchase_time)) AS avg_cart_to_purchase_min
    FROM user_journey_timestamps
),

/*
---------------------------------------------------------------------------------------
CTE 4: SOURCE-LEVEL METRICS (MARKETING ATTRIBUTION)
---------------------------------------------------------------------------------------
Finally, we break down the funnel metrics by 'traffic_source' (organic, social, etc.) 
to see which acquisition channels drive the highest intent and revenue.
---------------------------------------------------------------------------------------
*/
source_metrics AS (
    SELECT
        traffic_source,
        COUNT(DISTINCT CASE WHEN event_type = 'page_view' THEN user_id END) AS source_views,
        COUNT(DISTINCT CASE WHEN event_type = 'add_to_cart' THEN user_id END) AS source_carts,
        COUNT(DISTINCT CASE WHEN event_type = 'purchase' THEN user_id END) AS source_purchasers,
        SUM(CASE WHEN event_type = 'purchase' THEN amount END) AS source_revenue
    FROM user_events
    WHERE event_date BETWEEN DATE_SUB('2026-02-09', INTERVAL 30 DAY) AND '2026-02-09'
    GROUP BY traffic_source
)

/*
---------------------------------------------------------------------------------------
FINAL OUTPUT: JOINING THE CTEs
---------------------------------------------------------------------------------------
We now JOIN everything together! Since 'global_metrics' and 'global_friction' are 
single-row summary tables, we use a CROSS JOIN to attach these global baselines to 
every row in our 'source_metrics' table. 

This allows us to perform powerful mathematical comparisons, such as calculating 
exactly what percentage of total revenue each marketing channel is responsible for.
---------------------------------------------------------------------------------------
*/
SELECT
    -- Channel Identifying Info
    sm.traffic_source,
    
    -- Channel Specific Funnel Metrics
    sm.source_views,
    sm.source_carts,
    sm.source_purchasers,
    ROUND((sm.source_carts * 100.0) / sm.source_views, 2) AS source_cart_conversion_pct,
    ROUND((sm.source_purchasers * 100.0) / sm.source_views, 2) AS source_purchase_conversion_pct,
    
    -- Revenue & Performance Comparisons (Using Joined Data)
    ROUND(sm.source_revenue, 2) AS source_revenue,
    ROUND((sm.source_revenue * 100.0) / gm.global_revenue, 2) AS pct_of_global_revenue,
    
    -- Appending the Global Friction Metrics (Constant across rows for reference)
    ROUND(gf.avg_view_to_cart_min, 2) AS global_avg_view_to_cart_min,
    ROUND(gf.avg_cart_to_purchase_min, 2) AS global_avg_cart_to_purchase_min

FROM source_metrics sm
CROSS JOIN global_metrics gm
CROSS JOIN global_friction gf
ORDER BY sm.source_revenue DESC;
