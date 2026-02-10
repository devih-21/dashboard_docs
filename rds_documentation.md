# Tài liệu Dashboard Amazon RDS

## Tổng quan
Dashboard này được thiết kế để giám sát và theo dõi các metrics của Amazon RDS (Relational Database Service). Dashboard cung cấp cái nhìn toàn diện về hiệu suất, tài nguyên, I/O performance, và connections của tất cả RDS instances trong một region.

**Nguồn dữ liệu**: CloudWatch  
**Khoảng thời gian mặc định**: 2 ngày gần nhất  
**Auto-refresh**: Có thể cấu hình (5s - 1d intervals)  

---

## Cấu hình

### Biến Template (Template Variables)

Dashboard sử dụng các biến động để cho phép người dùng tùy chỉnh view:

1. **datasource**: Nguồn dữ liệu CloudWatch
2. **region**: Vùng AWS (AWS Region)
3. **period**: Chu kỳ lấy mẫu metrics (seconds)
   - 60 seconds (1 minute) - Mặc định
   - 300 seconds (5 minutes)
   - 3600 seconds (1 hour)

**Lưu ý**: Dashboard hiển thị **TẤT CẢ** RDS instances trong region được chọn (wildcard `*`).

---

## Chi tiết các Dashboard Panel

### 1. CPU utilization per instance [%]

**Vị trí**: Hàng 1 (sau row header), chiều cao 8 đơn vị, chiều rộng 24 đơn vị  
**Loại biểu đồ**: Time series (Line chart)

**Mô tả**:  
Hiển thị CPU utilization của TẤT CẢ RDS instances trong region. Cho phép so sánh CPU usage giữa các instances và identify performance issues.

**Metrics**:
- Namespace: `AWS/RDS`
- Metric: `CPUUtilization`
- Statistic: `Maximum`
- Dimension: `DBInstanceIdentifier` = `*` (all instances)
- Unit: Percent (%)

**Cấu hình hiển thị**:
- Line chart (không fill)
- Legend: Bảng hiển thị mean value bên phải
- Multi-tooltip: Xem nhiều instances cùng lúc
- Giá trị không có min/max limit

**Ý nghĩa & Phân tích**:

### CPU Utilization Thresholds

**Health Levels**:
- **< 40%**: Healthy - Cluster có dư capacity
- **40-60%**: Normal load - Operating efficiency
- **60-75%**: Elevated - Monitor closely
- **75-85%**: High utilization - Plan for scaling
- **85-95%**: Very high - Performance may degrade
- **> 95%**: Critical - Immediate action needed

### Nguyên nhân CPU cao

**Query-Related**:
- Unoptimized queries (missing indexes)
- Full table scans
- Complex JOIN operations
- Heavy aggregations (GROUP BY, ORDER BY)
- Large result sets processing

**Workload-Related**:
- Too many concurrent connections
- High transaction volume
- Batch processing jobs
- ETL operations
- Backup/maintenance operations running

**Configuration Issues**:
- Instance undersized
- Inefficient database design
- Missing query optimization
- Poor connection pooling
- Locking/blocking queries

### Optimization Strategies

**Immediate Actions**:
```sql
-- Find CPU-intensive queries (PostgreSQL)
SELECT pid, query, state, 
       (now() - query_start) as duration
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

-- Find CPU-intensive queries (MySQL)
SELECT * FROM performance_schema.processlist
WHERE command != 'Sleep'
ORDER BY time DESC;
```

**Query Optimization**:
```sql
-- Add missing indexes
CREATE INDEX idx_customer_email ON customers(email);

-- Analyze query execution plan
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 123;

-- Update statistics
ANALYZE TABLE orders;
```

**Instance Scaling**:
- Vertical scaling: Upgrade instance class (more CPU/RAM)
- Read replicas: Offload read traffic
- Connection pooling: RDS Proxy
- Query caching: Application-level or database-level

**Monitoring Tips**:
- Compare CPU across instances
- Identify patterns (peak hours, batch jobs)
- Correlate với DatabaseConnections
- Alert when > 80% sustained for 10+ minutes

---

### 2. Database connections [count sum]

**Vị trí**: Hàng 2, chiều cao 8 đơn vị, chiều rộng 24 đơn vị  
**Loại biểu đồ**: Time series (Line chart)

**Mô tả**:  
Hiển thị tổng số database connections đang active cho tất cả RDS instances. Critical metric để monitor connection pool health và identify connection leaks.

**Metrics**:
- Namespace: `AWS/RDS`
- Metric: `DatabaseConnections`
- Statistic: `Sum`
- Dimension: `DBInstanceIdentifier` = `*` (all instances)
- Unit: Count

**Cấu hình hiển thị**:
- Line chart (không fill)
- Legend: Bảng hiển thị mean value bên phải
- Min value: 0
- Multi-tooltip enabled

**Ý nghĩa & Phân tích**:

### Connection Limits by Engine

**PostgreSQL**:
- Formula: `(DBInstanceClassMemory/9531392) - 2`
- Example: db.t3.medium (4GB) ≈ 442 max connections
- Reserved: 3 connections for maintenance

**MySQL**:
- Default: Based on instance size
- Example: db.t3.small ≈ 150 max connections
- Can be configured: `max_connections` parameter

**MariaDB**:
- Similar to MySQL
- Default: Based on instance size
- Configurable via parameter group

**Oracle**:
- License-dependent
- Typically higher limits
- Configurable: `processes` parameter

**SQL Server**:
- Edition-dependent
- Standard: Up to 32,767 connections
- Configurable: `user connections`

### Connection Monitoring

**Healthy Connection Patterns**:
- Stable baseline during off-peak
- Predictable spikes during business hours
- Connections < 70% of max limit
- No sudden unexplained spikes

**Problematic Patterns**:
- Steadily increasing (connection leak)
- Sudden spikes (connection storm)
- Sustained high (> 80% of max)
- Never drops to baseline

### Common Connection Issues

**Connection Pool Exhaustion**:
```python
# Bad: Creating new connection each time
def get_user(user_id):
    conn = connect_to_db()  # New connection
    result = conn.query("SELECT * FROM users WHERE id = %s", user_id)
    conn.close()
    return result

# Good: Using connection pooling
from sqlalchemy import create_engine
engine = create_engine('postgresql://...', pool_size=20, max_overflow=0)
```

