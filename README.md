
# SQL Funnel Analysis — Query Cookbook

## What This Is

A structured collection of SQL query patterns used in funnel analysis,
written in MySQL with a realistic sample e-commerce dataset.

This is a personal practice repository documenting  popular
funnel analysis query pattern that is used in real world product
analytics teams and tested in data analyst interviews.

The goal is simple — understand how funnel analysis is done in SQL,
document every pattern clearly, and build a reference that can be
used during interviews and on the job.

---

## Why Funnel Analysis

Funnel analysis is one of the most frequently asked topics in
data analyst interviews at product companies like Swiggy, Flipkart,
Razorpay, PhonePe, Meesho, and FAANG.

It is also one of the most used techniques in real product analytics
work — understanding where users drop off, why conversion is low,
and what to fix first.

---

## Dataset

A sample e-commerce event dataset built to simulate realistic
user behavior including:

→ 4 funnel steps — page_view, product_view, add_to_cart, purchase
→ 2 devices — mobile and desktop
→ 6 cities — Hyderabad, Mumbai, Delhi, Bangalore, Chennai, Pune
→ Mix of full funnel completers and drop off users 

### Dataset Schema

```sql
events (
    event_id    INT AUTO_INCREMENT PRIMARY KEY,
    user_id     INT,
    event_name  VARCHAR(100),
    event_time  DATETIME,
    device      VARCHAR(50),
    city        VARCHAR(50)
)
```

### Funnel Steps

page_view → product_view → add_to_cart → purchase

### User Journey Summary

8 users → completed full funnel — converted
5 users → dropped after add_to_cart
4 users → dropped after product_view
3 users → dropped after page_view
Overall conversion → 40%


---

## Query Patterns Covered

| # | Pattern | Business Question | Key SQL Concepts |
|---|---------|-------------------|-----------------|
| 1 | Basic Funnel Count | How many users touched each step | COUNT DISTINCT CASE WHEN |
| 2 | Ordered Sequential Funnel | How many users completed each step in order | ROW_NUMBER, Pivot, t2>t1 |
| 3 | Conversion Rate and Drop off | What is conversion and drop off at each step | LAG, FIRST_VALUE, NULLIF, COALESCE |
| 4 | Time Bound Funnel | How many users converted within 60 minutes | TIMESTAMPDIFF filter |
| 5 | Time to Convert | Which transition has the most friction | LAG on raw event stream |
| 6 | Next Event Analysis | What do users do after each step | LEAD on raw event stream |
| 7 | Segmented Funnel by Device | Does mobile convert differently from desktop | GROUP BY device |
| 7B | Segmented Funnel by City | Which cities convert best and worst | GROUP BY city |
| 8 | User Level Funnel Tracking | Where did each individual user drop off | MAX CASE WHEN, CASE status |
| 9 | Cohort Funnel | Do different entry date cohorts convert differently | DATE cohort, JOIN |
| 10 | WoW Funnel Degradation | Is conversion improving or degrading over time | LAG on weekly aggregation |
| 11 | User Level Time to Convert | How long did each converter take | TIMESTAMPDIFF per user |

---

## Core SQL Concepts Practiced

### Window Functions

ROW_NUMBER() → deduplication, first event per user per event type
LAG() → previous event analysis, drop off, WoW comparison
LEAD() → next event analysis, forward funnel view
FIRST_VALUE() → overall conversion denominator always from step 1


### Aggregate Functions

COUNT DISTINCT CASE WHEN → unique user counts per step
MAX CASE WHEN → pivot pattern — vertical to horizontal
AVG, MIN, MAX → time gap analysis across transitions


### Date and Time Functions

TIMESTAMPDIFF → time between funnel steps in minutes
DATE() → cohort date extraction
WEEK() → weekly aggregation for WoW analysis


### Logic and Safety

NULLIF → division by zero protection
COALESCE → convert NULL to 0 at first step
CASE WHEN → conditional flags and funnel status labels
CONCAT → readable transition labels like page_view -> product_view


### Query Structure

CTEs → modular query building, one job per CTE
UNION ALL → stacking step counts into rows for LAG to work


---

## Master Template

Every query in this cookbook follows the same 4 step template:

