# Tài liệu Dashboard AWS ECS

## Tổng quan
Dashboard này được thiết kế để giám sát và theo dõi các metrics của Amazon ECS (Elastic Container Service). Dashboard cung cấp cái nhìn toàn diện về CPU utilization, memory usage, và resource reservation của ECS clusters và services.

**Nguồn dữ liệu**: CloudWatch  
**Grafana Dashboard ID**: 551  
**Khoảng thời gian mặc định**: 24 giờ gần nhất  

---

## Kiến trúc AWS ECS

### ECS Components

**Cluster**:
- Logical grouping của tasks và services
- Có thể sử dụng EC2 hoặc Fargate launch type
- Chứa nhiều services và tasks
- Network và security boundary

**Service**:
- Định nghĩa và maintain số lượng tasks running
- Load balancing và service discovery
- Auto-scaling capabilities
- Rolling updates và deployment strategies

**Task**:
- Đơn vị thực thi nhỏ nhất
- Chạy một hoặc nhiều containers
- Defined by Task Definition
- Ephemeral hoặc long-running

**Task Definition**:
- Blueprint cho tasks
- Định nghĩa container images, CPU, memory
- Environment variables, volumes
- Networking configuration

**Container**:
- Docker container instance
- Runs application code
- Isolated execution environment
- Defined in Task Definition

### Launch Types

**Fargate**:
- Serverless container execution
- AWS manages infrastructure
- Pay per vCPU và memory
- No EC2 instance management
- Simplified operations

**EC2**:
- Containers run on EC2 instances
- More control over infrastructure
- ECS Agent manages container placement
- Cost optimization với Reserved Instances
- Access to instance-level features

---

## Cấu hình

### Biến Template (Template Variables)

Dashboard sử dụng các biến động để cho phép người dùng tùy chỉnh view:

1. **datasource**: Nguồn dữ liệu CloudWatch
2. **region**: Vùng AWS (AWS Region)
3. **cluster**: ECS Cluster name cần giám sát
4. **service**: ECS Service name trong cluster đã chọn

**Workflow**:
1. Chọn Region → Load danh sách Clusters
2. Chọn Cluster → Load danh sách Services
3. Chọn Service → Hiển thị service-specific metrics

---

## Chi tiết các Dashboard Panel

### 1. CPUUtilization

**Vị trí**: Hàng 1, chiều cao 7 đơn vị, chiều rộng 24 đơn vị  
**Loại biểu đồ**: Time series (Line chart)

**Mô tả**:  
Hiển thị CPU utilization của ECS service đã chọn. Metric này cho biết phần trăm CPU được sử dụng so với tổng CPU đã allocated cho service.

**Metrics**:
- Namespace: `AWS/ECS`
- Metric: `CPUUtilization`
- Statistic: `Average`
- Dimensions: 
  - `ClusterName`: $cluster
  - `ServiceName`: $service
- Unit: Percent (%)

**Cấu hình hiển thị**:
- Range: 0-100%
- Line chart với fill opacity 10%
- Legend: Hiển thị mean, max, min values
- Alert threshold hiển thị (màu đỏ tại 80%)

**Ý nghĩa & Phân tích**:

### CPU Utilization Thresholds

**Health Levels**:
- **< 40%**: Under-utilized - Consider downsizing
- **40-60%**: Optimal range - Healthy utilization
- **60-75%**: Elevated - Monitor closely
- **75-85%**: High - Plan for scaling
- **85-95%**: Very high - Scale immediately
- **> 95%**: Critical - Performance degradation

### CPU vs CPU Reservation

**Important Distinction**:
- **CPUUtilization**: Actual CPU usage (this panel)
- **CPUReservation**: How much CPU is reserved/allocated

**Example**:
```
Task Definition: 512 CPU units (0.5 vCPU)
Actual Usage: 256 CPU units
CPUUtilization: 50% (256/512)
```

### Nguyên nhân CPU cao

**Application-Level**:
- Inefficient code
- CPU-intensive operations (encoding, compression)
- Synchronous processing
- Memory leaks causing thrashing
- Infinite loops or deadlocks

**Container-Level**:
- Too many containers per task
- Insufficient CPU allocation
- Resource contention
- Noisy neighbor (EC2 launch type)

**Traffic-Related**:
- High request volume
- DDoS attacks
- Load balancer misconfiguration
- No auto-scaling configured

### Optimization Strategies

