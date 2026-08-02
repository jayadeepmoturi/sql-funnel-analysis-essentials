use funnel_practice_db ;

-- ====================================================================================================================================================
-- Business Question: How many unique users touched each step?
-- SQL Concepts: COUNT DISTINCT CASE WHEN
-- Approach: Conditional Aggregation
-- ====================================================================================================================================================

SELECT
    COUNT(DISTINCT CASE WHEN event_name = 'page_view'
          THEN user_id END)   AS step1_page_view,
    COUNT(DISTINCT CASE WHEN event_name = 'product_view'
          THEN user_id END)   AS step2_product_view,
    COUNT(DISTINCT CASE WHEN event_name = 'add_to_cart'
          THEN user_id END)   AS step3_add_to_cart,
    COUNT(DISTINCT CASE WHEN event_name = 'purchase'
          THEN user_id END)   AS step4_purchase
FROM events;

-- ====================================================================================================================================================
-- Business Question: How many users completed each step IN ORDER?
-- SQL Concepts: ROW_NUMBER, MAX CASE WHEN pivot, t2>t1 ordering
-- Approach: Window Function
-- ====================================================================================================================================================
WITH

deduplicated AS (
    SELECT
        user_id, event_name, event_time, device, city,
        ROW_NUMBER() OVER (
            PARTITION BY user_id, event_name
            ORDER BY event_time, event_id
        ) AS rn
    FROM events
),

first_touch AS (
    SELECT user_id, event_name, event_time, device, city
    FROM deduplicated
    WHERE rn = 1
),

pivoted AS (
    SELECT
        user_id, device, city,
        MAX(CASE WHEN event_name = 'page_view'
            THEN event_time END) AS t1,
        MAX(CASE WHEN event_name = 'product_view'
            THEN event_time END) AS t2,
        MAX(CASE WHEN event_name = 'add_to_cart'
            THEN event_time END) AS t3,
        MAX(CASE WHEN event_name = 'purchase'
            THEN event_time END) AS t4
    FROM first_touch
    GROUP BY user_id, device, city
)

SELECT
    COUNT(CASE WHEN t1 IS NOT NULL
               THEN 1 END)                                AS step1,
    COUNT(CASE WHEN t1 IS NOT NULL AND t2 > t1
               THEN 1 END)                                AS step2,
    COUNT(CASE WHEN t2 > t1 AND t3 > t2
               THEN 1 END)                                AS step3,
    COUNT(CASE WHEN t2 > t1 AND t3 > t2 AND t4 > t3
               THEN 1 END)                                AS step4
FROM pivoted;


-- ====================================================================================================================================================
-- Business Question: What is conversion rate and drop off at each step?
-- SQL Concepts: LAG, FIRST_VALUE, NULLIF, COALESCE
-- Approach: Window Function on step counts
-- ====================================================================================================================================================
WITH

deduplicated AS (
    SELECT
        user_id, event_name, event_time,
        ROW_NUMBER() OVER (
            PARTITION BY user_id, event_name
            ORDER BY event_time, event_id
        ) AS rn
    FROM events
),

first_touch AS (
    SELECT user_id, event_name, event_time
    FROM deduplicated WHERE rn = 1
),

pivoted AS (
    SELECT
        user_id,
        MAX(CASE WHEN event_name = 'page_view'
            THEN event_time END) AS t1,
        MAX(CASE WHEN event_name = 'product_view'
            THEN event_time END) AS t2,
        MAX(CASE WHEN event_name = 'add_to_cart'
            THEN event_time END) AS t3,
        MAX(CASE WHEN event_name = 'purchase'
            THEN event_time END) AS t4
    FROM first_touch
    GROUP BY user_id
),

step_counts AS (
    SELECT 1 AS step_number, 'page_view' AS step_name,
           COUNT(CASE WHEN t1 IS NOT NULL THEN 1 END) AS users
    FROM pivoted
    UNION ALL
    SELECT 2, 'product_view',
           COUNT(CASE WHEN t1 IS NOT NULL AND t2 > t1 THEN 1 END)
    FROM pivoted
    UNION ALL
    SELECT 3, 'add_to_cart',
           COUNT(CASE WHEN t2 > t1 AND t3 > t2 THEN 1 END)
    FROM pivoted
    UNION ALL
    SELECT 4, 'purchase',
           COUNT(CASE WHEN t2 > t1 AND t3 > t2 AND t4 > t3 THEN 1 END)
    FROM pivoted
)

