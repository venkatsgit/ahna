## Dev Database Configuration & Scaling Guidance (Telemetry/Early Warning)

### 1. Current Configuration
- **Platform:** Azure Citus (PostgreSQL distributed)
- **Citus Version:** 11.3
- **Cluster:**
  - 1 Coordinator node (4 vCores, 16GB RAM)
  - 2 Worker nodes (each 4 vCores, 32GB RAM)
  - **Production has:** 3 Worker nodes (scalable)
- **Database:** citus
- **Table:** `public.telemetry_sensors_partitioned`

### 2. Usage Pattern
- **High-frequency writes:** Spark cluster inserts new telemetry records continuously (no updates/deletes).
- **Frequent reads:** Real-time/near-real-time queries for early warning and analytics.
- **Hot partition:** Today’s partition receives both heavy writes and reads.

### 3. Partitioning
- **Partitioned by:** `eventdatetime` (1-day partitions)
- **Partition key:** `eventdatetime`
- **Query pattern:** Most queries target today’s partition (recent data).

### 4. Sharding
- **Sharding key:** `sensorid` (Citus distribution column)
- **Shards:** Data is distributed across worker nodes by `sensorid`.
- **Query parallelism:**
  - If querying many `sensorid` values, Citus parallelizes across nodes.
  - If querying a single `sensorid`, only one node is used.

### 5. Indexing
- **Index on:** `sensorid`
- **Recommended composite index:** `(eventdatetime, sensorid)` on each partition for best performance on time+sensor queries.

### 6. Special Situation
- **Same partition (today) is both write-heavy and read-heavy.**
  - This can cause resource contention (CPU, disk, memory) and variable query times.
  - No dead tuple bloat (inserts only), but index and cache pressure can be high.

### 7. Scaling Guidance
- **Current dev:** 2 worker nodes.
- **Scaling out:**
  - Adding a new node distributes new shards to the new node.
  - **To fully utilize new nodes, run Citus rebalance:**
    ```sql
    SELECT rebalance_table_shards();
    ```
  - This spreads existing data and query load across all nodes, improving performance for wide queries.
- **Scaling up:**
  - Increase vCores/RAM per node for more CPU/memory per shard.
- **Monitor:**
  - Use Azure metrics for CPU, memory, disk, and query times.
  - Use `EXPLAIN (ANALYZE)` to check query plans and parallelism.

### 8. Recommendations
- **For hot partitions:**
  - Ensure composite index on `(eventdatetime, sensorid)`.
  - Keep statistics up to date (`ANALYZE`).
  - Monitor resource usage and scale up/out as needed.
- **For scaling out:**
  - Always run rebalance after adding nodes for immediate benefit.
- **For real-time workloads:**
  - Consider reducing query time windows (e.g., 1 hour instead of 2) to reduce load.

### 9. Scaling Options

**Option 1: Scaling Up (Vertical Scaling)**
- Increase CPU and/or RAM on existing worker nodes.
- Best if queries target a single sensorid or if a single node is consistently maxed out.
- Simple to implement, no data movement required.
- Limited by the maximum VM size available in Azure.

**Option 2: Scaling Out (Horizontal Scaling)**
- Add more worker nodes to the Citus cluster.
- Best if queries target many sensorids at once, or if you have many concurrent queries.
- Increases parallelism and throughput, and prepares for future growth.
- After adding a node, run:
  ```sql
  SELECT rebalance_table_shards();
  ```
  to distribute existing data and load evenly.

### 10. Scaling Strategy Recommendation

**For this telemetry/early warning workload, Option 2 (scaling out and rebalancing) is the best approach:**

- The table is sharded by `sensorid`, and most queries target many sensorids at once.
- Scaling out allows Citus to parallelize both reads and writes across more nodes, increasing throughput and reducing query times for analytics and real-time dashboards.
- This approach also prepares your system for future growth and higher concurrency.

**Scaling up (option 1) is helpful if a single node is maxed out, but scaling out is the right approach for your case.**

> **In summary:**
> **Option 2 (scaling out and rebalancing) is the recommended scaling strategy for your workload.**

---

**This setup is optimized for high-ingest, real-time telemetry with early warning analytics. For best results, combine scaling, indexing, and regular monitoring.**

## Current Workload Query

The following query is central to the current workload and is used in two main places:
- **Alert Engine** (for anomaly detection and alerting)
- **Trends Display** (for real-time and historical sensor data visualization)

```sql
SELECT 
    a.asset_name as asset,
    a.asset_id as asset_id,
    s.sensor_name as sensor,
    ts.sensorid as sensor_id,
    CAST(ts.value AS FLOAT) as value,
    ts.eventdatetime as time,
    ts.messagetime as messagetime
FROM (
    SELECT assetid, sensorid, value, eventdatetime, messagetime
    FROM public.telemetry_sensors_partitioned
    WHERE
        eventdatetime BETWEEN '2025-07-21T16:22:00' AND '2025-07-21T17:22:00'
        AND sensorid IN (
            SELECT sensor_id FROM aimldev.sensor_master WHERE sensor_name IN ('Apparent Current L1')
        )
) ts
JOIN aimldev.asset_master a ON ts.assetid = a.asset_id
JOIN aimldev.sensor_master s ON ts.sensorid = s.sensor_id
ORDER BY ts.eventdatetime DESC
LIMIT 1000
```

**This query is optimized to filter by time and sensor, and only then join with asset and sensor master tables.**
