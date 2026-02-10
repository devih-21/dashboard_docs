# Tài liệu Dashboard AWS S3

## Tổng quan
Dashboard này được thiết kế để giám sát và theo dõi các metrics của Amazon S3 (Simple Storage Service) thông qua CloudWatch. Dashboard cung cấp cái nhìn toàn diện về dung lượng lưu trữ, số lượng requests, hiệu suất, băng thông và lỗi của S3 buckets.

**Nguồn dữ liệu**: CloudWatch  
**Grafana Dashboard ID**: 575  
**Khoảng thời gian mặc định**: 30 ngày gần nhất  

---

## Cấu hình

### Biến Template (Template Variables)

Dashboard sử dụng các biến động để cho phép người dùng tùy chỉnh view:

1. **datasource**: Nguồn dữ liệu CloudWatch
2. **region**: Vùng AWS (AWS Region)
3. **bucket**: Tên S3 bucket cần giám sát
4. **filterid**: Filter ID để lọc metrics theo cấu hình request metrics cụ thể
   - Có thể chọn "All" để xem tất cả
   - Hoặc chọn Filter ID cụ thể nếu đã cấu hình Request Metrics trong S3

---

## Chi tiết các Dashboard Panel

### 1. BucketSizeBytes (Dung lượng Bucket)

**Vị trí**: Hàng 1, chiều cao 7 đơn vị, chiều rộng 24 đơn vị  
**Loại biểu đồ**: Time series (Line chart)

**Mô tả**:  
Hiển thị tổng dung lượng lưu trữ (tính bằng bytes) của S3 bucket theo thời gian. Metric này cho biết lượng dữ liệu đang được lưu trữ trong bucket.

**Metrics**:
- Namespace: `AWS/S3`
- Metric: `BucketSizeBytes`
- Statistic: `Average`
- Dimensions: 
  - `BucketName`: $bucket
  - `StorageType`: StandardStorage

**Cấu hình hiển thị**:
- Đơn vị: bytes (tự động chuyển đổi sang KB, MB, GB, TB)
- Giá trị tối thiểu: 0
- Hiển thị: Đường line (continuous)
- Legend: Hiển thị các giá trị mean, max, min

**Ý nghĩa**:  
- Theo dõi xu hướng tăng trưởng dung lượng lưu trữ
- Dự đoán nhu cầu lưu trữ và chi phí
- Phát hiện tăng đột biến dung lượng (có thể do lỗi hoặc attack)
- Planning capacity và budget cho storage

**Lưu ý**:
- Metric này chỉ cập nhật 1 lần mỗi ngày (không real-time)
- Chỉ áp dụng cho StandardStorage class

---

### 2. NumberOfObjects (Số lượng Objects)

**Vị trí**: Hàng 2, chiều cao 7 đơn vị  
**Loại biểu đồ**: Time series (Line chart)

**Mô tả**:  
Hiển thị tổng số lượng objects (files) được lưu trữ trong S3 bucket theo thời gian.

**Metrics**:
- Namespace: `AWS/S3`
- Metric: `NumberOfObjects`
- Statistic: `Average`
- Dimensions: 
  - `BucketName`: $bucket
  - `StorageType`: AllStorageTypes

**Cấu hình hiển thị**:
- Đơn vị: none (count)
- Giá trị tối thiểu: 0
- Hiển thị: Đường line (continuous)
- Legend: Hiển thị các giá trị mean, max, min

**Ý nghĩa**:  
- Theo dõi số lượng files trong bucket
- Phát hiện pattern upload/delete bất thường
- Planning cho S3 Inventory và lifecycle policies
- Monitoring data retention compliance

**Lưu ý**:
- Metric này cập nhật 1 lần mỗi ngày
- Bao gồm tất cả storage classes (Standard, IA, Glacier, etc.)

---

### 3. Filtered Requests (Các loại Requests được lọc)

**Vị trí**: Hàng 3, chiều cao 7 đơn vị  
**Loại biểu đồ**: Time series (Line chart - Multi-series)

**Mô tả**:  
Hiển thị chi tiết các loại HTTP requests được gửi đến S3 bucket theo thời gian. Panel này cho phép phân tích pattern sử dụng và traffic của bucket.