SELECT
    step_number,
    step_name,
    users,
    ROUND(users * 100.0 /
          NULLIF(LAG(users) OVER (ORDER BY step_number), 0), 2)
                                                        AS step_conv_pct,
    ROUND(users * 100.0 /
          NULLIF(FIRST_VALUE(users) OVER (ORDER BY step_number), 0), 2)
                                                        AS overall_conv_pct,
    COALESCE(LAG(users) OVER (ORDER BY step_number)
             - users, 0)                                AS users_dropped,
    ROUND(COALESCE(LAG(users) OVER (ORDER BY step_number)
          - users, 0) * 100.0 /
          NULLIF(LAG(users) OVER (ORDER BY step_number), 0), 2)
                                                        AS dropoff_pct
FROM step_counts
ORDER BY step_number;


-- ====================================================================================================================================================
-- Business Question: How many users completed the full funnel within 60 minutes?
-- SQL Concepts: TIMESTAMPDIFF filter
-- Approach: Window Function + Time Window Condition
-- ====================================================================================================================================================
WITH

deduplicated AS (
    SELECT
        user_id, event_name, event_time,
        ROW_NUMBER() OVER (
            PARTITION BY user_id, event_name
            ORDER BY event_time, event_id
        ) AS rn
    FROM events
),

first_touch AS (
    SELECT user_id, event_name, event_time
    FROM deduplicated WHERE rn = 1
),

pivoted AS (
    SELECT
        user_id,
        MAX(CASE WHEN event_name = 'page_view'
            THEN event_time END) AS t1,
        MAX(CASE WHEN event_name = 'product_view'
            THEN event_time END) AS t2,
        MAX(CASE WHEN event_name = 'add_to_cart'
            THEN event_time END) AS t3,
        MAX(CASE WHEN event_name = 'purchase'
            THEN event_time END) AS t4
    FROM first_touch
    GROUP BY user_id
)

SELECT
    COUNT(CASE WHEN t1 IS NOT NULL
               THEN 1 END)                              AS step1,
    COUNT(CASE WHEN t2 > t1
               THEN 1 END)                              AS step2,
    COUNT(CASE WHEN t2 > t1 AND t3 > t2
               THEN 1 END)                              AS step3,
    COUNT(CASE WHEN t2 > t1 AND t3 > t2 AND t4 > t3
               AND TIMESTAMPDIFF(MINUTE, t1, t4) <= 60
               THEN 1 END)                              AS step4_within_60mins,
    COUNT(CASE WHEN t2 > t1 AND t3 > t2 AND t4 > t3
               THEN 1 END)                              AS step4_all_time
FROM pivoted;

-- ====================================================================================================================================================
-- Business Question: How long do users spend between each step?
-- Which transition has the most friction?
-- SQL Concepts: LAG, TIMESTAMPDIFF, CONCAT, AVG
-- Approach: LAG on raw event stream
-- ====================================================================================================================================================
-- Part 1: Per user event stream with time gaps
SELECT
    user_id,
    event_name,
    event_time,
    LAG(event_name) OVER (
        PARTITION BY user_id ORDER BY event_time, event_id
    )                                                   AS prev_event,
    LAG(event_time) OVER (
        PARTITION BY user_id ORDER BY event_time, event_id
    )                                                   AS prev_event_time,
    TIMESTAMPDIFF(
        MINUTE,
        LAG(event_time) OVER (
            PARTITION BY user_id ORDER BY event_time, event_id),
        event_time
    )                                                   AS mins_since_prev_event
FROM events
ORDER BY user_id, event_time;

-- Part 2: Average time between each step transition
WITH event_stream AS (
    SELECT
        user_id,
        event_name,
        event_time,
        LAG(event_name) OVER (
            PARTITION BY user_id ORDER BY event_time, event_id
        ) AS prev_event,
        TIMESTAMPDIFF(
            MINUTE,
            LAG(event_time) OVER (
                PARTITION BY user_id ORDER BY event_time, event_id),
            event_time
        ) AS mins_since_prev
    FROM events
)
SELECT
    CONCAT(prev_event, ' -> ', event_name) AS transition,
    COUNT(*)                               AS occurrences,
    ROUND(AVG(mins_since_prev), 2)         AS avg_mins,
    MIN(mins_since_prev)                   AS min_mins,
    MAX(mins_since_prev)                   AS max_mins
