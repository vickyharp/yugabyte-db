---
title: Bucket-based indexes in YugabyteDB YSQL
headerTitle: Bucket-based indexes
linkTitle: Bucket-based indexes
description: Using bucket-based indexes in YSQL
headContent: Use bucket-based distribution of indexes to avoid hotspots
menu:
  stable_develop:
    identifier: bucket-based-index-ysql
    parent: data-modeling
    weight: 310
tags:
  other: ysql
  feature: early-access
type: docs
---

Bucket-based indexes give you the write distribution of hash sharding and the read ordering of range sharding — without sacrificing either. The tension they resolve is fundamental to how distributed databases handle monotonically increasing keys like timestamps or sequential IDs. Range sharding keeps range reads simple — adjacent keys land on the same tablet — but concentrates all writes on whichever tablet holds the newest keys. Hash sharding solves that write hotspot by distributing rows evenly across tablets, but destroys ordering in the process: even a small range read has to scan every tablet in full and perform a global sort to reconstruct sequence.

Let's take an example of a monitoring table that stores readings by timestamp, and whose most common query reads the most recent 1000 rows. If this is range-sharded, the most recent data always goes to the same tablet, and the most common query is always contending with active writes because it is reading from the same place — a pattern known as a hot tablet. Using hash sharding would solve the write hotspot, but to satisfy that query for the most recent 1000 rows the database would have to:

1. Query each tablet in its entirety (since the hash does not store data in order)
2. Collect all results from all tablets into one place.
3. Perform a final global sort to ensure the total result set is correctly ordered and apply the LIMIT clause.

To make this into a bucket-based index, you put a deterministic hash bucket column at the head of the index key and let the rest of the key stay range-ordered. In YugabyteDB, this is done with a modulo on a hash column `yb_hash_code(col) % N` where N is the bucket count. For our example, let's further specify that each bucket is assigned its own tablet using `SPLIT AT`. Writes now spread across N tablets because each row's bucket is determined by the hash; within each tablet, the bucket column is constant, so rows stay ordered by the second index column.

So for the most recent 1000 rows, the database now only needs to:

1. Query the last 1000 rows on each tablet
2. Merge the pre-sorted results across the streams and apply the LIMIT

What makes this work without query changes is the combination of the schema and the query planner. When you ask for ordered results with a LIMIT as above, the planner detects the bucket-based schema and pushes down the sort-and-limit to all N tablets in parallel. It then treats each stream as locally sorted, and merges them into a single globally-ordered set to apply the LIMIT and return results. The query is the same as what would be written for a single-node B-tree in a non-distributed database.


## When to use it

Use bucket-based scans for the following workloads:

- Timestamp-ordered inserts.
- Sequence-based IDs.
- "Latest N items" queries (feeds, time-series, audit tables).

## Syntax

```sql
CREATE INDEX index_name ON table_name(yb_hash_code(<key_columns>) % <buckets>) ASC, column_name ASC)
SPLIT AT VALUES ((1), (2));
```

- Only [yb_hash_code()](../../../api/ysql/exprs/func_yb_hash_code/) can be used to benefit from scan optimizations. (Issue {{<issue 31072>}})

- For unique indexes, columns in `yb_hash_code()` must be a subset of the remaining columns in the key. Non-unique indexes can have anything in `yb_hash_code()`.

- The bucket column or index expression column should generally be the first column of the key.

- What you set the number of buckets to depends on how much you want to spread the previously-hot-shard load. The simplest recommendation is to make it equal to the number of nodes.

- You should take care that the load is spread to other nodes. For example, with 3 nodes and 9 tablets, if there are 3 buckets, each bucket having 3 tablets, it could be the case that the hottest tablet of each bucket is on the same node.

    To reduce the chance of this happening, increase number of buckets and/or keep tablet-to-bucket ratio low (or increase number of nodes).

- You can't change the number of buckets after creation (not an issue for secondary indexes, which can be recreated).