**Metrics** (7 metrics):
- `AllRequests`: Tổng tất cả requests
- `GetRequests`: Requests đọc/download objects
- `PutRequests`: Requests upload/ghi objects
- `DeleteRequests`: Requests xóa objects
- `HeadRequests`: Requests kiểm tra metadata
- `PostRequests`: Requests POST (multipart upload initiation)
- `ListRequests`: Requests liệt kê objects

**Cấu hình**:
- Namespace: `AWS/S3`
- Statistic: `Sum`
- Dimensions: 
  - `BucketName`: $bucket
  - `FilterId`: $filterid

**Cấu hình hiển thị**:
- Đơn vị: none (count)
- Giá trị tối thiểu: 0
- Hiển thị: Multiple lines (mỗi loại request một đường)
- Legend: Hiển thị các giá trị mean, max, min

**Ý nghĩa**:  
- **AllRequests**: Tổng traffic, giúp đánh giá tải tổng thể
- **GetRequests**: Cao = nhiều traffic đọc dữ liệu (download, data retrieval)
- **PutRequests**: Cao = nhiều traffic ghi dữ liệu (upload, data ingestion)
- **DeleteRequests**: Tracking data lifecycle và cleanup operations
- **HeadRequests**: Thường dùng để check object existence hoặc metadata
- **PostRequests**: Multipart upload operations
- **ListRequests**: Listing operations, có thể tốn chi phí nếu quá nhiều

**Use Cases**:
- Phát hiện unusual traffic patterns
- Tối ưu hóa chi phí (LIST operations tốn phí)
- Phân tích user behavior
- Capacity planning cho API rate limits

**Lưu ý**:
- Yêu cầu enable **Request Metrics** trong S3 bucket configuration
- Có thể filter theo prefix/tag sử dụng FilterId
- Metrics này có độ trễ vài phút (near real-time)

---

### 4. Filtered Latency (Độ trễ được lọc)

**Vị trí**: Hàng 4, chiều cao 7 đơn vị  
**Loại biểu đồ**: Time series (Line chart)

**Mô tả**:  
Hiển thị độ trễ (latency) của các requests đến S3 bucket, giúp đánh giá hiệu suất và trải nghiệm người dùng.

**Metrics** (2 metrics):
- **FirstByteLatency**: Thời gian từ khi S3 nhận request đến khi trả về byte đầu tiên
- **TotalRequestLatency**: Tổng thời gian từ khi nhận request đến khi hoàn thành response

**Cấu hình**:
- Namespace: `AWS/S3`
- Statistic: `Average`
- Dimensions: 
  - `BucketName`: $bucket
  - `FilterId`: $filterid

**Cấu hình hiển thị**:
- Đơn vị: milliseconds (ms)
- Giá trị tối thiểu: 0
- Hiển thị: Line charts
- Legend: Hiển thị các giá trị mean, max, min

**Ý nghĩa**:  

**FirstByteLatency (Time to First Byte - TTFB)**:
- Đo lường thời gian S3 xử lý request và bắt đầu trả dữ liệu
- Thấp = tốt (thường < 100ms cho Standard storage)
- Cao có thể do:
  - Object size lớn
  - Sử dụng Glacier/Deep Archive (cần thời gian restore)
  - Network congestion
  - S3 internal processing

**TotalRequestLatency**:
- Đo lường toàn bộ thời gian từ request đến response hoàn tất
- Bao gồm: FirstByteLatency + thời gian transfer data
- Phụ thuộc vào object size và network bandwidth

**Performance Benchmarks**:
- **Excellent**: FirstByteLatency < 50ms, TotalRequestLatency < 200ms
- **Good**: FirstByteLatency < 100ms, TotalRequestLatency < 500ms
- **Acceptable**: FirstByteLatency < 200ms, TotalRequestLatency < 1000ms
- **Poor**: FirstByteLatency > 200ms, TotalRequestLatency > 1000ms

**Troubleshooting**:
- Latency cao đột ngột: Check AWS Service Health Dashboard
- Latency cao liên tục: 
  - Xem xét sử dụng CloudFront CDN
  - Enable Transfer Acceleration
  - Optimize object size
  - Review S3 bucket region placement

**Lưu ý**:
- Yêu cầu enable Request Metrics
- Metrics average, cần xem p95/p99 percentiles để có cái nhìn chính xác hơn

---

### 5. Filtered Bytes (Lưu lượng dữ liệu được lọc)