FROM event_stream
WHERE prev_event IS NOT NULL
AND prev_event != event_name
GROUP BY prev_event, event_name
ORDER BY avg_mins DESC;

-- ====================================================================================================================================================
-- Business Question: After each step what do users do next?
-- Are users going forward or backward in the funnel?
-- SQL Concepts: LEAD, TIMESTAMPDIFF, CONCAT
-- Approach: LEAD on raw event stream
-- ====================================================================================================================================================
WITH next_event AS (
    SELECT
        user_id,
        event_name                                      AS current_event,
        event_time                                      AS current_event_time,
        LEAD(event_name) OVER (
            PARTITION BY user_id ORDER BY event_time, event_id
        )                                               AS next_event,
        LEAD(event_time) OVER (
            PARTITION BY user_id ORDER BY event_time, event_id
        )                                               AS next_event_time,
        TIMESTAMPDIFF(
            MINUTE,
            event_time,
            LEAD(event_time) OVER (
                PARTITION BY user_id ORDER BY event_time, event_id)
        )                                               AS mins_to_next_event
    FROM events
)
SELECT
    current_event,
    next_event,
    COUNT(*)                           AS occurrences,
    ROUND(AVG(mins_to_next_event), 2)  AS avg_mins_to_next
FROM next_event
WHERE next_event IS NOT NULL
AND current_event != next_event
GROUP BY current_event, next_event
ORDER BY current_event, occurrences DESC;

-- ====================================================================================================================================================
-- Business Question: After each step what do users do next?
-- Are users going forward or backward in the funnel?
-- SQL Concepts: LEAD, TIMESTAMPDIFF, CONCAT
-- Approach: LEAD on raw event stream
-- ====================================================================================================================================================
-- Business Question: After each step what do users do next?
-- Are users going forward or backward in the funnel?
-- SQL Concepts: LEAD, TIMESTAMPDIFF, CONCAT
-- Approach: LEAD on raw event stream

WITH next_event AS (
    SELECT
        user_id,
        event_name                                      AS current_event,
        event_time                                      AS current_event_time,
        LEAD(event_name) OVER (
            PARTITION BY user_id ORDER BY event_time, event_id
        )                                               AS next_event,
        LEAD(event_time) OVER (
            PARTITION BY user_id ORDER BY event_time, event_id
        )                                               AS next_event_time,
        TIMESTAMPDIFF(
            MINUTE,
            event_time,
            LEAD(event_time) OVER (
                PARTITION BY user_id ORDER BY event_time, event_id)
        )                                               AS mins_to_next_event
    FROM events
)
SELECT
    current_event,
    next_event,
    COUNT(*)                           AS occurrences,
    ROUND(AVG(mins_to_next_event), 2)  AS avg_mins_to_next
FROM next_event
WHERE next_event IS NOT NULL
AND current_event != next_event
GROUP BY current_event, next_event
ORDER BY current_event, occurrences DESC;

-- ====================================================================================================================================================
-- Business Question: How does funnel conversion differ between mobile and desktop?
-- SQL Concepts: GROUP BY device
-- Approach: Window Function + Segmentation
-- ====================================================================================================================================================
WITH
deduplicated AS (
    SELECT
        user_id, event_name, event_time, device, city,
        ROW_NUMBER() OVER (
            PARTITION BY user_id, event_name
            ORDER BY event_time, event_id
        ) AS rn
    FROM events
),

first_touch AS (
    SELECT user_id, event_name, event_time, device, city
    FROM deduplicated WHERE rn = 1
),

pivoted AS (
    SELECT
        user_id, device, city,
        MAX(CASE WHEN event_name = 'page_view'
            THEN event_time END) AS t1,
        MAX(CASE WHEN event_name = 'product_view'
            THEN event_time END) AS t2,
        MAX(CASE WHEN event_name = 'add_to_cart'
            THEN event_time END) AS t3,
        MAX(CASE WHEN event_name = 'purchase'
            THEN event_time END) AS t4
    FROM first_touch
    GROUP BY user_id, device, city
)