- The SPLIT clause is optional but recommended to cleanly distribute each bucket to its own tablet. For example, with 3 buckets, if tablets were dynamically split to ((0, 99999), (1, 99999)), then the scan for buckets 0 and 1 may query two tablets. If tablets were presplit to ((1), (2)), all 3 buckets would query one tablet.

## Parameters

You configure bucket-based scan optimizations in the [query planner](../../../architecture/query-layer/planner-optimizer/) using the following configuration parameters:

- yb_enable_derived_equalities: Set to `true`.
- yb_enable_derived_saops: Set to `true`.
- yb_max_merge_scan_streams: Maximum number of buckets to process in parallel. The recommended value is 64.
- yb_max_saop_merge_streams: Used in v2025.2.1.0 and v2025.2.2.0; deprecated in v2025.2.3.0. Use `yb_max_merge_scan_streams` instead.

For more information, see [Bucket-based index scan optimization](../../../reference/configuration/yb-tserver/#bucket-based-index-scan-optimization) parameters.

In addition, the [cost-based optimizer](../../../best-practices-operations/ysql-yb-enable-cbo/) (CBO) must be enabled (CBO is enabled by default when you deploy your universe using yugabyted, YugabyteDB Anywhere, or YugabyteDB Aeon).

## Setup

Follow the [setup instructions](../../../explore/cluster-setup-local/#multi-node-universe) to start a local multi-node universe with a replication factor of 3, and connect to universe using ysqlsh.

The following example assumes you are running v2025.2.3.0 or later.

### Configure bucket-based indexing

Enable features that preserve global ordering across the buckets by setting the configuration parameters:

```sql
SET yb_max_merge_scan_streams=64;
SET yb_enable_derived_saops=true;
SET yb_enable_derived_equalities=true;
ALTER DATABASE yugabyte SET yb_max_merge_scan_streams=64;
ALTER DATABASE yugabyte SET yb_enable_derived_saops=true;
ALTER DATABASE yugabyte SET yb_enable_derived_equalities=true;
```

## Point lookups

Single-row lookups by exact key work with bucket-based indexes because the planner automatically adds the bucket predicate to the index condition. As a result, the query can target the correct tablet instead of scanning all buckets.

Create a table with a generated column:

```sql
CREATE TABLE foo (
  r1 int,
  r2 int,
  v1 int,
  v2 int,
  bucket_id int GENERATED ALWAYS AS (yb_hash_code(r1, r2) % 3) STORED,
  PRIMARY KEY (bucket_id ASC, r1, r2))
SPLIT AT VALUES ((1), (2));
```

Using the primary key (r1, r2), the `bucket_id` clause is automatically added:

```sql
EXPLAIN
    SELECT *
    FROM foo
    WHERE r1 = 1
    AND r2 = 1;
```

```output
 Index Scan using foo_pkey on foo  (cost=20.00..21.10 rows=1 width=20)
   Index Cond: ((bucket_id = (yb_hash_code(1, 1) % 3)) AND (r1 = 1) AND (r2 = 1))
```

Create a secondary index:

```sql
CREATE INDEX foo_bucket_idx_v1_v2 ON foo (
  (yb_hash_code(v1, v2) % 3) ASC,
  v1,
  v2)
SPLIT AT VALUES ((1), (2));
```

Run an EXPLAIN on a point lookup query:

```sql
EXPLAIN SELECT * FROM foo WHERE v1 = 1 AND v2 = 1;
```

```output
 Index Scan using foo_bucket_idx_v1_v2 on foo  (cost=40.02..68.42 rows=1 width=20)
   Index Cond: (((yb_hash_code(v1, v2) % 3) = (yb_hash_code(1, 1) % 3)) AND (v1 = 1) AND (v2 = 1))
```

The query planner automatically calculates the bucket ID for the search keys and adds the bucket predicate to the index condition. This allows the database to go directly to a single, specific tablet (bucket).

## Range scans

The following example walks through using a bucket-based index to avoid write hot spots on a timestamp column. You create a table, define an index that distributes writes across three buckets (tablets), insert sample rows with timestamps over a range, and enable the planner settings for bucket-based merge. Then you run queries with ORDER BY and LIMIT and confirm in the plan that there is no Sort node—ordering comes from merging streams from each bucket.

Create a table with a monotonic column:

```sql
CREATE TABLE te (
    id SERIAL PRIMARY KEY,
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Create an index that always writes to the 3 nodes by adding a bucket column and SPLIT AT:

```sql
CREATE INDEX yb_nothotspot
    ON te ((yb_hash_code(timestamp) % 3) ASC, timestamp) INCLUDE (id)
    SPLIT AT VALUES ((1), (2));
```

SPLIT AT ensures each bucket (0, 1, 2) maps to its own tablet, so writes are spread across three tablets.

As a reminder, an ordinary "hot-spot" index would use only the timestamp column (no bucket column and no SPLIT AT).

```output.sql
ON te (timestamp ASC) INCLUDE (id);
```

Insert rows with increasing timestamps:

```sql
INSERT INTO te (timestamp)
SELECT
    '2026-01-01 00:00:00+00'::timestamptz
    + (gs * (('2024-01-01 00:00:00+00'::timestamptz - '2026-01-01 00:00:00+00'::timestamptz) / 100000))
FROM generate_series(0, 99999) AS gs;
```

Observe planner changes that provide global ordering with no SQL or application changes:

```sql
ANALYZE;
EXPLAIN (ANALYZE, COSTS off)
    SELECT timestamp
    FROM te
    WHERE timestamp >= '2020-01-01'
        AND timestamp <  '2030-01-01'
    ORDER BY timestamp;
```

```output
 Index Only Scan using yb_nothotspot on te (actual time=17.731..105.882 rows=100000 loops=1)
   Index Cond: (("timestamp" >= '2020-01-01 00:00:00-05'::timestamp with time zone) AND ("timestamp" < '2030-01-01 00:00:00-05'::timestamp with time zone) AND (((yb_hash_code("timestamp") % 3)) = ANY ('{0,1,2}'::integer[])))
   Merge Sort Key: "timestamp"
   Merge Stream Key: (yb_hash_code("timestamp") % 3)
   Merge Streams: 3
   Heap Fetches: 0
 Planning Time: 5.969 ms
 Execution Time: 123.060 ms
 Peak Memory Usage: 144 kB
```

When the optimization is enabled, EXPLAIN output includes the following additional properties:

- Merge Sort Key: columns that the index uses for the merge.
- Merge Stream Key: columns involved in forming buckets.
- Merge Streams: number of streams that the index merges, generally the cardinality of the cross product of all merge stream key values.

Notice that no sort is present and that the `yb_enable_derived_saops` feature passed the "bucket" column into the index condition. The index has then merged the streams and returned sorted data, eliminating the need for a sort.

### Add a LIMIT

You really see how powerful this feature is when you use the LIMIT clause.

First turn off the optimization:

```sql
SET yb_max_merge_scan_streams = 0;
```

Run the following query:

```sql
EXPLAIN (ANALYZE, COSTS off, TIMING on)
    SELECT timestamp
    FROM te
    WHERE timestamp >= '2020-01-01'
    AND timestamp <  '2030-01-01'
    ORDER BY timestamp
    LIMIT 1000;
```

```output
 Limit (actual time=71.180..71.485 rows=1000 loops=1)
   ->  Sort (actual time=71.178..71.262 rows=1000 loops=1)
         Sort Key: "timestamp"
         Sort Method: top-N heapsort  Memory: 49kB
         ->  Index Only Scan using yb_nothotspot on te (actual time=2.655..52.299 rows=100000 loops=1)
               Index Cond: (("timestamp" >= '2020-01-01 00:00:00-05'::timestamp with time zone) AND ("timestamp" < '2030-01-01 00:00:00-05'::timestamp with time zone))
               Heap Fetches: 0
 Planning Time: 0.144 ms
 Execution Time: 71.710 ms
 Peak Memory Usage: 129 kB
```

Without the optimization, the planner performs a sort as expected.

Turn the optimization back on:

```sql
SET yb_max_merge_scan_streams = 64;
```

Run the query again:

```sql
EXPLAIN (ANALYZE, COSTS off, TIMING on)
    SELECT timestamp
    FROM te
    WHERE timestamp >= '2020-01-01'
    AND timestamp <  '2030-01-01'
    ORDER BY timestamp
    LIMIT 1000;
```

```output
 Limit (actual time=2.907..3.923 rows=1000 loops=1)
   ->  Index Only Scan using yb_nothotspot on te (actual time=2.904..3.663 rows=1000 loops=1)
         Index Cond: (("timestamp" >= '2020-01-01 00:00:00-05'::timestamp with time zone) AND ("timestamp" < '2030-01-01 00:00:00-05'::timestamp with time zone) AND (((yb_hash_code("timestamp") % 3)) = ANY ('{0,1,2}'::integer[])))
         Merge Sort Key: "timestamp"
         Merge Stream Key: (yb_hash_code("timestamp") % 3)
         Merge Streams: 3
         Heap Fetches: 0
 Planning Time: 0.164 ms
 Execution Time: 4.184 ms
 Peak Memory Usage: 8 kB
```

The SQL is asking for 1000 globally ordered rows, and the index returns it in a fraction of the time, without the sort, while scanning 1000 rows per bucket.

### Keyset pagination

Bucket-based scans also work well for more complex OLTP top-N queries, such as keyset pagination.

Create a more complicated index and an extra predicate:

```sql
ALTER TABLE te ADD COLUMN key_id integer NOT NULL DEFAULT 123;
ANALYZE;
CREATE INDEX scalable_key_timestamp ON te (
    (yb_hash_code(timestamp) % 3) ASC,
    key_id,
    timestamp ASC,
    id
) SPLIT AT VALUES ((1), (2));
```

Run the following query:

```sql
EXPLAIN (ANALYZE, COSTS off, TIMING on)
SELECT *
    FROM te
    WHERE 1=1
        AND key_id = 123
        AND timestamp >= '2025-05-05 08:00:00'
        AND (timestamp, id) > ('2025-05-05 08:00:00', 1)
    ORDER BY timestamp ASC, id ASC
    LIMIT 1000;
```

```output
 Limit (actual time=2.464..3.393 rows=1000 loops=1)
   ->  Index Only Scan using scalable_key_timestamp on te (actual time=2.462..3.152 rows=1000 loops=1)
         Index Cond: ((key_id = 123) AND ("timestamp" >= '2025-05-05 08:00:00-04'::timestamp with time zone) AND (ROW("timestamp", id) > ROW('2025-05-05 08:00:00-04'::timestamp with time zone, 1)) AND (((yb_hash_code("timestamp") % 3)) = ANY ('{0,1,2}'::integer[])))
         Merge Sort Key: "timestamp", id
         Merge Stream Key: (yb_hash_code("timestamp") % 3)
         Merge Streams: 3
         Heap Fetches: 0
 Planning Time: 10.316 ms
 Execution Time: 5.938 ms
 Peak Memory Usage: 132 kB
```

The global ordering is still preserved on this keyset pagination without any changes to the SQL.

## Learn more

- [Hot shards](../hot-shards-ysql/)
- [Scaling writes](../../../explore/linear-scalability/scaling-writes)
- [Cost-based optimizer (CBO)](../../../best-practices-operations/ysql-yb-enable-cbo)
- [EXPLAIN and ANALYZE](../../../launch-and-manage/monitor-and-alert/query-tuning/explain-analyze)