**Vị trí**: Hàng 5, chiều cao 7 đơn vị  
**Loại biểu đồ**: Time series (Line chart)

**Mô tả**:  
Hiển thị lưu lượng dữ liệu (bandwidth) upload và download từ S3 bucket theo thời gian.

**Metrics** (2 metrics):
- **BytesDownloaded**: Tổng số bytes được download từ bucket
- **BytesUploaded**: Tổng số bytes được upload vào bucket

**Cấu hình**:
- Namespace: `AWS/S3`
- Statistic: `Sum`
- Dimensions: 
  - `BucketName`: $bucket
  - `FilterId`: $filterid

**Cấu hình hiển thị**:
- Đơn vị: bytes (tự động chuyển đổi sang KB, MB, GB, TB)
- Giá trị tối thiểu: 0
- Hiển thị: Line charts
- Legend: Hiển thị các giá trị mean, max, min

**Ý nghĩa**:  

**BytesDownloaded**:
- Lượng dữ liệu được transfer ra khỏi S3
- Ảnh hưởng trực tiếp đến **data transfer cost**
- Cao = nhiều users download hoặc data serving
- Use cases: Media streaming, file distribution, backup restore

**BytesUploaded**:
- Lượng dữ liệu được ghi vào S3
- Upload to S3 thường miễn phí (free inbound transfer)
- Cao = data ingestion, backups, log aggregation
- Use cases: Data lake ingestion, application logs, user uploads

**Cost Implications**:
- Data Transfer OUT từ S3 đến Internet: **$0.09/GB** (first 10TB/month)
- Data Transfer IN: **FREE**
- Data Transfer giữa S3 và EC2 cùng region: **FREE**
- Data Transfer cross-region: Có phí

**Optimization Tips**:
- Sử dụng CloudFront để giảm data transfer costs
- Review download patterns để identify unnecessary transfers
- Enable S3 Transfer Acceleration cho upload nhanh hơn
- Compress data trước khi upload

**Monitoring & Alerts**:
- Alert khi BytesDownloaded vượt threshold (unusual spike)
- Track ratio BytesUploaded/BytesDownloaded để hiểu usage pattern
- Correlate với cost metrics để budget planning

**Lưu ý**:
- Yêu cầu enable Request Metrics
- Metrics này useful cho cost optimization và capacity planning

---

### 6. Filtered Errors (Lỗi được lọc)

**Vị trí**: Hàng 6, chiều cao 7 đơn vị  
**Loại biểu đồ**: Time series (Line chart)

**Mô tả**:  
Hiển thị số lượng HTTP errors xảy ra khi request đến S3 bucket, giúp phát hiện vấn đề về permissions, configuration, hoặc client errors.

**Metrics** (2 metrics):
- **4xxErrors**: Client-side errors (lỗi từ phía client/application)
- **5xxErrors**: Server-side errors (lỗi từ phía S3/AWS)

**Cấu hình**:
- Namespace: `AWS/S3`
- Statistic: `Sum`
- Dimensions: 
  - `BucketName`: $bucket
  - `FilterId`: $filterid

**Cấu hình hiển thị**:
- Đơn vị: none (count)
- Giá trị tối thiểu: 0
- Hiển thị: Line charts
- Legend: Hiển thị các giá trị mean, max, min
- 4xxErrors: Hiển thị trên trục phải (secondary axis)

**Ý nghĩa**:  

### 4xxErrors (Client Errors)

**Các loại lỗi 4xx thường gặp**:
- **400 Bad Request**: Invalid request syntax
- **403 Forbidden**: Không có quyền truy cập (IAM, bucket policy, ACL)
- **404 Not Found**: Object không tồn tại
- **405 Method Not Allowed**: HTTP method không được phép
- **409 Conflict**: Version conflict, object already exists

**Nguyên nhân**:
- IAM permissions không đúng
- Bucket policy quá restrictive
- Application bug (wrong object key, invalid parameters)
- Timing issues (try to access just-deleted object)

**Hành động**:
- Review CloudTrail logs để xem chi tiết requests bị lỗi
- Check IAM policies và bucket policies
- Verify application code
- Common fix: Update permissions, fix object keys

### 5xxErrors (Server Errors)

**Các loại lỗi 5xx thường gặp**:
- **500 Internal Server Error**: S3 internal error
- **503 Service Unavailable**: S3 temporarily unavailable
- **504 Gateway Timeout**: Request timeout

