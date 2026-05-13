# BigQuery Optimization

> A practical guide to cost and performance optimization in BigQuery. All information is based on official Google Cloud documentation.

## When will it look at this file?

- BigQuery query is running slowly
- Query cost (bytes processed) is high
- A new BigQuery table is being designed
- An existing dbt model is being refactored
- Storage cost is high

---

## Partition Pruning 

**What's all about Partition?**

Basically, a *partition* means a way of splitting large tables into smaller, manageable segments (such as a date, an integer range, or an ingestion time), so this way BigQuery only reads the relevant segments instead of scanning the entire table, which reduces both cost and query time.

**Partition Pruning?**

If a query uses a qualified filter on the partition column value, BigQuery can only scan for partitions that match the filter and discard the remaining partitions. This pruning method significantly reduces query costs. This whole mechanism is called partition pruning.

**Why does it matter?**

Because BigQuery billing is based on processed bytes, it directly reduces the amount of data the query scans and shortens response time. Especially with TB-sized tables, precise filters narrows the scanned data down to only the relevant sections of the table; this can significantly reduce costs.

**Bad example:**
```sql
SELECT user_id, event_name
FROM `project.dataset.events`
WHERE user_id = 123
```

**Good Example:**
```sql
SELECT user_id, event_name
FROM `project.dataset.events`
WHERE event_date = '2026-05-08'
  AND user_id = 123
```

**Reference:** https://cloud.google.com/bigquery/docs/querying-partitioned-tables

---

## 2. Clustering

### What is clustering?

A clustered table automatically sorts its data based on the values of one or more user-defined columns (called clustering columns). This co-locates related rows in storage, which improves query performance and reduces cost when those columns are used as filters.

### How is it different from partitioning?

While *clustering* doesn't split the table; instead it sorts data within storage blocks by the clustering columns. *Partitioning* split the table into separate physical segments based on a column's value.

### Which columns should be clustered?

- Prioritize aggregation columns. 
- Columns with unique values are then ideal for clustering. 
- Sort clustering columns from most frequently filtered to least frequently filtered. 
- BigQuery allows a maximum of 4 clustering columns. Therefore, choose carefully; adding unnecessary columns will create more complexity than benefit. 
- Clustering columns can only be of certain types (`STRING`, `INT64`, `DATE`, `TIMESTAMP`, `BOOL`, `NUMERIC`, `BIGNUMERIC`, `DATETIME`, `GEOGRAPHY`, `RANGE`). Types like `FLOAT64`, `ARRAY`, or `STRUCTURE` cannot be clustered. 
- If your table is partitioned by a single column (e.g., order_date), clustering that same column is pointless.

### Why does it matter?

It matters because clustering only saves you money and time when BigQuery can actually skip blocks during a query. When a query filters or aggregates on clustering columns, BigQuery can skip storage blocks that don't contain matching values. This means less data scanned, lower cost, and faster queries — especially noticeable on large tables (TB+) where full scans are expensive. If the query doesn't use the clustering columns, clustering provides no benefit.

### Example

```sql
CREATE TABLE `project.dataset.events`
(
  user_id INT64,
  event_name STRING,
  event_date DATE,
  country STRING
)
PARTITION BY event_date
CLUSTER BY user_id, country;
```

This table is partitioned by `event_date` and clustered by `user_id` and `country`. Queries filtering on these columns will scan less data.

**Reference:** https://cloud.google.com/bigquery/docs/clustered-tables

---

## 3. SELECT * Avoidance

### Why is `SELECT *` expensive in BigQuery?

BigQuery stores data in a column-based format. The values of each column are stored sequentially in its own block. Using `SELECT *` reads all column blocks in the query. To put it more practically, if you query 5 columns in a 1TB table with 100 columns, you'll roughly only scan 50 GB of data (assuming similar column sizes). If you run the same query with `SELECT *`, you'll read the entire 1TB — that's 20 times more data and at a higher cost. This is where the power of column-based storage comes into play.

### What should you do instead?

- Use the data preview options
- Query specific columns (do not use `LIMIT` clause to a `SELECT *` query cause does not affect the amount of data read.)
- Use partitioned tables
- Use `SELECT * EXCEPT`

