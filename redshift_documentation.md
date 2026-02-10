# Tài liệu Dashboard AWS Redshift

## Tổng quan
Dashboard này được thiết kế để giám sát và theo dõi các metrics của Amazon Redshift - data warehouse service của AWS. Dashboard cung cấp cái nhìn toàn diện về hiệu suất cluster, node-level performance, network throughput, disk I/O, và health status của Redshift cluster.

**Nguồn dữ liệu**: CloudWatch  
**Grafana Dashboard ID**: 8050  
**Khoảng thời gian mặc định**: 24 giờ gần nhất  

---

## Cấu hình

### Biến Template (Template Variables)

Dashboard sử dụng các biến động để cho phép người dùng tùy chỉnh view:

1. **datasource**: Nguồn dữ liệu CloudWatch
2. **region**: Vùng AWS (AWS Region)
3. **clusteridentifier**: Redshift Cluster Identifier cần giám sát
4. **nodeid**: Node ID cụ thể trong cluster để xem node-level metrics
   - Ví dụ: Compute-0, Compute-1, Leader, Shared, etc.

---

## Kiến trúc Redshift

### Redshift Cluster Components

**Leader Node**:
- Nhận queries từ clients
- Parse và tạo query execution plan
- Phân phối workload đến compute nodes
- Aggregate kết quả từ compute nodes

**Compute Nodes**:
- Thực thi queries
- Lưu trữ data
- Xử lý song song (parallel processing)
- Types: Dense Compute (DC2) hoặc Dense Storage (DS2)

**Node Slices**:
- Mỗi compute node chia thành nhiều slices
- Mỗi slice có portion of memory và disk
- Parallel processing tại slice level

---

## Chi tiết các Dashboard Panel

### 1. CPUUtilization / DatabaseConnections

**Vị trí**: Hàng 1, chiều cao 7 đơn vị, chiều rộng 24 đơn vị  
**Loại biểu đồ**: Time series (Line chart - Dual axis)

**Mô tả**:  
Hiển thị CPU utilization của cluster và số lượng database connections đồng thời. Đây là 2 metrics quan trọng nhất để đánh giá cluster load và capacity.

**Metrics**:

#### CPUUtilization
- Namespace: `AWS/Redshift`
- Metric: `CPUUtilization`
- Statistic: `Average`
- Dimension: `ClusterIdentifier`
- Unit: Percent (%)

#### DatabaseConnections
- Namespace: `AWS/Redshift`
- Metric: `DatabaseConnections`
- Statistic: `Average`
- Dimension: `ClusterIdentifier`
- Unit: Count (hiển thị trên trục phải)

**Cấu hình hiển thị**:
- CPUUtilization: Trục trái (%), giá trị min = 0
- DatabaseConnections: Trục phải (count), giá trị min = 0
- Legend: Hiển thị mean, max, min

**Ý nghĩa & Phân tích**:

### CPUUtilization

**CPU Usage Thresholds**:
- **< 50%**: Healthy, cluster có capacity dư
- **50-70%**: Normal load
- **70-85%**: High utilization, cần monitoring
- **85-95%**: Very high, cân nhắc scaling
- **> 95%**: Critical, queries bị slow, cần scale ngay

**Nguyên nhân CPU cao**:
- Complex queries (large joins, aggregations)
- Insufficient VACUUM/ANALYZE
- Inefficient query plans
- Too many concurrent queries
- Cluster undersized

**Optimization Tips**:
- Review WLM (Workload Management) configuration
- Optimize queries (EXPLAIN ANALYZE)
- Scale cluster (resize, add nodes)
- Implement query queuing
- Use materialized views

### DatabaseConnections

**Connection Limits**:
- Default: 500 connections per cluster
- User connections: ~495 (5 reserved for superuser)
- Recommendations: Monitor < 80% of limit

**High Connection Count Issues**:
- Connection pooling không đúng
- Applications không close connections
- Too many concurrent users
- Connection leaks

**Best Practices**:
- Use connection pooling (PgBouncer, application-level)
- Set reasonable connection timeouts
- Monitor connection states (idle, active, idle in transaction)
- Alert khi > 400 connections (80% threshold)

**Correlation Analysis**:
- High CPU + High Connections = Overloaded cluster
- High CPU + Low Connections = Expensive queries
- Low CPU + High Connections = Idle connections or light queries

---

### 2. MaintenanceMode / HealthStatus

**Vị trí**: Hàng 2, chiều cao 7 đơn vị  
**Loại biểu đồ**: Time series (Line chart - Dual axis)

**Mô tả**:  
Giám sát trạng thái maintenance và health của Redshift cluster. Critical metrics để biết khi nào cluster đang trong maintenance window hoặc có vấn đề về health.

**Metrics**:

#### MaintenanceMode
- Namespace: `AWS/Redshift`
- Metric: `MaintenanceMode`
- Statistic: `Average`
- Dimension: `ClusterIdentifier`
- Unit: None (binary: 0 hoặc 1)
- Range: 0-1

#### HealthStatus
- Namespace: `AWS/Redshift`
- Metric: `HealthStatus`
- Statistic: `Average`
- Dimension: `ClusterIdentifier`
- Unit: None (binary: 0 hoặc 1)
- Range: 0-1
- Hiển thị trên trục phải

**Cấu hình hiển thị**:
- Giá trị: 0 (OFF/Unhealthy) hoặc 1 (ON/Healthy)
- Max = 1, Min = 0
- Legend: Hiển thị mean, max, min

**Ý nghĩa & Phân tích**:

### MaintenanceMode

**Values**:
- **1**: Cluster đang trong maintenance mode
- **0**: Cluster đang hoạt động bình thường

**Maintenance Mode Behavior**:
- Cluster unavailable cho queries
- AWS đang apply patches, upgrades
- Thường xảy ra trong maintenance window đã cấu hình
- Duration: Thường 15-60 phút

**Maintenance Window Planning**:
- Cấu hình maintenance window vào off-peak hours
- Default: Weekly, 30-minute window
- Có thể defer maintenance (giới hạn)
- Alert users trước maintenance

**Actions During Maintenance**:
- Applications cần handle connection failures
- Implement retry logic
- Use read replicas nếu có
- Schedule around critical business operations

### HealthStatus

