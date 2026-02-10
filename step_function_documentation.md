# Tài liệu Dashboard AWS Step Functions

## Tổng quan
Dashboard này được thiết kế để giám sát và theo dõi các metrics của AWS Step Functions thông qua CloudWatch. Dashboard cung cấp cái nhìn toàn diện về hiệu suất, trạng thái thực thi và các vấn đề tiềm ẩn của State Machine.

**Nguồn dữ liệu**: CloudWatch  
**Grafana Dashboard ID**: 11099  
**Khoảng thời gian mặc định**: 24 giờ gần nhất  

---

## Cấu hình

### Biến Template (Template Variables)

Dashboard sử dụng các biến động để cho phép người dùng tùy chỉnh view:

1. **datasource**: Nguồn dữ liệu CloudWatch
2. **agg (Aggregation)**: Khoảng thời gian tổng hợp dữ liệu
   - Tự động: auto (mặc định)
   - Các giá trị khác: 1s, 5s, 10s, 30s, 1m, 5m, 15m, 1h, 6h, 1d, 7d, 30d
3. **region**: Vùng AWS (AWS Region)
4. **stateMachineArn**: ARN của State Machine cần giám sát

---

## Chi tiết các Dashboard Panel

### 1. ExecutionsSucceeded (Thực thi thành công)

**Vị trí**: Hàng 1, chiều cao 7 đơn vị  
**Loại biểu đồ**: Time series (Bar chart)

**Mô tả**:  
Hiển thị số lượng executions (lần thực thi) của Step Function đã hoàn thành thành công theo thời gian.

**Metrics**:
- Namespace: `AWS/States`
- Metric: `ExecutionsSucceeded`
- Statistic: `Sum`
- Dimension: `StateMachineArn`

**Cấu hình hiển thị**:
- Đơn vị: Không có (count)
- Số thập phân: 0
- Giá trị tối thiểu: 0
- Hiển thị: Biểu đồ cột (bars)
- Legend: Hiển thị các giá trị mean, max, min, sum

**Ý nghĩa**:  
Đây là metric quan trọng nhất để đánh giá sức khỏe của Step Function. Số lượng execution thành công cao cho thấy hệ thống hoạt động tốt.

---

### 2. ExecutionTime (Thời gian thực thi)

**Vị trí**: Hàng 2, chiều cao 7 đơn vị  
**Loại biểu đồ**: Time series (Kết hợp bar và line chart)

**Mô tả**:  
Hiển thị thời gian thực thi trung bình và tối đa của các executions theo thời gian.

**Metrics**:
- Namespace: `AWS/States`
- Metric: `ExecutionTime`
- Statistics: 
  - `Average` (Biểu đồ cột)
  - `Maximum` (Đường line, hiển thị trên trục phải)
- Dimension: `StateMachineArn`

