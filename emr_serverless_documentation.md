# Tài liệu Dashboard AWS EMR Serverless

## Tổng quan
Dashboard này được thiết kế để giám sát và theo dõi các metrics của AWS EMR Serverless thông qua Amazon CloudWatch. Dashboard cung cấp cái nhìn toàn diện về hiệu suất, tài nguyên và trạng thái của các ứng dụng EMR Serverless.

## Thông tin Dashboard
- **Tiêu đề**: AWS EMR Serverless
- **UID**: `emr-serverless-dashboard`
- **Phiên bản**: 4
- **Tags**: cloudwatch
- **Mô tả**: Visualize AWS EMR Serverless metrics

## Biến Template (Variables)

Dashboard sử dụng các biến template động để lọc và tùy chỉnh dữ liệu hiển thị:

### 1. **Datasource** (`$datasource`)
- **Loại**: CloudWatch datasource
- **Mô tả**: Nguồn dữ liệu CloudWatch để lấy metrics
- **Giá trị**: cloudwatch

### 2. **Region** (`$region`)
- **Loại**: Query
- **Mô tả**: AWS Region chứa EMR Serverless application
- **Nguồn**: CloudWatch regions
- **Mặc định**: default

### 3. **ApplicationId** (`$applicationid`)
- **Loại**: Query
- **Mô tả**: ID của EMR Serverless application
- **Hỗ trợ**: Multi-select với "All" option
- **Query**: `dimension_values($region,AWS/EMRServerless,RunningWorkerCount,ApplicationId)`

### 4. **ApplicationName** (`$applicationname`)
- **Loại**: Query
- **Mô tả**: Tên của EMR Serverless application
- **Hỗ trợ**: Multi-select với "All" option
- **Query**: `dimension_values($region,AWS/EMRServerless,RunningWorkerCount,ApplicationName)`

### 5. **WorkerType** (`$workertype`)
- **Loại**: Query
- **Mô tả**: Loại worker (Driver/Executor)
- **Hỗ trợ**: Multi-select với "All" option
- **Query**: `dimension_values($region,AWS/EMRServerless,RunningWorkerCount,WorkerType)`

### 6. **CapacityAllocationType** (`$capacityallocationtype`)
- **Loại**: Query
- **Mô tả**: Loại phân bổ capacity
- **Hỗ trợ**: Multi-select với "All" option
- **Query**: `dimension_values($region,AWS/EMRServerless,RunningWorkerCount,CapacityAllocationType)`

### 7. **JobId** (`$jobid`)
- **Loại**: Query
- **Mô tả**: ID của job đang chạy
- **Hỗ trợ**: Multi-select với "All" option
- **Query**: `dimension_values($region,AWS/EMRServerless,WorkerCpuAllocated,JobId)`

### 8. **JobName** (`$jobname`)
- **Loại**: Query
- **Mô tả**: Tên của job
- **Hỗ trợ**: Multi-select với "All" option
- **Query**: `dimension_values($region,AWS/EMRServerless,WorkerCpuAllocated,JobName)`

## Panels và Metrics

Dashboard bao gồm 10 panels chính, mỗi panel theo dõi các metrics khác nhau:

### Panel 1: Workers (Application Level)
**Vị trí**: Row 1, Full width (24 units), Height: 10
**Loại**: Time Series

**Metrics theo dõi**:
1. **RunningWorkerCount** (Average)
   - Số lượng workers đang chạy
   
2. **TotalWorkerCount** (Average)
   - Tổng số workers

3. **IdleWorkerCount** (Average)
   - Số workers đang rảnh

4. **PendingCreationWorkerCount** (Average)
   - Số workers đang chờ khởi tạo

**Dimensions**: ApplicationId, ApplicationName, CapacityAllocationType, WorkerType

**Cấu hình**:
- Min value: 0
- Unit: none
- Legend: Hiển thị mean, max, min

**Mục đích**: Theo dõi trạng thái và số lượng workers để đánh giá khả năng xử lý của application.

---

### Panel 2: Jobs (Application Level)
**Vị trí**: Row 2, Full width (24 units), Height: 18
**Loại**: Time Series

**Metrics theo dõi**:
1. **SubmittedJobs** (Average)
   - Jobs đã được submit

2. **PendingJobs** (Average)
   - Jobs đang chờ xử lý

3. **ScheduledJobs** (Average)
   - Jobs đã được lên lịch

4. **RunningJobs** (Average)
   - Jobs đang chạy

5. **SuccessJobs** (Average)
   - Jobs hoàn thành thành công

6. **FailedJobs** (Average)
   - Jobs thất bại

7. **CancellingJobs** (Average)
   - Jobs đang được hủy

8. **CancelledJobs** (Average)
   - Jobs đã bị hủy

