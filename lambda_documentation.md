# Tài Liệu Dashboard AWS Lambda

## Tổng Quan

Dashboard AWS Lambda (Grafana ID: 19734) cung cấp giám sát toàn diện cho các Lambda functions, bao gồm metrics về errors, performance, saturation, utilization và logs. Dashboard này sử dụng CloudWatch datasource để thu thập metrics từ AWS Lambda và Lambda Insights.

## Biến Template

Dashboard sử dụng các biến sau:
- **$datasource**: CloudWatch datasource
- **$region**: AWS region của Lambda functions
- **$period**: Khoảng thời gian tổng hợp metrics (auto, 1s, 5s, 10s, 30s, 1m, 5m, 15m, 1h, 6h, 1d, 7d, 30d)
- **$function_name**: Tên của Lambda function cần giám sát
- **$lambda_log_groups**: Nhóm logs của Lambda (hỗ trợ multi-select)
- **$log_filter**: Bộ lọc text cho logs

## Các Panels Chi Tiết

### 1. Errors (Errors Row)

**Metric CloudWatch**: `AWS/Lambda > Errors`
- **Statistic**: Sum
- **Dimensions**: FunctionName, Resource
- **Visualization**: Bar chart (màu dark-red)
- **Grid Position**: Row 1, Height 7

**Ý nghĩa**: 
Hiển thị tổng số lỗi xảy ra trong Lambda function. Errors được tính khi function throw exception hoặc return error response.

**Ngưỡng cảnh báo**:
- **Warning**: > 5 errors trong period
- **Critical**: > 20 errors trong period
- **Acceptable**: 0 errors

**Best Practices**:
- Implement proper error handling và retry logic
- Sử dụng Dead Letter Queue (DLQ) để xử lý failed invocations
- Log chi tiết error stack trace để troubleshooting
- Cấu hình X-Ray tracing để phân tích error flows
- Monitor error rate % (errors/invocations)

**Troubleshooting**:
```bash
# Xem error logs trong CloudWatch
aws logs filter-log-events \
  --log-group-name /aws/lambda/your-function-name \
  --filter-pattern "ERROR" \
  --start-time $(date -u -d '1 hour ago' +%s)000 \
  --limit 50

# Lấy function configuration để kiểm tra timeout/memory
aws lambda get-function-configuration \
  --function-name your-function-name

# Test function với payload cụ thể
aws lambda invoke \
  --function-name your-function-name \
  --payload '{"test": "data"}' \
  response.json
```

**Tối ưu hóa**:
- Xử lý exceptions gracefully trong code
- Validate input data trước khi xử lý
- Implement circuit breaker pattern cho external dependencies
- Sử dụng appropriate timeout settings
- Enable reserved concurrency để tránh resource exhaustion

---

### 2. Shutdown Reason Logs (Errors Row)

**Data Source**: CloudWatch Logs Insights
- **Query Type**: Logs visualization
- **Filter**: `shutdown_reason` field
- **Grid Position**: Row 1, Y: 8-13

**Ý nghĩa**: 
Hiển thị lý do shutdown của Lambda execution environment (cold start, timeout, memory limit, etc.).

**Các shutdown reasons phổ biến**:
- **timeout**: Function execution vượt quá configured timeout
- **memoryLimitExceeded**: Function sử dụng quá allocated memory
- **outOfMemory**: Out of memory error trong runtime
- **reason**: Lý do khác (cold start, environment cleanup)

**CloudWatch Insights Query**:
```
fields @timestamp, @message, shutdown_reason
| filter shutdown_reason like /./
| sort @timestamp desc
| limit 100
```

**Best Practices**:
- Monitor timeout patterns để adjust timeout configuration
- Track memory usage trends để optimize memory allocation
- Analyze cold start frequency để implement warming strategies
- Document và alert trên abnormal shutdown patterns

**Troubleshooting**:
- **Timeout**: Tăng timeout hoặc tối ưu code performance
- **Memory limit**: Tăng memory allocation hoặc optimize memory usage
- **Cold starts**: Implement provisioned concurrency hoặc warming schedule