**Cấu hình hiển thị**:
- Đơn vị: milliseconds (ms)
- Giá trị tối thiểu: 0
- Màu sắc:
  - Maximum: Đỏ (#e24d42)
  - Average: Màu mặc định
- Legend: Hiển thị các giá trị mean, max, min, sum

**Ý nghĩa**:  
Giúp theo dõi hiệu suất của Step Function. Thời gian thực thi tăng đột biến hoặc liên tục cao có thể chỉ ra vấn đề về hiệu năng cần được khắc phục.

---

### 3. ExecutionsStarted (Thực thi được bắt đầu)

**Vị trí**: Hàng 3, chiều cao 7 đơn vị  
**Loại biểu đồ**: Time series (Bar chart)

**Mô tả**:  
Hiển thị số lượng executions được khởi động theo thời gian.

**Metrics**:
- Namespace: `AWS/States`
- Metric: `ExecutionsStarted`
- Statistic: `Sum`
- Dimension: `StateMachineArn`

**Cấu hình hiển thị**:
- Đơn vị: Không có (count)
- Giá trị tối thiểu: 0
- Hiển thị: Biểu đồ cột (bars)
- Legend: Hiển thị các giá trị mean, max, min, sum

**Ý nghĩa**:  
Cho biết tần suất Step Function được trigger. So sánh metric này với ExecutionsSucceeded và ExecutionsFailed để tính tỷ lệ thành công/thất bại.

---

### 4. ExecutionsFailed (Thực thi thất bại)

**Vị trí**: Hàng 4, chiều cao 7 đơn vị  
**Loại biểu đồ**: Time series (Bar chart)

**Mô tả**:  
Hiển thị số lượng executions thất bại theo thời gian.

**Metrics**:
- Namespace: `AWS/States`
- Metric: `ExecutionsFailed`
- Statistic: `Sum`
- Dimension: `StateMachineArn`

**Cấu hình hiển thị**:
- Đơn vị: Không có (count)
- Số thập phân: 0
- Giá trị tối thiểu: 0
- Màu sắc: Đỏ (red) - nhấn mạnh cảnh báo
- Hiển thị: Biểu đồ cột (bars)
- Legend: Hiển thị các giá trị mean, max, min, sum

**Ý nghĩa**:  
Metric quan trọng để phát hiện lỗi. Cần thiết lập alert khi giá trị này tăng đột biến. Mọi giá trị khác 0 đều cần được điều tra nguyên nhân.

---

### 5. ExecutionsTimedOut (Thực thi bị timeout)

**Vị trí**: Hàng 5, chiều cao 7 đơn vị  
**Loại biểu đồ**: Time series (Bar chart)

**Mô tả**:  
Hiển thị số lượng executions bị timeout (vượt quá thời gian cho phép) theo thời gian.

**Metrics**:
- Namespace: `AWS/States`
- Metric: `ExecutionsTimedOut`
- Statistic: `Sum`
- Dimension: `StateMachineArn`

**Cấu hình hiển thị**:
- Đơn vị: Không có (count)
- Số thập phân: 0
- Giá trị tối thiểu: 0
- Hiển thị: Biểu đồ cột (bars)
- Legend: Hiển thị các giá trị mean, max, min, sum

**Ý nghĩa**:  
Chỉ ra các executions bị dừng do vượt quá thời gian timeout được cấu hình. Nếu metric này thường xuyên xuất hiện, cần xem xét:
- Tăng timeout setting
- Tối ưu hóa các bước xử lý trong workflow
- Kiểm tra các service phụ thuộc

---

### 6. ExecutionsAborted (Thực thi bị hủy)

**Vị trí**: Hàng 6, chiều cao 7 đơn vị  
**Loại biểu đồ**: Time series (Bar chart)

**Mô tả**:  
Hiển thị số lượng executions bị hủy bỏ (aborted) theo thời gian.

**Metrics**:
- Namespace: `AWS/States`
- Metric: `ExecutionsAborted`
- Statistic: `Sum`
- Dimension: `StateMachineArn`

**Cấu hình hiển thị**:
- Đơn vị: Không có (count)
- Số thập phân: 0
- Giá trị tối thiểu: 0
- Màu sắc: Vàng (yellow) - cảnh báo
- Hiển thị: Biểu đồ cột (bars)
- Legend: Hiển thị các giá trị mean, max, min, sum

**Ý nghĩa**:  
Executions có thể bị abort do:
- User manual intervention
- Lỗi logic trong workflow
- External system request
Cần điều tra nguyên nhân khi metric này xuất hiện.

---

### 7. ExecutionThrottled (Thực thi bị giới hạn)

**Vị trí**: Hàng 7, chiều cao 7 đơn vị  
**Loại biểu đồ**: Time series (Bar chart)

**Mô tả**:  
Hiển thị số lượng executions bị throttle (giới hạn tốc độ) do vượt quá giới hạn API của AWS Step Functions.

**Metrics**:
- Namespace: `AWS/States`
- Metric: `ExecutionThrottled`
- Statistic: `Sum`
- Dimension: `StateMachineArn`

**Cấu hình hiển thị**:
- Đơn vị: Không có (count)
- Số thập phân: 0
- Giá trị tối thiểu: 0
- Màu sắc: Vàng/Cam (#EAB839)
- Hiển thị: Biểu đồ cột (bars)
- Legend: Hiển thị các giá trị mean, max, min, sum

**Ý nghĩa**:  
Throttling xảy ra khi vượt quá giới hạn API calls của AWS Step Functions. Khi metric này xuất hiện, cần:
- Request tăng service quota với AWS
- Implement retry logic với exponential backoff
- Cân nhắc batch processing hoặc queueing
- Tối ưu hóa số lượng executions đồng thời

---

## Hướng dẫn sử dụng

### 1. Chọn State Machine để giám sát
- Sử dụng dropdown **StateMachineArn** ở phía trên dashboard
- Chọn ARN của State Machine cần theo dõi

### 2. Điều chỉnh khoảng thời gian
- Sử dụng time picker (góc trên bên phải) để chọn khoảng thời gian
- Mặc định: 24 giờ gần nhất
- Có thể chọn: Last 1h, Last 7d, Last 30d hoặc custom range

### 3. Điều chỉnh aggregation
- Sử dụng dropdown **Aggregation** để thay đổi độ chi tiết của dữ liệu
- Auto: Tự động điều chỉnh dựa trên time range
- Các giá trị cụ thể: 1m, 5m, 15m, 1h, etc.

### 4. Chọn Region
- Sử dụng dropdown **Region** để chọn AWS Region
- Dashboard sẽ hiển thị metrics từ region được chọn

---

## Best Practices

### 1. Giám sát proactive
- Thường xuyên kiểm tra ExecutionsFailed và ExecutionsTimedOut
- Thiết lập alerts cho các metrics quan trọng
- Theo dõi ExecutionTime để phát hiện sớm vấn đề hiệu năng

### 2. Phân tích xu hướng
- So sánh ExecutionsStarted vs ExecutionsSucceeded để tính success rate
- Theo dõi xu hướng ExecutionTime để phát hiện performance degradation
- Kiểm tra correlation giữa các metrics

### 3. Troubleshooting
Khi phát hiện vấn đề:
1. Kiểm tra ExecutionsFailed - lỗi xảy ra
2. Xem ExecutionsTimedOut - timeout issues
3. Kiểm tra ExecutionThrottled - API limit issues
4. Phân tích ExecutionTime - performance bottlenecks

### 4. Capacity Planning
- Theo dõi ExecutionsStarted để dự đoán tải
- Sử dụng ExecutionThrottled để biết khi nào cần tăng quota
- Phân tích ExecutionTime để optimize resource allocation

---

## Metrics Reference

| Metric | Unit | Mô tả | Statistic | Màu sắc |
|--------|------|-------|-----------|---------|
| ExecutionsSucceeded | Count | Số lần thực thi thành công | Sum | Mặc định |
| ExecutionTime | Milliseconds | Thời gian thực thi | Average, Maximum | Maximum: Đỏ |
| ExecutionsStarted | Count | Số lần thực thi được bắt đầu | Sum | Mặc định |
| ExecutionsFailed | Count | Số lần thực thi thất bại | Sum | Đỏ |
| ExecutionsTimedOut | Count | Số lần thực thi bị timeout | Sum | Mặc định |
| ExecutionsAborted | Count | Số lần thực thi bị hủy | Sum | Vàng |
| ExecutionThrottled | Count | Số lần bị giới hạn API | Sum | Vàng/Cam |

---

## Alerting Recommendations

### Critical Alerts (Ưu tiên cao)
1. **ExecutionsFailed > 0**: Alert ngay lập tức
2. **Success Rate < 95%**: (ExecutionsSucceeded / ExecutionsStarted) * 100
3. **ExecutionTime Maximum > threshold**: Tùy thuộc vào SLA

### Warning Alerts (Cảnh báo)
1. **ExecutionsTimedOut > 0**: Cần điều tra
2. **ExecutionThrottled > 0**: Approaching API limits
3. **ExecutionTime Average tăng > 20%**: So với baseline

### Informational
1. **ExecutionsAborted**: Theo dõi và ghi log
2. **ExecutionsStarted**: Monitoring traffic patterns

---

## Troubleshooting Common Issues

### Issue 1: High Execution Failures
**Triệu chứng**: ExecutionsFailed cao  
**Kiểm tra**:
- CloudWatch Logs của Step Function
- Error messages trong execution history
- Dependencies (Lambda, API Gateway, etc.)

**Giải pháp**:
- Fix code logic errors
- Add error handling và retry logic
- Validate input data

### Issue 2: Slow Execution Time
**Triệu chứng**: ExecutionTime tăng cao  
**Kiểm tra**:
- Lambda function duration
- External API response time
- Database query performance

**Giải pháp**:
- Optimize Lambda code
- Add caching
- Parallel processing where possible
- Optimize database queries

### Issue 3: Throttling
**Triệu chứng**: ExecutionThrottled > 0  
**Kiểm tra**:
- Current API limits
- Execution rate
- Concurrent executions

**Giải pháp**:
- Request service quota increase
- Implement queue-based processing
- Add exponential backoff retry
- Distribute load across multiple State Machines

### Issue 4: Timeouts
**Triệu chứng**: ExecutionsTimedOut > 0  
**Kiểm tra**:
- Timeout configuration
- Long-running tasks
- Blocking operations

**Giải pháp**:
- Increase timeout if legitimate
- Break down long tasks
- Implement async processing
- Add progress checkpoints

---

## Technical Details

### Data Source
- **Type**: CloudWatch
- **Namespace**: AWS/States
- **Refresh**: Manual (có thể cấu hình auto-refresh)

### Dashboard Configuration
- **Schema Version**: 39
- **Timezone**: Browser timezone
- **Refresh**: Manual
- **Tags**: cloudwatch

### Panel Configuration
- **Visualization**: Time series (bars và lines)
- **Legend**: Table format với calcs (mean, max, min, sum)
- **Tooltip**: Multi-series mode
- **Axis**: Auto-placement, linear scale

---

## Kết luận

Dashboard AWS Step Functions này cung cấp cái nhìn toàn diện về:
- **Hiệu suất** (ExecutionTime)
- **Độ tin cậy** (ExecutionsSucceeded, ExecutionsFailed)
- **Capacity** (ExecutionsStarted, ExecutionThrottled)
- **Vấn đề tiềm ẩn** (ExecutionsTimedOut, ExecutionsAborted)

Sử dụng dashboard này thường xuyên để:
- Đảm bảo Step Functions hoạt động ổn định
- Phát hiện và khắc phục sự cố nhanh chóng
- Tối ưu hóa hiệu năng
- Planning capacity cho tương lai

---

**Lưu ý**: Dashboard này yêu cầu:
- CloudWatch datasource đã được cấu hình trong Grafana
- IAM permissions để đọc CloudWatch metrics
- AWS Step Functions đang chạy và có dữ liệu metrics

**Version**: 1.0  
**Last Updated**: February 2026