**Nguyên nhân**:
- AWS infrastructure issues (rất hiếm)
- Rate limiting / throttling
- Network issues
- Request timeout do large objects

**Hành động**:
- Check AWS Service Health Dashboard
- Implement exponential backoff retry logic
- Consider request rate optimization
- For persistent issues: Open AWS Support ticket

**Error Monitoring Best Practices**:

1. **Thresholds & Alerts**:
   - 4xxErrors: Alert nếu > 1% của tổng requests
   - 5xxErrors: Alert ngay lập tức (critical)

2. **Analysis**:
   - High 403: Permissions issues
   - High 404: Wrong object keys hoặc lifecycle policies đang xóa objects
   - Any 5xx: AWS infrastructure issues

3. **Correlation**:
   - Cross-reference với AllRequests để tính error rate
   - Check timing: errors spike cùng lúc với deployment?
   - Review CloudTrail logs cho detailed diagnosis

**Error Rate Calculation**:
```
Error Rate (%) = (4xxErrors + 5xxErrors) / AllRequests * 100
```

**Healthy Benchmarks**:
- **Excellent**: Error rate < 0.1%
- **Good**: Error rate < 0.5%
- **Acceptable**: Error rate < 1%
- **Poor**: Error rate > 1% (needs investigation)

**Lưu ý**:
- Yêu cầu enable Request Metrics
- 4xxErrors thường do application issues, dễ fix
- 5xxErrors nghiêm trọng hơn, thường do AWS infrastructure
- Enable S3 Server Access Logging để detailed error analysis

---

## Hướng dẫn sử dụng

### 1. Chọn S3 Bucket để giám sát
- Sử dụng dropdown **Bucket** ở phía trên dashboard
- Chọn tên bucket cần theo dõi
- Dropdown tự động load danh sách buckets từ region đã chọn

### 2. Chọn AWS Region
- Sử dụng dropdown **Region** để chọn AWS Region
- Chỉ hiển thị metrics của buckets trong region được chọn

### 3. Chọn Filter ID (Optional)
- Nếu đã cấu hình Request Metrics với filters trong S3
- Chọn Filter ID cụ thể để xem metrics theo prefix hoặc tags
- Chọn "All" để xem tất cả metrics

### 4. Điều chỉnh khoảng thời gian
- Sử dụng time picker (góc trên bên phải) để chọn khoảng thời gian
- Mặc định: 30 ngày gần nhất
- Recommended ranges:
  - **Real-time monitoring**: Last 1h - Last 6h
  - **Daily analysis**: Last 24h - Last 7d
  - **Trend analysis**: Last 30d - Last 90d

### 5. Refresh Dashboard
- Dashboard không tự động refresh (manual refresh)
- Click nút refresh hoặc F5 để cập nhật dữ liệu mới nhất

---

## Yêu cầu cấu hình

### CloudWatch Metrics cho S3

S3 cung cấp 2 loại metrics:

#### 1. Storage Metrics (Miễn phí, tự động)
- **BucketSizeBytes**: Tự động enable
- **NumberOfObjects**: Tự động enable
- Update: 1 lần/ngày (không real-time)
- Không cần cấu hình gì thêm

#### 2. Request Metrics (Có phí, cần enable)
- **AllRequests, GetRequests, PutRequests, etc.**
- **FirstByteLatency, TotalRequestLatency**
- **BytesDownloaded, BytesUploaded**
- **4xxErrors, 5xxErrors**

**Cách enable Request Metrics**:

1. **Via AWS Console**:
   ```
   S3 Console → Chọn Bucket → Metrics tab → 
   Request metrics → Create filter
   ```

2. **Filter Configuration**:
   - **Entire bucket**: Apply to all objects
   - **Prefix/Tags**: Apply to specific objects
   - Mỗi filter tạo ra một FilterId

3. **Pricing**:
   - $0.01 per 1,000 metrics requested
   - Có chi phí nhưng rất hữu ích cho production monitoring

4. **Via AWS CLI**:
   ```bash
   aws s3api put-bucket-metrics-configuration \
     --bucket my-bucket \
     --id EntireBucket \
     --metrics-configuration '{"Id":"EntireBucket"}'
   ```

### IAM Permissions

User/Role chạy Grafana cần các permissions sau:

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
        "s3:ListAllMyBuckets",
        "s3:GetBucketLocation"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## Best Practices