**Right-Sizing**:
```json
// Task Definition - CPU allocation
{
  "containerDefinitions": [{
    "name": "app",
    "cpu": 512,  // 0.5 vCPU
    "memory": 1024,
    "essential": true
  }]
}
```

**CPU Units Mapping**:
- 256 CPU units = 0.25 vCPU
- 512 CPU units = 0.5 vCPU
- 1024 CPU units = 1 vCPU
- 2048 CPU units = 2 vCPU

**Auto-Scaling Configuration**:
```json
// Target Tracking Scaling Policy
{
  "TargetValue": 70.0,
  "PredefinedMetricSpecification": {
    "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
  },
  "ScaleInCooldown": 300,
  "ScaleOutCooldown": 60
}
```

**Application Optimization**:
```python
# Use async/concurrent processing
import asyncio

async def process_item(item):
    # Non-blocking I/O operations
    result = await external_api_call(item)
    return result

async def main():
    tasks = [process_item(item) for item in items]
    results = await asyncio.gather(*tasks)
```

**Container Best Practices**:
- Use multi-stage Docker builds
- Minimize layer count
- Use Alpine base images where possible
- Remove unnecessary dependencies
- Implement health checks

**Monitoring Tips**:
- Set alert when CPU > 80% for 5+ minutes
- Track CPU patterns (peak hours)
- Compare with task count (scale correlation)
- Monitor ECS Agent CPU (EC2 launch type)
- Review CloudWatch Container Insights

---

### 2. MemoryUtilization

**Vị trí**: Hàng 2, chiều cao 7 đơn vị, chiều rộng 24 đơn vị  
**Loại biểu đồ**: Time series (Line chart)

**Mô tả**:  
Hiển thị memory utilization của ECS service. Cho biết phần trăm memory được sử dụng so với memory đã allocated.

**Metrics**:
- Namespace: `AWS/ECS`
- Metric: `MemoryUtilization`
- Statistic: `Average`
- Dimensions: 
  - `ClusterName`: $cluster
  - `ServiceName`: $service
- Unit: Percent (%)

**Cấu hình hiển thị**:
- Range: 0-100%
- Line chart với fill opacity 10%
- Legend: Hiển thị mean, max, min values
- Alert threshold tại 80%

**Ý nghĩa & Phân tích**:

### Memory Utilization Thresholds

**Health Levels**:
- **< 50%**: Under-utilized - Consider optimization
- **50-70%**: Optimal range
- **70-80%**: Elevated - Monitor
- **80-90%**: High - Scale or optimize
- **90-95%**: Very high - Immediate action
- **> 95%**: Critical - OOM risk

### Memory Issues & Impact

**Out of Memory (OOM)**:
- Task gets killed by ECS
- Service restarts task automatically
- Application unavailability
- Data loss (if not persistent)
- Cascading failures

**Memory Pressure Symptoms**:
- Increased latency
- Slow response times
- Swapping (EC2 launch type)
- Garbage collection pauses
- Container restarts

### Memory vs Memory Reservation

**Understanding Metrics**:
- **MemoryUtilization**: Actual memory usage (this panel)
- **MemoryReservation**: Memory allocated/reserved

**Example**:
```
Task Definition: 2048 MB memory
Actual Usage: 1536 MB
MemoryUtilization: 75% (1536/2048)
```

### Memory Optimization

**Task Definition Memory**:
```json
{
  "containerDefinitions": [{
    "name": "app",
    "memory": 2048,  // Hard limit (MB)
    "memoryReservation": 1024,  // Soft limit (MB)
    "essential": true
  }]
}
```

**Memory Configuration**:
- **memory (hard limit)**: Container killed if exceeded
- **memoryReservation (soft limit)**: Guaranteed memory
- Set hard limit 20-30% above typical usage
- Use soft limit for burstable workloads

**Application-Level Optimization**:

**Java Applications**:
```dockerfile
# Set JVM heap size appropriately
ENV JAVA_OPTS="-Xms512m -Xmx1536m -XX:+UseG1GC"

# If container has 2048MB, leave ~512MB for non-heap
```

**Node.js Applications**:
```dockerfile
# Set V8 heap size
ENV NODE_OPTIONS="--max-old-space-size=1536"

# If container has 2048MB, leave ~512MB for buffers
```

**Python Applications**:
```python
# Limit memory usage
import resource

# Set memory limit (bytes)
resource.setrlimit(resource.RLIMIT_AS, (1610612736, 1610612736))  # 1.5GB
```