**Dimensions**: ApplicationId, ApplicationName

**Cấu hình**:
- Min value: 0
- Unit: none
- Legend: Hiển thị mean, max, min

**Mục đích**: Giám sát lifecycle và trạng thái của các jobs để phát hiện vấn đề và tối ưu hiệu suất.

---

### Panel 3: CPU (Application Level)
**Vị trí**: Row 3 Left, Width: 12 units, Height: 7
**Loại**: Time Series

**Metrics theo dõi**:
1. **CPUAllocated** (Average)
   - Số CPU đã được phân bổ

2. **MaxCPUAllowed** (Average)
   - Số CPU tối đa được phép

**Dimensions**:
- CPUAllocated: ApplicationId, ApplicationName, CapacityAllocationType, WorkerType
- MaxCPUAllowed: ApplicationId, ApplicationName

**Cấu hình**:
- Min value: 0
- Unit: short
- Legend: Hiển thị mean, max, min

**Mục đích**: Theo dõi việc sử dụng CPU để đảm bảo không vượt quá giới hạn và tối ưu resource allocation.

---

### Panel 4: Memory (Application Level)
**Vị trí**: Row 3 Right, Width: 12 units, Height: 7
**Loại**: Time Series

**Metrics theo dõi**:
1. **MemoryAllocated** (Average)
   - Memory đã được phân bổ

2. **MaxMemoryAllowed** (Average)
   - Memory tối đa được phép

**Dimensions**:
- MemoryAllocated: ApplicationId, ApplicationName, CapacityAllocationType, WorkerType
- MaxMemoryAllowed: ApplicationId, ApplicationName

**Cấu hình**:
- Min value: 0
- Unit: decgbytes (GB)
- Legend: Hiển thị mean, max, min

**Mục đích**: Giám sát việc sử dụng bộ nhớ để tránh OOM (Out of Memory) và tối ưu performance.

---

### Panel 5: Storage (Application Level)
**Vị trí**: Row 4, Full width (24 units), Height: 7
**Loại**: Time Series

**Metrics theo dõi**:
1. **StorageAllocated** (Average)
   - Storage đã được phân bổ

2. **MaxStorageAllowed** (Average)
   - Storage tối đa được phép

**Dimensions**:
- StorageAllocated: ApplicationId, ApplicationName, CapacityAllocationType, WorkerType
- MaxStorageAllowed: ApplicationId, ApplicationName

**Cấu hình**:
- Min value: 0
- Unit: decgbytes (GB)
- Legend: Hiển thị mean, max, min

**Mục đích**: Theo dõi việc sử dụng storage để đảm bảo đủ dung lượng cho các jobs.

---

### Panel 6: Job Worker CPU
**Vị trí**: Row 5 Left, Width: 12 units, Height: 7
**Loại**: Time Series

**Metrics theo dõi**:
1. **WorkerCpuAllocated** (Sum)
   - Tổng CPU được phân bổ cho workers

2. **WorkerCpuUsed** (Sum)
   - Tổng CPU đang được sử dụng bởi workers

**Dimensions**: ApplicationId, ApplicationName, CapacityAllocationType, JobId, JobName, WorkerType

**Cấu hình**:
- Min value: 0
- Unit: short
- Legend: Hiển thị mean, max, min

**Mục đích**: Theo dõi hiệu suất CPU ở mức worker để xác định bottlenecks và optimize resource usage.

---

### Panel 7: Job Worker Memory
**Vị trí**: Row 5 Right, Width: 12 units, Height: 7
**Loại**: Time Series

**Metrics theo dõi**:
1. **WorkerMemoryAllocated** (Sum)
   - Tổng memory được phân bổ cho workers

2. **WorkerMemoryUsed** (Sum)
   - Tổng memory đang được sử dụng bởi workers

**Dimensions**: ApplicationId, ApplicationName, CapacityAllocationType, JobId, JobName, WorkerType

**Cấu hình**:
- Min value: 0
- Unit: decgbytes (GB)
- Legend: Hiển thị mean, max, min

**Mục đích**: Giám sát việc sử dụng memory của workers để tối ưu allocation và phát hiện memory leaks.

---

### Panel 8: Job Worker Ephemeral Storage
**Vị trí**: Row 6 Left, Width: 12 units, Height: 7
**Loại**: Time Series

**Metrics theo dõi**:
1. **WorkerEphemeralStorageAllocated** (Sum)
   - Tổng ephemeral storage được phân bổ

2. **WorkerEphemeralStorageUsed** (Sum)
   - Tổng ephemeral storage đang được sử dụng

**Dimensions**: ApplicationId, ApplicationName, CapacityAllocationType, JobId, JobName, WorkerType

