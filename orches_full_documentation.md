# Tài Liệu Dashboard Orches Full Layer

## Tổng Quan

Dashboard **Orches Full Layer** là một công cụ giám sát và quản lý data orchestration cho các DBT jobs và Raw jobs (Glue Catalog Ingestion). Dashboard này cung cấp khả năng trực quan hóa dependencies, tracking execution status, và phân tích job performance theo thời gian.

**Dashboard UID**: `afagos32k5wjkc`  
**Datasource**: PostgreSQL (Grafana PostgreSQL Datasource)  
**Database Schema**: `olap_dev`

## Kiến Trúc Hệ Thống

Dashboard này quản lý 2 loại jobs:

### 1. DBT Jobs
- Được lưu trong table `olap_dev.job_config`
- Execution logs trong `olap_dev.execution_log`
- Có dependencies được định nghĩa trong field `depend_ons` (JSONB array)

### 2. Raw Jobs (Glue Catalog Ingestion)
- External jobs KHÔNG có configuration trong `job_config`
- Được referenced như dependencies của DBT jobs
- Tracking logs trong `olap_dev.glue_catalog_ingestion_tracking`
- Thường là data sources: S3, Glue Catalog, external databases

## Biến Template

Dashboard sử dụng các biến sau để filter và customize views:

### $job_id (Multi-select)
- **Nguồn**: Query từ `olap_dev.job_config`
- **Query**: `select id from olap_dev.job_config;`
- **Mô tả**: Chọn job cụ thể để xem dependencies
- **Hỗ trợ**: Multi-select, có thể chọn "All"
- **Use case**: Filter jobs cần analyze trong node graph và tables

### $data_date (Single select)
- **Nguồn**: Query từ `olap_dev.execution_log`
- **Query**: `select data_date::text from olap_dev.execution_log;`
- **Mô tả**: Chọn ngày execution cần xem
- **Sort**: Descending (mới nhất trước)
- **Hỗ trợ**: Include "All" để xem multiple dates
- **Use case**: Track job status theo specific date

### $status (Multi-select)
- **Nguồn**: Query từ execution logs
- **Query**: 
  ```sql
  select distinct status::text from olap_dev.execution_log
  UNION ALL
  select 'NOT_STARTED' as status;
  ```
- **Mô tả**: Filter jobs theo execution status
- **Values**:
  - `Completed` - Job hoàn thành thành công
  - `SUCCESS` - Job thành công (Glue jobs)
  - `FAILED` - Job thất bại
  - `RUNNING` - Job đang chạy
  - `PENDING` - Job chờ execution
  - `QUEUE` - Job trong queue
  - `NOT_STARTED` - Job chưa được trigger
- **Hỗ trợ**: Multi-select, Include "All"

### $level (Single select)
- **Type**: Interval variable
- **Options**: 1, 2, 3, 4, 5, 6, 7, 8
- **Default**: 1
- **Mô tả**: Số level của dependency graph để hiển thị
  - Level 1: Direct dependencies (parent/child trực tiếp)
  - Level 2: Dependencies của dependencies (grandparent/grandchild)
  - Level 3+: Deep dependency chains
- **Use case**: Control complexity của node graph visualization

## Các Panels Chi Tiết

### Row 1: Job Summary per Day

---

#### Panel 1: Job Full Layer (Node Graph)

**Visualization Type**: Node Graph  
**Grid Position**: Row 1, Width 18, Height 16  
**Queries**: 2 queries (Nodes + Edges)

**Mục đích**: 
Hiển thị dependency graph của jobs với visual representation của relationships và execution status.

**Query A - Nodes** (Job nodes với status):
```sql
WITH RECURSIVE
dbt_job_id as (
    select id, depend_ons from olap_dev.job_config
),
all_raw_ids as (
  -- Extract raw job IDs from depend_ons
  SELECT distinct elem as id
  FROM olap_dev.job_config jc
  LEFT JOIN LATERAL jsonb_array_elements_text(
      CASE WHEN jsonb_typeof(jc.depend_ons) = 'array'
          THEN jc.depend_ons
          ELSE '[]'::jsonb
      END
  ) AS elem ON TRUE
  where elem not in (select id from dbt_job_id)
),
all_config_ids as (
    -- Combine DBT jobs + Raw jobs
    select id, '[]'::jsonb AS depend_ons from all_raw_ids
    union all
    select id, depend_ons from dbt_job_id
),
downstream AS (
    -- Level 0: Root jobs
    SELECT id, 0 AS level
    FROM all_config_ids
    WHERE ('All' IN ($job_id) OR id IN ($job_id))
    
    UNION ALL
    
    -- Level n: Downstream dependencies
    SELECT c.id, d.level + 1 AS level
    FROM olap_dev.job_config c
    JOIN downstream d ON c.depend_ons ? d.id
    WHERE d.level < $level
),
upstream AS (
    -- Level 0: Root jobs
    SELECT id, depend_ons, 0 AS level
    FROM all_config_ids
    WHERE ('All' IN ($job_id) OR id IN ($job_id))
    
    UNION ALL
    
    -- Level n: Upstream dependencies
    SELECT c.id, c.depend_ons, u.level + 1 AS level
    FROM all_config_ids c
    JOIN upstream u ON u.depend_ons ? c.id
    WHERE u.level < $level
),
all_logs as (
    -- Combine logs from both sources
    select job_id, data_date, status from olap_dev.execution_log 
    union all
    select data_source_name as job_id, data_date, status 
    from olap_dev.glue_catalog_ingestion_tracking
),
all_ids AS (
    -- Filter by level
    SELECT id FROM downstream WHERE level <= $level
    UNION
    SELECT id FROM upstream WHERE level <= $level
)
SELECT DISTINCT ON (conf.id)
    conf.id AS id,
    conf.id AS title,
    COALESCE(log.status, 'NOT_STARTED') AS "mainStat",
    CASE 
        WHEN log.status = 'Completed' THEN 'green'
        WHEN log.status = 'SUCCESS' THEN 'green'
        WHEN log.status = 'FAILED' THEN 'red'
        WHEN log.status = 'RUNNING' THEN 'blue'
        WHEN log.status = 'PENDING' THEN 'orange'
        WHEN log.status = 'QUEUE' THEN 'purple'
        ELSE 'gray' 
    END AS color
FROM all_config_ids conf
LEFT JOIN all_logs log 
    ON conf.id = log.job_id 
   AND ('All' IN ($data_date) OR log.data_date::text IN ($data_date))
WHERE conf.id IN (SELECT id FROM all_ids);
```

**Query B - Edges** (Dependency links):
```sql
-- Returns relationships between jobs
SELECT 
    parent_id || '->' || child_id AS id,
    parent_id AS source,
    child_id AS target
FROM (
    SELECT 
        jsonb_array_elements_text(
            CASE WHEN jsonb_typeof(depend_ons) = 'array' 
                THEN depend_ons 
                ELSE '[]'::jsonb 
            END
        ) AS parent_id,
        id AS child_id
    FROM all_config_ids
) all_links
WHERE parent_id IN (SELECT id FROM all_ids)
  AND child_id IN (SELECT id FROM all_ids);
```