**Values**:
- **1**: HEALTHY - Cluster hoạt động tốt
- **0**: UNHEALTHY - Cluster có vấn đề

**HealthStatus = 0 Causes**:
- Node failures
- Disk failures
- Network issues
- Storage full
- Internal AWS issues

**Troubleshooting HealthStatus Issues**:

1. **Check AWS Service Health Dashboard**
   - Regional outages?
   - Known issues?

2. **Review CloudWatch Logs**
   - Connection errors
   - Query failures
   - Error messages

3. **Check Cluster Events**
   ```bash
   aws redshift describe-events \
     --source-identifier my-cluster \
     --source-type cluster
   ```

4. **Contact AWS Support**
   - If health = 0 for extended period
   - Provide cluster identifier và timeline

**Monitoring Best Practices**:
- Alert immediately khi HealthStatus = 0
- Alert trước maintenance window (notification)
- Track maintenance duration patterns
- Correlate với query performance metrics

---

### 3. NetworkReceiveThroughput / NetworkTransmitThroughput

**Vị trí**: Hàng 3, chiều cao 7 đơn vị  
**Loại biểu đồ**: Time series (Line chart - Dual axis)

**Mô tả**:  
Giám sát network throughput của cluster - data được receive và transmit qua network. Critical để hiểu data transfer patterns và identify bottlenecks.

**Metrics**:

#### NetworkReceiveThroughput
- Namespace: `AWS/Redshift`
- Metric: `NetworkReceiveThroughput`
- Statistic: `Average`
- Dimension: `ClusterIdentifier`
- Unit: Bytes per second (Bps)

#### NetworkTransmitThroughput
- Namespace: `AWS/Redshift`
- Metric: `NetworkTransmitThroughput`
- Statistic: `Average`
- Dimension: `ClusterIdentifier`
- Unit: Bytes per second (Bps)
- Hiển thị trên trục phải

**Cấu hình hiển thị**:
- Unit: Bps (auto-scale to KBps, MBps, GBps)
- Giá trị min = 0
- Legend: Hiển thị mean, max, min

**Ý nghĩa & Phân tích**:

### NetworkReceiveThroughput (Inbound)

**What's Being Received**:
- Client query requests
- COPY commands (data loading)
- Data từ S3 during COPY operations
- Inter-node communication (queries)
- Replication traffic

**High Receive Throughput Scenarios**:
- Large-scale ETL operations
- Bulk data loads (COPY from S3)
- Complex distributed queries
- Many concurrent query submissions

### NetworkTransmitThroughput (Outbound)

**What's Being Transmitted**:
- Query results to clients
- UNLOAD commands (data export)
- Inter-node communication
- Backup/snapshot data
- Logs to CloudWatch

**High Transmit Throughput Scenarios**:
- Large result sets returned to clients
- UNLOAD operations to S3
- Frequent reporting queries
- Data extracts và exports

**Network Capacity by Node Type**:

| Node Type | Network Performance |
|-----------|-------------------|
| dc2.large | Up to 10 Gbps |
| dc2.8xlarge | 10 Gbps |
| ra3.xlplus | Up to 10 Gbps |
| ra3.4xlarge | Up to 10 Gbps |
| ra3.16xlarge | 25 Gbps |

**Performance Analysis**:

**Balanced Traffic**:
- Receive ≈ Transmit: Good sign
- Data processing và retrieval cân bằng

**High Receive, Low Transmit**:
- Data loading operations
- Aggregation queries (return small results)
- ETL workloads

**Low Receive, High Transmit**:
- Large result sets
- UNLOAD operations
- Reporting queries
- Possibly inefficient queries (returning too much data)

**Optimization Tips**:

1. **For High Network Usage**:
   - Use COPY with multiple files (parallel loading)
   - Compress data before COPY/UNLOAD
   - Use result set caching
   - Implement pagination for large results
   - Use Redshift Spectrum for external data

2. **For Network Bottlenecks**:
   - Upgrade to larger node types (better network)
   - Use Enhanced VPC Routing for better S3 access
   - Optimize queries to return less data
   - Use materialized views

3. **Monitor Patterns**:
   - Peak hours identification
   - ETL window timing
   - Query result size trends

**Alerts**:
- Network throughput approaching node capacity
- Unusual spikes (possible data exfiltration)
- Sustained high throughput (capacity planning)

---

### 4. ReadIOPS / WriteIOPS - NodeId

**Vị trí**: Hàng 4, chiều cao 7 đơn vị  
**Loại biểu đồ**: Time series (Line chart - Dual axis)

**Mô tả**:  
Giám sát disk I/O operations per second tại node level. Critical để hiểu disk performance và identify storage bottlenecks trên từng node cụ thể.

**Metrics**:

#### ReadIOPS
- Namespace: `AWS/Redshift`
- Metric: `ReadIOPS`
- Statistic: `Average`
- Dimensions: `ClusterIdentifier`, `NodeID`
- Unit: Operations per second (rps)

#### WriteIOPS
- Namespace: `AWS/Redshift`
- Metric: `WriteIOPS`
- Statistic: `Average`
- Dimensions: `ClusterIdentifier`, `NodeID`
- Unit: Operations per second (wps)
- Hiển thị trên trục phải

**Cấu hình hiển thị**:
- ReadIOPS: rps (reads per second)
- WriteIOPS: wps (writes per second)
- Node-specific: Filtered by $nodeid variable
- Giá trị min = 0
- Legend: Hiển thị mean, max, min

**Ý nghĩa & Phân tích**:

### ReadIOPS

**What Triggers Read Operations**:
- SELECT queries (table scans)
- JOIN operations
- Aggregations
- Sorting operations
- Loading data to memory
- Query compilation

**Normal Read Patterns**:
- Analytical queries: High ReadIOPS
- OLAP workloads: Burst patterns
- Column-store reads: Efficient, lower IOPS

**High ReadIOPS Scenarios**:
- Full table scans (missing dist/sort keys)
- Complex queries với multiple joins
- Cold cache (data not in memory)
- Inefficient queries
- Large result sets processing

### WriteIOPS

**What Triggers Write Operations**:
- INSERT/UPDATE/DELETE operations
- COPY commands
- Temporary tables creation
- Intermediate query results
- VACUUM operations
- Materialized view refreshes