---

### 3. Error Logs (Errors Row)

**Data Source**: CloudWatch Logs Insights
- **Query Type**: Logs visualization
- **Filter Pattern**: `WARN|ERROR|Exception`
- **Grid Position**: Row 1, Y: 13-20

**Ý nghĩa**: 
Hiển thị tất cả log entries chứa WARN, ERROR hoặc Exception keywords.

**CloudWatch Insights Query**:
```
fields @timestamp, @message, @logStream
| filter @message like /WARN|ERROR|Exception/
| sort @timestamp desc
| display @timestamp, @message
```

**Log patterns cần monitor**:
- Application exceptions
- Runtime errors
- Dependency connection failures
- Timeout warnings
- Memory pressure warnings

**Best Practices**:
- Implement structured logging (JSON format)
- Include correlation IDs trong logs
- Log với appropriate levels (DEBUG, INFO, WARN, ERROR)
- Không log sensitive data (passwords, tokens)
- Use contextual information (request ID, user ID)

**Troubleshooting**:
```bash
# Query specific error pattern
aws logs filter-log-events \
  --log-group-name /aws/lambda/your-function-name \
  --filter-pattern "Exception" \
  --start-time $(date -u -d '24 hours ago' +%s)000

# Tail logs real-time
aws logs tail /aws/lambda/your-function-name --follow
```

---

### 4. Concurrent Executions (Saturation Row)

**Metrics CloudWatch**: 
- `AWS/Lambda > ConcurrentExecutions` (Maximum) - Per function
- `AWS/Lambda > UnreservedConcurrentExecutions` (Maximum) - Account-level

**Dimensions**: 
- FunctionName, Resource (for function-level)
- Global (for unreserved)

**Visualization**: Time series line chart
**Grid Position**: Row 2 (Saturation), Y: 21-30

**Ý nghĩa**: 
- **ConcurrentExecutions**: Số lượng instances của function đang chạy đồng thời
- **UnreservedConcurrentExecutions**: Unreserved concurrency còn available ở account level

**Ngưỡng cảnh báo**:
- **Account limit**: 1000 concurrent executions (default, có thể request tăng)
- **Warning**: > 80% của reserved concurrency
- **Critical**: > 95% của reserved concurrency

**Best Practices**:
- Set reserved concurrency cho critical functions
- Monitor unreserved concurrency để tránh account-level throttling
- Distribute load evenly với batch processing
- Implement exponential backoff cho retries
- Use SQS để queue requests khi high concurrency

**AWS CLI Commands**:
```bash
# Xem concurrent executions metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name ConcurrentExecutions \
  --dimensions Name=FunctionName,Value=your-function-name \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Maximum

# Set reserved concurrency
aws lambda put-function-concurrency \
  --function-name your-function-name \
  --reserved-concurrent-executions 100

# Remove reserved concurrency
aws lambda delete-function-concurrency \
  --function-name your-function-name

# Get account concurrency limit
aws servicequotas get-service-quota \
  --service-code lambda \
  --quota-code L-B99A9384
```

**Tối ưu hóa**:
- Use provisioned concurrency cho predictable traffic
- Implement request queuing với SQS/EventBridge
- Optimize cold start time
- Consider Step Functions cho complex workflows
- Monitor và adjust reserved concurrency based on patterns

---

### 5. Throttles (Saturation Row)

**Metric CloudWatch**: `AWS/Lambda > Throttles`
- **Statistic**: Sum
- **Dimensions**: FunctionName, Resource
- **Visualization**: Bar chart (màu dark-orange)
- **Grid Position**: Row 2, X: 11, Y: 21-30

**Ý nghĩa**: 
Số lượng invocation requests bị throttle do vượt quá concurrency limits. Throttled requests return 429 error.

**Ngưỡng cảnh báo**:
- **Acceptable**: 0 throttles
- **Warning**: > 1% throttle rate
- **Critical**: > 5% throttle rate