**Node Colors**:
- 🟢 **Green**: Completed/SUCCESS - Job chạy thành công
- 🔴 **Red**: FAILED - Job thất bại
- 🔵 **Blue**: RUNNING - Job đang thực thi
- 🟠 **Orange**: PENDING - Job chờ được execute
- 🟣 **Purple**: QUEUE - Job trong hàng đợi
- ⚪ **Gray**: NOT_STARTED - Job chưa được trigger

**Best Practices**:
- Start với level 1 để xem direct dependencies
- Tăng level dần để explore deep dependencies
- Sử dụng để identify bottlenecks trong pipeline
- Click vào nodes để drill down vào specific jobs
- Identify failed jobs (red nodes) và upstream dependencies

**Use Cases**:
1. **Root cause analysis**: Tìm failed job và trace upstream để xem dependency nào gây ra
2. **Impact analysis**: Select một job và xem downstream để biết impact nếu job fail
3. **Pipeline visualization**: Hiểu toàn bộ data flow từ raw sources đến final tables
4. **Performance optimization**: Identify long chains và opportunities để parallelize

---

#### Panel 2: Total DBT Job (Stat)

**Visualization Type**: Stat panel  
**Grid Position**: Row 1, X: 18, Y: 1, Width 3, Height 3

**Query**:
```sql
select count(1)
from olap_dev.job_config
```

**Ý nghĩa**: 
Tổng số DBT jobs được configured trong hệ thống.

**Monitoring**:
- Track growth của DBT jobs over time
- Baseline metric cho capacity planning
- Compare với jobs executed để calculate coverage

---

#### Panel 3: Total Raw Jobs (Stat)

**Visualization Type**: Stat panel  
**Grid Position**: Row 1, X: 21, Y: 1, Width 3, Height 3

**Query**:
```sql
with dbt_job_id as (
    select id from olap_dev.job_config
)
SELECT count(distinct elem)
FROM olap_dev.job_config jc
LEFT JOIN LATERAL jsonb_array_elements_text(
    CASE
        WHEN jsonb_typeof(jc.depend_ons) = 'array'
        THEN jc.depend_ons
        ELSE '[]'::jsonb
    END
) AS elem ON TRUE
where elem not in (select id from dbt_job_id);
```

**Ý nghĩa**: 
Tổng số Raw jobs (external dependencies) được referenced trong DBT jobs nhưng KHÔNG có configuration trong `job_config`.

**Raw jobs thường là**:
- S3 ingestion jobs
- Glue Catalog crawlers
- External database syncs
- API data pulls
- File uploads

**Monitoring**:
- Track external dependencies
- Ensure all raw jobs có proper tracking
- Identify missing configurations

---

#### Panel 4: Total DBT Jobs not run (Stat)

**Visualization Type**: Stat panel (Dark Red color)  
**Grid Position**: Row 1, X: 18, Y: 4, Width 3, Height 3

**Query**:
```sql
WITH
dbt_job_id as (
    select id from olap_dev.job_config
),
total_job AS (
    SELECT COUNT(1) AS total_job
    FROM olap_dev.job_config
),
total_job_run AS (
    SELECT COUNT(1) AS total_job_run
    FROM olap_dev.execution_log log
    WHERE log.data_date::text IN ($data_date)
    and job_id in (select id from dbt_job_id)
)
SELECT
    total_job.total_job - total_job_run.total_job_run AS not_run_job
FROM total_job, total_job_run;
```

**Ý nghĩa**: 
Số lượng DBT jobs CHƯA được execute trong selected date.

**Ngưỡng cảnh báo**:
- **Normal**: 0 jobs not run (all scheduled jobs executed)
- **Warning**: < 10% jobs not run
- **Critical**: > 10% jobs not run hoặc critical jobs missing

**Troubleshooting**:
```sql
-- Find which jobs didn't run
WITH dbt_job_id as (
    select id from olap_dev.job_config
)
SELECT jc.id
FROM olap_dev.job_config jc
WHERE jc.id NOT IN (
    SELECT job_id 
    FROM olap_dev.execution_log 
    WHERE data_date = '2026-01-12'
)
ORDER BY jc.id;
```

**Common reasons**:
1. **Not scheduled**: Jobs không có cron schedule cho date đó
2. **Dependency not met**: Upstream jobs failed hoặc chưa complete
3. **Manual trigger only**: Jobs chỉ run on-demand
4. **Disabled**: Jobs tạm thời disabled
5. **Configuration issue**: Scheduler không pick up job

**Best Practices**:
- Set alerts khi > threshold
- Review daily để ensure completeness
- Document expected vs actual job runs
- Investigate patterns (same jobs missing daily?)

---

#### Panel 5: Total Raw Jobs not run (Stat)

**Visualization Type**: Stat panel (Dark Red color)  
**Grid Position**: Row 1, X: 21, Y: 4, Width 3, Height 3

**Query**:
```sql
WITH dbt_job_id AS (
    SELECT id FROM olap_dev.job_config
),
raw_job_id AS (
    -- Dependencies NOT in job_config
    SELECT DISTINCT elem
    FROM olap_dev.job_config jc
    LEFT JOIN LATERAL jsonb_array_elements_text(
        CASE
            WHEN jsonb_typeof(jc.depend_ons) = 'array'
            THEN jc.depend_ons
            ELSE '[]'::jsonb
        END
    ) AS elem ON TRUE
    WHERE elem IS NOT NULL
      AND NOT EXISTS (
          SELECT 1 FROM dbt_job_id d WHERE d.id = elem
      )
),
raw_job_total_count AS (
    SELECT COUNT(*) AS total_raw_job
    FROM raw_job_id
),
count_job_run AS (
    SELECT COUNT(DISTINCT log.data_source_name) AS total_run
    FROM olap_dev.glue_catalog_ingestion_tracking log
    WHERE log.data_date::text IN ($data_date)
      AND log.data_source_name IN (SELECT elem FROM raw_job_id)
)
SELECT
    r.total_raw_job - c.total_run AS not_run_raw_job
FROM raw_job_total_count r
CROSS JOIN count_job_run c;
```

**Ý nghĩa**: 
Số lượng Raw jobs (external dependencies) CHƯA được execute trong selected date.

**Ngưỡng cảnh báo**:
- **Critical**: > 0 raw jobs not run (có thể block downstream DBT jobs)
- **High priority**: Raw jobs thường là foundation của pipeline

**Impact**:
Raw jobs not running → DBT jobs không có fresh data → Pipeline incomplete