**Memory Leak Detection**:
```python
# Monitor memory growth
import tracemalloc

tracemalloc.start()

# Your application code

snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics('lineno')

for stat in top_stats[:10]:
    print(stat)
```

**Caching Strategies**:
```python
# Use LRU cache to limit memory
from functools import lru_cache

@lru_cache(maxsize=1000)
def expensive_function(arg):
    # Function implementation
    pass

# Clear cache when needed
expensive_function.cache_clear()
```

**Auto-Scaling on Memory**:
```json
{
  "TargetValue": 75.0,
  "PredefinedMetricSpecification": {
    "PredefinedMetricType": "ECSServiceAverageMemoryUtilization"
  },
  "ScaleInCooldown": 300,
  "ScaleOutCooldown": 60
}
```

**Monitoring Best Practices**:
- Alert when memory > 80% sustained
- Track memory growth rate
- Monitor OOM kill events
- Review application memory profiling
- Use Container Insights for detailed metrics
- Check for memory leaks regularly

---

### 3. CPUReservation

**Vị trí**: Hàng 3, chiều cao 7 đơn vị, chiều rộng 24 đơn vị  
**Loại biểu đồ**: Time series (Line chart)

**Mô tả**:  
Hiển thị CPU reservation của ECS cluster (không phải service). Cho biết phần trăm CPU resources của cluster đã được reserved/allocated bởi các tasks.

**Metrics**:
- Namespace: `AWS/ECS`
- Metric: `CPUReservation`
- Statistic: `Average`
- Dimensions: 
  - `ClusterName`: $cluster
  - **Note**: Không có ServiceName dimension (cluster-level metric)
- Unit: Percent (%)