**Các nguyên nhân throttling**:
1. **Reserved concurrency limit**: Function đạt reserved concurrent executions
2. **Account-level limit**: Tổng concurrent executions vượt account limit
3. **Burst limit**: Quá 3000 requests/second (hoặc 1000 trong us-east-1)
4. **Regional limits**: Vượt regional concurrency quota

**Best Practices**:
- Monitor throttle rate thường xuyên
- Set appropriate reserved concurrency
- Implement retry logic với exponential backoff
- Use SQS để buffer requests
- Request limit increase nếu cần thiết

**Troubleshooting**:
```bash
# Check throttle metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Throttles \
  --dimensions Name=FunctionName,Value=your-function-name \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Sum

# Check concurrent executions vs throttles
aws cloudwatch get-metric-data \
  --metric-data-queries file://throttle-query.json \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S)
```

**Giải pháp throttling**:
1. **Increase reserved concurrency**: Nếu function cần higher limits
2. **Implement queuing**: Sử dụng SQS Standard/FIFO queue
3. **Distribute load**: Spread invocations over time
4. **Optimize execution time**: Reduce duration để free up concurrency faster
5. **Request quota increase**: Contact AWS Support

---

### 6. Duration (Saturation Row)

**Metrics CloudWatch**: `AWS/Lambda > Duration`
- **Statistics**: Average, Maximum, Minimum
- **Dimensions**: FunctionName, Resource
- **Visualization**: Time series với log scale
- **Unit**: Milliseconds (dtdurationms)
- **Grid Position**: Row 2, Y: 30-38

**Ý nghĩa**: 
Thời gian thực thi của Lambda function từ khi bắt đầu execution cho đến khi return response.

**Ngưỡng cảnh báo**:
- **Good**: < 1000ms (1 second)
- **Warning**: > 5000ms (5 seconds)
- **Critical**: Gần timeout limit

**Visualization colors**:
- **Average**: Dark green (main metric)
- **Maximum**: Dark yellow (upper bound)
- **Minimum**: Super light yellow (lower bound)

**Best Practices**:
- Keep duration dưới 3 seconds cho API responses
- Optimize cold start time (< 1 second)
- Monitor p99 duration cho worst-case scenarios
- Set timeout = 2x expected max duration
- Track duration trends để detect performance regressions

**Các yếu tố ảnh hưởng duration**:
1. **Cold start overhead**: Khởi tạo runtime environment
2. **Code complexity**: Algorithm efficiency
3. **Dependencies**: External API calls, database queries
4. **Memory allocation**: Higher memory = more CPU power
5. **Package size**: Larger deployment packages = slower cold starts

**Tối ưu hóa duration**:
```bash
# Increase memory để có more CPU power
aws lambda update-function-configuration \
  --function-name your-function-name \
  --memory-size 1024

# Enable X-Ray tracing để analyze performance
aws lambda update-function-configuration \
  --function-name your-function-name \
  --tracing-config Mode=Active

# Optimize cold start với SnapStart (Java)
aws lambda update-function-configuration \
  --function-name your-function-name \
  --snap-start ApplyOn=PublishedVersions

# Set appropriate timeout
aws lambda update-function-configuration \
  --function-name your-function-name \
  --timeout 30
```

**Performance optimization strategies**:
1. **Minimize cold starts**: 
   - Use provisioned concurrency
   - Implement warming schedules
   - Reduce package size
   - Optimize initialization code

2. **Optimize runtime execution**:
   - Reuse connections (database, HTTP clients)
   - Cache data khi có thể
   - Use efficient algorithms và data structures
   - Minimize external API calls

3. **Right-size memory**:
   - Test với different memory settings
   - Monitor memory usage vs duration
   - Balance cost vs performance

4. **Async processing**:
   - Use Step Functions cho long-running tasks
   - Implement async invocations khi appropriate
   - Offload heavy processing sang background jobs

---

### 7. Network (Utilization Row)

**Metrics Lambda Insights**: 
- `LambdaInsights > tx_bytes` (p99) - Transmitted bytes
- `LambdaInsights > rx_bytes` (p99) - Received bytes