**Troubleshooting**:
```sql
-- Find which raw jobs didn't run
WITH dbt_job_id AS (
    SELECT id FROM olap_dev.job_config
),
raw_job_id AS (
    SELECT DISTINCT elem
    FROM olap_dev.job_config jc
    LEFT JOIN LATERAL jsonb_array_elements_text(
        CASE
            WHEN jsonb_typeof(jc.depend_ons) = 'array'
            THEN jc.depend_ons
            ELSE '[]'::jsonb
        END
    ) AS elem ON TRUE
    WHERE elem IS NOT NULL
      AND NOT EXISTS (SELECT 1 FROM dbt_job_id d WHERE d.id = elem)
)
SELECT r.elem as raw_job_id
FROM raw_job_id r
WHERE r.elem NOT IN (
    SELECT data_source_name 
    FROM olap_dev.glue_catalog_ingestion_tracking
    WHERE data_date = '2026-01-12'
)
ORDER BY r.elem;
```

**Common causes**:
1. **S3 data not arrived**: Source files chưa được upload
2. **Glue crawler failed**: Catalog update failed
3. **External system down**: Source database/API unavailable
4. **Network issues**: Connectivity problems
5. **Permission issues**: IAM roles hoặc credentials expired

**Resolution steps**:
1. Check source data availability
2. Verify Glue Crawler/Job status
3. Check CloudWatch logs cho raw job executions
4. Validate IAM permissions
5. Manual trigger nếu cần thiết

---

#### Panel 6: Total Job Raw per Day (Table)

**Visualization Type**: Table  
**Grid Position**: Row 1, X: 18, Y: 7, Width 6, Height 5

**Query**:
```sql
with dbt_job_id as (
    select id from olap_dev.job_config
),
raw_job_id as (
    SELECT distinct elem
    FROM olap_dev.job_config jc
    LEFT JOIN LATERAL jsonb_array_elements_text(
        CASE
            WHEN jsonb_typeof(jc.depend_ons) = 'array'
            THEN jc.depend_ons
            ELSE '[]'::jsonb
        END
    ) AS elem ON TRUE
    where elem not in (select id from dbt_job_id)
)
SELECT 
    status as "Job Status", 
    count(1) as "Job Count" 
FROM olap_dev.glue_catalog_ingestion_tracking log
where 
  log.data_date::text IN ($data_date)
  and log.data_source_name in (select elem from raw_job_id)
group by status
```

**Columns**:
- **Job Status**: Status của raw job execution
- **Job Count**: Số lượng jobs với status đó

**Status values**:
- `SUCCESS`: Raw job ingestion thành công
- `FAILED`: Raw job ingestion thất bại
- `RUNNING`: Raw job đang thực thi

**Ý nghĩa**: 
Distribution của raw job statuses trong selected date.

**Analysis**:
- Monitor success rate của raw ingestion jobs
- Identify trends (increasing failures?)
- Compare across dates
- Alert on high failure rates

**Success Rate Calculation**:
```
Success Rate = SUCCESS / (SUCCESS + FAILED) × 100%
Target: > 95% success rate
```

---

#### Panel 7: Total Job DBT per Day (Table)

**Visualization Type**: Table  
**Grid Position**: Row 1, X: 18, Y: 12, Width 6, Height 5

**Query**:
```sql
with dbt_job_id as (
    select id from olap_dev.job_config
)
SELECT 
    status as "Job Status", 
    count(1) as "Job Count" 
FROM olap_dev.execution_log log
where 
  log.data_date::text IN ($data_date)
  and job_id in (select id from dbt_job_id)
group by status
```

**Columns**:
- **Job Status**: Status của DBT job execution
- **Job Count**: Số lượng jobs với status đó

**Status values**:
- `Completed`: DBT job hoàn thành thành công
- `FAILED`: DBT job thất bại
- `RUNNING`: DBT job đang thực thi
- `PENDING`: DBT job chờ dependencies
- `QUEUE`: DBT job trong queue

**Ý nghĩa**: 
Distribution của DBT job statuses trong selected date.

**Key Metrics**:
```
Success Rate = Completed / (Completed + FAILED) × 100%
Target: > 98% success rate

Average Queue Time = Jobs in QUEUE / Total Jobs
Target: < 5%

Failure Rate = FAILED / Total Jobs × 100%
Target: < 2%
```

**Monitoring Strategy**:
1. **Daily Review**: Check status distribution mỗi ngày
2. **Trend Analysis**: Compare với previous days
3. **Alert Thresholds**: Set alerts cho abnormal patterns
4. **Root Cause**: Investigate failed jobs immediately

---

### Row 2: Job Dependencies

---

#### Panel 8: Execution Log (Table)

**Visualization Type**: Table  
**Grid Position**: Row 2, Y: 18, Width 24, Height 8

**Query**:
```sql
WITH RECURSIVE 
downstream AS (
    SELECT id FROM olap_dev.job_config 
    WHERE ('All' IN ($job_id) OR id IN ($job_id))
    UNION
    SELECT c.id 
    FROM olap_dev.job_config c
    JOIN downstream d ON c.depend_ons ? d.id
),
upstream AS (
    SELECT id, depend_ons 
    FROM olap_dev.job_config 
    WHERE ('All' IN ($job_id) OR id IN ($job_id))
    UNION
    SELECT c.id, c.depend_ons
    FROM olap_dev.job_config c
    JOIN upstream u ON u.depend_ons ? c.id
),
all_ids AS (
    SELECT id FROM downstream
    UNION
    SELECT id FROM upstream
)
SELECT 
    execution_id,
    job_id,
    data_date::text,
    depend_ons,
    dependency_status,
    job_config,
    status,
    detail_status,
    error_message,
    max_retry,
    retry,
    start_time::text,
    end_time::text,
    data_date_type,
    duration,
    cron_expression,
    run_dependencies,
    schedule_type,
    log_retention_days,
    engine_run_id,
    redshift_write_status,
    redshift_write_result,
    execution_steps,
    test_results,
    test_status
FROM olap_dev.execution_log log
WHERE log.job_id IN (SELECT id FROM all_ids) 
  AND ('All' IN ($data_date) OR log.data_date::text IN ($data_date))
  AND ('All' IN ($status) OR log.status IN ($status))
```

**Key Columns**:

**Execution Information**:
- `execution_id`: Unique ID cho mỗi execution
- `job_id`: DBT job identifier
- `data_date`: Business date của data được process
- `status`: Execution status (Completed, FAILED, RUNNING, PENDING, QUEUE)
- `detail_status`: Detailed status information

**Dependency Tracking**:
- `depend_ons`: JSONB array của upstream job IDs
- `dependency_status`: Status của dependencies
- `run_dependencies`: Dependencies được run trong execution này

**Timing & Performance**:
- `start_time`: Execution start timestamp
- `end_time`: Execution end timestamp
- `duration`: Execution duration (seconds)

**Error Handling**:
- `error_message`: Error details nếu job failed
- `max_retry`: Maximum số retry attempts
- `retry`: Current retry count