SELECT
    device,
    COUNT(CASE WHEN t1 IS NOT NULL
               THEN 1 END)                              AS step1,
    COUNT(CASE WHEN t2 > t1
               THEN 1 END)                              AS step2,
    COUNT(CASE WHEN t2 > t1 AND t3 > t2
               THEN 1 END)                              AS step3,
    COUNT(CASE WHEN t2 > t1 AND t3 > t2 AND t4 > t3
               THEN 1 END)                              AS step4,
    ROUND(
        COUNT(CASE WHEN t2 > t1 AND t3 > t2 AND t4 > t3
                   THEN 1 END) * 100.0 /
        NULLIF(COUNT(CASE WHEN t1 IS NOT NULL
                          THEN 1 END), 0)
    , 2)                                                AS overall_conv_pct
FROM pivoted
GROUP BY device
ORDER BY overall_conv_pct DESC;

-- ====================================================================================================================================================
-- Business Question: Which cities convert best and worst?
-- SQL Concepts: GROUP BY city
-- Approach: Same as device segmentation, replace device with city
-- ====================================================================================================================================================
WITH

deduplicated AS (
    SELECT
        user_id, event_name, event_time, device, city,
        ROW_NUMBER() OVER (
            PARTITION BY user_id, event_name
            ORDER BY event_time, event_id
        ) AS rn
    FROM events
),

first_touch AS (
    SELECT user_id, event_name, event_time, device, city
    FROM deduplicated WHERE rn = 1
),

pivoted AS (
    SELECT
        user_id, device, city,
        MAX(CASE WHEN event_name = 'page_view'
            THEN event_time END) AS t1,
        MAX(CASE WHEN event_name = 'product_view'
            THEN event_time END) AS t2,
        MAX(CASE WHEN event_name = 'add_to_cart'
            THEN event_time END) AS t3,
        MAX(CASE WHEN event_name = 'purchase'
            THEN event_time END) AS t4
    FROM first_touch
    GROUP BY user_id, device, city
)

SELECT
    city,
    COUNT(CASE WHEN t1 IS NOT NULL
               THEN 1 END)                              AS step1,
    COUNT(CASE WHEN t2 > t1
               THEN 1 END)                              AS step2,
    COUNT(CASE WHEN t2 > t1 AND t3 > t2
               THEN 1 END)                              AS step3,
    COUNT(CASE WHEN t2 > t1 AND t3 > t2 AND t4 > t3
               THEN 1 END)                              AS step4,
    ROUND(
        COUNT(CASE WHEN t2 > t1 AND t3 > t2 AND t4 > t3
                   THEN 1 END) * 100.0 /
        NULLIF(COUNT(CASE WHEN t1 IS NOT NULL
                          THEN 1 END), 0)
    , 2)                                                AS overall_conv_pct
FROM pivoted
GROUP BY city
ORDER BY overall_conv_pct DESC;

-- ====================================================================================================================================================
-- Business Question: Where did each individual user drop off?
-- SQL Concepts: MAX CASE WHEN, CASE status
-- Approach: Conditional Aggregation per user
-- ====================================================================================================================================================
WITH

deduplicated AS (
    SELECT
        user_id, event_name, event_time,
        ROW_NUMBER() OVER (
            PARTITION BY user_id, event_name
            ORDER BY event_time, event_id
        ) AS rn
    FROM events
),

first_touch AS (
    SELECT user_id, event_name, event_time
    FROM deduplicated WHERE rn = 1
),

pivoted AS (
    SELECT
        user_id,
        MAX(CASE WHEN event_name = 'page_view'
            THEN event_time END) AS t1,
        MAX(CASE WHEN event_name = 'product_view'
            THEN event_time END) AS t2,
        MAX(CASE WHEN event_name = 'add_to_cart'
            THEN event_time END) AS t3,
        MAX(CASE WHEN event_name = 'purchase'
            THEN event_time END) AS t4
    FROM first_touch
    GROUP BY user_id
)