**Dimensions**: function_name
**Visualization**: Time series với negative-Y transform cho tx_bytes
**Unit**: Bytes
**Grid Position**: Row 3 (Utilization), X: 0, Y: 39-46

**Ý nghĩa**: 
Lưu lượng network traffic của Lambda function (data sent và received).

**Ngưỡng cảnh báo**:
- **Normal**: < 1 MB per invocation
- **Warning**: > 5 MB per invocation
- **Critical**: Consistent high network usage (có thể indicate inefficient data transfer)

**Best Practices**:
- Minimize data transfer giữa function và external services
- Use compression cho large payloads
- Implement pagination cho large result sets
- Cache frequently accessed data
- Monitor network costs (data transfer charges)

**Tối ưu hóa network usage**:
- Compress request/response payloads
- Use binary protocols thay vì JSON khi appropriate
- Implement edge caching với CloudFront
- Batch requests để reduce network overhead
- Use VPC endpoints để avoid internet gateway costs

**Note**: Lambda Insights metrics yêu cầu Lambda Insights extension được enabled trên function.

```bash
# Enable Lambda Insights
aws lambda update-function-configuration \
  --function-name your-function-name \
  --layers arn:aws:lambda:region:580247275435:layer:LambdaInsightsExtension:14
```

---

### 8. Invocations (Utilization Row)

**Metric CloudWatch**: `AWS/Lambda > Invocations`
- **Statistic**: Sum
- **Dimensions**: FunctionName, Resource
- **Visualization**: Stacked bar chart
- **Grid Position**: Row 3, X: 5, Y: 39-46

**Ý nghĩa**: 
Tổng số lần Lambda function được invoke trong period.

**Ngưỡng**:
- Track invocation patterns để capacity planning
- Monitor invocation trends để detect anomalies
- Compare với billing để optimize costs

**Best Practices**:
- Monitor invocation rate để detect traffic spikes
- Set CloudWatch alarms cho abnormal invocation counts
- Analyze invocation patterns để implement caching
- Track invocations by version/alias
- Monitor cost per invocation

**Cost optimization**:
```bash
# Calculate invocation costs
# Cost = (Invocations × Duration × Memory) × Price per GB-second

# Get invocation count
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Invocations \
  --dimensions Name=FunctionName,Value=your-function-name \
  --start-time $(date -u -d '30 days ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 86400 \
  --statistics Sum

# Get average duration
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Duration \
  --dimensions Name=FunctionName,Value=your-function-name \
  --start-time $(date -u -d '30 days ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 86400 \
  --statistics Average
```

**Invocation patterns phổ biến**:
1. **Synchronous invocation**: API Gateway, ALB, direct invoke
2. **Asynchronous invocation**: S3 events, SNS, EventBridge
3. **Stream-based invocation**: DynamoDB Streams, Kinesis
4. **Scheduled invocation**: EventBridge rules (cron)

**Tối ưu hóa invocations**:
- Implement intelligent caching để reduce unnecessary calls
- Batch process events khi possible
- Use appropriate invocation type (sync vs async)
- Implement idempotency để handle retries safely
- Monitor và eliminate duplicate invocations

---

### 9. CPU / Init Time (Utilization Row)

**Metrics Lambda Insights**:
- `LambdaInsights > cpu_total_time` (p99) - Total CPU time
- `LambdaInsights > init_duration` (p99) - Initialization duration

**Dimensions**: function_name
**Visualization**: Time series line chart
**Unit**: Milliseconds
**Grid Position**: Row 3, X: 11, Y: 39-46

**Ý nghĩa**: 
- **cpu_total_time**: Tổng CPU time sử dụng trong execution
- **init_duration**: Thời gian khởi tạo runtime environment (cold start overhead)

**Ngưỡng**:
- **Init duration acceptable**: < 1000ms
- **Init duration warning**: > 3000ms
- **CPU time**: Nên tương đương với duration cho CPU-bound tasks

**Cold Start Optimization**:

**1. Minimize package size**:
```bash
# Remove unused dependencies
npm prune --production

# Use webpack/esbuild để bundle
npm install --save-dev esbuild
esbuild index.js --bundle --platform=node --target=node18 --outfile=dist/index.js

# Check package size
du -sh .aws-sam/build/YourFunction/
```

**2. Optimize initialization code**:
```javascript
// ❌ Bad: Initialize in handler
exports.handler = async (event) => {
  const dbClient = new DatabaseClient(); // Cold start every time
  return await dbClient.query();
};

// ✅ Good: Initialize outside handler
const dbClient = new DatabaseClient(); // Reused across invocations

exports.handler = async (event) => {
  return await dbClient.query();
};
```

**3. Use provisioned concurrency**:
```bash
# Create function version
aws lambda publish-version \
  --function-name your-function-name

# Configure provisioned concurrency
aws lambda put-provisioned-concurrency-config \
  --function-name your-function-name \
  --qualifier 1 \
  --provisioned-concurrent-executions 5
```

**4. Enable SnapStart (Java 11+)**:
```bash
aws lambda update-function-configuration \
  --function-name your-function-name \
  --snap-start ApplyOn=PublishedVersions
```

**5. Optimize runtime selection**:
- **Python**: Faster cold starts than Node.js
- **Node.js**: Good balance của performance và cold start
- **Java**: Slower cold starts, use SnapStart
- **Go/Rust**: Fastest cold starts và execution
- **Custom runtime**: Consider compiled languages

**Best Practices**:
- Move initialization code outside handler
- Lazy load dependencies khi possible
- Use lightweight frameworks
- Minimize import statements
- Implement connection pooling
- Monitor init_duration trends
- Use Lambda Insights để analyze initialization overhead

---

### 10. Memory (Utilization Row)

**Metrics Lambda Insights**:
- `LambdaInsights > total_memory` (p99) - Configured memory
- `LambdaInsights > memory_utilization` (Average) - Memory usage percentage

**Dimensions**: function_name
**Visualization**: Time series với dual axis
**Units**: 
- total_memory: MB
- memory_utilization: Percent
**Grid Position**: Row 3, X: 17, Y: 39-46

**Ý nghĩa**: 
- **total_memory**: Memory allocation cho function (configured)
- **memory_utilization**: Percentage của memory đang được sử dụng

**Ngưỡng cảnh báo**:
- **Acceptable**: < 70% memory utilization
- **Warning**: 70-85% memory utilization
- **Critical**: > 85% memory utilization (risk của OOM errors)

**Best Practices**:
- Monitor memory utilization để right-size allocation
- Set memory = 1.2x peak usage để có buffer
- Higher memory = More CPU power (proportional)
- Test performance với different memory settings
- Balance memory cost vs execution speed

**Memory optimization strategies**:

**1. Right-sizing memory**:
```bash
# Test với different memory settings
for mem in 512 1024 1536 2048 3008; do
  aws lambda update-function-configuration \
    --function-name your-function-name \
    --memory-size $mem
  
  # Test performance
  aws lambda invoke \
    --function-name your-function-name \
    --payload '{"test": "data"}' \
    response.json
  
  # Check duration và cost
done

# Use Lambda Power Tuning tool
# https://github.com/alexcasalboni/aws-lambda-power-tuning
```

**2. Monitor memory trends**:
```bash
# Get memory utilization metrics
aws cloudwatch get-metric-statistics \
  --namespace LambdaInsights \
  --metric-name memory_utilization \
  --dimensions Name=function_name,Value=your-function-name \
  --start-time $(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 3600 \
  --statistics Average,Maximum
```

**3. Code optimization**:
```python
# ❌ Bad: Load large data into memory
with open('large_file.json', 'r') as f:
    data = json.load(f)  # Entire file in memory

# ✅ Good: Stream processing
with open('large_file.json', 'r') as f:
    for line in f:
        process_line(line)  # Process incrementally

# ❌ Bad: Keep all results in memory
results = [process(item) for item in large_list]

# ✅ Good: Generator pattern
results = (process(item) for item in large_list)
```