### 1. Cost Optimization

**Monitoring Costs**:
- Storage Metrics: FREE
- Request Metrics: ~$0.01/1000 metrics
- CloudWatch API calls: Có chi phí nhỏ

**Optimization Tips**:
- Enable Request Metrics chỉ cho critical buckets
- Sử dụng filters để monitor specific prefixes thay vì entire bucket
- Set appropriate dashboard refresh intervals

**Reduce S3 Costs**:
- Monitor BytesDownloaded để identify expensive data transfers
- Use CloudFront để giảm S3 data transfer costs
- Implement S3 Lifecycle policies based on NumberOfObjects trends
- Review GetRequests vs PutRequests ratio để optimize access patterns

### 2. Performance Optimization

**Latency Optimization**:
- Target: FirstByteLatency < 100ms
- If higher:
  - Enable S3 Transfer Acceleration
  - Use CloudFront for frequent access
  - Consider Multi-Region Access Points

**Request Rate Optimization**:
- S3 supports 3,500 PUT/COPY/POST/DELETE và 5,500 GET/HEAD requests per second per prefix
- If hitting limits: Use more prefixes (sharding)
- Monitor ListRequests: Expensive and slow, consider caching

### 3. Security & Compliance

**Error Monitoring**:
- Alert on 403 errors: Potential unauthorized access attempts
- Alert on 404 spikes: Could indicate data deletion issues
- Monitor 5xx errors: AWS infrastructure issues

**Access Patterns**:
- Review unusual spikes in GetRequests (potential data exfiltration)
- Monitor DeleteRequests (accidental/malicious deletion)
- Track BytesDownloaded for compliance (data transfer monitoring)

**Enable Additional Logging**:
- S3 Server Access Logs: Detailed request logs
- CloudTrail: API-level logging for S3 management operations
- S3 Object-level logging in CloudTrail for data events

### 4. Capacity Planning

**Storage Growth**:
- Track BucketSizeBytes trend
- Project future storage needs
- Plan budget based on growth rate
- Consider S3 Intelligent-Tiering

**Request Volume**:
- Monitor AllRequests trend
- Plan for scaling (if approaching limits)
- Consider caching strategies
- Review architecture if consistently high

### 5. Alert Configuration

**Critical Alerts** (Immediate response):
- 5xxErrors > 0: AWS infrastructure issues
- Error rate > 5%: Serious problems
- FirstByteLatency > 1000ms: Performance degradation

**Warning Alerts** (Investigation needed):
- 4xxErrors > 1% of requests
- Error rate > 1%
- FirstByteLatency > 500ms
- Unusual spike in DeleteRequests

**Informational Alerts**:
- BucketSizeBytes growth > 20% week-over-week
- BytesDownloaded > monthly budget threshold
- NumberOfObjects reached lifecycle policy thresholds

---

## Troubleshooting Common Issues

### Issue 1: No Data Showing for Request Metrics

**Triệu chứng**: Panels 3-6 (Requests, Latency, Bytes, Errors) không có dữ liệu

**Nguyên nhân**:
- Request Metrics chưa được enable
- FilterId không đúng
- Bucket chưa có traffic
- Độ trễ CloudWatch metrics (vài phút)

**Giải pháp**:
1. Enable Request Metrics trong S3 Console
2. Đợi 15-30 phút để metrics xuất hiện
3. Thử chọn FilterId = "All"
4. Generate test traffic: upload/download một số files
5. Check IAM permissions

### Issue 2: BucketSizeBytes Not Updating

**Triệu chứng**: Panel 1 không cập nhật hoặc giá trị cũ

**Nguyên nhân**:
- Storage metrics chỉ update 1 lần/ngày
- Cần đợi UTC midnight để thấy update

**Giải pháp**:
- Đây là behavior bình thường của S3
- Không real-time, dùng cho trend analysis
- For real-time: Monitor NumberOfObjects và PutRequests

### Issue 3: High Latency

**Triệu chứng**: FirstByteLatency hoặc TotalRequestLatency cao

**Kiểm tra**:
1. Object size: Large objects = higher latency
2. Storage class: Glacier/Deep Archive cần restore
3. Region: Cross-region access = higher latency
4. AWS Service Health Dashboard