**Connection Leaks**:
```python
# Bad: Not closing connections
def update_user(user_id, name):
    conn = connect_to_db()
    conn.execute("UPDATE users SET name = %s WHERE id = %s", name, user_id)
    # Missing: conn.close()

# Good: Using context manager
def update_user(user_id, name):
    with connect_to_db() as conn:
        conn.execute("UPDATE users SET name = %s WHERE id = %s", name, user_id)
    # Auto-closes
```

### Optimization Strategies

**RDS Proxy**:
```bash
# Benefits:
# - Connection pooling (reduces connection overhead)
# - Automatic failover (faster failover time)
# - IAM authentication support
# - Reduced connection storm impact

aws rds create-db-proxy \
  --db-proxy-name my-proxy \
  --engine-family POSTGRESQL \
  --auth '{"IAMAuth":"REQUIRED"}' \
  --role-arn arn:aws:iam::account:role/RDSProxyRole \
  --vpc-subnet-ids subnet-xxx subnet-yyy
```

**Application-Level Connection Pooling**:
```python
# PostgreSQL with psycopg2
import psycopg2.pool

connection_pool = psycopg2.pool.SimpleConnectionPool(
    minconn=5,
    maxconn=20,
    host='rds-endpoint',
    database='mydb'
)

# MySQL with mysql-connector-python
import mysql.connector.pooling

connection_pool = mysql.connector.pooling.MySQLConnectionPool(
    pool_name="mypool",
    pool_size=20,
    host='rds-endpoint',
    database='mydb'
)
```

**Query Idle Connections**:
```sql
-- PostgreSQL: Find idle connections
SELECT pid, usename, application_name, 
       state, state_change
FROM pg_stat_activity
WHERE state = 'idle'
AND state_change < now() - interval '5 minutes';

-- Terminate idle connections
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
AND state_change < now() - interval '1 hour';

-- MySQL: Find idle connections
SELECT id, user, host, db, command, time, state
FROM information_schema.processlist
WHERE command = 'Sleep'
AND time > 300;

-- Kill idle connections
KILL <connection_id>;
```

**Set Connection Timeouts**:
```sql
-- PostgreSQL parameter group
idle_in_transaction_session_timeout = 600000  -- 10 minutes

-- MySQL parameter group
wait_timeout = 600  -- 10 minutes
interactive_timeout = 600
```

**Monitoring & Alerts**:
- Alert when connections > 70% of max limit
- Alert on sudden spikes (> 50% increase in 5 min)
- Track connection creation rate
- Monitor idle connections percentage

---

### 3. Available storage space [bytes]

**Vị trí**: Hàng 3 (trái), chiều cao 8 đơn vị, chiều rộng 12 đơn vị  
**Loại biểu đồ**: Time series (Line chart)

**Mô tả**:  
Hiển thị dung lượng storage còn trống của tất cả RDS instances. Critical để tránh out-of-space issues có thể gây database downtime.

**Metrics**:
- Namespace: `AWS/RDS`
- Metric: `FreeStorageSpace`
- Statistic: `Average`
- Dimension: `DBInstanceIdentifier` = `*`
- Unit: Bytes (tự động convert to GB/TB)

**Cấu hình hiển thị**:
- Line chart
- Unit: decbytes (decimal bytes)
- Legend: Bảng hiển thị mean value bên phải
- Min value: 0

**Ý nghĩa & Phân tích**:

### Storage Capacity Thresholds

**Alert Levels**:
- **> 50% free**: Healthy
- **30-50% free**: Monitor growth
- **20-30% free**: Plan expansion
- **10-20% free**: Urgent - Expand soon
- **< 10% free**: Critical - Immediate action

**Impact of Low Storage**:
- Database stops accepting writes
- Query performance degradation
- Backup failures
- Replication lag
- Potential database unavailability

### Storage Growth Analysis

**Calculate Growth Rate**:
```
Daily Growth = (Storage Used Today - Storage Used Yesterday)
Monthly Projection = Daily Growth * 30
Capacity Runway = Free Storage / Daily Growth (days until full)
```

**Example**:
- Total Storage: 100 GB
- Free Storage: 40 GB (60 GB used)
- Yesterday Used: 58 GB
- Daily Growth: 2 GB/day
- **Runway**: 40 GB / 2 GB = 20 days until full

### Storage Optimization

**Identify Large Tables**:
```sql
-- PostgreSQL
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;

-- MySQL
SELECT table_schema, table_name,
       ROUND(((data_length + index_length) / 1024 / 1024), 2) AS size_mb
FROM information_schema.tables
ORDER BY (data_length + index_length) DESC
LIMIT 20;
```

**Data Cleanup Strategies**:

1. **Archive Old Data**:
```sql
-- Move old records to archive table
INSERT INTO orders_archive 
SELECT * FROM orders 
WHERE order_date < '2023-01-01';

DELETE FROM orders 
WHERE order_date < '2023-01-01';

-- Vacuum (PostgreSQL)
VACUUM FULL orders;
```

2. **Drop Unused Indexes**:
```sql
-- PostgreSQL: Find unused indexes
SELECT schemaname, tablename, indexname,
       idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- Drop unused index
DROP INDEX IF EXISTS idx_unused_index;
```

3. **Compress Large Columns**:
```sql
-- PostgreSQL: Use TOAST compression
ALTER TABLE large_table 
ALTER COLUMN description SET STORAGE EXTENDED;
```

4. **Purge Old Logs**:
```sql
-- MySQL: Purge binary logs
PURGE BINARY LOGS BEFORE '2024-01-01';

-- Check binary log space
SHOW BINARY LOGS;
```

### Storage Scaling

**Increase Storage**:
```bash
# AWS CLI
aws rds modify-db-instance \
  --db-instance-identifier mydb \
  --allocated-storage 200 \
  --apply-immediately

# Note: Can only increase, not decrease
# Minimum increase: 10% or 10 GB (whichever is larger)
```

**Storage Auto-Scaling**:
```bash
# Enable storage autoscaling
aws rds modify-db-instance \
  --db-instance-identifier mydb \
  --max-allocated-storage 500 \
  --apply-immediately

# Benefits:
# - Automatically increases storage when threshold hit
# - Prevents out-of-space issues
# - Scales in 10% increments
```