**4. Memory leak prevention**:
- Clear references sau khi sử dụng
- Avoid global state accumulation
- Monitor memory across warm invocations
- Implement proper cleanup trong finally blocks
- Use weak references khi appropriate

**Memory vs Duration tradeoff**:
```
Memory (MB) | CPU Power | Price Multiplier | Sweet Spot
128         | 0.08 vCPU | 1x               | Very light tasks
512         | 0.33 vCPU | 4x               | Light processing
1024        | 0.67 vCPU | 8x               | ⭐ Most common
1536        | 1.0 vCPU  | 12x              | Compute-intensive
3008        | 1.83 vCPU | 23.5x            | Heavy processing
10240       | 6.0 vCPU  | 80x              | Maximum
```

**Cost optimization**:
- Higher memory có thể reduce execution time
- Faster execution = lower total cost
- Use Lambda Power Tuning để find optimal memory
- Monitor GB-seconds metric cho actual cost

---

### 11. Lambda Logs (Logs Row)

**Data Source**: CloudWatch Logs Insights
**Query Type**: Full text logs với filter
**Grid Position**: Row 4 (Logs), Y: 47-54

**CloudWatch Insights Query**:
```
fields @timestamp, @message, @logStream, @log, function_name
| filter  (@log LIKE "$function_name" OR function_name LIKE "$function_name")
| filter @message LIKE "${log_filter}"
| sort @timestamp desc
| display @timestamp, @message
```

**Ý nghĩa**: 
Hiển thị all logs từ Lambda function với ability để filter theo custom patterns.

**Log levels**:
- **START**: Invocation started
- **END**: Invocation completed
- **REPORT**: Execution summary (duration, memory, billed duration)
- **INFO**: Application info logs
- **WARN**: Warning messages
- **ERROR**: Error messages
- **DEBUG**: Debug information

**REPORT line example**:
```
REPORT RequestId: abc123-def456 Duration: 1234.56 ms 
Billed Duration: 1300 ms Memory Size: 1024 MB 
Max Memory Used: 512 MB Init Duration: 234.56 ms
```

**Structured logging best practices**:
```python
import json
import logging

# Configure structured logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    # Structured log với context
    logger.info(json.dumps({
        'event': 'processing_started',
        'request_id': context.request_id,
        'function_name': context.function_name,
        'input_size': len(json.dumps(event))
    }))
    
    try:
        result = process_data(event)
        
        logger.info(json.dumps({
            'event': 'processing_completed',
            'request_id': context.request_id,
            'result_count': len(result)
        }))
        
        return {'statusCode': 200, 'body': result}
        
    except Exception as e:
        logger.error(json.dumps({
            'event': 'processing_failed',
            'request_id': context.request_id,
            'error_type': type(e).__name__,
            'error_message': str(e)
        }), exc_info=True)
        
        raise
```

**Advanced log queries**:
```
# Find slow invocations
fields @timestamp, @requestId, @duration
| filter @type = "REPORT"
| filter @duration > 5000
| sort @duration desc

# Calculate average duration by version
fields @timestamp, @duration
| filter @type = "REPORT"
| stats avg(@duration) as avg_duration by @functionVersion

# Find memory usage patterns
fields @timestamp, @maxMemoryUsed, @memorySize
| filter @type = "REPORT"
| stats max(@maxMemoryUsed) as peak_memory, avg(@maxMemoryUsed) as avg_memory by bin(5m)

# Error analysis
fields @timestamp, @message
| filter @message like /ERROR|Exception|failed/
| stats count() as error_count by bin(5m)

# Cold start analysis
fields @timestamp, @initDuration
| filter @type = "REPORT" and @initDuration > 0
| stats avg(@initDuration) as avg_cold_start, count() as cold_start_count by bin(1h)
```

**Log retention & costs**:
```bash
# Set log retention
aws logs put-retention-policy \
  --log-group-name /aws/lambda/your-function-name \
  --retention-in-days 7

# Export logs to S3 for long-term storage
aws logs create-export-task \
  --log-group-name /aws/lambda/your-function-name \
  --from $(date -u -d '30 days ago' +%s)000 \
  --to $(date -u +%s)000 \
  --destination your-s3-bucket \
  --destination-prefix lambda-logs/
```