**Cấu hình hiển thị**:
- Range: 0-100%
- Line chart màu vàng (#EAB839)
- Legend: Hiển thị mean, max, min values

**Ý nghĩa & Phân tích**:

### CPU Reservation vs CPU Utilization

**Key Differences**:

| Metric | Level | What It Measures |
|--------|-------|------------------|
| CPUUtilization | Service | Actual CPU usage by running containers |
| CPUReservation | Cluster | Percentage of cluster CPU allocated to tasks |

**Example Scenario**:
```
Cluster Capacity: 4 vCPU (4096 CPU units)
Task 1: Reserved 1 vCPU (1024 units)
Task 2: Reserved 2 vCPU (2048 units)
Task 3: Reserved 0.5 vCPU (512 units)

CPUReservation = (1024 + 2048 + 512) / 4096 = 87.5%

Even if tasks are idle (CPUUtilization = 0%), 
CPUReservation remains 87.5%
```

### CPU Reservation Thresholds

**Cluster Capacity Planning**:
- **< 50%**: Under-utilized cluster
- **50-70%**: Healthy utilization
- **70-80%**: Monitor capacity
- **80-90%**: Plan for scaling
- **> 90%**: Add capacity immediately

**Impact of High Reservation**:
- No room for new tasks
- Tasks pending placement
- Service scaling blocked
- Deployment failures

### Launch Type Considerations

**Fargate Launch Type**:
- CPUReservation not as critical
- AWS manages capacity automatically
- Tasks get resources when requested
- Pay per task resources

**EC2 Launch Type**:
- CPUReservation directly affects capacity
- Must monitor cluster capacity
- Need to scale EC2 instances
- Bin-packing optimization important

### Capacity Planning

**Calculate Required Capacity**:
```python
# Desired task count
desired_tasks = 10
cpu_per_task = 512  # CPU units

# Total CPU needed
total_cpu_needed = desired_tasks * cpu_per_task
# = 5120 CPU units = 5 vCPU

# If each EC2 instance has 2 vCPU
instances_needed = total_cpu_needed / 2048
# = 2.5 → Need 3 instances minimum
```

**Reservation Optimization**:

1. **Right-Size Task Definitions**:
```json
// Don't over-allocate
{
  "cpu": 256,  // Start small
  "memory": 512
}

// Monitor actual usage
// Scale up if needed
```

2. **Use Spot Instances** (EC2):
```bash
# Mix On-Demand and Spot
# 50% cost savings on Spot
# ECS handles interruptions gracefully
```

3. **Implement Bin-Packing**:
```json
// Task placement strategy
{
  "placementStrategy": [{
    "type": "binpack",
    "field": "cpu"
  }]
}
```

### Cluster Scaling (EC2 Launch Type)

**Auto Scaling Group**:
```json
{
  "TargetTrackingConfiguration": {
    "TargetValue": 75.0,
    "CustomizedMetricSpecification": {
      "MetricName": "CPUReservation",
      "Namespace": "AWS/ECS",
      "Statistic": "Average"
    }
  }
}
```

**Capacity Providers** (Recommended):
```bash
# Create capacity provider
aws ecs create-capacity-provider \
  --name my-capacity-provider \
  --auto-scaling-group-provider \
    autoScalingGroupArn=arn:aws:autoscaling:...,\
    managedScaling={status=ENABLED,targetCapacity=80}

# Associate with cluster
aws ecs put-cluster-capacity-providers \
  --cluster my-cluster \
  --capacity-providers my-capacity-provider \
  --default-capacity-provider-strategy \
    capacityProvider=my-capacity-provider,weight=1
```

**Manual Calculation**:
```bash
# Check current capacity
aws ecs describe-clusters --clusters my-cluster

# Check container instances
aws ecs list-container-instances --cluster my-cluster

# Describe instance resources
aws ecs describe-container-instances \
  --cluster my-cluster \
  --container-instances <instance-id>
```

**Monitoring & Alerts**:
- Alert when CPUReservation > 80%
- Track reservation trends
- Monitor task pending count
- Review capacity provider metrics
- Alert on task placement failures

---

### 4. MemoryReservation

**Vị trí**: Hàng 4, chiều cao 7 đơn vị, chiều rộng 24 đơn vị  
**Loại biểu đồ**: Time series (Line chart)

**Mô tả**:  
Hiển thị memory reservation của ECS cluster. Cho biết phần trăm memory resources của cluster đã được reserved/allocated bởi các tasks.

**Metrics**:
- Namespace: `AWS/ECS`
- Metric: `MemoryReservation`
- Statistic: `Average`
- Dimensions: 
  - `ClusterName`: $cluster
  - **Note**: Cluster-level metric (không có ServiceName)
- Unit: Percent (%)

**Cấu hình hiển thị**:
- Range: 0-100%
- Line chart màu vàng (#EAB839)
- Legend: Hiển thị mean, max, min values

**Ý nghĩa & Phân tích**:

### Memory Reservation vs Memory Utilization

**Key Differences**:

| Metric | Level | What It Measures |
|--------|-------|------------------|
| MemoryUtilization | Service | Actual memory usage by containers |
| MemoryReservation | Cluster | Percentage of cluster memory allocated to tasks |

**Example Scenario**:
```
Cluster Memory Capacity: 16 GB (16384 MB)
Task 1: Reserved 2 GB (2048 MB)
Task 2: Reserved 4 GB (4096 MB)
Task 3: Reserved 1 GB (1024 MB)

MemoryReservation = (2048 + 4096 + 1024) / 16384 = 43.75%

Even if actual usage is lower,
Memory is reserved and unavailable for other tasks
```

### Memory Reservation Thresholds

**Cluster Capacity Planning**:
- **< 50%**: Under-utilized
- **50-70%**: Healthy range
- **70-80%**: Monitor closely
- **80-90%**: Add capacity soon
- **> 90%**: Critical - Add capacity now

**Consequences of High Reservation**:
- Cannot place new tasks
- Service scaling failures
- Deployment blocked
- Tasks in PENDING state
- Increased deployment time

### Hard Limit vs Soft Limit

**Task Definition Memory Configuration**:

```json
{
  "containerDefinitions": [{
    "name": "app",
    "memory": 2048,          // Hard limit
    "memoryReservation": 1024 // Soft limit (reservation)
  }]
}
```

**Behavior**:
- **memoryReservation (soft)**: Guaranteed memory, used for placement
- **memory (hard)**: Maximum memory, container killed if exceeded
- If only hard limit set: Both reservation and limit are same
- If only soft limit set: No hard limit (can use more if available)

**Best Practices**:
```json
// Recommended configuration
{
  "memory": 2048,           // Set 20-30% above typical
  "memoryReservation": 1536 // Set to typical usage
}

// Allows:
// - Guaranteed 1536 MB
// - Can burst to 2048 MB if available
// - Better bin-packing
```

### Memory Reservation Optimization

**Right-Sizing Strategy**:

1. **Monitor Actual Usage**:
```bash
# Get task memory stats
aws ecs describe-tasks \
  --cluster my-cluster \
  --tasks <task-id> \
  --include METRICS

# Review Container Insights
# Check CloudWatch Logs Insights
```

2. **Adjust Reservations**:
```python
# Analysis script
actual_usage = 1200  # MB observed
buffer = 1.3  # 30% buffer

recommended_reservation = actual_usage
recommended_hard_limit = actual_usage * buffer

print(f"memoryReservation: {recommended_reservation}")
print(f"memory: {recommended_hard_limit}")
```

3. **Use Appropriate Values**:
```
Application Type → Recommended Settings

Static website:
  memoryReservation: 128
  memory: 256

API service:
  memoryReservation: 512
  memory: 1024

Data processing:
  memoryReservation: 2048
  memory: 4096
```

### Cluster Capacity Management

**Calculate Memory Needs**:
```python
# Service requirements
desired_count = 20
memory_per_task = 1024  # MB

# Total memory needed
total_memory = desired_count * memory_per_task
# = 20480 MB = 20 GB

# If each EC2 instance has 8 GB
instances_needed = total_memory / 8192
# = 2.5 → Need 3 instances
```

**Capacity Provider Auto-Scaling**:
```json
{
  "managedScaling": {
    "status": "ENABLED",
    "targetCapacity": 80,  // Scale at 80% reservation
    "minimumScalingStepSize": 1,
    "maximumScalingStepSize": 10
  }
}
```

**Mixed Instance Types**:
```bash
# Use different instance types for flexibility
# Memory-optimized: r6g, r6i, r5
# General purpose: t3, t4g, m5, m6g
# Compute-optimized: c6g, c6i, c5

# Example Auto Scaling Group
Instance types:
- m5.large (8 GB)
- m5.xlarge (16 GB)
- m5.2xlarge (32 GB)
```

### Task Placement Strategies

**Bin-Packing for Memory**:
```json
{
  "placementStrategy": [{
    "type": "binpack",
    "field": "memory"
  }]
}
```

**Benefits**:
- Efficient resource utilization
- Fewer EC2 instances needed
- Lower costs
- Better consolidation

**Spread Strategy** (for HA):
```json
{
  "placementStrategy": [{
    "type": "spread",
    "field": "attribute:ecs.availability-zone"
  }]
}
```

### Monitoring & Troubleshooting

**Key Metrics to Track**:
```
MemoryReservation + RegisteredContainerInstancesCount
→ Understand total capacity

MemoryReservation + PendingTasksCount
→ Identify capacity issues

MemoryUtilization vs MemoryReservation
→ Find over-provisioning
```

**Common Issues**:

1. **Tasks Pending**:
```bash
# Check why tasks aren't placing
aws ecs describe-tasks \
  --cluster my-cluster \
  --tasks <task-id>

# Look for placement failures
# Common: Insufficient memory
```

2. **Over-Reservation**:
```
MemoryReservation: 85%
Actual MemoryUtilization: 45%

→ Tasks are over-provisioned
→ Reduce memory allocations
→ Can fit more tasks
```

3. **Fragmentation**:
```
Total cluster memory: 100 GB
Available: 40 GB
MemoryReservation: 60%

Cannot place 8GB task because:
- Memory fragmented across instances
- No single instance has 8GB free

Solution: Consolidate or add capacity
```

**Alerts Configuration**:
- Alert: MemoryReservation > 80%
- Alert: Tasks in PENDING state
- Alert: Capacity provider scaling events
- Monitor: Task placement failures
- Track: Memory reservation trends

---

## Hướng dẫn sử dụng

### 1. Chọn Data Source
- Dropdown **Datasource**: Chọn CloudWatch
- Mặc định: cloudwatch

### 2. Chọn AWS Region
- Dropdown **Region**: Chọn region
- Dashboard sẽ load clusters trong region

### 3. Chọn ECS Cluster
- Dropdown **Cluster**: Chọn cluster name
- Load danh sách từ ECS API

### 4. Chọn ECS Service
- Dropdown **Service**: Chọn service trong cluster
- Panels 1-2: Service-level metrics
- Panels 3-4: Cluster-level metrics (không thay đổi)

### 5. Điều chỉnh Time Range
- Time picker: Góc trên bên phải
- Mặc định: 24 giờ gần nhất
- Recommendations:
  - Real-time: Last 1h - Last 6h
  - Daily: Last 24h
  - Trends: Last 7d - Last 30d

### 6. Analyze Metrics
- Compare service metrics (CPU, Memory)
- Review cluster capacity (Reservation)
- Identify scaling needs
- Detect performance issues

---

## Best Practices

### 1. Resource Allocation

**Task Definition Best Practices**:
```json
{
  "family": "my-app",
  "cpu": "512",  // Start with minimum needed
  "memory": "1024",
  "containerDefinitions": [{
    "name": "app",
    "cpu": 512,
    "memory": 896,  // Leave headroom
    "memoryReservation": 768,
    "essential": true,
    "portMappings": [{
      "containerPort": 8080,
      "protocol": "tcp"
    }]
  }]
}
```

**Resource Sizing Guidelines**:
- Start small, scale up based on metrics
- Set hard limit 20-30% above typical usage
- Use soft limit for guaranteed resources
- Leave 10-20% headroom for bursts
- Monitor actual usage regularly

### 2. Auto-Scaling Configuration

**Service Auto-Scaling**:
```json
{
  "ServiceName": "my-service",
  "MinCapacity": 2,
  "MaxCapacity": 10,
  "TargetTrackingScalingPolicies": [
    {
      "PolicyName": "cpu-scale",
      "TargetValue": 70.0,
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization",
      "ScaleInCooldown": 300,
      "ScaleOutCooldown": 60
    },
    {
      "PolicyName": "memory-scale",
      "TargetValue": 75.0,
      "PredefinedMetricType": "ECSServiceAverageMemoryUtilization",
      "ScaleInCooldown": 300,
      "ScaleOutCooldown": 60
    }
  ]
}
```

**Capacity Provider Auto-Scaling** (EC2):
```bash
aws ecs create-capacity-provider \
  --name my-capacity-provider \
  --auto-scaling-group-provider \
    autoScalingGroupArn=arn:aws:autoscaling:region:account:asg-name,\
    managedScaling={status=ENABLED,targetCapacity=80,minimumScalingStepSize=1,maximumScalingStepSize=10},\
    managedTerminationProtection=ENABLED
```

### 3. Monitoring Strategy

**Daily Checks**:
- CPUUtilization: All services < 80%
- MemoryUtilization: All services < 80%
- CPUReservation: Cluster < 85%
- MemoryReservation: Cluster < 85%
- No tasks in PENDING state

**Weekly Reviews**:
- Resource utilization trends
- Scaling events analysis
- Cost optimization opportunities
- Task failure patterns
- Deployment success rate

**Monthly Planning**:
- Capacity forecasting
- Instance type optimization
- Reserved Instance purchases
- Cost vs performance trade-offs
- Architecture improvements

### 4. Alert Configuration

**Critical Alerts**:
```yaml
# CPU Utilization High
- Metric: CPUUtilization
  Threshold: > 85%
  Duration: 5 minutes
  Action: Page on-call engineer

# Memory Utilization High
- Metric: MemoryUtilization
  Threshold: > 85%
  Duration: 5 minutes
  Action: Page on-call engineer

# Cluster Capacity Low
- Metric: CPUReservation or MemoryReservation
  Threshold: > 90%
  Duration: 5 minutes
  Action: Alert operations team

# Tasks Not Running
- Metric: RunningTasksCount
  Threshold: < DesiredCount
  Duration: 2 minutes
  Action: Alert on-call
```

**Warning Alerts**:
```yaml
# Elevated CPU
- Metric: CPUUtilization
  Threshold: > 70%
  Duration: 15 minutes
  Action: Notification

# Elevated Memory
- Metric: MemoryUtilization
  Threshold: > 75%
  Duration: 15 minutes
  Action: Notification

# Capacity Planning
- Metric: CPUReservation or MemoryReservation
  Threshold: > 80%
  Duration: 30 minutes
  Action: Notification
```

### 5. Cost Optimization

**Fargate vs EC2 Analysis**:
```python
# Calculate costs
fargate_vcpu_hour = 0.04048  # us-east-1
fargate_gb_hour = 0.004445

ec2_m5_large_hour = 0.096  # On-Demand
ec2_m5_large_vcpu = 2
ec2_m5_large_gb = 8

# Example workload
tasks = 10
vcpu_per_task = 0.5
gb_per_task = 1

# Fargate cost per hour
fargate_cost = tasks * (vcpu_per_task * fargate_vcpu_hour + 
                        gb_per_task * fargate_gb_hour)

# EC2 cost per hour
instances_needed = (tasks * vcpu_per_task) / ec2_m5_large_vcpu
ec2_cost = instances_needed * ec2_m5_large_hour

print(f"Fargate: ${fargate_cost}/hour")
print(f"EC2: ${ec2_cost}/hour")
```

**Cost Optimization Strategies**:

1. **Use Fargate Spot**:
```json
{
  "capacityProviderStrategy": [{
    "capacityProvider": "FARGATE_SPOT",
    "weight": 1,
    "base": 0
  }]
}
// Up to 70% cost savings
```

2. **Use Savings Plans**:
- Compute Savings Plans: Up to 66% discount
- Commit to 1 or 3 years
- Flexible across instance types

3. **Right-Size Resources**:
```
Review metrics weekly:
- Over-provisioned? → Reduce allocation
- Under-utilized? → Consolidate tasks
- Peak patterns? → Use scheduled scaling
```

4. **Use Spot Instances** (EC2):
```bash
# Mix instance purchasing options
# 50% On-Demand: Baseline capacity
# 50% Spot: Cost savings
# ECS drains tasks before Spot termination
```

### 6. High Availability

**Multi-AZ Deployment**:
```json
{
  "desiredCount": 4,
  "placementStrategy": [{
    "type": "spread",
    "field": "attribute:ecs.availability-zone"
  }]
}
```

**Health Checks**:
```json
{
  "healthCheck": {
    "command": ["CMD-SHELL", "curl -f http://localhost/health || exit 1"],
    "interval": 30,
    "timeout": 5,
    "retries": 3,
    "startPeriod": 60
  }
}
```

**Load Balancing**:
```bash
# Use Application Load Balancer
# Target group health checks
# Drain connections before task stop
# Connection draining: 300 seconds
```

---

## Troubleshooting Common Issues

### Issue 1: High CPU/Memory Utilization

**Symptoms**: CPU or Memory > 85% sustained

**Diagnosis**:
```bash
# Check task details
aws ecs describe-tasks \
  --cluster my-cluster \
  --tasks <task-arn>

# Check CloudWatch Container Insights
# Review application logs
aws logs tail /ecs/my-app --follow
```

**Solutions**:
1. Increase CPU/memory allocation
2. Optimize application code
3. Scale out (add more tasks)
4. Enable auto-scaling
5. Review resource leaks

### Issue 2: Tasks Not Placing

**Symptoms**: Tasks stuck in PENDING

**Diagnosis**:
```bash
# Check task status
aws ecs describe-tasks \
  --cluster my-cluster \
  --tasks <task-arn>

# Look for errors:
# - Insufficient CPU/memory
# - No container instances available
# - IAM permission issues
```

**Solutions**:
1. Increase cluster capacity
2. Reduce task resource requirements
3. Check capacity provider configuration
4. Review task placement constraints
5. Verify IAM roles

### Issue 3: High Reservation, Low Utilization

**Symptoms**: Reservation > 80%, Utilization < 50%

**Diagnosis**:
```
Tasks are over-provisioned
Allocated resources > Actual usage
Wasting capacity and money
```

**Solutions**:
1. Right-size task definitions
2. Reduce CPU/memory allocations
3. Monitor actual usage patterns
4. Adjust incrementally
5. Use soft limits for flexibility

### Issue 4: Frequent Task Restarts

**Symptoms**: Tasks restarting repeatedly

**Diagnosis**:
```bash
# Check stopped tasks
aws ecs list-tasks \
  --cluster my-cluster \
  --desired-status STOPPED

# Review stop reasons
aws ecs describe-tasks \
  --cluster my-cluster \
  --tasks <stopped-task-arns>
```

**Common Causes**:
- Application crashes
- Health check failures
- Out of memory (OOM)
- Failed container commands
- Missing dependencies

**Solutions**:
1. Fix application bugs
2. Adjust health check settings
3. Increase memory allocation
4. Review startup commands
5. Check environment variables

### Issue 5: Slow Deployments

**Symptoms**: Deployments taking too long

**Diagnosis**:
```bash
# Check deployment configuration
aws ecs describe-services \
  --cluster my-cluster \
  --services my-service

# Review:
# - minimumHealthyPercent
# - maximumPercent
# - deployment circuit breaker
```

**Optimization**:
```json
{
  "deploymentConfiguration": {
    "minimumHealthyPercent": 50,
    "maximumPercent": 200,
    "deploymentCircuitBreaker": {
      "enable": true,
      "rollback": true
    }
  }
}
```

### Issue 6: Container Instance Registration Issues

**Symptoms**: EC2 instances not showing in cluster

**Diagnosis**:
```bash
# Check ECS agent logs
ssh into instance
cat /var/log/ecs/ecs-agent.log

# Common issues:
# - ECS agent not running
# - IAM role missing permissions
# - Network connectivity
# - Wrong cluster name
```

**Solutions**:
1. Verify ECS agent running
2. Check IAM instance profile
3. Review security groups
4. Validate cluster name in user data
5. Restart ECS agent

---

## Advanced Monitoring

### Container Insights

**Enable Container Insights**:
```bash
# For cluster
aws ecs update-cluster-settings \
  --cluster my-cluster \
  --settings name=containerInsights,value=enabled

# Provides:
# - Task-level CPU/Memory
# - Network metrics
# - Storage metrics
# - Container-level details
```

**Query Insights**:
```sql
-- CloudWatch Logs Insights query
fields @timestamp, @message
| filter @message like /ERROR/
| filter TaskDefinitionFamily = "my-app"
| stats count() by bin(5m)
```

### Custom Metrics

**Application Metrics**:
```python
# Publish custom metrics
import boto3

cloudwatch = boto3.client('cloudwatch')

cloudwatch.put_metric_data(
    Namespace='MyApp/ECS',
    MetricData=[{
        'MetricName': 'RequestCount',
        'Value': 100,
        'Unit': 'Count',
        'Dimensions': [{
            'Name': 'ServiceName',
            'Value': 'my-service'
        }]
    }]
)
```

### Service Discovery Monitoring

**Monitor Service Discovery**:
```bash
# Check service discovery services
aws servicediscovery list-services

# Check service health
aws servicediscovery get-instances-health-status \
  --service-id srv-xxx
```

---

## Integration với AWS Services

### CloudWatch Alarms

**Create Alarms**:
```bash
# High CPU alarm
aws cloudwatch put-metric-alarm \
  --alarm-name ecs-high-cpu \
  --alarm-description "ECS CPU > 80%" \
  --namespace AWS/ECS \
  --metric-name CPUUtilization \
  --dimensions Name=ServiceName,Value=my-service Name=ClusterName,Value=my-cluster \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:region:account:my-topic
```

### EventBridge Rules

**Task State Change Events**:
```json
{
  "source": ["aws.ecs"],
  "detail-type": ["ECS Task State Change"],
  "detail": {
    "clusterArn": ["arn:aws:ecs:region:account:cluster/my-cluster"],
    "lastStatus": ["STOPPED"],
    "stoppedReason": [{
      "exists": true
    }]
  }
}
```

### SNS Notifications

**Setup Notifications**:
```bash
# Create SNS topic
aws sns create-topic --name ecs-alerts

# Subscribe
aws sns subscribe \
  --topic-arn arn:aws:sns:region:account:ecs-alerts \
  --protocol email \
  --notification-endpoint ops@example.com
```

---

## Kết luận

Dashboard AWS ECS này cung cấp monitoring cơ bản nhưng essential cho:

### Service Performance
- **CPUUtilization**: Computational load per service
- **MemoryUtilization**: Memory usage per service
- Identify performance bottlenecks
- Trigger auto-scaling decisions

### Cluster Capacity
- **CPUReservation**: Available compute capacity
- **MemoryReservation**: Available memory capacity
- Prevent task placement failures
- Capacity planning insights

### Operational Excellence
- Real-time monitoring
- Historical trend analysis
- Quick issue identification
- Informed scaling decisions

**Sử dụng dashboard này để**:
✅ Ensure optimal ECS performance  
✅ Proactive capacity management  
✅ Cost optimization opportunities  
✅ Auto-scaling configuration  
✅ Resource right-sizing  
✅ Troubleshoot task issues  
✅ Plan cluster capacity  

---

**Lưu ý quan trọng**:
- Dashboard cung cấp 4 metrics cơ bản
- Enable Container Insights cho detailed monitoring
- Service metrics (Panels 1-2) require service selection
- Cluster metrics (Panels 3-4) independent of service
- Consider creating separate dashboards cho:
  - Task-level metrics
  - Container-level metrics
  - Network metrics
  - Application-specific metrics

**Recommendations**:
- Add network metrics (NetworkRx/TxBytes)
- Add task count metrics (RunningTasks, PendingTasks)
- Add deployment metrics (SuccessfulDeployments)
- Implement custom application metrics
- Use Container Insights for comprehensive view

**Version**: 1.0  
**Last Updated**: February 2026  
**Maintainer**: Container Platform Team