### What if I really need most columns?

- Use `SELECT * EXCEPT` . You exclude the raw data columns of type `STRING` or `JSON`, which account for most of the cost. Often, 70-80% of the table's cost comes from 2-3 large columns; excluding them provides significant savings.

### Bad example

```sql
SELECT *
FROM `project.dataset.events`
WHERE event_date = '2026-05-08'
```

Reads all columns even if you only need a few.

### Good example

```sql
SELECT user_id, event_name, event_time
FROM `project.dataset.events`
WHERE event_date = '2026-05-08'
```

Reads only the 3 columns needed.

### When you need most columns but want to exclude a few

```sql
SELECT * EXCEPT(internal_debug_col, raw_payload)
FROM `project.dataset.events`
WHERE event_date = '2026-05-08'
```

Excludes large/unused columns explicitly.

**Reference:** https://cloud.google.com/bigquery/docs/best-practices-performance-input

---

## 4. Materialized Views

### What is a materialized view?

Materialized views mean "pre-calculating and storing the answer to a frequently asked but expensive-to-calculate question." If the same aggregation or filtering operation is performed repeatedly, materializing it and reading the ready-made result instead of calculating it from scratch each time will significantly reduce both cost and response time.

### When should you use it?

- **If the same aggregate query is run frequently:** Ideal for metrics that BI dashboards and reports constantly ask for, such as daily sales totals and active user numbers.
- **If there are repetitive, costly queries on large tables:** Significant cost savings are achieved by reading a small, pre-calculated result instead of scanning a TB-scale table every time.
- **If real-time analytics are required on streaming data:** Used to quickly provide instant summary metrics on tables that constantly receive data; BigQuery automatically updates the view in the background.

### Cost-benefit tradeoff

Materialized views consume both storage (the precomputed result) and refresh compute (BigQuery automatically re-runs the underlying query when the base table changes). Use them when `query cost × daily run count > refresh cost + storage cost`. For rarely-queried results, a logical view or an ad-hoc query is cheaper.

### Limitations

- **Not every query can be materialized:** Classic (incremental) materialized views don't support some SQL features — for example, window functions (`ROW_NUMBER`, `RANK`) won't work. For more complex queries, you can use the non-incremental version, but then you have to accept that the data will be somewhat "stale".
- **Restrictions on base tables:** You cannot build a materialized view on top of external tables, wildcard tables, logical views, or other materialized views.
- **Once created, it cannot be modified:** You cannot update the underlying SQL query later — you would need to delete and recreate it to make changes. Also, `INSERT` / `UPDATE` / `DELETE` operations are not allowed; the MV is read-only.
- **Hard limits:** A table can have a maximum of 100 materialized views per project and a maximum of 500 per organization.
- **Streaming gotcha:** If you don't refresh the view for more than 3 days on tables that receive streaming data, your queries may return errors.

### Example

```sql
CREATE MATERIALIZED VIEW `project.dataset.daily_user_event_counts` AS
SELECT
  user_id,
  event_date,
  COUNT(*) AS event_count
FROM `project.dataset.events`
GROUP BY user_id, event_date;
```

This materialized view precomputes daily event counts per user. Queries that aggregate over `events` for these dimensions will be automatically routed to this MV by BigQuery's query optimizer.

**Reference:** https://cloud.google.com/bigquery/docs/materialized-views-intro

----

## 5. JOIN Optimization

`JOIN`s are the key to preventing queries from running too slowly or never finishing at all. Four core rules — *filter/aggregate first, sort tables from largest to smallest, use matching data types, and materialize repeated joins* — will cut most BigQuery bills by more than half.

### Place the largest table first

The order in which tables are combined matters. The best strategy is to place the table with the most rows first, followed by the table with the fewest rows, and then the remaining tables in descending order of size.

### Broadcast JOINs

A broadcast join occurs when there is a large table on the left side of the join and a small table on the right side. BigQuery sends all the data from the small table to each slot processing the large table, which avoids expensive shuffles. Broadcast joins are the preferred strategy when one side is small enough to fit in memory.