**Normal Write Patterns**:
- ETL windows: High WriteIOPS
- Real-time ingestion: Steady writes
- VACUUM: Periodic burst writes

**High WriteIOPS Scenarios**:
- Large-scale data loads
- Frequent small writes (anti-pattern)
- VACUUM operations
- Heavy DML workload
- Materialized view maintenance

**IOPS Capacity by Node Type**:

| Node Type | Storage | Max IOPS |
|-----------|---------|----------|
| dc2.large | 160 GB NVMe SSD | ~20,000 |
| dc2.8xlarge | 2.56 TB NVMe SSD | ~100,000 |
| ra3.xlplus | Managed Storage | Variable (high) |
| ra3.4xlarge | Managed Storage | Variable (high) |
| ra3.16xlarge | Managed Storage | Variable (high) |

**Performance Analysis**:

**Read vs Write Ratio**:
- **High Read, Low Write**: Normal analytical workload
- **High Write, Low Read**: ETL/loading operations
- **Both High**: Heavy mixed workload, potential bottleneck
- **Both Low**: Idle cluster hoặc cached queries

**Node-Level Analysis**:
- Compare IOPS across nodes
- Identify hotspots (unbalanced dist keys)
- Detect failing nodes (abnormally low IOPS)

**Optimization Strategies**:

### For High ReadIOPS:

1. **Optimize Distribution Keys**:
   ```sql
   -- Check table distribution
   SELECT * FROM svv_table_info WHERE "table" = 'your_table';
   
   -- Redistribute nếu cần
   ALTER TABLE your_table ALTER DISTSTYLE KEY DISTKEY(customer_id);
   ```

2. **Optimize Sort Keys**:
   ```sql
   -- Add sort key để reduce scan
   ALTER TABLE your_table ALTER SORTKEY(date_column);
   ```

3. **Use Result Caching**:
   - Enable result caching
   - Reuse query results (automatic)

4. **Implement Materialized Views**:
   ```sql
   CREATE MATERIALIZED VIEW mv_sales_summary AS
   SELECT date, product, SUM(amount) as total
   FROM sales
   GROUP BY date, product;
   ```

### For High WriteIOPS:

1. **Batch Writes**:
   - Use COPY instead of individual INSERTs
   - Batch size: 100MB - 1GB per file optimal

2. **Optimize VACUUM**:
   ```sql
   -- Schedule during off-peak
   VACUUM FULL your_table;
   
   -- Or automatic vacuum
   ANALYZE COMPRESSION your_table;
   ```

3. **Reduce Unnecessary Writes**:
   - Review update patterns
   - Use INSERT instead of UPDATE where possible
   - Consider append-only design

4. **Distribution Strategy**:
   - EVEN distribution cho write-heavy tables
   - Avoid ALL distribution for large tables

**Monitoring & Alerts**:
- Alert khi IOPS > 80% of node capacity
- Monitor sustained high IOPS (> 1 hour)
- Track IOPS during ETL windows
- Compare across nodes (identify imbalance)

**Troubleshooting High IOPS**:

1. **Identify Expensive Queries**:
   ```sql
   SELECT query, substring(querytxt,1,100), elapsed, rows
   FROM stl_query
   WHERE userid > 1
   ORDER BY elapsed DESC
   LIMIT 10;
   ```

2. **Check Table Statistics**:
   ```sql
   SELECT * FROM svv_table_info
   WHERE unsorted > 10 OR stats_off > 5;
   ```

3. **Review Workload**:
   ```sql
   SELECT service_class, num_queued_queries, num_executing_queries
   FROM stv_wlm_service_class_state;
   ```

---

### 5. ReadLatency / WriteLatency - NodeId

**Vị trí**: Hàng 5, chiều cao 7 đơn vị  
**Loại biểu đồ**: Time series (Line chart - Dual axis)

**Mô tả**:  
Giám sát disk I/O latency tại node level - thời gian trung bình để hoàn thành disk operations. Critical indicator của storage performance và query execution speed.

**Metrics**:

#### ReadLatency
- Namespace: `AWS/Redshift`
- Metric: `ReadLatency`
- Statistic: `Average`
- Dimensions: `ClusterIdentifier`, `NodeID`
- Unit: Seconds (s)

#### WriteLatency
- Namespace: `AWS/Redshift`
- Metric: `WriteLatency`
- Statistic: `Average`
- Dimensions: `ClusterIdentifier`, `NodeID`
- Unit: Seconds (s)
- Hiển thị trên trục phải

**Cấu hình hiển thị**:
- Unit: Seconds (tự động convert to ms nếu < 1s)
- Node-specific: Filtered by $nodeid variable
- Giá trị min = 0
- Legend: Hiển thị mean, max, min

**Ý nghĩa & Phân tích**:

### ReadLatency

**What It Measures**:
- Average time để complete một read operation
- End-to-end disk read performance
- Includes queue time + actual read time

**Latency Thresholds**:
- **< 0.001s (1ms)**: Excellent (NVMe SSD)
- **0.001-0.005s (1-5ms)**: Good (SSD)
- **0.005-0.020s (5-20ms)**: Acceptable
- **> 0.020s (20ms)**: Poor, needs investigation

**High Read Latency Causes**:
- Disk contention (high concurrent reads)
- Storage capacity issues
- Hardware degradation
- Large sequential scans
- Network-attached storage delays (RA3)

### WriteLatency

**What It Measures**:
- Average time để complete một write operation
- Includes buffering và flushing to disk
- VACUUM operations significantly impact

**Latency Thresholds**:
- **< 0.002s (2ms)**: Excellent
- **0.002-0.010s (2-10ms)**: Good
- **0.010-0.030s (10-30ms)**: Acceptable
- **> 0.030s (30ms)**: Poor, needs investigation

**High Write Latency Causes**:
- VACUUM operations running
- High concurrent writes
- Large batch inserts
- Storage nearing capacity
- Table bloat (needs VACUUM)

**Performance Benchmarks by Node Type**:

| Node Type | Storage Type | Typical Read Latency | Typical Write Latency |
|-----------|--------------|---------------------|----------------------|
| dc2.large | Local NVMe SSD | 0.5-2 ms | 1-3 ms |
| dc2.8xlarge | Local NVMe SSD | 0.5-2 ms | 1-3 ms |
| ra3.xlplus | Managed Storage | 2-5 ms | 3-8 ms |
| ra3.4xlarge | Managed Storage | 2-5 ms | 3-8 ms |
| ra3.16xlarge | Managed Storage | 2-5 ms | 3-8 ms |