**Storage Types**:
- **General Purpose (gp3)**: Cost-effective, up to 16,000 IOPS
- **General Purpose (gp2)**: Baseline 3 IOPS/GB, burst to 3,000
- **Provisioned IOPS (io1)**: High performance, up to 64,000 IOPS
- **Magnetic**: Legacy, not recommended

**Monitoring Best Practices**:
- Alert when < 20% free storage
- Track growth trends weekly
- Review large tables monthly
- Enable storage autoscaling
- Monitor after autoscale events

---

### 4. Available RAM [bytes]

**Vị trí**: Hàng 3 (phải), chiều cao 8 đơn vị, chiều rộng 12 đơn vị  
**Loại biểu đồ**: Time series (Line chart)

**Mô tả**:  
Hiển thị RAM còn trống (freeable memory) của tất cả RDS instances. Memory được sử dụng cho buffer cache, query execution, connections, và database operations.

**Metrics**:
- Namespace: `AWS/RDS`
- Metric: `FreeableMemory`
- Statistic: `Average`
- Dimension: `DBInstanceIdentifier` = `*`
- Unit: Bytes (tự động convert to MB/GB)

**Cấu hình hiển thị**:
- Line chart
- Unit: decbytes
- Legend: Bảng hiển thị mean value bên phải
- Min value: 0

**Ý nghĩa & Phân tích**:

### Memory Usage Patterns

**Understanding Freeable Memory**:
- **Not the same as "free memory" in OS**
- Memory that can be freed for other operations
- Excludes buffer cache (which is good)
- Low value doesn't always mean problem

**Healthy Memory Patterns**:
- Database uses memory for caching (good!)
- Freeable memory stabilizes at low value
- No memory pressure warnings
- Query performance remains good

**Problematic Memory Patterns**:
- Freeable memory approaching 0
- Frequent swapping (SwapUsage metric)
- OOM (Out of Memory) events
- Query performance degradation

### Memory Thresholds

**Alert Levels**:
- **> 500 MB free**: Healthy
- **200-500 MB free**: Monitor
- **100-200 MB free**: Warning
- **< 100 MB free**: Critical - Check for issues

**Note**: Thresholds depend on instance size. Larger instances should have more free memory.

### Memory Optimization

**Query Memory Usage**:
```sql
-- PostgreSQL: Check memory settings
SHOW shared_buffers;
SHOW work_mem;
SHOW maintenance_work_mem;
SHOW effective_cache_size;

-- MySQL: Check memory settings
SHOW VARIABLES LIKE '%buffer%';
SHOW VARIABLES LIKE '%cache%';
```

**Identify Memory-Heavy Queries**:
```sql
-- PostgreSQL: Find queries using temp space
SELECT pid, datname, usename, 
       query,
       temp_files, temp_bytes
FROM pg_stat_activity
WHERE temp_files > 0
ORDER BY temp_bytes DESC;

-- MySQL: Find queries with large sorts/joins
SELECT * FROM sys.statements_with_full_table_scans
ORDER BY total_latency DESC
LIMIT 20;
```

**Optimize Buffer Cache**:
```sql
-- PostgreSQL: Check buffer cache hit ratio
SELECT 
  sum(heap_blks_read) as heap_read,
  sum(heap_blks_hit)  as heap_hit,
  sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) as ratio
FROM pg_statio_user_tables;

-- Target: > 0.99 (99% cache hit rate)

-- MySQL: Check buffer pool hit ratio
SHOW STATUS LIKE 'Innodb_buffer_pool_read%';
```

**Parameter Tuning** (via Parameter Group):

PostgreSQL:
```ini
# Increase shared buffers (25% of RAM recommended)
shared_buffers = 8GB

# Increase work_mem for complex queries
work_mem = 64MB

# Maintenance operations
maintenance_work_mem = 1GB

# Query planner
effective_cache_size = 24GB  # ~75% of RAM
```

MySQL:
```ini
# InnoDB buffer pool (70-80% of RAM)
innodb_buffer_pool_size = 10G

# Number of buffer pool instances
innodb_buffer_pool_instances = 8

# Query cache (deprecated in MySQL 8.0)
query_cache_size = 0  # Disable on 5.7

# Sort buffer
sort_buffer_size = 2M
```

### Memory Scaling

**Vertical Scaling**:
```bash
# Upgrade instance class for more memory
aws rds modify-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.r6g.xlarge \
  --apply-immediately
```

**Memory-Optimized Instances**:
- **r6g family**: Graviton2, best price/performance
- **r6i family**: Intel, high memory
- **r5 family**: Previous generation, cost-effective
- **x2 family**: Extreme memory (up to 4TB)

**Monitoring Best Practices**:
- Alert when freeable memory < 200 MB
- Monitor SwapUsage metric (should be near 0)
- Track memory usage trends
- Correlate with query performance
- Review after parameter changes

---

### 5. Read throughput [bytes/sec]

**Vị trí**: Hàng 4 (trái), chiều cao 8 đơn vị, chiều rộng 12 đơn vị  
**Loại biểu đồ**: Time series (Line chart - Step after)

**Mô tả**:  
Hiển thị disk read throughput (bytes per second) của tất cả RDS instances. Indicates volume of data being read from storage.

**Metrics**:
- Namespace: `AWS/RDS`
- Metric: `ReadThroughput`
- Statistic: `Average`
- Dimension: `DBInstanceIdentifier` = `*`
- Unit: Bytes per second

**Cấu hình hiển thị**:
- Line chart (step after interpolation)
- Unit: short (auto-scale to KB/s, MB/s)
- Legend: Bảng hiển thị mean value bên phải
- Min value: 0

**Ý nghĩa & Phân tích**:

### What Drives Read Throughput

**High Read Throughput Scenarios**:
- SELECT queries with large result sets
- Full table scans
- Sequential scans (missing indexes)
- Report generation
- Analytical queries
- Backup operations
- Read replica lag catch-up

**Throughput vs IOPS**:
- **High throughput + Low IOPS**: Large sequential reads (good)
- **Low throughput + High IOPS**: Small random reads (may need optimization)
- **Both high**: Heavy read workload

### Throughput Capacity by Storage Type

**General Purpose SSD (gp3)**:
- Base throughput: 125 MiB/s
- Max throughput: 1,000 MiB/s (configurable)
- Independent of storage size