```sql
-- Step 1: Deduplicate
-- Handle duplicate events — keep only first occurrence per user per event
WITH deduplicated AS (
    SELECT *, ROW_NUMBER() OVER (
        PARTITION BY user_id, event_name
        ORDER BY event_time, event_id
    ) AS rn
    FROM events
),

-- Step 2: First Touch
-- Filter to keep only rn = 1
first_touch AS (
    SELECT * FROM deduplicated WHERE rn = 1
),

-- Step 3: Pivot
-- Convert vertical event rows into one horizontal row per user
pivoted AS (
    SELECT
        user_id,
        MAX(CASE WHEN event_name = 'page_view'    THEN event_time END) AS t1,
        MAX(CASE WHEN event_name = 'product_view' THEN event_time END) AS t2,
        MAX(CASE WHEN event_name = 'add_to_cart'  THEN event_time END) AS t3,
        MAX(CASE WHEN event_name = 'purchase'     THEN event_time END) AS t4
    FROM first_touch
    GROUP BY user_id
)

-- Step 4: Apply Condition
-- Count users at each step enforcing strict chronological order
SELECT
    COUNT(CASE WHEN t1 IS NOT NULL THEN 1 END)                  AS step1,
    COUNT(CASE WHEN t2 > t1 THEN 1 END)                         AS step2,
    COUNT(CASE WHEN t2 > t1 AND t3 > t2 THEN 1 END)             AS step3,
    COUNT(CASE WHEN t2 > t1 AND t3 > t2 AND t4 > t3 THEN 1 END) AS step4
FROM pivoted;
```

Master this template and every other query is just one addition on top of it.

---

## Key Findings From This Dataset

Overall funnel conversion → 40% — 8 out of 20 users purchased
Biggest drop off step → Purchase — 33.33% drop off rate
Best performing device → Desktop — 50% vs Mobile — 30%
Best performing cities → Hyderabad and Bangalore — 50% each
Worst performing cities → Delhi, Mumbai, Chennai, Pune — 33% each
Average time to convert → 38.75 minutes
Biggest friction point → Add to cart to purchase — 18 mins average
Users to retarget urgently → Users 2, 6, 9, 14, 19 — dropped after cart
Best performing cohort → Jan 2nd and Jan 3rd — 42.86% conversion


---

## What I Learned
Always deduplicate before building a funnel
Real event data has duplicate events
ROW_NUMBER with PARTITION BY user_id, event_name ORDER BY event_time
is the industry standard way to handle this
COUNT DISTINCT CASE WHEN is better than UNION ALL
One table scan vs four table scans
More efficient and cleaner to read
The pivot pattern is the foundation of ordered funnel analysis
MAX CASE WHEN + GROUP BY converts a vertical event stream
into one horizontal row per user with timestamps as columns
This enables t2 > t1 ordering logic
NULL comparisons fail automatically in SQL
t2 > t1 already handles cases where t2 is NULL
No need to write t2 IS NOT NULL explicitly
LAG has two distinct use cases in funnel analysis
On aggregated step counts — to calculate drop off between steps
On raw event stream — to calculate time between consecutive events
These are different queries serving different purposes
LEAD tells the forward story LAG cannot
After each step what did users do next
Are users going forward or going backward in the funnel
The master template is everything
deduplicate → first_touch → pivot → apply condition
Every funnel query is a variation of this four step pattern
Business interpretation matters as much as the query
A query without an insight is just a number
An insight without a recommendation is just an observation
The full chain is — query → result → insight → recommendation

---

## How to Run

Step 1 → Install MySQL on your machine
Step 2 → Open MySQL Workbench or any SQL client
Step 3 → Run data/create_and_insert.sql to create the database and load data
Step 4 → Run any query file from the queries folder
Step 5 → Compare your output with the expected results in each query file


---

## Tools Used

Database → MySQL 8.0
Client → MySQL Workbench
Language → SQL
Concepts → Window Functions, CTEs, Conditional Aggregation


---

## Repository Structure

```
funnel-analysis-sql/
│
├── README.md
├── data/
│   └── create_and_insert.sql
├── queries/
│   ├── 01_basic_funnel_count.sql
│   ├── 02_ordered_funnel.sql
│   ├── 03_conversion_rate_dropoff.sql
│   ├── 04_time_bound_funnel.sql
│   ├── 05_time_to_convert_lag.sql
│   ├── 06_next_event_lead.sql
│   ├── 07_segmented_funnel_device.sql
│   ├── 07b_segmented_funnel_city.sql
│   ├── 08_user_level_tracking.sql
│   ├── 09_cohort_funnel.sql
│   ├── 10_wow_degradation.sql
│   └── 11_user_level_time_to_convert.sql
└── results/
    └── screenshots of query outputs
```