**Best Practices**:
- Use structured logging (JSON format)
- Include correlation IDs
- Log appropriate detail levels
- Avoid logging sensitive data
- Set appropriate retention policies
- Use CloudWatch Logs Insights cho analysis
- Implement log sampling cho high-traffic functions
- Consider log aggregation tools (Datadog, New Relic)

---

## CloudWatch Alarms Recommendations

### Critical Alarms

**1. High Error Rate**:
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name lambda-high-error-rate \
  --alarm-description "Lambda error rate > 5%" \
  --metric-name Errors \
  --namespace AWS/Lambda \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=FunctionName,Value=your-function-name \
  --alarm-actions arn:aws:sns:region:account:alert-topic
```

**2. Function Throttling**:
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name lambda-throttles \
  --alarm-description "Lambda throttles detected" \
  --metric-name Throttles \
  --namespace AWS/Lambda \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 1 \
  --threshold 1 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=FunctionName,Value=your-function-name \
  --alarm-actions arn:aws:sns:region:account:alert-topic
```

**3. High Duration**:
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name lambda-high-duration \
  --alarm-description "Lambda duration approaching timeout" \
  --metric-name Duration \
  --namespace AWS/Lambda \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 25000 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=FunctionName,Value=your-function-name \
  --alarm-actions arn:aws:sns:region:account:alert-topic
```

**4. High Concurrent Executions**:
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name lambda-high-concurrency \
  --alarm-description "Concurrent executions > 80% of limit" \
  --metric-name ConcurrentExecutions \
  --namespace AWS/Lambda \
  --statistic Maximum \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=FunctionName,Value=your-function-name \
  --alarm-actions arn:aws:sns:region:account:alert-topic
```

**5. High Memory Utilization**:
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name lambda-high-memory \
  --alarm-description "Memory utilization > 85%" \
  --metric-name memory_utilization \
  --namespace LambdaInsights \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 85 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=function_name,Value=your-function-name \
  --alarm-actions arn:aws:sns:region:account:alert-topic
```

---

## Performance Optimization Checklist

### Cold Start Optimization
- [ ] Minimize deployment package size
- [ ] Move initialization code outside handler
- [ ] Use provisioned concurrency cho critical functions
- [ ] Enable SnapStart (Java)
- [ ] Optimize dependency loading
- [ ] Consider compiled languages (Go, Rust)
- [ ] Implement lazy loading
- [ ] Use Lambda layers cho shared dependencies

### Execution Optimization
- [ ] Right-size memory allocation
- [ ] Reuse connections (DB, HTTP clients)
- [ ] Implement caching strategies
- [ ] Optimize algorithm complexity
- [ ] Minimize external API calls
- [ ] Use async/parallel processing
- [ ] Implement proper error handling
- [ ] Enable X-Ray tracing

### Concurrency Management
- [ ] Set appropriate reserved concurrency
- [ ] Monitor unreserved concurrency
- [ ] Implement exponential backoff retries
- [ ] Use SQS cho request queuing
- [ ] Distribute load evenly
- [ ] Track throttle metrics
- [ ] Request limit increases khi cần

### Cost Optimization
- [ ] Monitor invocation patterns
- [ ] Optimize memory vs duration tradeoff
- [ ] Implement intelligent caching
- [ ] Batch process events
- [ ] Set appropriate timeout values
- [ ] Use appropriate invocation types
- [ ] Monitor và eliminate waste
- [ ] Set log retention policies

### Monitoring & Observability
- [ ] Enable Lambda Insights
- [ ] Configure CloudWatch alarms
- [ ] Implement structured logging
- [ ] Enable X-Ray tracing
- [ ] Track custom metrics
- [ ] Set up dashboards
- [ ] Implement health checks
- [ ] Monitor cost metrics

---

## Common Issues & Solutions

### Issue 1: High Cold Start Times

**Symptoms**: init_duration > 3 seconds

**Solutions**:
1. Reduce package size
2. Use provisioned concurrency
3. Optimize imports và initialization
4. Consider different runtime
5. Enable SnapStart (Java)

### Issue 2: Function Timeouts

**Symptoms**: Errors với timeout message

**Solutions**:
1. Increase timeout setting
2. Optimize code performance
3. Implement async processing
4. Use Step Functions cho long workflows
5. Break into smaller functions

### Issue 3: High Memory Usage

**Symptoms**: memory_utilization > 85%

**Solutions**:
1. Increase memory allocation
2. Optimize memory usage trong code
3. Implement streaming processing
4. Clear references properly
5. Fix memory leaks

### Issue 4: Throttling Issues

**Symptoms**: Throttles > 0

**Solutions**:
1. Increase reserved concurrency
2. Implement request queuing
3. Distribute load
4. Request quota increase
5. Optimize execution time

### Issue 5: High Error Rate

**Symptoms**: Errors/Invocations > 5%

**Solutions**:
1. Implement proper error handling
2. Add retry logic với exponential backoff
3. Validate input data
4. Use DLQ cho failed events
5. Enable detailed logging
6. Monitor dependencies health

---

## Cost Optimization Strategies

### 1. Memory Optimization
```bash
# Use AWS Lambda Power Tuning
# GitHub: alexcasalboni/aws-lambda-power-tuning