**Performance Analysis**:

**Latency Patterns**:

1. **Spikes During VACUUM**:
   - Normal behavior
   - Schedule VACUUM during off-peak
   - Consider VACUUM BOOST for faster completion

2. **Gradual Increase**:
   - Table bloat accumulating
   - Needs regular VACUUM
   - Consider automatic VACUUM settings

3. **Sustained High Latency**:
   - Storage capacity issues
   - Hardware problems
   - Need to scale cluster

4. **Node-Specific High Latency**:
   - Unbalanced data distribution
   - Hardware issue on specific node
   - Hot partition/slice

**Correlation with Other Metrics**:
- High Latency + High IOPS = Storage bottleneck
- High Latency + Low IOPS = Hardware issue hoặc large operations
- High Latency + High CPU = Query complexity issue

**Optimization Strategies**:

### Reduce Read Latency:

1. **Optimize Queries**:
   ```sql
   -- Use EXPLAIN để analyze query plan
   EXPLAIN SELECT * FROM large_table WHERE date > '2024-01-01';
   
   -- Add appropriate filters
   -- Use columnar storage advantage
   SELECT specific_columns FROM table;  -- Better than SELECT *
   ```

2. **Improve Data Layout**:
   ```sql
   -- Set appropriate sort keys
   ALTER TABLE events 
   ALTER SORTKEY (event_date);
   
   -- Compress columns
   ANALYZE COMPRESSION events;
   ```

3. **Use Zone Maps**:
   - Automatic trong Redshift
   - Benefits from sorted data
   - Helps skip data blocks

### Reduce Write Latency:

1. **Optimize VACUUM Operations**:
   ```sql
   -- Full vacuum during maintenance window
   VACUUM FULL table_name;
   
   -- Or incremental vacuum
   VACUUM table_name BOOST;
   
   -- Check vacuum progress
   SELECT * FROM svv_vacuum_progress;
   ```

2. **Batch Writes**:
   ```sql
   -- Use COPY for bulk loads
   COPY table_name FROM 's3://bucket/data/'
   IAM_ROLE 'arn:aws:iam::account:role/RedshiftRole'
   FORMAT AS PARQUET;
   ```

3. **Table Maintenance**:
   ```sql
   -- Regular ANALYZE
   ANALYZE table_name;
   
   -- Check table health
   SELECT * FROM svv_table_info 
   WHERE "table" = 'your_table';
   ```

4. **Monitor Storage Usage**:
   ```sql
   -- Check disk usage
   SELECT node, SUM(used) as used_mb, 
          SUM(capacity) as capacity_mb,
          (SUM(used) * 100.0 / SUM(capacity)) as pct_used
   FROM stv_partitions
   GROUP BY node
   ORDER BY node;
   ```

**Monitoring & Alerts**:

**Critical Alerts**:
- ReadLatency > 20ms for > 5 minutes
- WriteLatency > 30ms for > 5 minutes
- Latency spike > 3x baseline

**Warning Alerts**:
- ReadLatency > 10ms sustained
- WriteLatency > 15ms sustained
- Latency trending upward (week-over-week)

**Troubleshooting Steps**:

1. **Check Running Queries**:
   ```sql
   -- Active queries
   SELECT pid, user_name, starttime, query
   FROM stv_recents
   WHERE status = 'Running'
   ORDER BY starttime;
   ```

2. **Check VACUUM Operations**:
   ```sql
   -- Current VACUUM
   SELECT * FROM svv_vacuum_progress;
   
   -- VACUUM history
   SELECT * FROM stl_vacuum
   WHERE endtime > DATEADD(hour, -24, GETDATE())
   ORDER BY endtime DESC;
   ```

3. **Check Disk Space**:
   ```sql
   -- Per-node disk usage
   SELECT node, used, capacity,
          ROUND((used::float / capacity) * 100, 2) as pct_used
   FROM stv_partitions
   WHERE capacity > 0
   GROUP BY node, used, capacity
   ORDER BY pct_used DESC;
   ```

4. **Review Workload Management**:
   ```sql
   -- WLM queue stats
   SELECT service_class, 
          num_queued_queries,
          num_executing_queries,
          avg_execution_time
   FROM stv_wlm_service_class_state;
   ```

**Best Practices**:
- Regular VACUUM (weekly for active tables)
- Monitor disk usage (< 80% capacity)
- Schedule heavy operations during off-peak
- Use appropriate node types for workload
- Regular ANALYZE for statistics

---

### 6. ReadThroughput / WriteThroughput - NodeId

**Vị trí**: Hàng 6, chiều cao 7 đơn vị  
**Loại biểu đồ**: Time series (Line chart - Dual axis)

**Mô tả**:  
Giám sát disk I/O throughput tại node level - lượng data được read/write per second. Complementary với IOPS metrics để có complete picture của disk performance.

**Metrics**:

#### ReadThroughput
- Namespace: `AWS/Redshift`
- Metric: `ReadThroughput`
- Statistic: `Average`
- Dimensions: `ClusterIdentifier`, `NodeID`
- Unit: Bytes per second (Bps)

#### WriteThroughput
- Namespace: `AWS/Redshift`
- Metric: `WriteThroughput`
- Statistic: `Average`
- Dimensions: `ClusterIdentifier`, `NodeID`
- Unit: Bytes (display unit differs: Bps vs bytes)
- Hiển thị trên trục phải

**Cấu hình hiển thị**:
- ReadThroughput: Bps (auto-scale to KBps, MBps, GBps)
- WriteThroughput: bytes (note: có thể là configuration inconsistency)
- Node-specific: Filtered by $nodeid variable
- Giá trị min = 0
- Legend: Hiển thị mean, max, min

**Ý nghĩa & Phân tích**:

### ReadThroughput

**What It Measures**:
- Volume of data read from disk per second
- Aggregate throughput for all read operations
- Indicator of query data scanning volume

**What Drives High Read Throughput**:
- Large table scans (SELECT * queries)
- Complex analytical queries
- JOIN operations on large tables
- Aggregations over many rows
- UNLOAD operations reading data
- Columnar compression (lower throughput, more data)