**Giải pháp**:
- Enable Transfer Acceleration
- Use CloudFront CDN
- Move bucket closer to users
- Optimize object sizes
- Use multipart upload/download

### Issue 4: High 4xxErrors

**Triệu chứng**: Nhiều 4xx errors, đặc biệt 403 hoặc 404

**Troubleshooting**:

**For 403 Forbidden**:
```bash
# Check bucket policy
aws s3api get-bucket-policy --bucket my-bucket

# Check IAM permissions
aws iam get-user-policy --user-name my-user --policy-name my-policy

# Test access
aws s3 ls s3://my-bucket/
```

**For 404 Not Found**:
- Check object exists: `aws s3 ls s3://my-bucket/path/to/object`
- Check object key spelling (case-sensitive)
- Review lifecycle policies (objects được deleted?)
- Check versioning if enabled

**Giải pháp**:
- Fix IAM policies/bucket policies
- Update application code với correct object keys
- Review and adjust lifecycle rules

### Issue 5: Unexpected High Costs

**Triệu chứng**: S3 bill cao bất thường

**Investigation**:
1. Check BytesDownloaded: High data transfer costs
2. Check AllRequests: High request costs
3. Check ListRequests: Expensive operations
4. Review storage class distribution

**Cost Breakdown**:
- Storage: $0.023/GB/month (Standard)
- Requests: $0.0004/1000 PUT, $0.0004/1000 GET
- Data Transfer: $0.09/GB (out to internet)

**Optimization**:
- Use CloudFront to reduce data transfer
- Implement caching to reduce requests
- Use S3 Intelligent-Tiering
- Enable lifecycle policies to move old data to cheaper storage

### Issue 6: Missing Metrics After Filter Change

**Triệu chứng**: Thay đổi FilterId không thấy data

**Nguyên nhân**:
- Filter mới chưa có historical data
- CloudWatch chỉ collect metrics từ khi filter được enable

**Giải pháp**:
- Đợi metrics accumulate (15-30 phút)
- Historical data chỉ có cho filters đã enable từ trước
- Không thể backfill metrics data

---

## Metrics Reference

| Metric | Unit | Statistic | Frequency | Cost | Panel |
|--------|------|-----------|-----------|------|-------|
| BucketSizeBytes | Bytes | Average | Daily | FREE | 1 |
| NumberOfObjects | Count | Average | Daily | FREE | 2 |
| AllRequests | Count | Sum | 1-minute | Paid | 3 |
| GetRequests | Count | Sum | 1-minute | Paid | 3 |
| PutRequests | Count | Sum | 1-minute | Paid | 3 |
| DeleteRequests | Count | Sum | 1-minute | Paid | 3 |
| HeadRequests | Count | Sum | 1-minute | Paid | 3 |
| PostRequests | Count | Sum | 1-minute | Paid | 3 |
| ListRequests | Count | Sum | 1-minute | Paid | 3 |
| FirstByteLatency | Milliseconds | Average | 1-minute | Paid | 4 |
| TotalRequestLatency | Milliseconds | Average | 1-minute | Paid | 4 |
| BytesDownloaded | Bytes | Sum | 1-minute | Paid | 5 |
| BytesUploaded | Bytes | Sum | 1-minute | Paid | 5 |
| 4xxErrors | Count | Sum | 1-minute | Paid | 6 |
| 5xxErrors | Count | Sum | 1-minute | Paid | 6 |

---

## Use Cases

### 1. Media/Content Delivery

**Scenario**: Video streaming platform, image hosting

**Key Metrics**:
- **BytesDownloaded**: Monitor bandwidth usage
- **GetRequests**: Track content access
- **FirstByteLatency**: User experience quality
- **BucketSizeBytes**: Storage costs

**Optimization**:
- Use CloudFront CDN (reduce S3 costs by 70-90%)
- Enable Transfer Acceleration
- Monitor peak hours for capacity planning

### 2. Data Lake / Big Data

**Scenario**: Log aggregation, analytics data storage

**Key Metrics**:
- **PutRequests**: Data ingestion rate
- **NumberOfObjects**: Growth tracking
- **BucketSizeBytes**: Storage capacity
- **ListRequests**: Query patterns

**Optimization**:
- Use appropriate prefixes (partitioning)
- Implement lifecycle policies
- Consider S3 Intelligent-Tiering
- Use S3 Select for query optimization

### 3. Backup & Archive