**General Purpose SSD (gp2)**:
- Throughput: 128-250 MiB/s depending on size
- Smaller volumes: 128 MiB/s
- Larger volumes (> 334 GB): 250 MiB/s

**Provisioned IOPS SSD (io1/io2)**:
- Throughput: Up to 1,000 MiB/s (io1)
- Throughput: Up to 4,000 MiB/s (io2 Block Express)
- High-performance workloads

**Magnetic (Standard)**:
- Throughput: ~100 MiB/s
- Not recommended for production

### Read Throughput Optimization

**Reduce Unnecessary Reads**:
```sql
-- Bad: SELECT * reads all columns
SELECT * FROM large_table WHERE id = 123;

-- Good: Select only needed columns
SELECT id, name, email FROM large_table WHERE id = 123;
```

**Optimize Query Plans**:
```sql
-- PostgreSQL: Check query execution
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM orders WHERE customer_id = 123;

-- Look for:
-- - Seq Scan (should be Index Scan)
-- - High buffer reads
-- - Nested loops with large tables
```

**Add Missing Indexes**:
```sql
-- PostgreSQL: Find missing indexes
SELECT schemaname, tablename, 
       seq_scan, seq_tup_read,
       idx_scan, idx_tup_fetch
FROM pg_stat_user_tables
WHERE seq_scan > 1000
AND seq_tup_read > 100000
ORDER BY seq_tup_read DESC;

-- Create appropriate index
CREATE INDEX idx_customer_id ON orders(customer_id);
```

**Use Read Replicas**:
```bash
# Create read replica for read-heavy workloads
aws rds create-db-instance-read-replica \
  --db-instance-identifier mydb-replica \
  --source-db-instance-identifier mydb \
  --db-instance-class db.r6g.large
```

**Implement Caching**:
```python
# Application-level caching (Redis example)
import redis
r = redis.Redis(host='redis-endpoint')

def get_user(user_id):
    # Check cache first
    cached = r.get(f"user:{user_id}")
    if cached:
        return cached
    
    # Cache miss: query database
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    r.setex(f"user:{user_id}", 3600, user)  # Cache for 1 hour
    return user
```

**Monitoring Tips**:
- Compare throughput with IOPS metrics
- Identify peak read periods
- Correlate with query patterns
- Alert on sustained high throughput (> 80% capacity)

---

### 6. Write throughput [bytes/sec]

**Vị trí**: Hàng 4 (phải), chiều cao 8 đơn vị, chiều rộng 12 đơn vị  
**Loại biểu đồ**: Time series (Line chart - Step after)

**Mô tả**:  
Hiển thị disk write throughput (bytes per second) của tất cả RDS instances. Indicates volume of data being written to storage.

**Metrics**:
- Namespace: `AWS/RDS`
- Metric: `WriteThroughput`
- Statistic: `Average`
- Dimension: `DBInstanceIdentifier` = `*`
- Unit: Bytes per second

**Cấu hình hiển thị**:
- Line chart (step after interpolation)
- Unit: short (auto-scale)
- Legend: Bảng hiển thị mean value bên phải
- Min value: 0

**Ý nghĩa & Phân tích**:

### What Drives Write Throughput

**High Write Throughput Scenarios**:
- INSERT/UPDATE/DELETE operations
- Bulk data loads
- WAL (Write-Ahead Log) writes
- Transaction log writes
- Index updates
- Checkpoint operations
- Backup operations

**Write Patterns**:
- **Steady writes**: Normal transactional workload
- **Burst writes**: Batch operations, ETL
- **High sustained**: Heavy transactional system

### Write Throughput Capacity

Same capacity as Read Throughput (by storage type):
- **gp3**: 125-1,000 MiB/s
- **gp2**: 128-250 MiB/s
- **io1/io2**: Up to 1,000-4,000 MiB/s

### Write Optimization Strategies

**Batch Operations**:
```sql
-- Bad: Individual inserts
INSERT INTO orders VALUES (1, 'data1');
INSERT INTO orders VALUES (2, 'data2');
INSERT INTO orders VALUES (3, 'data3');

-- Good: Batch insert
INSERT INTO orders VALUES 
  (1, 'data1'),
  (2, 'data2'),
  (3, 'data3');
```

**Use Transactions**:
```sql
-- PostgreSQL
BEGIN;
INSERT INTO orders VALUES (1, 'data1');
INSERT INTO order_items VALUES (1, 1, 'item1');
INSERT INTO order_items VALUES (1, 2, 'item2');
COMMIT;

-- MySQL
START TRANSACTION;
INSERT INTO orders VALUES (1, 'data1');
INSERT INTO order_items VALUES (1, 1, 'item1');
COMMIT;
```

**Bulk Loading**:
```sql
-- PostgreSQL: COPY command
COPY orders FROM '/path/to/data.csv' CSV;

-- MySQL: LOAD DATA
LOAD DATA INFILE '/path/to/data.csv'
INTO TABLE orders
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n';
```

**Reduce Index Overhead**:
```sql
-- For bulk loads: Drop indexes, load data, recreate indexes
DROP INDEX idx_customer_id;
-- Load data here
CREATE INDEX idx_customer_id ON orders(customer_id);
```

**Optimize WAL/Redo Logs**:
```sql
-- PostgreSQL: Check WAL settings
SHOW wal_buffers;
SHOW checkpoint_timeout;
SHOW max_wal_size;

-- MySQL: Check redo log settings
SHOW VARIABLES LIKE 'innodb_log%';
```

**Async Replication**:
```sql
-- For read replicas where slight lag is acceptable
-- Master doesn't wait for replica confirmation
-- Improves write performance on master
```

**Monitoring Tips**:
- Monitor during ETL windows
- Compare with CPU and IOPS
- Track transaction rates
- Alert on unusual spikes
- Correlate with application deployments

---

### 7. Read latency [sec]

**Vị trí**: Hàng 5 (trái), chiều cao 8 đơn vị, chiều rộng 12 đơn vị  
**Loại biểu đồ**: Time series (Line chart - Step after)

**Mô tả**:  
Hiển thị average time (latency) để hoàn thành disk read operations. Critical indicator của storage performance và query responsiveness.

**Metrics**:
- Namespace: `AWS/RDS`
- Metric: `ReadLatency`
- Statistic: `Maximum`
- Dimension: `DBInstanceIdentifier` = `*`
- Unit: Seconds