**Scheduling**:
- `cron_expression`: Cron schedule nếu applicable
- `schedule_type`: Type của schedule (cron, manual, event-driven)
- `data_date_type`: Type của data date (current, historical, backfill)

**DBT Specific**:
- `job_config`: DBT job configuration (JSONB)
- `execution_steps`: Steps executed trong DBT run
- `test_results`: DBT test results
- `test_status`: Overall test status

**Data Warehouse Integration**:
- `engine_run_id`: dbt Cloud/Core run ID
- `redshift_write_status`: Status của Redshift write operation
- `redshift_write_result`: Results của Redshift operations

**Operations**:
- `log_retention_days`: Number of days để retain logs

**Use Cases**:

**1. Failed Job Analysis**:
```sql
-- Find all failed jobs và error messages
SELECT 
    job_id,
    data_date,
    error_message,
    retry,
    start_time,
    end_time
FROM olap_dev.execution_log
WHERE status = 'FAILED'
  AND data_date = '2026-01-12'
ORDER BY start_time DESC;
```

**2. Performance Analysis**:
```sql
-- Find slowest jobs
SELECT 
    job_id,
    AVG(duration) as avg_duration,
    MAX(duration) as max_duration,
    COUNT(*) as execution_count
FROM olap_dev.execution_log
WHERE status = 'Completed'
  AND data_date >= CURRENT_DATE - 7
GROUP BY job_id
ORDER BY avg_duration DESC
LIMIT 10;
```

**3. Retry Pattern Analysis**:
```sql
-- Jobs requiring multiple retries
SELECT 
    job_id,
    data_date,
    retry,
    max_retry,
    error_message
FROM olap_dev.execution_log
WHERE retry > 0
ORDER BY retry DESC, data_date DESC;
```

**4. Dependency Chain Tracking**:
```sql
-- Trace execution chain
SELECT 
    log.job_id,
    log.status,
    log.depend_ons,
    log.dependency_status,
    log.start_time
FROM olap_dev.execution_log log
WHERE log.data_date = '2026-01-12'
ORDER BY log.start_time;
```

**5. Test Results Analysis**:
```sql
-- Find jobs với failed tests
SELECT 
    job_id,
    data_date,
    test_status,
    test_results,
    status
FROM olap_dev.execution_log
WHERE test_status = 'FAILED'
  AND data_date >= CURRENT_DATE - 7
ORDER BY data_date DESC;
```

**Best Practices**:
- Review execution logs daily
- Monitor duration trends để detect performance degradation
- Set up alerts cho high retry counts
- Investigate patterns trong error messages
- Track dependency resolution times
- Monitor test results để ensure data quality
- Archive old logs theo retention policy

---

#### Panel 9: Job Config (Table)

**Visualization Type**: Table  
**Grid Position**: Row 2, Y: 26, Width 24, Height 8

**Query**:
```sql
WITH RECURSIVE 
downstream AS (
    SELECT id FROM olap_dev.job_config 
    WHERE ('All' IN ($job_id) OR id IN ($job_id))
    UNION
    SELECT c.id 
    FROM olap_dev.job_config c
    JOIN downstream d ON c.depend_ons ? d.id
),
upstream AS (
    SELECT id, depend_ons 
    FROM olap_dev.job_config 
    WHERE ('All' IN ($job_id) OR id IN ($job_id))
    UNION
    SELECT c.id, c.depend_ons
    FROM olap_dev.job_config c
    JOIN upstream u ON u.depend_ons ? c.id
),
all_ids AS (
    SELECT id FROM downstream
    UNION
    SELECT id FROM upstream
)
SELECT * FROM olap_dev.job_config conf
WHERE conf.id IN (SELECT id FROM all_ids);
```

**Ý nghĩa**: 
Hiển thị full configuration của all jobs trong dependency tree của selected job(s).

**Key Information trong job_config table**:

**Job Identity**:
- `id`: Unique job identifier
- `name`: Human-readable job name
- `description`: Job purpose và details

**Dependencies**:
- `depend_ons`: JSONB array of upstream job IDs
  ```json
  ["raw_job_1", "raw_job_2", "dbt_job_parent"]
  ```

**Scheduling**:
- `cron_expression`: Cron schedule
  - Examples:
    - `0 8 * * *` - Daily at 8 AM
    - `0 */4 * * *` - Every 4 hours
    - `0 0 * * 0` - Weekly on Sunday
- `schedule_type`: Manual, Cron, Event-driven

**DBT Configuration**:
- `project_name`: dbt project name
- `target`: dbt target environment (dev, prod)
- `models`: Models để run
- `vars`: dbt variables
- `threads`: Number of threads
- `full_refresh`: Full refresh flag

**Resource Configuration**:
- `timeout`: Job timeout (seconds)
- `memory`: Memory allocation
- `cpu`: CPU allocation

**Data Configuration**:
- `data_date_type`: current, historical, backfill
- `lookback_days`: Days to look back cho data
- `incremental_strategy`: append, merge, delete+insert

**Error Handling**:
- `max_retry`: Maximum retry attempts
- `retry_delay`: Delay between retries (seconds)
- `on_failure`: Action on failure (alert, skip, block)

**Notifications**:
- `alert_channels`: Slack, Email, PagerDuty
- `alert_on`: success, failure, all
- `owner`: Team/person responsible

**Use Cases**:

**1. Dependency Audit**:
```sql
-- Find jobs với most dependencies
SELECT 
    id,
    jsonb_array_length(
        CASE 
            WHEN jsonb_typeof(depend_ons) = 'array' 
            THEN depend_ons 
            ELSE '[]'::jsonb 
        END
    ) as dependency_count
FROM olap_dev.job_config
ORDER BY dependency_count DESC;
```

**2. Orphaned Jobs**:
```sql
-- Jobs không có dependencies và không là dependency của others
WITH all_dependencies AS (
    SELECT DISTINCT elem as job_id
    FROM olap_dev.job_config jc
    LEFT JOIN LATERAL jsonb_array_elements_text(
        CASE 
            WHEN jsonb_typeof(jc.depend_ons) = 'array'
            THEN jc.depend_ons
            ELSE '[]'::jsonb
        END
    ) AS elem ON TRUE
)
SELECT jc.id
FROM olap_dev.job_config jc
WHERE (
    jc.depend_ons IS NULL 
    OR jsonb_array_length(jc.depend_ons) = 0
)
AND jc.id NOT IN (SELECT job_id FROM all_dependencies WHERE job_id IS NOT NULL);
```

**3. Circular Dependencies Detection**:
```sql
-- Find potential circular dependencies
WITH RECURSIVE dep_chain AS (
    SELECT 
        id as root_job,
        id as current_job,
        depend_ons,
        1 as depth,
        ARRAY[id] as path
    FROM olap_dev.job_config
    
    UNION ALL
    
    SELECT 
        dc.root_job,
        jc.id as current_job,
        jc.depend_ons,
        dc.depth + 1,
        dc.path || jc.id
    FROM dep_chain dc
    JOIN olap_dev.job_config jc 
        ON dc.depend_ons ? jc.id
    WHERE dc.depth < 10
      AND NOT (jc.id = ANY(dc.path))
)
SELECT 
    root_job,
    current_job,
    path
FROM dep_chain
WHERE current_job = root_job
  AND depth > 1;
```