SELECT
    user_id,
    CASE WHEN t1 IS NOT NULL
         THEN 1 ELSE 0 END                              AS did_step1,
    CASE WHEN t2 > t1
         THEN 1 ELSE 0 END                              AS did_step2,
    CASE WHEN t2 > t1 AND t3 > t2
         THEN 1 ELSE 0 END                              AS did_step3,
    CASE WHEN t2 > t1 AND t3 > t2 AND t4 > t3
         THEN 1 ELSE 0 END                              AS did_step4,
    CASE
        WHEN t2 > t1 AND t3 > t2 AND t4 > t3
             THEN 'Converted'
        WHEN t2 > t1 AND t3 > t2
             THEN 'Dropped — After Add to Cart'
        WHEN t2 > t1
             THEN 'Dropped — After Product View'
        WHEN t1 IS NOT NULL
             THEN 'Dropped — After Page View'
        ELSE 'Never Entered'
    END                                                 AS funnel_status
FROM pivoted
ORDER BY user_id;

-- ====================================================================================================================================================
-- Business Question: Do users who entered on different days convert differently?
-- SQL Concepts: DATE() cohort, JOIN
-- Approach: Window Function + Cohort Grouping
-- ====================================================================================================================================================
WITH

deduplicated AS (
    SELECT
        user_id, event_name, event_time,
        ROW_NUMBER() OVER (
            PARTITION BY user_id, event_name
            ORDER BY event_time, event_id
        ) AS rn
    FROM events
),

first_touch AS (
    SELECT user_id, event_name, event_time
    FROM deduplicated WHERE rn = 1
),

cohort AS (
    SELECT user_id, DATE(event_time) AS cohort_date
    FROM first_touch
    WHERE event_name = 'page_view'
),

pivoted AS (
    SELECT
        f.user_id,
        c.cohort_date,
        MAX(CASE WHEN f.event_name = 'page_view'
            THEN f.event_time END) AS t1,
        MAX(CASE WHEN f.event_name = 'product_view'
            THEN f.event_time END) AS t2,
        MAX(CASE WHEN f.event_name = 'add_to_cart'
            THEN f.event_time END) AS t3,
        MAX(CASE WHEN f.event_name = 'purchase'
            THEN f.event_time END) AS t4
    FROM first_touch f
    JOIN cohort c ON f.user_id = c.user_id
    GROUP BY f.user_id, c.cohort_date
)

SELECT
    cohort_date,
    COUNT(CASE WHEN t1 IS NOT NULL
               THEN 1 END)                              AS step1,
    COUNT(CASE WHEN t2 > t1
               THEN 1 END)                              AS step2,
    COUNT(CASE WHEN t2 > t1 AND t3 > t2
               THEN 1 END)                              AS step3,
    COUNT(CASE WHEN t2 > t1 AND t3 > t2 AND t4 > t3
               THEN 1 END)                              AS step4,
    ROUND(
        COUNT(CASE WHEN t2 > t1 AND t3 > t2 AND t4 > t3
                   THEN 1 END) * 100.0 /
        NULLIF(COUNT(CASE WHEN t1 IS NOT NULL
                          THEN 1 END), 0)
    , 2)                                                AS conv_pct
FROM pivoted
GROUP BY cohort_date
ORDER BY cohort_date;

-- ====================================================================================================================================================
-- Business Question: Is funnel conversion improving or degrading over time?
-- SQL Concepts: WEEK, LAG on aggregated result, CASE trend label
-- Approach: Conditional Aggregation + LAG
-- ====================================================================================================================================================
WITH

weekly_counts AS (
    SELECT
        WEEK(event_time)                                AS week_num,
        DATE(MIN(event_time))                           AS week_start,
        COUNT(DISTINCT CASE WHEN event_name = 'page_view'
              THEN user_id END)                         AS s1,
        COUNT(DISTINCT CASE WHEN event_name = 'product_view'
              THEN user_id END)                         AS s2,
        COUNT(DISTINCT CASE WHEN event_name = 'add_to_cart'
              THEN user_id END)                         AS s3,
        COUNT(DISTINCT CASE WHEN event_name = 'purchase'
              THEN user_id END)                         AS s4
    FROM events
    GROUP BY WEEK(event_time)
),

with_conversion AS (
    SELECT
        week_num, week_start, s1, s2, s3, s4,
        ROUND(s4 * 100.0 / NULLIF(s1, 0), 2)           AS conv_pct
    FROM weekly_counts
)