**Cấu hình hiển thị**:
- Line chart (step after interpolation)
- Unit: short (seconds)
- Legend: Bảng hiển thị mean value bên phải
- Min value: 0

**Ý nghĩa & Phân tích**:

### Read Latency Thresholds

**Performance Levels**:
- **< 1 ms**: Excellent (SSD, cached data)
- **1-5 ms**: Good (SSD performance)
- **5-10 ms**: Acceptable
- **10-20 ms**: Degraded - Investigate
- **> 20 ms**: Poor - Immediate action

**What Causes High Read Latency**:
- Storage saturation (hitting IOPS limit)
- Disk queue depth high
- Large sequential scans
- Cold cache (data not in buffer)
- Storage type limitations
- Network latency (EBS over network)

### Read Latency by Storage Type

**Expected Latencies**:
- **gp3/gp2 SSD**: 1-5 ms typical
- **io1/io2 SSD**: < 1 ms (sub-millisecond)
- **Magnetic**: 100+ ms (not recommended)

### Read Latency Optimization

**Improve Cache Hit Ratio**:
```sql
-- PostgreSQL: Check cache effectiveness
SELECT 
  sum(heap_blks_read) as disk_reads,
  sum(heap_blks_hit) as cache_hits,
  round(sum(heap_blks_hit)::numeric / 
        (sum(heap_blks_hit) + sum(heap_blks_read)), 4) as hit_ratio
FROM pg_statio_user_tables;

-- Target: > 0.99 (99%+ cache hit)
```

**Optimize IOPS**:
```bash
# Upgrade to Provisioned IOPS for lower latency
aws rds modify-db-instance \
  --db-instance-identifier mydb \
  --storage-type io1 \
  --iops 10000 \
  --apply-immediately
```

**Query Optimization**:
```sql
-- Add covering indexes (include all columns in query)
CREATE INDEX idx_covering ON orders(customer_id, order_date, status);

-- Query can be satisfied from index alone
SELECT customer_id, order_date, status 
FROM orders 
WHERE customer_id = 123;
```

**Partitioning**:
```sql
-- PostgreSQL: Partition large tables
CREATE TABLE orders_2024 PARTITION OF orders
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Reduces data scanned, improves latency
```

**Monitoring & Troubleshooting**:
- Correlate with ReadIOPS metric
- Check DiskQueueDepth metric
- Review slow query logs
- Monitor buffer cache hit ratio
- Alert when latency > 10ms sustained

---

### 8. Write latency [sec]

**Vị trí**: Hàng 5 (phải), chiều cao 8 đơn vị, chiều rộng 12 đơn vị  
**Loại biểu đồ**: Time series (Line chart - Step after)

**Mô tả**:  
Hiển thị average time để hoàn thành disk write operations. Critical cho transaction performance và data durability.

**Metrics**:
- Namespace: `AWS/RDS`
- Metric: `WriteLatency`
- Statistic: `Maximum`
- Dimension: `DBInstanceIdentifier` = `*`
- Unit: Seconds

**Cấu hình hiển thị**:
- Line chart (step after interpolation)
- Unit: short (seconds)
- Legend: Bảng hiển thị mean value bên phải
- Min value: 0

**Ý nghĩa & Phân tích**:

### Write Latency Thresholds

**Performance Levels**:
- **< 2 ms**: Excellent (SSD)
- **2-10 ms**: Good
- **10-20 ms**: Acceptable
- **20-50 ms**: Degraded
- **> 50 ms**: Poor - Serious issue

**What Causes High Write Latency**:
- IOPS limit reached
- High checkpoint activity
- WAL/redo log contention
- Disk queue depth maxed
- Large transactions
- Index maintenance overhead

### Write Latency Impact

**Application Impact**:
- Slow INSERT/UPDATE/DELETE operations
- Transaction timeouts
- Application slowness
- User-facing delays
- Replication lag (for replicas)

**Database Impact**:
- Transaction log growth
- Checkpoint stalls
- Connection pile-up
- Lock contention increases

### Write Latency Optimization

**Reduce Write Volume**:
```sql
-- Use UPDATE only when needed
UPDATE users SET last_login = now() WHERE id = 123;

-- Better: Track separately or use cache
-- Avoid updating frequently-changing columns
```

**Batch Writes**:
```sql
-- Use transactions to batch writes
BEGIN;
UPDATE orders SET status = 'shipped' WHERE id IN (1,2,3,4,5);
UPDATE inventory SET quantity = quantity - 1 WHERE product_id IN (10,20,30);
COMMIT;
```

**Optimize Checkpoints**:
```sql
-- PostgreSQL: Tune checkpoint settings
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9
max_wal_size = 4GB

-- MySQL: Tune InnoDB settings
innodb_flush_log_at_trx_commit = 2  -- Faster, slight durability trade-off
innodb_log_file_size = 512M
```

**Upgrade Storage**:
```bash
# Move to Provisioned IOPS for consistent low latency
aws rds modify-db-instance \
  --db-instance-identifier mydb \
  --storage-type io1 \
  --iops 15000
```

**Async Commit** (when acceptable):
```sql
-- PostgreSQL: Faster writes, slight durability trade-off
SET synchronous_commit = off;

-- MySQL: Similar setting
SET GLOBAL innodb_flush_log_at_trx_commit = 2;
```

**Monitoring Tips**:
- Correlate with WriteIOPS
- Check DiskQueueDepth
- Monitor checkpoint frequency
- Review transaction sizes
- Alert when > 20ms sustained

---

### 9. Read operations [IOPS]

**Vị trí**: Hàng 6 (trái), chiều cao 8 đơn vị, chiều rộng 12 đơn vị  
**Loại biểu đồ**: Time series (Line chart - Step after)

**Mô tả**:  
Hiển thị số lượng disk read I/O operations per second. Indicates query activity và storage access patterns.

**Metrics**:
- Namespace: `AWS/RDS`
- Metric: `ReadIOPS`
- Statistic: `Average`
- Dimension: `DBInstanceIdentifier` = `*`
- Unit: Count/second (IOPS)

**Cấu hình hiển thị**:
- Line chart (step after interpolation)
- Unit: short
- Legend: Bảng hiển thị mean value bên phải
- Min value: 0