**4. Schedule Analysis**:
```sql
-- Group jobs by schedule pattern
SELECT 
    cron_expression,
    COUNT(*) as job_count,
    string_agg(id, ', ') as jobs
FROM olap_dev.job_config
WHERE cron_expression IS NOT NULL
GROUP BY cron_expression
ORDER BY job_count DESC;
```

**5. Resource Planning**:
```sql
-- Aggregate resource requirements
SELECT 
    SUM(threads) as total_threads,
    AVG(timeout) as avg_timeout,
    COUNT(*) as total_jobs
FROM olap_dev.job_config
WHERE schedule_type = 'cron';
```

**Best Practices**:
- Document all job configurations
- Regular dependency audit để detect issues
- Validate cron expressions
- Set appropriate timeouts based on historical duration
- Configure retries based on failure patterns
- Keep dependency chains shallow (< 5 levels)
- Use naming conventions cho job IDs
- Regular cleanup của orphaned jobs

---

## Database Schema

### Table: olap_dev.job_config

**Purpose**: Store DBT job configurations và dependencies

**Key Columns**:
```sql
CREATE TABLE olap_dev.job_config (
    id VARCHAR PRIMARY KEY,
    name VARCHAR,
    description TEXT,
    depend_ons JSONB,  -- Array of upstream job IDs
    cron_expression VARCHAR,
    schedule_type VARCHAR,
    project_name VARCHAR,
    target VARCHAR,
    models VARCHAR[],
    vars JSONB,
    threads INTEGER,
    timeout INTEGER,
    max_retry INTEGER,
    retry_delay INTEGER,
    on_failure VARCHAR,
    alert_channels JSONB,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    owner VARCHAR
);
```

**Indexes**:
```sql
-- For dependency queries
CREATE INDEX idx_job_config_depend_ons ON olap_dev.job_config USING GIN (depend_ons);

-- For schedule queries
CREATE INDEX idx_job_config_cron ON olap_dev.job_config (cron_expression);
```

---

### Table: olap_dev.execution_log

**Purpose**: Track DBT job execution history và results

**Key Columns**:
```sql
CREATE TABLE olap_dev.execution_log (
    execution_id VARCHAR PRIMARY KEY,
    job_id VARCHAR REFERENCES olap_dev.job_config(id),
    data_date DATE,
    status VARCHAR,
    detail_status VARCHAR,
    depend_ons JSONB,
    dependency_status JSONB,
    job_config JSONB,
    start_time TIMESTAMP,
    end_time TIMESTAMP,
    duration INTEGER,  -- seconds
    error_message TEXT,
    max_retry INTEGER,
    retry INTEGER,
    cron_expression VARCHAR,
    run_dependencies JSONB,
    schedule_type VARCHAR,
    data_date_type VARCHAR,
    log_retention_days INTEGER,
    engine_run_id VARCHAR,
    execution_steps JSONB,
    test_results JSONB,
    test_status VARCHAR,
    redshift_write_status VARCHAR,
    redshift_write_result JSONB,
    created_at TIMESTAMP
);
```

**Indexes**:
```sql
-- For date-based queries
CREATE INDEX idx_execution_log_data_date ON olap_dev.execution_log (data_date);

-- For status queries
CREATE INDEX idx_execution_log_status ON olap_dev.execution_log (status);

-- For job lookup
CREATE INDEX idx_execution_log_job_id ON olap_dev.execution_log (job_id);

-- Composite for common filters
CREATE INDEX idx_execution_log_job_date ON olap_dev.execution_log (job_id, data_date);
```

---

### Table: olap_dev.glue_catalog_ingestion_tracking

**Purpose**: Track Raw job (Glue Catalog ingestion) execution history

**Key Columns**:
```sql
CREATE TABLE olap_dev.glue_catalog_ingestion_tracking (
    id SERIAL PRIMARY KEY,
    data_source_name VARCHAR,  -- Raw job identifier
    data_date DATE,
    status VARCHAR,  -- SUCCESS, FAILED, RUNNING
    start_time TIMESTAMP,
    end_time TIMESTAMP,
    duration INTEGER,
    error_message TEXT,
    records_processed BIGINT,
    bytes_processed BIGINT,
    glue_job_name VARCHAR,
    glue_job_run_id VARCHAR,
    s3_path VARCHAR,
    catalog_database VARCHAR,
    catalog_table VARCHAR,
    partition_keys JSONB,
    created_at TIMESTAMP
);
```

**Indexes**:
```sql
-- For date-based queries
CREATE INDEX idx_glue_tracking_data_date ON olap_dev.glue_catalog_ingestion_tracking (data_date);

-- For source lookup
CREATE INDEX idx_glue_tracking_source ON olap_dev.glue_catalog_ingestion_tracking (data_source_name);

-- For status queries
CREATE INDEX idx_glue_tracking_status ON olap_dev.glue_catalog_ingestion_tracking (status);

-- Composite for common filters
CREATE INDEX idx_glue_tracking_source_date 
ON olap_dev.glue_catalog_ingestion_tracking (data_source_name, data_date);
```

---

## Recursive CTE Logic Explained

Dashboard sử dụng **Recursive CTEs** để traverse dependency graph. Đây là core logic:

### Downstream Traversal

```sql
WITH RECURSIVE downstream AS (
    -- Base case: Root job(s)
    SELECT id, 0 AS level
    FROM olap_dev.job_config
    WHERE id IN ($job_id)
    
    UNION ALL
    
    -- Recursive case: Find children
    SELECT c.id, d.level + 1 AS level
    FROM olap_dev.job_config c
    JOIN downstream d 
        ON c.depend_ons ? d.id  -- JSONB contains operator
    WHERE d.level < $level
)
SELECT * FROM downstream;
```

**Logic**:
1. Start với selected job(s) - level 0
2. Find all jobs DEPENDENT ON current jobs (children) - level 1
3. Recursively find dependencies của dependencies - level 2+
4. Stop at max $level

**Example**:
```
Selected: job_B (level 0)
├── job_D depends on job_B (level 1)
│   └── job_F depends on job_D (level 2)
└── job_E depends on job_B (level 1)
```

---

### Upstream Traversal

```sql
WITH RECURSIVE upstream AS (
    -- Base case: Root job(s)
    SELECT id, depend_ons, 0 AS level
    FROM olap_dev.job_config
    WHERE id IN ($job_id)
    
    UNION ALL
    
    -- Recursive case: Find parents
    SELECT c.id, c.depend_ons, u.level + 1 AS level
    FROM olap_dev.job_config c
    JOIN upstream u 
        ON u.depend_ons ? c.id  -- Current job in upstream's dependencies
    WHERE u.level < $level
)
SELECT * FROM upstream;
```