SELECT
    week_num,
    week_start,
    s1, s2, s3, s4,
    conv_pct,
    LAG(conv_pct) OVER (ORDER BY week_num)              AS prev_week_conv_pct,
    ROUND(conv_pct -
          LAG(conv_pct) OVER (ORDER BY week_num), 2)    AS wow_change,
    CASE
        WHEN conv_pct > LAG(conv_pct) OVER (ORDER BY week_num)
             THEN 'Improving'
        WHEN conv_pct < LAG(conv_pct) OVER (ORDER BY week_num)
             THEN 'Degrading'
        ELSE 'Stable'
    END                                                 AS trend
FROM with_conversion
ORDER BY week_num;

-- ====================================================================================================================================================
-- Business Question: How long did each converting user take to complete the funnel?
-- Who was fastest and slowest?
-- SQL Concepts: TIMESTAMPDIFF, MIN, MAX, AVG
-- Approach: Window Function + Time Calculation
-- ====================================================================================================================================================
WITH

deduplicated AS (
    SELECT
        user_id, event_name, event_time,
        ROW_NUMBER() OVER (
            PARTITION BY user_id, event_name
            ORDER BY event_time, event_id
        ) AS rn
    FROM events
),

first_touch AS (
    SELECT user_id, event_name, event_time
    FROM deduplicated WHERE rn = 1
),

pivoted AS (
    SELECT
        user_id,
        MAX(CASE WHEN event_name = 'page_view'
            THEN event_time END) AS t1,
        MAX(CASE WHEN event_name = 'product_view'
            THEN event_time END) AS t2,
        MAX(CASE WHEN event_name = 'add_to_cart'
            THEN event_time END) AS t3,
        MAX(CASE WHEN event_name = 'purchase'
            THEN event_time END) AS t4
    FROM first_touch
    GROUP BY user_id
)

SELECT
    user_id,
    t1                                                  AS page_view_time,
    t4                                                  AS purchase_time,
    TIMESTAMPDIFF(MINUTE, t1, t2)                       AS mins_t1_to_t2,
    TIMESTAMPDIFF(MINUTE, t2, t3)                       AS mins_t2_to_t3,
    TIMESTAMPDIFF(MINUTE, t3, t4)                       AS mins_t3_to_t4,
    TIMESTAMPDIFF(MINUTE, t1, t4)                       AS total_mins
FROM pivoted
WHERE t4 IS NOT NULL
ORDER BY total_mins;


-- ====================================================================================================================================================
-- Aggregated Summary
-- ====================================================================================================================================================
WITH
deduplicated AS (
    SELECT
        user_id, event_name, event_time,
        ROW_NUMBER() OVER (
            PARTITION BY user_id, event_name
            ORDER BY event_time, event_id
        ) AS rn
    FROM events
),

first_touch AS (
    SELECT user_id, event_name, event_time
    FROM deduplicated WHERE rn = 1
),

pivoted AS (
    SELECT
        user_id,
        MAX(CASE WHEN event_name = 'page_view'
            THEN event_time END) AS t1,
        MAX(CASE WHEN event_name = 'product_view'
            THEN event_time END) AS t2,
        MAX(CASE WHEN event_name = 'add_to_cart'
            THEN event_time END) AS t3,
        MAX(CASE WHEN event_name = 'purchase'
            THEN event_time END) AS t4
    FROM first_touch
    GROUP BY user_id
)

SELECT
    COUNT(*)                                            AS total_converters,
    ROUND(AVG(TIMESTAMPDIFF(MINUTE, t1, t4)), 2)       AS avg_mins_to_convert,
    MIN(TIMESTAMPDIFF(MINUTE, t1, t4))                 AS fastest_mins,
    MAX(TIMESTAMPDIFF(MINUTE, t1, t4))                 AS slowest_mins,
    ROUND(AVG(TIMESTAMPDIFF(MINUTE, t1, t2)), 2)       AS avg_mins_step1_to_2,
    ROUND(AVG(TIMESTAMPDIFF(MINUTE, t2, t3)), 2)       AS avg_mins_step2_to_3,
    ROUND(AVG(TIMESTAMPDIFF(MINUTE, t3, t4)), 2)       AS avg_mins_step3_to_4
FROM pivoted
WHERE t4 IS NOT NULL;