# Deploy power tuning tool
sam deploy --guided

# Run tuning
aws lambda invoke \
  --function-name power-tuning \
  --payload file://payload.json \
  response.json
```

### 2. Request Optimization
- Implement caching để reduce invocations
- Batch requests khi possible
- Use appropriate invocation types
- Eliminate duplicate calls
- Implement idempotency

### 3. Duration Optimization
- Optimize code performance
- Use appropriate memory setting
- Reuse connections
- Minimize cold starts
- Implement parallel processing

### 4. Log Cost Optimization
```bash
# Set retention policy
aws logs put-retention-policy \
  --log-group-name /aws/lambda/your-function-name \
  --retention-in-days 7

# Implement log sampling
import random

def should_log_debug():
    return random.random() < 0.1  # Log 10% of debug messages
```

### 5. Architecture Optimization
- Use Step Functions cho orchestration
- Implement event-driven architecture
- Use SQS cho decoupling
- Consider containerized workloads (ECS/EKS) cho consistent high load
- Use Lambda@Edge cho edge computing

---

## Best Practices Summary

### Security
- Follow least privilege principle cho IAM roles
- Encrypt environment variables
- Use VPC cho database access
- Enable AWS WAF cho API Gateway
- Rotate credentials regularly
- Scan dependencies cho vulnerabilities

### Reliability
- Implement proper error handling
- Use DLQ cho failed events
- Configure retries appropriately
- Monitor key metrics continuously
- Set up meaningful alarms
- Test failure scenarios

### Performance
- Right-size memory allocation
- Minimize cold starts
- Optimize initialization code
- Reuse connections
- Implement caching
- Monitor duration trends

### Observability
- Enable structured logging
- Configure X-Ray tracing
- Track custom metrics
- Set up dashboards
- Monitor costs
- Analyze patterns regularly

### Operations
- Use infrastructure as code (CloudFormation/Terraform)
- Implement CI/CD pipelines
- Use versions và aliases
- Implement canary deployments
- Maintain documentation
- Regular security audits

---

## Tài Liệu Tham Khảo

- **AWS Lambda Developer Guide**: https://docs.aws.amazon.com/lambda/
- **Lambda Best Practices**: https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html
- **Lambda Insights**: https://docs.aws.amazon.com/lambda/latest/dg/monitoring-insights.html
- **Lambda Pricing**: https://aws.amazon.com/lambda/pricing/
- **CloudWatch Logs Insights**: https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html
- **AWS Well-Architected Framework - Serverless**: https://docs.aws.amazon.com/wellarchitected/latest/serverless-applications-lens/

---

**Dashboard Version**: 1  
**Last Updated**: 2026-02-10  
**Grafana ID**: 19734