**Logic**:
1. Start với selected job(s) - level 0
2. Find all jobs that selected job DEPENDS ON (parents) - level 1
3. Recursively find dependencies của parents - level 2+
4. Stop at max $level

**Example**:
```
Selected: job_B (level 0) depends on [job_A]
├── job_A (level 1) depends on [raw_source_1]
│   └── raw_source_1 (level 2)
```

---

### Combined Traversal

```sql
WITH RECURSIVE 
downstream AS (...),
upstream AS (...),
all_ids AS (
    SELECT id FROM downstream WHERE level <= $level
    UNION
    SELECT id FROM upstream WHERE level <= $level
)
SELECT * FROM olap_dev.job_config
WHERE id IN (SELECT id FROM all_ids);
```

**Result**: Complete dependency tree (upstream + downstream) trong specified levels.

---

## Monitoring & Alerting Strategy

### 1. Daily Health Checks

**Morning Checklist** (review yesterday's execution):
```sql
-- Daily summary query
SELECT 
    COUNT(*) FILTER (WHERE status = 'Completed') as completed,
    COUNT(*) FILTER (WHERE status = 'FAILED') as failed,
    COUNT(*) FILTER (WHERE status IN ('RUNNING', 'PENDING', 'QUEUE')) as in_progress,
    ROUND(100.0 * COUNT(*) FILTER (WHERE status = 'Completed') / COUNT(*), 2) as success_rate
FROM olap_dev.execution_log
WHERE data_date = CURRENT_DATE - 1;
```

**Expected Output**:
- Success Rate: > 98%
- Failed Jobs: < 2% of total
- In Progress: Should be 0 by morning (all overnight jobs complete)

---

### 2. Real-time Alerts

**Alert Rules**:

**Critical Alerts** (Immediate response):
```sql
-- Critical job failures
SELECT job_id, error_message, start_time
FROM olap_dev.execution_log
WHERE status = 'FAILED'
  AND job_id IN (
      -- List of critical jobs
      'dim_customer', 'fact_transactions', 'daily_revenue'
  )
  AND data_date = CURRENT_DATE;
```

**Warning Alerts** (Review within hours):
```sql
-- Jobs exceeding normal duration
WITH avg_duration AS (
    SELECT 
        job_id,
        AVG(duration) as avg_dur,
        STDDEV(duration) as stddev_dur
    FROM olap_dev.execution_log
    WHERE status = 'Completed'
      AND data_date >= CURRENT_DATE - 30
    GROUP BY job_id
)
SELECT 
    log.job_id,
    log.duration,
    ad.avg_dur,
    log.duration - ad.avg_dur as deviation
FROM olap_dev.execution_log log
JOIN avg_duration ad ON log.job_id = ad.job_id
WHERE log.data_date = CURRENT_DATE
  AND log.status IN ('Completed', 'RUNNING')
  AND log.duration > ad.avg_dur + (2 * ad.stddev_dur);
```

**Info Alerts** (Daily review):
```sql
-- High retry count
SELECT job_id, retry, max_retry, error_message
FROM olap_dev.execution_log
WHERE data_date = CURRENT_DATE
  AND retry > 0
ORDER BY retry DESC;
```

---

### 3. CloudWatch Alarms (if using AWS)

```bash
# Alert on failed jobs
aws cloudwatch put-metric-alarm \
  --alarm-name dbt-job-failures \
  --alarm-description "Alert when DBT jobs fail" \
  --metric-name FailedJobs \
  --namespace DataPipeline \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 1 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:region:account:alert-topic

# Alert on stale data
aws cloudwatch put-metric-alarm \
  --alarm-name dbt-jobs-not-run \
  --alarm-description "Alert when expected jobs didn't run" \
  --metric-name JobsNotRun \
  --namespace DataPipeline \
  --statistic Sum \
  --period 3600 \
  --evaluation-periods 1 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:region:account:alert-topic
```

---

### 4. Slack Notifications

**Example webhook integration**:
```python
import requests
import json

def send_slack_alert(job_id, status, error_message):
    webhook_url = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
    
    message = {
        "text": f"🔴 DBT Job Alert",
        "attachments": [{
            "color": "danger" if status == "FAILED" else "warning",
            "fields": [
                {"title": "Job ID", "value": job_id, "short": True},
                {"title": "Status", "value": status, "short": True},
                {"title": "Error", "value": error_message, "short": False}
            ],
            "footer": "Data Pipeline Monitor",
            "ts": int(time.time())
        }]
    }
    
    requests.post(webhook_url, json=message)
```

---

## Troubleshooting Guide

### Issue 1: Job Not Running

**Symptoms**: Job shows "NOT_STARTED" status

**Diagnosis**:
```sql
-- Check if job is scheduled
SELECT id, cron_expression, schedule_type
FROM olap_dev.job_config
WHERE id = 'your_job_id';

-- Check dependency status
WITH job_deps AS (
    SELECT jsonb_array_elements_text(depend_ons) as dep_id
    FROM olap_dev.job_config
    WHERE id = 'your_job_id'
)
SELECT 
    jd.dep_id,
    log.status,
    log.end_time
FROM job_deps jd
LEFT JOIN olap_dev.execution_log log 
    ON jd.dep_id = log.job_id 
    AND log.data_date = CURRENT_DATE
ORDER BY log.end_time DESC;
```

**Common Causes & Solutions**:

1. **Dependencies not met**:
   - Check upstream job statuses
   - Verify dependency_status in execution_log
   - Manual trigger upstream jobs if needed

2. **Not scheduled for today**:
   - Check cron_expression
   - Verify schedule_type
   - Manual trigger if needed

3. **Scheduler not running**:
   - Check orchestrator service status
   - Review scheduler logs
   - Restart scheduler if necessary

4. **Configuration issue**:
   - Validate job_config
   - Check for recent config changes
   - Verify IAM permissions

---

### Issue 2: Job Stuck in PENDING

**Symptoms**: Job status = 'PENDING' for extended period

**Diagnosis**:
```sql
-- Check how long job has been pending
SELECT 
    job_id,
    status,
    start_time,
    NOW() - start_time as pending_duration,
    dependency_status
FROM olap_dev.execution_log
WHERE status = 'PENDING'
  AND data_date = CURRENT_DATE
ORDER BY start_time;
```

**Common Causes & Solutions**:

1. **Waiting for dependencies**:
   ```sql
   -- Check dependency completion
   SELECT 
       log.job_id,
       log.depend_ons,
       log.dependency_status
   FROM olap_dev.execution_log log
   WHERE log.status = 'PENDING'
     AND log.data_date = CURRENT_DATE;
   ```
   - Wait for upstream jobs
   - Investigate stuck upstream jobs

2. **Resource constraints**:
   - Check concurrent execution limits
   - Review system resources (CPU, memory)
   - Check database connection pool

3. **Queue backlog**:
   ```sql
   -- Count jobs in queue
   SELECT COUNT(*) as queue_depth
   FROM olap_dev.execution_log
   WHERE status IN ('PENDING', 'QUEUE')
     AND data_date = CURRENT_DATE;
   ```
   - Increase worker capacity
   - Optimize slow jobs
   - Implement priority queues

---

### Issue 3: Job Failures

**Symptoms**: Job status = 'FAILED'

**Diagnosis**:
```sql
-- Get failure details
SELECT 
    job_id,
    error_message,
    retry,
    max_retry,
    start_time,
    end_time,
    duration,
    execution_steps,
    test_results
FROM olap_dev.execution_log
WHERE status = 'FAILED'
  AND data_date = CURRENT_DATE
ORDER BY start_time DESC;
```

**Common Causes & Solutions**:

1. **Data quality issues**:
   ```sql
   -- Check test results
   SELECT 
       job_id,
       test_status,
       test_results
   FROM olap_dev.execution_log
   WHERE test_status = 'FAILED'
     AND data_date = CURRENT_DATE;
   ```
   - Review test failures
   - Investigate source data quality
   - Fix upstream data issues

2. **Schema changes**:
   - Check for upstream schema changes
   - Validate model definitions
   - Update dbt models if needed

3. **Resource exhaustion**:
   - Check for out-of-memory errors
   - Review query complexity
   - Increase resource allocation

4. **Timeout**:
   ```sql
   -- Check if duration exceeded timeout
   SELECT 
       job_id,
       duration,
       timeout,
       CASE 
           WHEN duration >= timeout THEN 'Timeout likely'
           ELSE 'Other failure'
       END as failure_reason
   FROM olap_dev.execution_log log
   JOIN olap_dev.job_config conf ON log.job_id = conf.id
   WHERE log.status = 'FAILED'
     AND log.data_date = CURRENT_DATE;
   ```
   - Increase timeout setting
   - Optimize query performance
   - Break into smaller jobs

5. **Connection issues**:
   - Check database connectivity
   - Verify credentials
   - Review network issues

---

### Issue 4: Slow Job Performance

**Symptoms**: Job duration significantly higher than normal

**Diagnosis**:
```sql
-- Compare with historical performance
WITH historical AS (
    SELECT 
        job_id,
        AVG(duration) as avg_duration,
        STDDEV(duration) as stddev_duration,
        MIN(duration) as min_duration,
        MAX(duration) as max_duration
    FROM olap_dev.execution_log
    WHERE status = 'Completed'
      AND data_date >= CURRENT_DATE - 30
    GROUP BY job_id
)
SELECT 
    log.job_id,
    log.duration as current_duration,
    h.avg_duration,
    h.min_duration,
    h.max_duration,
    ROUND((log.duration - h.avg_duration) / h.avg_duration * 100, 2) as pct_deviation
FROM olap_dev.execution_log log
JOIN historical h ON log.job_id = h.job_id
WHERE log.data_date = CURRENT_DATE
  AND log.duration > h.avg_duration * 1.5
ORDER BY pct_deviation DESC;
```

**Solutions**:

1. **Optimize queries**:
   - Add indexes
   - Rewrite inefficient SQL
   - Use incremental models

2. **Increase parallelization**:
   ```sql
   -- Update threads in config
   UPDATE olap_dev.job_config
   SET threads = 8
   WHERE id = 'slow_job_id';
   ```

3. **Partition large tables**:
   - Implement date partitioning
   - Use appropriate partition keys
   - Optimize partition pruning

4. **Check data volume**:
   ```sql
   -- Check if input data volume increased
   SELECT 
       data_source_name,
       data_date,
       records_processed,
       bytes_processed
   FROM olap_dev.glue_catalog_ingestion_tracking
   WHERE data_date >= CURRENT_DATE - 7
   ORDER BY data_date DESC, records_processed DESC;
   ```

---

### Issue 5: Circular Dependencies

**Symptoms**: Jobs stuck in PENDING, infinite loop

**Diagnosis**:
```sql
-- Detect circular dependencies
WITH RECURSIVE dep_chain AS (
    SELECT 
        id as root_job,
        id as current_job,
        depend_ons,
        1 as depth,
        ARRAY[id] as path
    FROM olap_dev.job_config
    
    UNION ALL
    
    SELECT 
        dc.root_job,
        jsonb_array_elements_text(dc.depend_ons)::text as current_job,
        jc.depend_ons,
        dc.depth + 1,
        dc.path || jsonb_array_elements_text(dc.depend_ons)::text
    FROM dep_chain dc
    JOIN olap_dev.job_config jc 
        ON jc.id = jsonb_array_elements_text(dc.depend_ons)::text
    WHERE dc.depth < 20
      AND NOT (jsonb_array_elements_text(dc.depend_ons)::text = ANY(dc.path))
)
SELECT 
    root_job,
    current_job,
    path,
    depth
FROM dep_chain
WHERE current_job = root_job
  AND depth > 1;
```

**Solutions**:
1. Review and fix dependency configuration
2. Remove circular references
3. Restructure job dependencies
4. Implement dependency validation

---

## Performance Optimization

### 1. Query Optimization

**Optimize Node Graph Query**:
```sql
-- Add indexes for faster lookups
CREATE INDEX IF NOT EXISTS idx_job_config_id 
ON olap_dev.job_config (id);

CREATE INDEX IF NOT EXISTS idx_execution_log_job_date_status 
ON olap_dev.execution_log (job_id, data_date, status);

CREATE INDEX IF NOT EXISTS idx_glue_tracking_source_date_status
ON olap_dev.glue_catalog_ingestion_tracking (data_source_name, data_date, status);
```

**Materialized View for Job Statistics**:
```sql
CREATE MATERIALIZED VIEW olap_dev.mv_daily_job_stats AS
SELECT 
    data_date,
    COUNT(*) as total_jobs,
    COUNT(*) FILTER (WHERE status = 'Completed') as completed_jobs,
    COUNT(*) FILTER (WHERE status = 'FAILED') as failed_jobs,
    AVG(duration) as avg_duration,
    MAX(duration) as max_duration
FROM olap_dev.execution_log
GROUP BY data_date;

-- Refresh daily
REFRESH MATERIALIZED VIEW olap_dev.mv_daily_job_stats;
```

---

### 2. Dashboard Performance

**Best Practices**:
1. **Limit date range**: Don't query "All" dates unnecessarily
2. **Use specific job filters**: Avoid "All" jobs when possible
3. **Keep level low**: Start with level 1-2, increase only if needed
4. **Set refresh rate appropriately**: 5s refresh only for active monitoring

**Query Caching**:
```sql
-- Cache frequently accessed data
CREATE TABLE olap_dev.cache_job_dependencies AS
SELECT 
    parent.id as parent_job,
    child.id as child_job,
    1 as depth
FROM olap_dev.job_config parent
CROSS JOIN olap_dev.job_config child
WHERE child.depend_ons ? parent.id;

-- Refresh cache periodically
TRUNCATE olap_dev.cache_job_dependencies;
INSERT INTO olap_dev.cache_job_dependencies 
SELECT ...;
```

---

### 3. Database Maintenance

**Regular Maintenance Tasks**:
```sql
-- Vacuum and analyze tables
VACUUM ANALYZE olap_dev.job_config;
VACUUM ANALYZE olap_dev.execution_log;
VACUUM ANALYZE olap_dev.glue_catalog_ingestion_tracking;

-- Update statistics
ANALYZE olap_dev.job_config;
ANALYZE olap_dev.execution_log;

-- Reindex periodically
REINDEX TABLE olap_dev.job_config;
REINDEX TABLE olap_dev.execution_log;
```

**Archive Old Data**:
```sql
-- Archive logs older than retention period
INSERT INTO olap_dev.execution_log_archive
SELECT * FROM olap_dev.execution_log
WHERE data_date < CURRENT_DATE - INTERVAL '90 days';

DELETE FROM olap_dev.execution_log
WHERE data_date < CURRENT_DATE - INTERVAL '90 days';
```

---

## Best Practices

### 1. Job Configuration

**Naming Conventions**:
```
raw_<source>_<table>          - Raw ingestion jobs
stg_<source>_<table>          - Staging models
int_<domain>_<description>    - Intermediate models
dim_<entity>                  - Dimension tables
fct_<event>                   - Fact tables
mart_<business_area>          - Business marts
```

**Dependency Management**:
- Keep dependency chains shallow (< 5 levels)
- Group related jobs into logical layers
- Minimize cross-layer dependencies
- Document complex dependency rationale

**Resource Allocation**:
```sql
-- Set appropriate timeouts based on historical data
UPDATE olap_dev.job_config
SET timeout = CEIL(avg_duration * 2)
FROM (
    SELECT 
        job_id,
        AVG(duration) as avg_duration
    FROM olap_dev.execution_log
    WHERE status = 'Completed'
      AND data_date >= CURRENT_DATE - 30
    GROUP BY job_id
) hist
WHERE job_config.id = hist.job_id;
```

---

### 2. Monitoring

**Daily Review Checklist**:
- [ ] Check job completion rate (target: > 98%)
- [ ] Review failed jobs và error messages
- [ ] Verify critical jobs completed
- [ ] Check jobs not run vs expected
- [ ] Review performance outliers
- [ ] Monitor retry patterns
- [ ] Validate test results
- [ ] Check raw job ingestion status

**Weekly Review**:
- [ ] Analyze performance trends
- [ ] Review dependency structure
- [ ] Update timeout settings
- [ ] Optimize slow jobs
- [ ] Clean up orphaned jobs
- [ ] Update documentation
- [ ] Review alert thresholds

**Monthly Review**:
- [ ] Capacity planning
- [ ] Archive old logs
- [ ] Database maintenance
- [ ] Security audit
- [ ] Cost optimization
- [ ] Update runbooks

---

### 3. Incident Response

**Severity Levels**:

**P0 - Critical** (Immediate response):
- Critical business jobs failed
- Complete pipeline down
- Data corruption detected
- Security breach

**P1 - High** (Response within 1 hour):
- Multiple jobs failed
- SLA at risk
- Performance degradation
- Data quality issues

**P2 - Medium** (Response within 4 hours):
- Single job failure
- Minor performance issues
- Non-critical test failures

**P3 - Low** (Response within 1 day):
- Warnings
- Optimization opportunities
- Documentation updates

**Incident Response Process**:
1. **Detect**: Alert received or issue discovered
2. **Assess**: Determine severity và impact
3. **Communicate**: Notify stakeholders
4. **Investigate**: Use dashboard để diagnose
5. **Resolve**: Fix root cause
6. **Validate**: Verify resolution
7. **Document**: Update runbook
8. **Post-mortem**: Prevent recurrence

---

### 4. Documentation

**Required Documentation**:
- Job purpose và business logic
- Dependencies rationale
- Data sources và targets
- SLA requirements
- On-call procedures
- Runbooks for common issues
- Architecture diagrams
- Contact information

---

## Integration với Other Tools

### 1. dbt Cloud Integration

```python
# Trigger dbt Cloud job from orchestrator
import requests

def trigger_dbt_job(job_id, cause):
    url = f"https://cloud.getdbt.com/api/v2/accounts/{account_id}/jobs/{job_id}/run/"
    headers = {
        "Authorization": f"Token {dbt_api_token}",
        "Content-Type": "application/json"
    }
    payload = {
        "cause": cause
    }
    response = requests.post(url, headers=headers, json=payload)
    return response.json()
```

---

### 2. Airflow Integration

```python
from airflow import DAG
from airflow.operators.postgres_operator import PostgresOperator

# Check dependencies before running
check_dependencies = PostgresOperator(
    task_id='check_dependencies',
    postgres_conn_id='olap_dev',
    sql="""
        SELECT COUNT(*) as pending_deps
        FROM olap_dev.execution_log
        WHERE job_id IN (
            SELECT jsonb_array_elements_text(depend_ons)
            FROM olap_dev.job_config
            WHERE id = '{{ params.job_id }}'
        )
        AND data_date = '{{ ds }}'
        AND status != 'Completed'
    """
)
```

---

### 3. Slack Notifications

```python
from slack_sdk import WebClient

def notify_job_status(job_id, status, data_date):
    client = WebClient(token=slack_token)
    
    color = {
        'Completed': 'good',
        'FAILED': 'danger',
        'RUNNING': 'warning'
    }.get(status, '#808080')
    
    client.chat_postMessage(
        channel='#data-pipeline',
        attachments=[{
            'color': color,
            'title': f'Job {job_id} - {status}',
            'fields': [
                {'title': 'Date', 'value': data_date, 'short': True},
                {'title': 'Status', 'value': status, 'short': True}
            ]
        }]
    )
```

---

## Tài Liệu Tham Khảo

### Internal Resources
- **Job Config Documentation**: Confluence link
- **Runbooks**: Wiki link  
- **On-call Guide**: PagerDuty link
- **Architecture Diagrams**: Lucidchart link

### External Resources
- **dbt Documentation**: https://docs.getdbt.com/
- **PostgreSQL Recursive CTE**: https://www.postgresql.org/docs/current/queries-with.html
- **Grafana Node Graph**: https://grafana.com/docs/grafana/latest/panels/visualizations/node-graph/
- **JSONB Operators**: https://www.postgresql.org/docs/current/functions-json.html

---

**Dashboard Version**: 27  
**Last Updated**: 2026-02-10  
**Maintained By**: Data Engineering Team  
**Support**: #data-pipeline-support