**Ý nghĩa & Phân tích**:

### IOPS Capacity by Storage Type

**General Purpose SSD (gp3)**:
- Base: 3,000 IOPS
- Max: 16,000 IOPS (configurable)
- Independent of storage size

**General Purpose SSD (gp2)**:
- Baseline: 3 IOPS per GB
- Min: 100 IOPS
- Max: 16,000 IOPS
- Burst: Up to 3,000 IOPS (for volumes < 1TB)
- Burst credits: Accumulate when under baseline

**Provisioned IOPS SSD (io1)**:
- Configurable: 1,000 - 64,000 IOPS
- Ratio: Up to 50 IOPS per GB
- Consistent performance
- No burst concept

**Provisioned IOPS SSD (io2 Block Express)**:
- Configurable: 1,000 - 256,000 IOPS
- Ratio: Up to 1,000 IOPS per GB
- Highest performance
- Sub-millisecond latency

### Read IOPS Patterns

**Analytical Workload**:
- High read IOPS
- Low write IOPS
- Peak during business hours
- Reports/dashboards usage

**OLTP Workload**:
- Moderate read IOPS
- Frequent small reads
- Random access pattern
- Steady throughout day

**Batch Processing**:
- Burst read IOPS
- Sequential reads
- Scheduled patterns
- ETL jobs

### IOPS Optimization

**Identify Hot Tables**:
```sql
-- PostgreSQL: Table I/O statistics
SELECT schemaname, tablename,
       heap_blks_read, heap_blks_hit,
       idx_blks_read, idx_blks_hit
FROM pg_statio_user_tables
ORDER BY (heap_blks_read + idx_blks_read) DESC
LIMIT 20;

-- MySQL: Table I/O statistics  
SELECT * FROM sys.io_global_by_file_by_bytes
LIMIT 20;
```

**Reduce IOPS with Indexes**:
```sql
-- Full table scan: High IOPS
SELECT * FROM orders WHERE order_date > '2024-01-01';

-- With index: Lower IOPS
CREATE INDEX idx_order_date ON orders(order_date);
```

**Increase Buffer Cache**:
```sql
-- More data cached in memory = fewer disk reads = lower IOPS
-- PostgreSQL
ALTER SYSTEM SET shared_buffers = '8GB';

-- MySQL
SET GLOBAL innodb_buffer_pool_size = 10737418240;  -- 10GB
```

**Use Read Replicas**:
- Distribute read load across replicas
- Reduces IOPS on primary
- Improves primary write performance

**Upgrade Storage**:
```bash
# Increase IOPS capacity
aws rds modify-db-instance \
  --db-instance-identifier mydb \
  --storage-type io1 \
  --iops 20000
```

**Monitoring Tips**:
- Track IOPS vs capacity (% utilization)
- Monitor gp2 burst credit balance
- Correlate with query patterns
- Alert when approaching IOPS limit

---

### 10. Write operations [IOPS]

**Vị trí**: Hàng 6 (phải), chiều cao 8 đơn vị, chiều rộng 12 đơn vị  
**Loại biểu đồ**: Time series (Line chart - Step after)

**Mô tả**:  
Hiển thị số lượng disk write I/O operations per second. Critical cho transactional systems và data modification workloads.

**Metrics**:
- Namespace: `AWS/RDS`
- Metric: `WriteIOPS`
- Statistic: `Average`
- Dimension: `DBInstanceIdentifier` = `*`
- Unit: Count/second (IOPS)

**Cấu hình hiển thị**:
- Line chart (step after interpolation)
- Unit: short
- Legend: Bảng hiển thị mean value bên phải
- Min value: 0

**Ý nghĩa & Phân tích**:

### Write IOPS Patterns

**Transactional System**:
- Steady write IOPS
- Frequent small writes
- Proportional to user activity
- Peak during business hours

**Batch Loading**:
- Burst write IOPS
- Large bulk writes
- Scheduled patterns
- ETL windows

**Mixed Workload**:
- Variable write IOPS
- Combination of transactional + batch
- Multiple peaks throughout day

### What Drives Write IOPS

**Database Operations**:
- INSERT/UPDATE/DELETE statements
- Transaction commits (WAL writes)
- Index updates
- Checkpoint operations
- VACUUM/OPTIMIZE operations
- Temporary table writes

**Background Processes**:
- Autovacuum (PostgreSQL)
- Buffer flushing
- Transaction log archiving
- Statistics collection

### Write IOPS Optimization

**Batch Modifications**:
```sql
-- Bad: Many small writes
FOR EACH row IN rows:
    UPDATE orders SET status = 'processed' WHERE id = row.id;

-- Good: One batch update
UPDATE orders SET status = 'processed' 
WHERE id IN (1, 2, 3, 4, 5, ...);
```

**Reduce Index Writes**:
```sql
-- Review indexes: Each write to table writes to all indexes
SELECT tablename, indexname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE idx_scan < 100  -- Rarely used
ORDER BY pg_relation_size(indexrelid) DESC;

-- Drop unused indexes
DROP INDEX IF EXISTS idx_rarely_used;
```

**Optimize Checkpoints**:
```sql
-- PostgreSQL: Spread checkpoint I/O
checkpoint_completion_target = 0.9  -- Use 90% of checkpoint interval

-- MySQL: Tune InnoDB flushing
innodb_flush_neighbors = 0  -- On SSD, no need to flush neighbors
innodb_io_capacity = 2000   -- Set based on IOPS capacity
innodb_io_capacity_max = 4000
```

**Partitioning for Write Performance**:
```sql
-- Partition table by date
CREATE TABLE orders_2024_01 PARTITION OF orders
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Writes go to specific partition
-- Improves index maintenance
-- Easier data archival
```

**Use Fillfactor**:
```sql
-- PostgreSQL: Leave room for HOT updates
ALTER TABLE orders SET (fillfactor = 80);

-- Reduces write IOPS for updates
-- Prevents page splits
```

**Monitoring & Capacity Planning**:
- Track write IOPS trends
- Monitor during batch operations
- Compare with IOPS capacity
- Review burst credit balance (gp2)
- Plan capacity before scaling events

**Alerts**:
- Write IOPS > 80% of capacity
- gp2 burst credits depleted
- Sustained high write IOPS (> 1 hour)
- Sudden unusual spikes

---