### Filter and aggregate before joining

Performing aggregation and filtering before the `JOIN` reduces the amount of data being joined, which directly lowers cost and speeds up the query. The `GROUP BY` clause should only be used when actually needed — unnecessary grouping adds compute.

### Use matching data types on JOIN keys

JOIN keys should use matching, simple types. Mismatches like `STRING` vs `INT64` force BigQuery to apply implicit casts on every row, which slows down the join and prevents some optimizations.

### Bad example

```sql
SELECT *
FROM small_table s
JOIN large_table l ON s.id = l.id
WHERE l.country = 'TR'
```

Small table on the left + filtering applied after JOIN — both inefficient.

### Good example

```sql
SELECT l.user_id, l.event_name, s.country_name
FROM large_table l
JOIN small_table s ON l.id = s.id
WHERE l.country = 'TR'
```

Large table on the left + filter on `l.country` reduces data scanned. (Even better: pre-filter inside a CTE.)

### Better with CTE pre-filtering

```sql
WITH filtered_events AS (
  SELECT user_id, event_name, country, id
  FROM large_table
  WHERE country = 'TR'
    AND event_date = '2026-05-08'
)
SELECT f.user_id, f.event_name, s.country_name
FROM filtered_events f
JOIN small_table s ON f.id = s.id
```

Filters first, joins on a much smaller dataset.

**Reference:** https://cloud.google.com/bigquery/docs/best-practices-performance-compute

---

## 6. Query Plan & Slot Analysis

### What is a query plan?

When BigQuery executes a query, it converts the SQL into an execution graph that consists of stages. Stages are composed of steps, the elemental operations that perform the query's logic. The query plan uses the terms work units and workers to describe stage parallelism.

### Where to find it

In the BigQuery Console, after running a query, click **Execution details** or **Job information**. You can also fetch it programmatically via `INFORMATION_SCHEMA.JOBS_BY_PROJECT`.

### Key concepts

- **Stage:** A unit of work in the query execution graph — it represents a logical step, such as a table scan, join, or aggregate, and is executed in parallel by multiple workers.

- **Slot:** BigQuery's unit of parallel computing capacity is an abstract representation of the CPU, memory, and I/O resources that execute the work units of a phase in parallel.

- **Slot time:** This is the total CPU time consumed by all parallel workers for a query; it shows how much computational resource the query is using, rather than the actual time.

- **Bytes shuffled:** Shuffle is the amount of data moved through memory between stages; high values ​​can indicate data skew, inefficient joins, or unnecessarily large intermediate results.

### What metrics to look at

- **Wait / Read / Compute / Write ratios:** These four ratios show how workers spend their time in a stage — Wait is waiting for the scheduled time, Read is reading the input, Compute is computation on the CPU, and Write is writing the output. In a healthy query, Compute is dominant; high Wait indicates slot competition, high Read indicates inefficient scanning or lack of filtering, and high Write indicates excessively large intermediate results.

- **Slot contention:** When there are many concurrent queries sharing the same slot pool, the queries queue for processing, increasing wait times and extending the overall execution time.

- **Skew:** This is a situation where one worker processes significantly more data than the others — recognizable by computeMsMax being noticeably higher than computeMsAvg in the query plan, and reducing the entire process to the speed of the slowest worker.

### Common bottleneck signals

- **High "Wait" time:** Workers are sitting idle, waiting to be assigned a slot instead of doing actual work. This usually means too many queries are running at the same time and competing for the same slots. Fix: reduce concurrency or get more slot capacity.

- **High "Compute" time on one stage:** That stage is CPU-heavy — likely doing complex math, regex, `DISTINCT` aggregations, or window functions. If `computeMsMax` is much higher than `computeMsAvg`, you have data skew: a few workers are doing most of the work while others sit idle.

- **High "Bytes shuffled":** Too much data is being moved between stages. Common causes: joining before filtering, unnecessary `GROUP BY`, or carrying too many columns into the join. If the data doesn't fit in memory, it spills to disk (`shuffleOutputBytesSpilled`), which makes things much slower. Fix: filter and aggregate before the join, and only keep the columns you need.