**Throughput Capacity by Node Type**:

| Node Type | Storage | Read Throughput |
|-----------|---------|-----------------|
| dc2.large | 160 GB NVMe | ~400 MBps |
| dc2.8xlarge | 2.56 TB NVMe | ~3,500 MBps |
| ra3.xlplus | Managed Storage | Variable (high) |
| ra3.4xlarge | Managed Storage | Variable (high) |
| ra3.16xlarge | Managed Storage | Very high |

### WriteThroughput

**What It Measures**:
- Volume of data written to disk per second
- Includes all write operations
- Critical for ETL performance

**What Drives High Write Throughput**:
- COPY commands (bulk loading)
- INSERT operations
- UPDATE/DELETE operations
- Temporary table creation
- VACUUM operations (rewriting blocks)
- Materialized view refreshes

**Throughput Capacity by Node Type**:

| Node Type | Storage | Write Throughput |
|-----------|---------|------------------|
| dc2.large | 160 GB NVMe | ~400 MBps |
| dc2.8xlarge | 2.56 TB NVMe | ~3,500 MBps |
| ra3.xlplus | Managed Storage | Variable (high) |
| ra3.4xlarge | Managed Storage | Variable (high) |
| ra3.16xlarge | Managed Storage | Very high |

**Performance Analysis**:

### Throughput vs IOPS Relationship:

**High Throughput + Low IOPS**:
- Large sequential reads/writes
- Efficient I/O pattern
- Good performance
- Example: COPY large files, full table scans

**Low Throughput + High IOPS**:
- Small random reads/writes
- Inefficient I/O pattern
- Potential performance issue
- Example: Many small transactions, row-by-row processing

**High Throughput + High IOPS**:
- Heavy workload
- Mix of large and small operations
- Approaching capacity limits
- Need to monitor for bottlenecks

**Throughput Patterns**:

1. **ETL Window Pattern**:
   - High WriteThroughput during loading
   - Predictable, scheduled spikes
   - Normal behavior

2. **Query-Heavy Pattern**:
   - High ReadThroughput throughout day
   - Lower WriteThroughput
   - Typical analytical workload

3. **Real-Time Ingestion**:
   - Steady WriteThroughput
   - Moderate ReadThroughput
   - Continuous data flow

**Optimization Strategies**:

### Optimize Read Throughput:

1. **Compression**:
   ```sql
   -- Check compression encodings
   SELECT "column", type, encoding
   FROM pg_table_def
   WHERE tablename = 'your_table';
   
   -- Apply optimal compression
   ANALYZE COMPRESSION your_table;
   ALTER TABLE your_table 
   ALTER COLUMN col1 ENCODE LZO;
   ```

2. **Column Selection**:
   ```sql
   -- BAD: Reads all columns
   SELECT * FROM large_table;
   
   -- GOOD: Reads only needed columns
   SELECT id, name, date FROM large_table;
   ```

3. **Predicate Pushdown**:
   ```sql
   -- Use WHERE clauses effectively
   SELECT * FROM sales 
   WHERE date >= '2024-01-01'
   AND region = 'APAC';
   ```

4. **Zone Maps**:
   - Automatic với sorted data
   - Helps skip blocks
   - Reduces data scanned

### Optimize Write Throughput:

1. **Bulk Loading**:
   ```sql
   -- Use COPY for best performance
   COPY table_name FROM 's3://bucket/prefix/'
   IAM_ROLE 'role_arn'
   FORMAT AS PARQUET
   COMPUPDATE ON
   STATUPDATE ON;
   ```

2. **Multiple Files**:
   ```sql
   -- Split data into multiple files (parallel loading)
   -- Optimal: 1 file per slice
   -- File size: 100 MB - 1 GB each
   
   COPY table_name FROM 's3://bucket/prefix/'
   IAM_ROLE 'role_arn'
   MANIFEST;
   ```

3. **Compression**:
   ```sql
   -- Load compressed data
   COPY table_name FROM 's3://bucket/data.gz'
   IAM_ROLE 'role_arn'
   GZIP;
   ```

4. **Batch vs Streaming**:
   ```sql
   -- Batch loading (faster)
   -- Load every hour with 1000 rows per file
   
   -- Instead of:
   -- INSERT single rows continuously
   ```

**Monitoring & Capacity Planning**:

### Calculate Storage Bandwidth Utilization:

```
Utilization % = (Current Throughput / Max Capacity) * 100
```

**Example**:
- Node: dc2.8xlarge
- Current ReadThroughput: 2,800 MBps
- Max Capacity: 3,500 MBps
- Utilization: 80% → Approaching limit

### Throughput Alerts:

**Critical**:
- Throughput > 90% of node capacity
- Sustained high throughput > 2 hours
- Throttling detected

**Warning**:
- Throughput > 70% of capacity
- Increasing trend week-over-week
- Unbalanced across nodes

### Troubleshooting High Throughput:

1. **Identify Heavy Queries**:
   ```sql
   -- Queries with most I/O
   SELECT query, segment, 
          SUM(blocks_read) as total_blocks,
          SUM(bytes) as total_bytes
   FROM svl_query_summary
   WHERE query > 0
   GROUP BY query, segment
   ORDER BY total_bytes DESC
   LIMIT 20;
   ```

2. **Check Data Distribution**:
   ```sql
   -- Node-level data skew
   SELECT node, SUM(num_values) as rows
   FROM stv_blocklist
   WHERE tbl = (SELECT oid FROM pg_class WHERE relname = 'your_table')
   GROUP BY node
   ORDER BY rows DESC;
   ```

3. **Review COPY Operations**:
   ```sql
   -- Recent COPY performance
   SELECT query, filename, curtime, 
          rows, bytes, duration
   FROM stl_load_commits
   WHERE curtime > DATEADD(day, -1, GETDATE())
   ORDER BY curtime DESC;
   ```

4. **Monitor Concurrent Operations**:
   ```sql
   -- Current disk I/O
   SELECT slice, col, 
          SUM(blocks_read) as read_blocks,
          SUM(blocks_written) as write_blocks
   FROM svl_query_summary
   WHERE query IN (
     SELECT query FROM stv_recents WHERE status = 'Running'
   )
   GROUP BY slice, col
   ORDER BY read_blocks + write_blocks DESC;
   ```

**Best Practices**:

### Data Loading:
- Use COPY instead of INSERT
- Split into multiple files (parallel loading)
- Compress before loading
- Use appropriate file formats (Parquet, ORC)
- Monitor COPY performance regularly

### Query Optimization:
- Select only needed columns
- Use WHERE clauses effectively
- Leverage sort keys
- Use distribution keys properly
- Avoid SELECT * queries

### Capacity Planning:
- Monitor peak throughput periods
- Compare with node capacity
- Plan for growth (seasonal spikes)
- Consider node type upgrades
- Use RA3 for scalable storage

### Node Balancing:
- Check throughput across all nodes
- Identify hot nodes
- Review distribution keys
- Rebalance if needed

---

## Hướng dẫn sử dụng

### 1. Chọn Redshift Cluster
- Sử dụng dropdown **ClusterIdentifier** để chọn cluster cần monitor
- Dashboard sẽ hiển thị cluster-level metrics

### 2. Chọn AWS Region
- Sử dụng dropdown **Region** để chọn AWS Region
- Chỉ hiển thị clusters trong region được chọn

### 3. Chọn Node để phân tích chi tiết
- Sử dụng dropdown **NodeID** để chọn node cụ thể
- Các panels 4-6 sẽ hiển thị node-level metrics
- Node types:
  - **Compute-0, Compute-1, etc.**: Compute nodes
  - **Leader**: Leader node (nếu có metrics)
  - **Shared**: Shared metrics

### 4. Điều chỉnh khoảng thời gian
- Time picker: Góc trên bên phải
- Mặc định: 24 giờ gần nhất
- Recommended ranges:
  - **Real-time monitoring**: Last 1h - Last 6h
  - **Daily analysis**: Last 24h
  - **Trend analysis**: Last 7d - Last 30d

### 5. Refresh Dashboard
- Manual refresh (dashboard không auto-refresh)
- Click nút refresh hoặc F5

---

## Yêu cầu cấu hình

### CloudWatch Metrics

Redshift tự động gửi metrics đến CloudWatch (không cần configuration).

**Cluster-Level Metrics**:
- CPUUtilization
- DatabaseConnections
- HealthStatus
- MaintenanceMode
- NetworkReceiveThroughput
- NetworkTransmitThroughput

**Node-Level Metrics**:
- ReadIOPS, WriteIOPS
- ReadLatency, WriteLatency
- ReadThroughput, WriteThroughput
- PercentageDiskSpaceUsed
- CPUUtilization (per node)

**Metric Frequency**:
- 1-minute intervals
- Near real-time (2-3 minute delay)

### IAM Permissions

User/Role chạy Grafana cần permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:GetMetricData",
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:ListMetrics",
        "redshift:DescribeClusters",
        "redshift:DescribeClusterParameters",
        "redshift:DescribeEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

### Additional Monitoring

**Enable Enhanced VPC Routing**:
- Better network performance monitoring
- More detailed metrics

**Enable Audit Logging**:
```sql
-- Enable audit logging
ALTER SYSTEM SET enable_user_activity_logging TO true;
```

**Configure CloudWatch Logs**:
- Connection logs
- User activity logs
- User logs

---

## Best Practices

### 1. Performance Monitoring

**Daily Checks**:
- CPUUtilization: Should be < 85%
- DatabaseConnections: Should be < 400 (80% of limit)
- HealthStatus: Always = 1
- Disk space: < 80% used

**Weekly Reviews**:
- Performance trends
- Query performance degradation
- VACUUM/ANALYZE schedules
- Storage growth

**Monthly Analysis**:
- Capacity planning
- Cost optimization opportunities
- Workload patterns
- Scaling needs

### 2. Query Optimization

**Identify Slow Queries**:
```sql
-- Top 10 longest running queries
SELECT query, TRIM(querytxt) as SQL, 
       starttime, endtime, 
       DATEDIFF(seconds, starttime, endtime) as duration_sec
FROM stl_query
WHERE userid > 1
AND starttime > DATEADD(day, -7, GETDATE())
ORDER BY duration_sec DESC
LIMIT 10;
```

**Query Monitoring Rules (QMR)**:
```sql
-- Set up query monitoring rules
-- In parameter group:
-- query_execution_time: 300000 (5 minutes)
-- max_execution_time: 3600000 (1 hour)
```

**Workload Management (WLM)**:
```json
[
  {
    "query_group": "etl",
    "memory_percent_to_use": 40,
    "max_concurrency": 5
  },
  {
    "query_group": "reporting",
    "memory_percent_to_use": 30,
    "max_concurrency": 10
  },
  {
    "query_group": "adhoc",
    "memory_percent_to_use": 30,
    "max_concurrency": 5
  }
]
```

### 3. Maintenance Best Practices

**VACUUM Schedule**:
```sql
-- Weekly full vacuum for critical tables
VACUUM FULL critical_table;

-- Daily sort-only vacuum
VACUUM SORT ONLY active_table;

-- Check vacuum needs
SELECT "table", unsorted, vacuum_sort_benefit
FROM svv_table_info
WHERE unsorted > 5 OR vacuum_sort_benefit > 50
ORDER BY vacuum_sort_benefit DESC;
```

**ANALYZE Schedule**:
```sql
-- After significant data changes
ANALYZE table_name;

-- Regularly for all tables
ANALYZE;

-- Check statistics
SELECT "table", stats_off
FROM svv_table_info
WHERE stats_off > 10
ORDER BY stats_off DESC;
```

### 4. Capacity Planning

**Storage Growth Monitoring**:
```sql
-- Database size trend
SELECT TRUNC(recordtime) as date,
       AVG(size_mb) as avg_size_mb
FROM (
  SELECT recordtime,
         SUM(used) / 1024.0 as size_mb
  FROM stv_partitions
  WHERE recordtime > DATEADD(month, -3, GETDATE())
  GROUP BY recordtime
) s
GROUP BY TRUNC(recordtime)
ORDER BY date;
```

**Connection Trending**:
- Track peak connection times
- Plan for growth
- Implement connection pooling

**CPU Trending**:
- Monitor CPU patterns
- Identify growing workloads
- Plan scaling before hitting limits

### 5. Cost Optimization

**Reserved Instance Planning**:
- Monitor steady-state usage
- Consider Reserved Instances for predictable workload
- 1-year vs 3-year commitments