**Scenario**: Database backups, long-term archive

**Key Metrics**:
- **BucketSizeBytes**: Backup growth
- **PutRequests**: Backup frequency
- **BytesUploaded**: Backup sizes
- **NumberOfObjects**: Retention compliance

**Optimization**:
- Use Glacier for long-term storage
- Implement lifecycle transitions
- Monitor retention policies
- Set up versioning for critical data

### 4. Static Website Hosting

**Scenario**: SPA, documentation sites

**Key Metrics**:
- **GetRequests**: Page views
- **BytesDownloaded**: Traffic volume
- **FirstByteLatency**: Page load speed
- **4xxErrors**: Broken links (404s)

**Optimization**:
- Use CloudFront for better performance
- Enable gzip compression
- Cache-Control headers
- Monitor 404s to fix broken links

### 5. Application Data Storage

**Scenario**: User uploads, app assets

**Key Metrics**:
- **AllRequests**: Overall traffic
- **PutRequests vs GetRequests**: Usage ratio
- **4xxErrors**: Permission issues
- **BytesUploaded/Downloaded**: Data flow

**Optimization**:
- Proper IAM policies
- Presigned URLs for uploads
- CDN for frequently accessed content
- Implement upload size limits

---

## Integration với AWS Services khác

### CloudFront Integration
- Reduce S3 costs (cache at edge locations)
- Improve latency globally
- Monitor: CloudFront metrics + S3 metrics
- Setup: Origin = S3 bucket

### Lambda Integration
- Monitor: S3 events triggering Lambda
- Check: PutRequests correlation với Lambda invocations
- Optimize: Batch processing vs real-time

### Athena/Glue Integration
- Monitor: ListRequests (Athena queries)
- Optimize: Partition data properly
- Cost: Balance between S3 requests và query performance

### CloudTrail Integration
- Detailed audit logs for security
- Combine với S3 metrics để full picture
- Enable for compliance requirements

---

## Advanced Monitoring

### Custom Metrics & Calculations

**Error Rate**:
```
(4xxErrors + 5xxErrors) / AllRequests * 100
```

**Read/Write Ratio**:
```
GetRequests / PutRequests
```

**Average Object Size**:
```
BucketSizeBytes / NumberOfObjects
```

**Data Transfer Cost Estimate**:
```
BytesDownloaded * $0.09/GB (first 10TB)
```

### Cross-Bucket Comparison
- Create variables cho multiple buckets
- Compare performance across buckets
- Identify outliers

### Multi-Region Monitoring
- Setup dashboard cho mỗi region
- Compare latency across regions
- Optimize placement strategy

---

## Kết luận

Dashboard AWS S3 này cung cấp insights toàn diện về:

### Storage & Capacity
- **BucketSizeBytes**: Dung lượng và growth trends
- **NumberOfObjects**: Số lượng files và data distribution

### Performance
- **FirstByteLatency**: Response time quality
- **TotalRequestLatency**: End-to-end performance
- **Optimization**: CloudFront, Transfer Acceleration

### Usage & Traffic
- **Filtered Requests**: Chi tiết request patterns
- **BytesDownloaded/Uploaded**: Bandwidth consumption
- **Analysis**: Usage patterns và capacity planning

### Reliability & Errors
- **4xxErrors**: Client-side issues (permissions, not found)
- **5xxErrors**: Server-side issues (AWS infrastructure)
- **Monitoring**: Error rates và quick response

### Cost Optimization
- Identify expensive operations
- Data transfer monitoring
- Request optimization opportunities

**Sử dụng dashboard này để**:
✅ Đảm bảo S3 performance tốt  
✅ Phát hiện và troubleshoot issues nhanh chóng  
✅ Tối ưu hóa costs (storage, requests, data transfer)  
✅ Capacity planning và scaling  
✅ Security monitoring (unusual access patterns)  
✅ Compliance tracking (data retention, access logs)  

---

**Lưu ý quan trọng**:
- Storage metrics (Panels 1-2): FREE, update daily
- Request metrics (Panels 3-6): **Có phí**, cần enable, near real-time
- IAM permissions: Cấu hình đúng để access CloudWatch
- Cost awareness: Request Metrics có chi phí nhỏ nhưng rất valuable

**Version**: 1.0  
**Last Updated**: February 2026  
**Maintainer**: DevOps Team
