# sql-funnel-analysis-essentials
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