**Pause/Resume**:
```bash
# Pause cluster when not needed
aws redshift pause-cluster --cluster-identifier my-cluster

# Resume when needed
aws redshift resume-cluster --cluster-identifier my-cluster
```

**Right-Sizing**:
- Review actual CPU/Memory usage
- Downsize if consistently under-utilized
- Upgrade if consistently over 85% CPU

**Concurrency Scaling**:
- Enable for burst workloads
- Free credits: 1 hour per day
- Monitor usage to optimize costs

---

## Troubleshooting Common Issues

### Issue 1: High CPU Utilization

**Triệu chứng**: CPUUtilization > 85% sustained

**Diagnosis**:
```sql
-- Find CPU-intensive queries
SELECT query, TRIM(querytxt) as SQL, 
       starttime, endtime,
       DATEDIFF(seconds, starttime, endtime) as duration
FROM stl_query
WHERE userid > 1
AND starttime > DATEADD(hour, -2, GETDATE())
ORDER BY duration DESC
LIMIT 20;
```

**Solutions**:
1. Optimize slow queries (use EXPLAIN)
2. Implement WLM queues
3. Scale cluster (add nodes)
4. Use concurrency scaling
5. Schedule heavy queries during off-peak

### Issue 2: Too Many Connections

**Triệu chứng**: DatabaseConnections approaching 500

**Diagnosis**:
```sql
-- Check connection distribution
SELECT user_name, 
       COUNT(*) as connection_count,
       SUM(CASE WHEN state = 'idle' THEN 1 ELSE 0 END) as idle_count
FROM pg_stat_activity
GROUP BY user_name
ORDER BY connection_count DESC;
```

**Solutions**:
1. Implement connection pooling (PgBouncer)
2. Close idle connections
3. Review application connection management
4. Set statement_timeout
```sql
SET statement_timeout = 600000; -- 10 minutes
```

### Issue 3: Cluster Unhealthy

**Triệu chứng**: HealthStatus = 0

**Diagnosis**:
```bash
# Check cluster events
aws redshift describe-events \
  --source-identifier my-cluster \
  --source-type cluster \
  --start-time 2024-01-01T00:00:00Z
```

**Solutions**:
1. Check AWS Service Health Dashboard
2. Review CloudWatch Logs
3. Check disk space usage
4. Contact AWS Support if persistent

### Issue 4: High Disk Latency

**Triệu chứng**: ReadLatency or WriteLatency > 20ms

**Diagnosis**:
```sql
-- Check disk usage
SELECT node,
       used/1024 as used_gb,
       capacity/1024 as capacity_gb,
       ROUND((used::float/capacity)*100, 2) as pct_used
FROM stv_partitions
WHERE capacity > 0
GROUP BY node, used, capacity
ORDER BY pct_used DESC;

-- Check if VACUUM needed
SELECT "table", unsorted, vacuum_sort_benefit
FROM svv_table_info
WHERE unsorted > 10
ORDER BY vacuum_sort_benefit DESC;
```

**Solutions**:
1. Run VACUUM on tables with high unsorted %
2. Delete old data / implement lifecycle
3. Check for disk failures (hardware)
4. Scale storage (move to RA3)

### Issue 5: Poor Query Performance

**Triệu chứng**: Queries running slower than expected

**Diagnosis**:
```sql
-- Query execution breakdown
SELECT query, segment, step, 
       rows, bytes, 
       SUM(DATEDIFF(ms, starttime, endtime)) as elapsed_ms
FROM svl_query_summary
WHERE query = your_query_id
GROUP BY query, segment, step, rows, bytes
ORDER BY segment, step;

-- Check for data skew
SELECT slice, COUNT(*) as row_count
FROM your_table
GROUP BY slice
ORDER BY row_count DESC;
```

**Solutions**:
1. Review and optimize distribution keys
2. Add or modify sort keys
3. Update table statistics (ANALYZE)
4. Optimize join order
5. Consider materialized views

### Issue 6: High Network Throughput

**Triệu chứng**: NetworkReceive or NetworkTransmit muito alto

**Diagnosis**:
```sql
-- Check for large result sets
SELECT query, rows, bytes,
       TRIM(querytxt) as SQL
FROM stl_query
WHERE rows > 1000000
AND starttime > DATEADD(hour, -6, GETDATE())
ORDER BY bytes DESC;
```

**Solutions**:
1. Implement pagination for large results
2. Use UNLOAD to S3 instead of returning large datasets
3. Add filters to reduce result set size
4. Use result caching
5. Review application data retrieval patterns

---

## Advanced Monitoring

### Custom Metrics & Calculations

**CPU Per Node**:
```sql
-- Requires node-level CPU metrics
SELECT node, AVG(cpu_utilization) as avg_cpu
FROM node_metrics
GROUP BY node;
```

**Query Throughput**:
```
Queries Per Second = Total Queries / Time Period (seconds)
```

**Storage Efficiency**:
```
Compression Ratio = Uncompressed Size / Compressed Size
```

**Cache Hit Ratio**:
```sql
SELECT 
  SUM(blocks_read) as total_reads,
  SUM(blocks_cached) as cache_hits,
  ROUND(SUM(blocks_cached) * 100.0 / NULLIF(SUM(blocks_read), 0), 2) as cache_hit_pct
FROM svl_query_summary
WHERE starttime > DATEADD(day, -1, GETDATE());
```

### System Tables for Deep Dive

**Query Performance**:
- `stl_query`: All queries
- `stl_wlm_query`: WLM queue assignment
- `svl_query_summary`: Query execution details
- `svl_qlog`: Query logging

**Table Information**:
- `svv_table_info`: Comprehensive table statistics
- `pg_table_def`: Table definitions
- `stv_blocklist`: Block-level storage info
- `stv_tbl_perm`: Table permissions

**Disk & I/O**:
- `stv_partitions`: Disk usage by node
- `svl_query_summary`: I/O for queries
- `stv_slices`: Slice configuration

**Connections & Sessions**:
- `pg_stat_activity`: Current connections
- `stl_connection_log`: Connection history
- `stl_sessions`: Session information

### Grafana Advanced Queries

**CPU Utilization Trend**:
```
Average CPU over 7 days with std deviation
```

**Connection Pool Efficiency**:
```
(Active Connections / Total Connections) * 100
```