- **Slow final stage:** Often happens when the last step runs on a single worker — for example, an `ORDER BY` without a `LIMIT` forces all data into one slot for sorting. Fix: add a `LIMIT`, move sorting into a window function, or write results directly to a destination table.

### How to act on it

Once you've identified the bottleneck, the fix usually points back to one of the earlier sections. If a stage scans too many bytes, revisit **Section 1 (Partition Pruning)** to ensure your `WHERE` clause filters on the partitioning column, and **Section 3 (SELECT * Avoidance)** to cut unnecessary columns. If skew or excessive shuffling shows up in a `JOIN` stage, reconsider your clustering keys **(Section 2)** and apply the `JOIN` best practices **(Section 5)** — filter before joining, place the largest table first, and use `INT64` keys instead of `STRING`. If the same expensive query keeps appearing across the workload, materializing it once **(Section 4: Materialized Views)** eliminates the bottleneck entirely instead of optimizing each individual run.

**Reference:** https://cloud.google.com/bigquery/docs/query-plan-explanation

---

## 7. Storage Optimization

While the previous sections focused on **query cost** (bytes scanned at runtime), this section is about **storage cost** — what you pay every month for tables sitting in BigQuery, even when no queries run against them. For large or long-lived datasets, storage can quietly become a significant portion of the bill.

### Active vs Long-term storage

BigQuery automatically applies a discounted rate to data that hasn't been modified for **90 consecutive days**: active storage is ~$0.02/GB/month, long-term storage is ~$0.01/GB/month — roughly a 50% discount. The discount happens **automatically per partition**, no action required. However, any modification (including `UPDATE`, `MERGE`, or even a partial backfill) resets the partition's clock back to active pricing. Be careful with backfill jobs that touch old partitions: they can silently revert long-term storage discounts you were already getting.

### Partition expiration

For time-series tables (logs, events, metrics), set `partition_expiration_days` to automatically delete partitions older than a threshold. This caps storage growth and removes the need for manual cleanup jobs.

```sql
ALTER TABLE `project.dataset.events`
SET OPTIONS (
  partition_expiration_days = 365
);
```

Partitions older than 365 days are deleted automatically, with no slot or scan cost.

You can also set a **default partition expiration at the dataset level**, which applies to all new partitioned tables created in that dataset.

### Table expiration

For temporary or sandbox tables (intermediate results, ad-hoc analyses, dev tables), set an `expiration_timestamp` so the entire table is removed automatically. This prevents the very common pattern of forgotten temp tables piling up storage cost.

```sql
ALTER TABLE `project.dataset.temp_analysis`
SET OPTIONS (
  expiration_timestamp = TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
);
```

The table self-destructs after 7 days.

### Time travel and fail-safe storage

BigQuery retains deleted/updated data for two overlapping windows:

- **Time travel:** default **7 days**, configurable down to **2 days**. You can query historical versions or restore deleted data within this window.
- **Fail-safe:** an additional **7 days** after the time travel window. Not user-accessible — only Google Support can restore data from fail-safe.

Under the **physical billing model**, both windows count toward your storage cost. For high-churn tables (frequent `UPDATE`/`DELETE`/`MERGE`), the time travel buffer can hold significantly more bytes than the live table itself. Reducing the window is a quick win when you have external backups elsewhere.

```sql
ALTER SCHEMA `project.dataset`
SET OPTIONS (
  max_time_travel_hours = 48
);
```

Cuts time travel from the default 168 hours (7 days) down to 48 hours.

### Billing models: logical vs physical

BigQuery offers two storage billing models:

- **Logical (default):** billed on uncompressed data size. Time travel and fail-safe are **not charged**.
- **Physical:** billed on compressed data size, which is typically much smaller, but the per-GB rate is higher. Time travel and fail-safe **are charged** at active physical rates.

Physical storage is often cheaper for **text-heavy tables** (JSON, logs, free-form strings) where compression ratios are high. It's usually a worse choice for high-churn tables, since the time travel/fail-safe buffer adds up. Run a comparison on `INFORMATION_SCHEMA.TABLE_STORAGE` before switching.