**Cấu hình**:
- Min value: 0
- Unit: decgbytes (GB)
- Legend: Hiển thị mean, max, min

**Mục đích**: Theo dõi việc sử dụng ephemeral storage (temporary storage) của workers.

---

### Panel 9: Job Worker Storage I/O
**Vị trí**: Row 6 Right, Width: 12 units, Height: 7
**Loại**: Time Series

**Metrics theo dõi**:
1. **WorkerStorageReadBytes** (Sum)
   - Tổng bytes đọc từ storage

2. **WorkerStorageWriteBytes** (Sum)
   - Tổng bytes ghi vào storage

**Dimensions**: ApplicationId, ApplicationName, CapacityAllocationType, JobId, JobName, WorkerType

**Cấu hình**:
- Min value: 0
- Unit: bytes
- Legend: Hiển thị mean, max, min

**Mục đích**: Giám sát I/O operations để xác định performance bottlenecks liên quan đến disk I/O.

---

### Panel 10: Documentation
**Vị trí**: Row 7, Full width (24 units), Height: 3
**Loại**: Text Panel (HTML)

**Nội dung**: Links đến documentation
- AWS EMR Serverless Documentation
- EMR Serverless CloudWatch Metrics
- AWS EMR Serverless CloudWatch Dashboard Template

**Mục đích**: Cung cấp quick access đến tài liệu tham khảo và resources.

---

## Cấu hình Time Range

- **Mặc định**: 24 giờ gần nhất (now-24h to now)
- **Refresh**: Manual (có thể cấu hình auto-refresh)
- **Timezone**: Browser timezone

## Namespace CloudWatch

Tất cả metrics được lấy từ namespace: **AWS/EMRServerless**

## Use Cases và Best Practices

### 1. **Monitoring Job Performance**
- Theo dõi **Jobs panel** để xem job lifecycle
- Kiểm tra **FailedJobs** metric để phát hiện lỗi
- So sánh **RunningJobs** với **SuccessJobs** để đánh giá throughput

### 2. **Resource Optimization**
- So sánh **Allocated** vs **Allowed** metrics để tối ưu resource limits
- Theo dõi **IdleWorkerCount** để điều chỉnh auto-scaling policies
- Kiểm tra **CPU/Memory Used** vs **Allocated** để tối ưu worker configuration

### 3. **Cost Management**
- Monitor **TotalWorkerCount** và **RunningWorkerCount**
- Theo dõi **Storage** metrics để quản lý storage costs
- Sử dụng **Worker metrics** để optimize instance sizing

### 4. **Troubleshooting**
- Kiểm tra **PendingCreationWorkerCount** để xác định capacity issues
- Monitor **FailedJobs** cùng với **Worker metrics** để root cause analysis
- Theo dõi **Storage I/O** để phát hiện I/O bottlenecks

### 5. **Capacity Planning**
- Analyze peak usage patterns từ **Workers** và **Jobs** panels
- So sánh **Max Allowed** với usage patterns để plan capacity
- Monitor trends để dự đoán future resource needs

## Alerting Recommendations

Nên thiết lập alerts cho các metrics sau:

1. **FailedJobs > threshold**: Cảnh báo khi có nhiều jobs fail
2. **CPUAllocated / MaxCPUAllowed > 80%**: Gần đạt CPU limit
3. **MemoryAllocated / MaxMemoryAllowed > 80%**: Gần đạt memory limit
4. **PendingJobs > threshold**: Có nhiều jobs pending chưa được xử lý
5. **WorkerStorageUsed / WorkerStorageAllocated > 90%**: Gần hết storage

## Tích hợp và Dependencies

- **Datasource**: Yêu cầu CloudWatch datasource được cấu hình trong Grafana
- **IAM Permissions**: Cần permissions để đọc CloudWatch metrics từ namespace AWS/EMRServerless
- **Grafana Version**: Compatible với Grafana 11.3.1+

## Customization

Dashboard có thể được tùy chỉnh:
- Thêm/bớt panels theo nhu cầu monitoring
- Điều chỉnh time range và refresh intervals
- Tùy chỉnh legend calculations (thêm percentiles, etc.)
- Thêm annotations để đánh dấu deployments hoặc incidents

## Tài liệu tham khảo

- [AWS EMR Serverless Documentation](https://docs.aws.amazon.com/emr/latest/EMR-Serverless-UserGuide/emr-serverless.html)
- [EMR Serverless CloudWatch Metrics](https://docs.aws.amazon.com/emr/latest/EMR-Serverless-UserGuide/app-job-metrics.html)
- [AWS EMR Serverless CloudWatch Dashboard Template](https://github.com/aws-samples/emr-serverless-samples/tree/main/cloudformation/emr-serverless-cloudwatch-dashboard/)