## Hướng dẫn sử dụng

### 1. Chọn Data Source
- Dropdown **Data source**: Chọn CloudWatch datasource
- Mặc định: cloudwatch

### 2. Chọn AWS Region
- Dropdown **Region**: Chọn region chứa RDS instances
- Dashboard sẽ hiển thị tất cả instances trong region

### 3. Điều chỉnh Period
- Dropdown **Period [sec]**: Chọn chu kỳ lấy mẫu
- 60s (1 min): Chi tiết, real-time monitoring
- 300s (5 min): Tổng quát hơn, giảm data points
- 3600s (1 hour): Long-term trends

### 4. Điều chỉnh Time Range
- Time picker: Góc trên bên phải
- Mặc định: 2 ngày gần nhất
- Recommended:
  - Real-time: Last 1h - Last 6h
  - Daily: Last 24h - Last 7d
  - Trends: Last 30d

### 5. Auto-Refresh
- Có thể enable auto-refresh
- Options: 5s, 10s, 30s, 1m, 5m, 15m, 30m, 1h, 2h, 1d
- Useful cho real-time monitoring

### 6. Compare Instances
- Legend hiển thị tất cả instances
- Click vào instance name để toggle visibility
- Shift+click để isolate một instance

---

## Best Practices

### 1. Regular Monitoring Routine

**Daily Checks**:
- CPU Utilization: All instances < 75%
- Database Connections: < 70% of limit
- Storage space: > 20% free
- Memory: Adequate freeable memory

**Weekly Reviews**:
- Performance trends
- Storage growth rate
- Connection patterns
- IOPS utilization

**Monthly Planning**:
- Capacity planning
- Cost optimization review
- Performance optimization opportunities
- Scaling decisions

### 2. Alert Configuration

**Critical Alerts**:
```
CPU > 85% for 10 minutes
FreeStorageSpace < 10% 
FreeableMemory < 100 MB
ReadLatency > 20ms sustained
WriteLatency > 50ms sustained
DatabaseConnections > 80% of max
```

**Warning Alerts**:
```
CPU > 70% for 30 minutes
FreeStorageSpace < 20%
FreeableMemory < 500 MB
ReadIOPS > 80% of capacity
WriteIOPS > 80% of capacity
```

### 3. Performance Optimization

**Query Optimization**:
- Enable slow query log
- Review query execution plans
- Add appropriate indexes
- Update table statistics regularly

**Connection Management**:
- Implement connection pooling
- Use RDS Proxy for serverless workloads
- Set appropriate connection timeouts
- Monitor idle connections

**Storage Optimization**:
- Enable storage autoscaling
- Archive old data
- Purge unnecessary logs
- Review table sizes regularly

### 4. Cost Optimization

**Right-Sizing**:
- Review CPU and memory utilization
- Downsize underutilized instances
- Use burstable instances (T3/T4g) for dev/test

**Storage**:
- Use gp3 instead of gp2 (better price/performance)
- Right-size IOPS allocation
- Enable storage autoscaling
- Archive old data to S3

**Reserved Instances**:
- Purchase RIs for predictable workloads
- 1-year vs 3-year commitment analysis
- All Upfront vs Partial Upfront savings

**Scheduling**:
- Stop non-production instances after hours
- Use automated start/stop schedules
- Consider Aurora Serverless for variable workloads

### 5. High Availability

**Multi-AZ Deployment**:
- Enable Multi-AZ for production
- Automatic failover (< 2 minutes)
- Synchronous replication
- No performance impact on reads

**Read Replicas**:
- Scale read capacity
- Disaster recovery standby
- Cross-region replication
- Promote to standalone instance

**Backup Strategy**:
- Enable automated backups (retention: 7-35 days)
- Test restore procedures regularly
- Use snapshots for long-term retention
- Cross-region backup copies

---

## Troubleshooting Common Issues

### Issue 1: High CPU Utilization

**Symptoms**: CPU > 80% sustained

**Diagnosis**:
```sql
-- PostgreSQL: Find CPU-intensive queries
SELECT pid, usename, query, 
       state, (now() - query_start) as duration
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

-- MySQL: Find CPU-intensive queries
SELECT * FROM performance_schema.processlist
WHERE command != 'Sleep'
ORDER BY time DESC;
```

**Solutions**:
1. Optimize slow queries
2. Add missing indexes
3. Scale up instance class
4. Use read replicas
5. Implement caching

### Issue 2: Connection Limit Reached

**Symptoms**: "Too many connections" errors

**Diagnosis**:
```sql
-- Check current connections
SELECT count(*) FROM pg_stat_activity;  -- PostgreSQL
SHOW PROCESSLIST;  -- MySQL

-- Find idle connections
SELECT * FROM pg_stat_activity 
WHERE state = 'idle' 
AND state_change < now() - interval '10 minutes';
```

**Solutions**:
1. Implement connection pooling
2. Use RDS Proxy
3. Close idle connections
4. Increase max_connections (via parameter group)
5. Fix connection leaks in application

### Issue 3: Running Out of Storage

**Symptoms**: FreeStorageSpace approaching 0

**Diagnosis**:
```sql
-- PostgreSQL: Find large tables
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;

-- MySQL: Find large tables
SELECT table_schema, table_name,
       ROUND(((data_length + index_length) / 1024 / 1024), 2) as size_mb
FROM information_schema.tables
ORDER BY (data_length + index_length) DESC
LIMIT 20;
```

**Solutions**:
1. Enable storage autoscaling
2. Increase allocated storage
3. Archive old data
4. Purge binary logs
5. Drop unused indexes

### Issue 4: High Latency

**Symptoms**: ReadLatency or WriteLatency > 20ms

**Diagnosis**:
```sql
-- Check IOPS utilization
-- Compare current IOPS with baseline/capacity

-- Check for storage bottleneck
-- Review DiskQueueDepth metric

-- Identify slow queries
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

**Solutions**:
1. Upgrade to Provisioned IOPS (io1/io2)
2. Increase IOPS allocation
3. Optimize queries
4. Increase buffer cache
5. Use read replicas for read-heavy workloads

### Issue 5: Replication Lag

**Symptoms**: Read replica lagging behind master

**Diagnosis**:
```sql
-- Check ReplicaLag metric in CloudWatch

-- PostgreSQL: Check replication status
SELECT * FROM pg_stat_replication;