**Disk I/O Pressure**:
```
(Current IOPS / Max IOPS) * 100
```

---

## Integration với AWS Services

### CloudWatch Alarms

**Setup Alarms**:
```bash
# High CPU alarm
aws cloudwatch put-metric-alarm \
  --alarm-name redshift-high-cpu \
  --alarm-description "CPU > 85% for 10 minutes" \
  --namespace AWS/Redshift \
  --metric-name CPUUtilization \
  --dimensions Name=ClusterIdentifier,Value=my-cluster \
  --statistic Average \
  --period 300 \
  --threshold 85 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2
```

### AWS Lambda for Automation

**Auto-VACUUM**:
```python
import boto3

def lambda_handler(event, context):
    client = boto3.client('redshift-data')
    response = client.execute_statement(
        ClusterIdentifier='my-cluster',
        Database='mydb',
        DbUser='admin',
        Sql='VACUUM;'
    )
    return response
```

### SNS Notifications

**Alert on Health Issues**:
```bash
aws sns create-topic --name redshift-alerts

aws cloudwatch put-metric-alarm \
  --alarm-name redshift-unhealthy \
  --alarm-actions arn:aws:sns:region:account:redshift-alerts \
  --namespace AWS/Redshift \
  --metric-name HealthStatus \
  --dimensions Name=ClusterIdentifier,Value=my-cluster \
  --statistic Average \
  --period 60 \
  --threshold 0.5 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 1
```

---

## Metrics Reference

| Metric | Level | Unit | Statistic | Description |
|--------|-------|------|-----------|-------------|
| CPUUtilization | Cluster | Percent | Average | CPU usage across cluster |
| DatabaseConnections | Cluster | Count | Average | Number of active connections |
| HealthStatus | Cluster | Binary | Average | Cluster health (1=healthy, 0=unhealthy) |
| MaintenanceMode | Cluster | Binary | Average | Maintenance status (1=in maintenance, 0=normal) |
| NetworkReceiveThroughput | Cluster | Bytes/sec | Average | Inbound network traffic |
| NetworkTransmitThroughput | Cluster | Bytes/sec | Average | Outbound network traffic |
| ReadIOPS | Node | Count/sec | Average | Read operations per second |
| WriteIOPS | Node | Count/sec | Average | Write operations per second |
| ReadLatency | Node | Seconds | Average | Average read operation latency |
| WriteLatency | Node | Seconds | Average | Average write operation latency |
| ReadThroughput | Node | Bytes/sec | Average | Data read throughput |
| WriteThroughput | Node | Bytes/sec | Average | Data write throughput |
| PercentageDiskSpaceUsed | Node | Percent | Average | Disk usage percentage (not in dashboard but available) |

---

## Redshift-Specific Optimization

### Distribution Keys

**Choose Distribution Key**:
```sql
-- For dimension tables (< 100K rows)
ALTER TABLE small_table ALTER DISTSTYLE ALL;

-- For fact tables
ALTER TABLE large_table ALTER DISTSTYLE KEY DISTKEY(customer_id);

-- For staging tables
ALTER TABLE staging_table ALTER DISTSTYLE EVEN;
```

**Check Distribution**:
```sql
SELECT "table", diststyle
FROM svv_table_info
ORDER BY "table";
```

### Sort Keys

**Single Column Sort Key**:
```sql
ALTER TABLE events ALTER SORTKEY(event_date);
```

**Compound Sort Key** (most common):
```sql
ALTER TABLE sales ALTER COMPOUND SORTKEY(date, region, product_id);
```

**Interleaved Sort Key** (multiple query patterns):
```sql
ALTER TABLE logs ALTER INTERLEAVED SORTKEY(timestamp, user_id, action);
```

### Compression Encoding

**Automatic Compression**:
```sql
COPY table_name FROM 's3://bucket/data'
COMPUPDATE ON;
```

**Manual Compression**:
```sql
-- Analyze compression
ANALYZE COMPRESSION table_name;

-- Apply recommended encoding
ALTER TABLE table_name 
ALTER COLUMN col1 ENCODE LZO,
ALTER COLUMN col2 ENCODE ZSTD;
```

### Workload Management (WLM)

**Configure WLM Queues**:
```json
[
  {
    "name": "etl",
    "memory_percent_to_use": 40,
    "max_concurrency": 5,
    "query_group": ["etl"]
  },
  {
    "name": "reporting",
    "memory_percent_to_use": 40,
    "max_concurrency": 15,
    "query_group": ["reporting"]
  },
  {
    "name": "default",
    "memory_percent_to_use": 20,
    "max_concurrency": 5
  }
]
```

**Assign Queries to Queues**:
```sql
SET query_group TO 'etl';
-- Run ETL queries
RESET query_group;
```

---

## Kết luận

Dashboard AWS Redshift này cung cấp insights toàn diện về:

### Performance Monitoring
- **CPUUtilization**: Cluster computational load
- **DatabaseConnections**: Concurrency và connection management
- **Latency Metrics**: Storage performance quality

### Health & Availability
- **HealthStatus**: Cluster operational status
- **MaintenanceMode**: Planned downtime tracking

### Network Performance
- **NetworkThroughput**: Data transfer patterns
- **Optimization**: CloudFront, VPC routing opportunities

### Storage Performance
- **IOPS**: Operation intensity per node
- **Throughput**: Data volume per node
- **Node-Level**: Identify bottlenecks và imbalances

### Capacity Planning
- Track growth trends
- Identify scaling needs
- Optimize resource utilization
- Cost management

**Sử dụng dashboard này để**:
✅ Ensure optimal Redshift performance  
✅ Proactive issue detection  
✅ Query optimization opportunities  
✅ Capacity planning và scaling decisions  
✅ Cost optimization strategies  
✅ Node-level performance troubleshooting  
✅ Maintenance window planning  

---

**Lưu ý quan trọng**:
- Metrics tự động gửi đến CloudWatch (no extra cost)
- 1-minute resolution
- IAM permissions required
- Node-level metrics crucial for troubleshooting
- Regular VACUUM và ANALYZE critical for performance
- Monitor disk space (< 80% threshold)
- Consider RA3 node types for scalable storage

**Version**: 1.0  
**Last Updated**: February 2026  
**Maintainer**: Data Engineering Team