### Require partition filter

For very large partitioned tables (especially shared with multiple analysts), set `require_partition_filter = TRUE`. Any query without a partition filter will fail rather than silently scanning the whole table.

```sql
ALTER TABLE `project.dataset.events`
SET OPTIONS (
  require_partition_filter = TRUE
);
```

Prevents accidental full-table scans by analysts who forgot to filter on the partition column.

### Aggregate, then drop row-level data

For long-term retention, you usually don't need raw row-level data — pre-aggregated metrics are enough. Build aggregation queries that compute daily/weekly summaries, store those, and let `partition_expiration_days` clean up the row-level partitions. This dramatically lowers long-term storage cost while keeping reporting metrics intact.

### Quick wins

- Set `partition_expiration_days` on every time-series table (logs, events, metrics).
- Set `expiration_timestamp` on sandbox / scratch / dev tables — never let them be permanent by accident.
- Reduce `max_time_travel_hours` for high-churn datasets if you have external backups.
- Set `require_partition_filter = TRUE` on large shared tables.
- Audit the largest tables monthly using `INFORMATION_SCHEMA.TABLE_STORAGE`.
- Avoid backfill jobs that rewrite old partitions just to fix a few rows — they reset long-term storage discounts.

**Reference:** https://cloud.google.com/bigquery/docs/best-practices-storage

---

## 8. How to Review a Query

When given a query to optimize, apply this systematic review instead of jumping to the first issue you notice. Complex queries often have multiple compounding issues — fixing one in isolation misses 80% of the gains.

### Step 1 — Decompose the query

Identify the structural pieces before anything else:
- How many CTEs are there?
- How many JOINs, and what's the size relationship between joined tables?
- How many subqueries, and are any of them in WHERE clauses?
- How many aggregations / GROUP BYs?
- Are there window functions?

A 5-line query and a 200-line query need different review depths. Don't apply Section 1 advice to a single line — apply it to every relevant piece.

### Step 2 — Scan-cost layer (Sections 1, 3, 7)

For every base table reference in the query:
- Is the table partitioned? Does the query filter on the partition column?
- Is the partition filter a literal (`WHERE event_date = '2026-05-08'`) or wrapped in a function (`WHERE DATE(event_ts) = ...`)?
- Is `SELECT *` used anywhere? Even inside a CTE?
- Are there columns being selected that aren't used downstream?

Flag every base table that scans more than needed. Don't stop at the first one.

### Step 3 — Join layer (Section 5)

For every JOIN in the query:
- What's the row count of left vs right? Is largest-first respected?
- Is the join key the same type on both sides?
- Is there pre-filtering before the join, or are unfiltered tables being joined?
- Could the join be eliminated entirely (denormalization, cached lookup)?

In complex queries with 5+ JOINs, one poorly-ordered join can dominate cost — find the worst one first.

### Step 4 — Aggregation layer (Sections 4, 6)

- Are there expensive aggregations being recomputed? (materialized view candidate)
- Is there a GROUP BY that could be skipped?
- Are window functions partitioning on a skewed key?
- Is the same expensive subquery written twice?

### Step 5 — Structural layer

- Is the same CTE referenced multiple times? (BigQuery may re-execute it each reference)
- Are there UNIONs that could be JOINs, or vice versa?
- Is there an ORDER BY without a LIMIT?
- Could any of this be moved to a scheduled summary table?

### Step 6 — Report findings systematically

When responding, structure the answer by impact, not by section:
- **Highest impact:** the issues that affect the most bytes scanned or compute time
- **Medium impact:** structural improvements
- **Low impact / style:** consistency fixes

Don't list everything you found and let the user prioritize — rank for them.

### When to ask for more information

If the query is over 100 lines, or involves tables you can't infer sizes for, **ask** instead of guessing:
- "What's the approximate row count of `events` and `users`?"
- "Is `dim_country` static or updated frequently?"
- "Do you have access to the query plan? It would tell us whether the bottleneck is scan, join, or aggregation."

A targeted question is more useful than a generic recommendation. Skip the guess.