-- MySQL: Check replica status
SHOW SLAVE STATUS\G
```

**Solutions**:
1. Upgrade replica instance class
2. Reduce write load on master
3. Check network connectivity
4. Review long-running transactions
5. Consider Multi-AZ instead of read replica

### Issue 6: Out of Memory

**Symptoms**: FreeableMemory approaching 0, SwapUsage > 0

**Diagnosis**:
```sql
-- Check memory settings
SHOW shared_buffers;  -- PostgreSQL
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';  -- MySQL

-- Find memory-heavy queries
SELECT * FROM pg_stat_activity
WHERE temp_files > 0
ORDER BY temp_bytes DESC;
```

**Solutions**:
1. Upgrade to larger instance class
2. Tune memory parameters
3. Optimize queries (reduce temp space)
4. Limit connection count
5. Use memory-optimized instance (r6g, r6i)

---

## Additional CloudWatch Metrics

### Metrics Not in Dashboard (But Available)

**Disk Queue Depth**:
```
Metric: DiskQueueDepth
Use: Number of outstanding I/O requests
Alert: If consistently > 5
```

**Swap Usage**:
```
Metric: SwapUsage
Use: Memory swapping to disk
Alert: Should be near 0
```

**Burst Balance** (for gp2):
```
Metric: BurstBalance
Use: gp2 burst credits remaining
Alert: If depleting (< 20%)
```

**Replica Lag**:
```
Metric: ReplicaLag
Use: Read replica delay behind source
Alert: If > 5 seconds
```

**Transaction Logs Disk Usage**:
```
Metric: TransactionLogsDiskUsage
Use: Space used by transaction logs
Alert: If growing unexpectedly
```

**Network Metrics**:
```
Metrics: NetworkReceiveThroughput, NetworkTransmitThroughput
Use: Network I/O monitoring
```

---

## RDS Engine-Specific Considerations

### PostgreSQL

**Key Parameters**:
- shared_buffers: 25% of RAM
- work_mem: 64MB for complex queries
- maintenance_work_mem: 1GB
- effective_cache_size: 75% of RAM
- max_connections: Based on workload

**Monitoring Tools**:
```sql
-- Check cache hit ratio
SELECT sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) 
FROM pg_statio_user_tables;

-- Check index usage
SELECT * FROM pg_stat_user_indexes 
WHERE idx_scan = 0;

-- Find bloat
SELECT schemaname, tablename, 
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename))
FROM pg_tables;
```

### MySQL

**Key Parameters**:
- innodb_buffer_pool_size: 70-80% of RAM
- innodb_log_file_size: 512MB - 1GB
- max_connections: Based on workload
- query_cache_size: 0 (disabled in 8.0+)

**Monitoring Tools**:
```sql
-- Check buffer pool efficiency
SHOW STATUS LIKE 'Innodb_buffer_pool%';

-- Check slow queries
SELECT * FROM mysql.slow_log
ORDER BY start_time DESC
LIMIT 20;

-- Table sizes
SELECT table_name, 
       ROUND((data_length + index_length) / 1024 / 1024, 2) AS size_mb
FROM information_schema.tables
ORDER BY (data_length + index_length) DESC;
```

### MariaDB

Similar to MySQL with some enhancements:
- Better thread pool
- Enhanced replication
- More storage engines
- Improved performance schema

### Oracle

**Licensing Considerations**:
- License Included vs BYOL
- Costs based on vCPU count
- Enterprise Edition features

**Key Metrics**:
- Session count
- Tablespace usage
- Wait events
- AWR reports

### SQL Server

**Edition Differences**:
- Express: Limited to 10GB
- Web: Limited features
- Standard: Full features, lower limits
- Enterprise: All features, highest limits

**Key Metrics**:
- Page life expectancy
- Buffer cache hit ratio
- Lock waits
- Deadlocks

---

## Integration với AWS Services

### CloudWatch Alarms

**Create Alarm**:
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name rds-high-cpu \
  --alarm-description "CPU > 80%" \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=mydb \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:region:account:my-topic
```

### SNS Notifications

**Setup SNS for Alerts**:
```bash
# Create SNS topic
aws sns create-topic --name rds-alerts

# Subscribe email
aws sns subscribe \
  --topic-arn arn:aws:sns:region:account:rds-alerts \
  --protocol email \
  --notification-endpoint admin@example.com
```

### Lambda Automation

**Auto-Stop Non-Production**:
```python
import boto3

def lambda_handler(event, context):
    rds = boto3.client('rds')
    
    # Stop dev databases after hours
    instances = rds.describe_db_instances()
    for instance in instances['DBInstances']:
        if 'dev' in instance['DBInstanceIdentifier']:
            rds.stop_db_instance(
                DBInstanceIdentifier=instance['DBInstanceIdentifier']
            )
```

### EventBridge Rules

**Automated Actions**:
```json
{
  "source": ["aws.rds"],
  "detail-type": ["RDS DB Instance Event"],
  "detail": {
    "EventCategories": ["failure", "availability"]
  }
}
```

---

## Kết luận

Dashboard Amazon RDS này cung cấp monitoring toàn diện cho:

### Performance Monitoring
- **CPU Utilization**: Computational load
- **Connections**: Concurrency management
- **Latency**: Response time quality
- **IOPS**: Storage performance

### Resource Management
- **Storage Space**: Capacity planning
- **Memory**: Buffer cache efficiency
- **Throughput**: Data volume handling

### Operational Excellence
- Multi-instance visibility
- Cross-region support
- Flexible time periods
- Real-time monitoring

**Sử dụng dashboard này để**:
✅ Ensure optimal RDS performance  
✅ Proactive issue detection  
✅ Capacity planning  
✅ Cost optimization  
✅ Query performance tuning  
✅ Connection management  
✅ Storage optimization  

---

**Lưu ý quan trọng**:
- Dashboard hiển thị **TẤT CẢ** instances (wildcard `*`)
- Metrics tự động gửi đến CloudWatch (no extra cost)
- 1-minute resolution available
- Compare across instances easily
- Consider instance-specific dashboards for detailed analysis
- Enable Enhanced Monitoring for OS-level metrics
- Use Performance Insights for query-level analysis

**Version**: 1.0  
**Last Updated**: February 2026  
**Maintainer**: Database Operations Team
