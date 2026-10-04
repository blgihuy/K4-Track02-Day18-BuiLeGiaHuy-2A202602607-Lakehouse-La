# ARCHITECTURE DESIGN BRIEF: LLM OBSERVABILITY LAKEHOUSE AT 1B REQS/DAY SCALE

- **Hệ thống thiết kế:** Nền tảng Lakehouse Observability & Telemetry cho Foundation-Model API
- **Quy mô:** 1.000.000.000 requests/ngày (~5 TB dữ liệu thô/ngày)
- **Tác giả:** Bùi Lê Gia Huy (MSSV: 2A202602607)
- **Repo bài làm:** `K4-Track02-Day18-BuiLeGiaHuy-2A202602607-Lakehouse-La`
- **Mã bài nộp:** `K4-Track02-Day18` (Bonus Challenge — Topic A)

---

## 1. Vấn đề kinh doanh & Các ràng buộc hệ thống

### 1.1. Bối cảnh & Mục tiêu
Một đội ngũ kỹ thuật vận hành mô hình ngôn ngữ lớn (Foundation-Model API) phục vụ hơn **10.000 khách hàng doanh nghiệp (tenants)** trên toàn cầu. Hệ thống ghi nhận mọi lượt gọi API với lưu lượng trung bình **11.574 requests/giây**, đạt đỉnh (peak) lên tới **25.000 requests/giây**. Mỗi request mang payload trung bình **5 KB** (bao gồm prompt, completion, model, latency, token usage, tenant_id, timestamp, HTTP headers).

Tổng khối lượng dữ liệu thô chạm mốc **5 TB/ngày** (khoảng **150 TB/tháng** dữ liệu chưa nén).

### 1.2. Ràng buộc cứng (Hard Constraints)
1. **Độ tươi Dashboard (SLA Freshness):** Dashboard theo dõi chi phí (token billing) và độ trễ ($p50, p95, p99$) theo từng tenant phải được làm mới **mỗi 5 phút**; độ trễ truy vấn điểm (point query theo `tenant_id`) trên dashboard phải dưới **1 giây**.
2. **Vòng đời dữ liệu (Lifecycle & Retention):**
   - Nội dung chi tiết (prompt/response đầy đủ) được lưu giữ **7 ngày** phục vụ điều tra sự cố (incident triage, debugging, hallucinations review).
   - Sau 7 ngày, toàn bộ text payload phải được loại bỏ an toàn; chỉ lưu giữ các chỉ số tổng hợp (aggregates) và siêu dữ liệu kỹ thuật trong vòng **1 năm (365 ngày)**.
3. **Bảo mật & Quyền riêng tư (Privacy & Compliance):** Dữ liệu cá nhân (PII như Email, SĐT, Số CCCD/Passport, API Keys) bắt buộc phải được khử định danh (redact/tokenize) **trước khi bất kỳ kỹ sư hoặc nhà phân tích nào có quyền truy cập**.
4. **Ràng buộc ngân sách (FinOps Cap):** Tổng chi phí lưu trữ (Storage budget) do CFO phê duyệt bị giới hạn ở mức trần **$\le \$5.000$ USD/tháng**.

---

## 2. Sơ đồ kiến trúc tổng thể (Architecture Diagram)

```
                    [ 1B REQS/DAY (Peak 25K/s) ]
                                 │
                                 ▼
                   ┌───────────────────────────┐
                   │ Kafka / Kinesis Ingestion │
                   │  Buffer (1-min microbatch)│
                   └─────────────┬─────────────┘
                                 │
     ┌───────────────────────────┴───────────────────────────┐
     │ STREAMING INGESTION WORKER (Structured Streaming)     │
     │ Tokenization & PII Masking Gateway (Presidio/Salt Hashing)
     └───────────────────────────┬───────────────────────────┘
                                 │
                                 ▼
┌────────────────────────────────────────────────────────────────────────┐
│                        BRONZE LAKEHOUSE LAYER                          │
│  Path: s3://lakehouse/bronze/llm_requests_raw/                         │
│  Format: Apache Iceberg v2 / Snappy Parquet                           │
│  Retention: 7 days TTL (S3 Lifecycle + Snapshot Expiry)                │
│  Payload: Raw encrypted tokens, unmasked headers, raw JSON             │
└────────────────────────────────┬───────────────────────────────────────┘
                                 │
                  Hourly Micro-ETL (Iceberg CDF)
                                 │
                                 ▼
┌────────────────────────────────────────────────────────────────────────┐
│                        SILVER LAKEHOUSE LAYER                          │
│  Path: s3://lakehouse/silver/llm_requests_cleansed/                    │
│  Format: Apache Iceberg v2 / zstandard Parquet                         │
│  Clustering: Z-ORDER BY (tenant_id, event_time)                        │
│  Partitions: day(event_time) (Hidden Partitioning)                     │
│  Payload: Redacted prompt/response, parsed tokens, latency, status     │
│  Retention: 7 days full text; 30 days telemetry in S3-IA               │
└────────────────────────────────┬───────────────────────────────────────┘
                                 │
               Incremental Streaming Aggregation (Every 5 mins)
                                 │
                                 ▼
┌────────────────────────────────────────────────────────────────────────┐
│                         GOLD LAKEHOUSE LAYER                           │
│  Path: s3://lakehouse/gold/tenant_metrics_5min/                        │
│  Format: Apache Iceberg v2 / Dictionary Parquet                        │
│  Partitions: month(event_time) | Z-ORDER: tenant_id                    │
│  Payload: p50/p95/p99 latency, cost_usd, error_rate, token sums        │
│  Retention: 365 days (1 year) in S3 Standard                           │
└────────────────────────────────┬───────────────────────────────────────┘
                                 │
                Zero-Copy Query Engine (DuckDB / Trino)
                                 ▼
         ┌───────────────────────────────────────────────┐
         │ Real-time Tenant Billing & Ops Dashboards     │
         │ (Point query latency < 500ms via File Pruning)│
         └───────────────────────────────────────────────┘
```

---

## 3. Năm quyết định kiến trúc cốt lõi (5 Architectural Decisions)

### Quyết định 1: Lựa chọn Định dạng Bảng & Catalog Control Plane
- **Lựa chọn:** **Apache Iceberg v2 kết hợp REST Catalog (Apache Polaris / Project Nessie)**.
- **Lý do lựa chọn:**
  - *Hidden Partitioning:* Cho phép phân vùng theo $day(event\_time)$ mà người dùng vẫn có thể truy vấn tự nhiên trên timestamp nguồn mà không bao giờ bị rơi vào thảm họa Full Table Scan.
  - *Partition Evolution:* Cho phép chuyển đổi linh hoạt kích thước phân vùng (từ 1 giờ sang 1 ngày) mà không cần rewrite toàn bộ các petabyte dữ liệu cũ.
  - *Vendor-neutral REST Catalog:* Đóng vai trò Control Plane độc lập, không bị ràng buộc vào nền tảng đóng của Databricks Unity Catalog hay Snowflake.
- **Phương án bị loại (Rejected Alternatives):**
  1. *Delta Lake kết hợp Unity Catalog:* Bị loại vì phụ thuộc chặt chẽ vào hệ sinh thái thương mại của Databricks; chi phí bản quyền catalog cho 1 tỷ sự kiện/ngày là quá lớn, không khả thi với trần ngân sách \$5.000/tháng.
  2. *Hive Metastore truyền thống:* Bị loại vì không hỗ trợ giao dịch ACID, cơ chế partition dựa trên cây thư mục vật lý gây nghẽn S3 metadata, và không có khả năng bảo vệ người dùng khỏi lỗi quên filter trên cột partition.

---

### Quyết định 2: Chiến lược Ingestion & Kiến trúc Compaction 2 cấp
- **Lựa chọn:** **Two-tier Compaction: Micro-batching 1 phút ghi đệm $\rightarrow$ Asynchronous Compaction theo giờ thành file 256 MB**.
- **Lý do lựa chọn:**
  - Với 1 tỷ requests/ngày, nếu ghi từng request sẽ sinh ra 1 tỷ file Parquet/ngày $\rightarrow$ hệ thống sập hoàn toàn do Small-file Problem.
  - Ingestion worker gom dữ liệu thành micro-batch 1 phút (sinh ra ~60 file/giờ, mỗi file ~3.5 MB).
  - Một cron job bảo trì nền (`rewrite_data_files`) chạy mỗi giờ gom các file 3.5 MB thành các file chuẩn **256 MB**, vừa tối ưu hóa khả năng nén zstandard vừa duy trì đủ số lượng file để cơ chế Z-order skipping hoạt động hiệu quả.
- **Phương án bị loại (Rejected Alternatives):**
  1. *Synchronous Compaction ngay khi ghi:* Bị loại vì gây xung đột ghi (write conflicts/optimistic locking failures) liên tục khi có 25.000 requests/giây đổ về; làm tăng độ trễ ingestion lên hàng chục giây.
  2. *Direct S3 Object PUT per Request:* Bị loại vì chi phí S3 PUT request cho 1 tỷ file là:
     $$\frac{1.000.000.000}{1.000} \times \$0,005 = \$5.000\text{/ngày} = \$150.000\text{/tháng}$$
     (Vượt ngân sách ngay lập tức chỉ riêng tiền request).

---

### Quyết định 3: Khử định danh PII (PII Tokenization & Redaction Boundary)
- **Lựa chọn:** **Tokenization Gateway tại biên Bronze $\rightarrow$ Silver bằng thư viện Microsoft Presidio kết hợp Salted Hashing**.
- **Lý do lựa chọn:**
  - Tầng Bronze lưu trữ payload thô trong trạng thái mã hóa KMS (Key Management Service) với quyền truy cập bị khóa hoàn toàn đối với con người.
  - Khi ETL chuyển dịch sang Silver, toàn bộ PII nhạy cảm (Email, SĐT, API Keys) được nhận diện và thay thế bằng mã băm giả danh (pseudonymized tokens).
  - Đảm bảo tính tuân thủ pháp lý (GDPR Article 17, Nghị định 13/2023/NĐ-CP): nhà phân tích và model training chỉ nhìn thấy tầng Silver đã sạch PII 100%.
- **Phương án bị loại (Rejected Alternatives):**
  1. *View-based Dynamic Masking tại thời điểm truy vấn:* Bị loại vì chi phí Regex scan trên 1 tỷ dòng mỗi khi query dashboard sẽ làm tăng latency lên hàng phút và ngốn hàng nghìn vCPU.
  2. *Client-side Pre-encryption toàn bộ:* Bị loại vì mã hóa toàn bộ văn bản ở client sẽ phá hủy khả năng full-text search, semantic vector search và điều tra lỗi chất lượng mô hình ở tầng incident review.

---

### Quyết định 4: Cơ chế Data Skipping & Sắp xếp dữ liệu đa chiều cho Tenant
- **Lựa chọn:** **Z-ORDER Clustering theo cặp thuộc tính `(tenant_id, event_time)`**.
- **Lý do lựa chọn:**
  - 95% các truy vấn của khách hàng là: *"Lấy toàn bộ logs/metrics của tenant X trong 3 giờ qua"*.
  - Z-ordering lấp đầy không gian giúp co hẹp dải $[min, max]$ của cả `tenant_id` và thời gian vào các file Parquet kề nhau.
  - Khi query theo tenant, engine đọc metadata và **bỏ qua ngay lập tức $\ge 98\%$ số file dữ liệu**, đạt tốc độ phản hồi $< 1$ giây mà không cần dựng chỉ mục ngoài (external index).
- **Phương án bị loại (Rejected Alternatives):**
  1. *Phân vùng thư mục theo `tenant_id`:* Bị loại vì có tới 10.000 tenants. Việc tạo 10.000 thư mục con mỗi ngày sẽ dẫn đến hiện tượng bùng nổ phân vùng (partition explosion), sinh ra hàng triệu file rác không thể nén.
  2. *Sử dụng Elasticsearch / OpenSearch làm chỉ mục ngoài:* Bị loại vì chi phí cụm RAM/SSD cho 1 tỷ bản ghi/ngày trên Elasticsearch sẽ tiêu tốn ít nhất \$15.000 – \$25.000/tháng, vi phạm ngân sách.

---

### Quyết định 5: Chiến lược Phân tầng Lưu trữ & Vòng đời Bất biến (Storage Tiering & Expiry)
- **Lựa chọn:** **Kết hợp Iceberg Snapshot Expiry + S3 Lifecycle Rules (7-day Hard Purge)**.
- **Lý do lựa chọn:**
  - Tầng Bronze & Silver (phần text payload đầy đủ) áp dụng cấu hình `retention = 7 days`.
  - Hàng ngày, job bảo trì chạy `expire_snapshots` kết hợp `remove_orphan_files` xóa hoàn toàn các Parquet file cũ hơn 7 ngày.
  - Tầng Gold (chỉ số tổng hợp siêu nhỏ) được giữ nguyên vẹn 365 ngày trên S3 Standard để phục vụ báo cáo tài chính năm.
- **Phương án bị loại (Rejected Alternatives):**
  1. *Giữ lịch sử Time Travel vĩnh viễn:* Bị loại vì sau 1 năm, 150 TB $\times$ 12 tháng = 1,8 Petabytes dữ liệu text. Chi phí lưu trữ S3 Standard sẽ là:
     $$1.800 \text{ TB} \times \$23 = \$41.400\text{/tháng} \quad (\text{vượt gấp 8 lần ngân sách CFO}).$$
  2. *Chạy lệnh `DELETE` từng dòng theo ngày:* Bị loại vì `DELETE` trên bảng Parquet sẽ sinh ra write amplification khổng lồ, làm ghi lại hàng triệu file và phá vỡ SLA hệ thống.

---

## 4. Bảng tính toán dung lượng & Chi phí chi tiết (FinOps Math)

### 4.1. Toán học Lưu trữ (Storage Arithmetic)

| Tầng dữ liệu | Khối lượng/ngày | Tỷ lệ nén | Lưu trữ thực tế/ngày | Thời gian lưu giữ (Retention) | Tổng dung lượng active | Chi phí S3/tháng |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **Bronze (Raw Encrypted)** | 5,0 TB | 4,0× (zstd) | 1,25 TB/ngày | 7 ngày | **8,75 TB** | **\$205,50** |
| **Silver (Cleansed + Text)** | 4,2 TB | 4,2× (zstd) | 1,00 TB/ngày | 7 ngày | **7,00 TB** | **\$164,40** |
| **Silver (Telemetry metadata)** | 0,8 TB | 8,0× (Parquet) | 0,10 TB/ngày | 30 ngày (S3-IA) | **3,00 TB** | **\$38,40** |
| **Gold (5-min Aggregates)** | ~15 GB | 10,0× (dict) | 1,50 GB/ngày | 365 ngày (1 năm) | **0,55 TB** | **\$12,90** |
| **Cộng dọn dẹp & PII Audit log**| — | — | — | 30 ngày | **2,00 TB** | **\$47,00** |
| **TỔNG LƯU TRỮ (STORAGE)** | — | — | — | — | **~21,3 TB** | **\$468,20 / tháng** |

*(Đơn giá cơ sở: S3 Standard = \$0,023/GB/tháng; S3 Standard-IA = \$0,0125/GB/tháng).*

---

### 4.2. Toán học Chi phí API Requests (S3 PUT / GET)
- **Ingestion Requests:** Gom 1-min micro-batch $\rightarrow$ $1.440\text{ batches/ngày} \times 4\text{ files} = 5.760\text{ PUTs/ngày}$.
- **Compaction Requests:** Gom thành các file 256 MB $\rightarrow$ $\approx 5.000\text{ PUTs/ngày}$.
- **Dashboard & API Read Requests:** 10.000 tenants query dashboard 5 phút/lần:
  - Nhờ có Z-order skipping trên Gold, mỗi lần đọc chỉ tốn 1 đến 2 GET requests.
  - Tổng số GET requests/tháng $\approx 100.000.000\text{ GETs/tháng}$.
  - Chi phí GET: $100.000 \times \$0,0004 = \mathbf{\$40,00\text{/tháng}}$.
  - Chi phí PUT: $350.000 \times \$0,005 = \mathbf{\$1,75\text{/tháng}}$.

---

### 4.3. Toán học Điện toán (Compute Infrastructure)
- **Streaming Ingest & Masking:** Cụm Kubernetes chạy 4 nodes `c6i.2xlarge` (Spot instances): ~\$480/tháng.
- **Hourly Compaction & Maintenance:** Chạy serverless Spark/DuckDB job định kỳ: ~\$350/tháng.
- **Query Serving Engine (Trino/DuckDB):** 2 nodes `r6i.xlarge` cho dashboard caching: ~\$380/tháng.

### 4.4. Bảng cân đối Ngân sách tổng thể (Total FinOps Summary)

$$\text{Tổng ngân sách yêu cầu} = \text{Storage } (\$468) + \text{Requests } (\$42) + \text{Compute } (\$1.210) = \mathbf{\$1.720\text{ USD/tháng}}$$

$$\text{Tỷ lệ sử dụng ngân sách} = \frac{\$1.720}{\$5.000} = \mathbf{34,4\%} \quad (\text{Dư địa dự phòng an toàn } \mathbf{65,6\%}).$$

---

## 5. Kịch bản Thất bại & Cơ chế Tự phục hồi (3 Failure Modes)

### Kịch bản 1: Bùng nổ file nhỏ do Ingestion bị tắc nghẽn (Compaction Lag Spike)
- **Hiện tượng:** Lưu lượng API tăng đột biến lên 50.000 req/s, job compaction hourly bị crash thiếu bộ nhớ (OOM) $\rightarrow$ số lượng file nhỏ tăng vọt từ vài nghìn lên $> 500.000$ files trong 6 giờ $\rightarrow$ truy vấn dashboard bị chậm từ 1s lên 45s.
- **Cơ chế Phát hiện:** Prometheus alert giám sát số lượng active data files trong bảng Silver thông qua API Iceberg `tbl.inspect.files()`. Nếu số lượng file vượt ngưỡng 50.000 file, trigger P1 incident.
- **Quy trình Phục hồi:** Kích hoạt cụm Compaction dự phòng tự động mở rộng (auto-scale) chạy phân vùng theo ngày độc lập (`rewrite_data_files` với filter `day = current_day`); sử dụng checkpoint Parquet để bảo đảm reader không bị crash.

---

### Kịch bản 2: Rò rỉ PII chưa được khử mã vào tầng Silver (PII Leak Incident)
- **Hiện tượng:** Một phiên bản client mới gửi định dạng JSON lồng nhau chưa đăng ký, khiến bộ lọc Presidio bỏ sót số thẻ tín dụng và ghi thẳng vào Silver.
- **Cơ chế Phát hiện:** Batch audit anomaly chạy mỗi 15 phút quét lấy mẫu 1% dữ liệu Silver bằng Regex phát hiện chuỗi thẻ tín dụng/CVV.
- **Quy trình Phục hồi:**
  1. Sử dụng tính năng **Time Travel / Branching** của Iceberg: chuyển hướng truy vấn của người dùng ngay lập tức về snapshot an toàn trước đó ($T_{-15\text{ min}}$).
  2. Chạy job ghi đè có điều kiện với `MERGE INTO` để mask các bản ghi vi phạm.
  3. Chạy `expire_snapshots` và `VACUUM` ngay lập tức để tiêu hủy vật lý các file Parquet chứa dữ liệu rò rỉ trên S3, bảo đảm tuân thủ điều luật GDPR.

---

### Kịch bản 3: Lệch pha vòng đời giữa Lakehouse và Bộ nhớ đệm Vector (Lifecycle Skew Bug)
- **Hiện tượng:** Một doanh nghiệp yêu cầu xóa tài khoản theo quyền được lãng quên (Right-to-be-forgotten). Lakehouse xóa sạch bản ghi của tenant, nhưng hệ thống Semantic Search Cache (lưu embedding phục vụ gợi ý prompt) vẫn giữ vector cũ, dẫn đến việc LLM tiếp tục trả về dữ liệu của tenant đã xóa.
- **Cơ chế Phát hiện:** Giám sát đối chiếu giữa Delta/Iceberg **Change Data Feed (CDF)** và Vector Index log.
- **Quy trình Phục hồi:** 
  - Hệ thống vector cache bắt buộc không đồng bộ một chiều (one-way upsert) mà phải là **Subscriber của Iceberg/Delta CDF**.
  - Khi có sự kiện `_change_type = 'delete'`, CDF phát ra payload chứa `tenant_id` và `request_id`, tự động kích hoạt lệnh evict trên vector cache để loại bỏ ngay lập tức.

---

## 6. Kế hoạch triển khai MVP trong 1 tuần (One-Week MVP Plan)

```
Thứ 2           Thứ 3           Thứ 4           Thứ 5           Thứ 6
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ Day 1:      │ │ Day 2:      │ │ Day 3:      │ │ Day 4:      │ │ Day 5:      │
│ Ingest Mock │ │ Presidio    │ │ Medallion   │ │ Z-Order &   │ │ Load Test & │
│ 10M samples │ │ Redaction & │ │ Silver/Gold │ │ Retention   │ │ FinOps Audit│
│ into Bronze │ │ Tokenizer   │ │ Pipelines   │ │ Verification│ │ Review      │
└─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘
```

- **Ngày 1 (Thứ Hai) — Hạ tầng lưu trữ & Ingestion Slice:** Tạo bảng Iceberg qua REST Catalog trên MinIO/S3 cục bộ; giả lập streaming nạp 10 triệu requests mẫu theo từng micro-batch 1 phút; đo đạc tốc độ ghi và kích thước file Parquet.
- **Ngày 2 (Thứ Ba) — Cơ chế Khử PII:** Tích hợp Microsoft Presidio; kiểm thử unit test với 1.000 mẫu prompt chứa số thẻ tín dụng, email và API key; bảo đảm tỷ lệ nhận diện và mask đạt $> 99,8\%$.
- **Ngày 3 (Thứ Tư) — Pipeline Medallion & Bảng Gold:** Viết câu lệnh tổng hợp Gold theo cửa sổ 5 phút ($p50/p95$ latency và chi phí token theo tenant); đo đạc thời gian tính toán hoàn tất trong $< 30$ giây.
- **Ngày 4 (Thứ Năm) — Xác minh Cơ chế Khó nhất (Hardest Mechanism):** 
  - Chạy `optimize.z_order(["tenant_id", "event_time"])` trên Silver.
  - Đo đạc tỷ lệ file skipping: chứng minh truy vấn theo tenant bỏ qua $\ge 90\%$ số file và có latency $< 500\text{ ms}$.
  - Chạy thử nghiệm xóa snapshot sau 7 ngày và thu hồi dung lượng thực tế qua `remove_orphan_files`.
- **Ngày 5 (Thứ Sáu) — Kiểm thử Chịu tải & Rà soát Ngân sách (Design Review):** Chạy kiểm thử tải giả lập 25.000 req/s trong 1 giờ; chốt báo cáo FinOps đối chiếu với trần ngân sách \$5.000/tháng của CFO; nghiệm thu tài liệu kiến trúc.

---

## 7. Kết luận & Cam kết Kiến trúc
Bản thiết kế trên giải quyết trọn vẹn bài toán quan sát 1 tỷ requests LLM mỗi ngày bằng cách áp dụng nhất quán các nguyên lý **Lakehouse 2026**:
1. *Kiểm soát chi phí bằng Medallion và Vòng đời 7 ngày:* Giảm 95% chi phí lưu trữ dài hạn.
2. *Bảo vệ hiệu năng bằng Compaction và Z-order:* Giữ tốc độ truy vấn dashboard dưới 1 giây.
3. *Đảm bảo tính pháp lý bằng Phân vùng PII và Change Data Feed:* Ngăn chặn rò rỉ dữ liệu và lỗi đồng bộ vòng đời.
4. *Bảo vệ trần tài chính FinOps:* Toàn bộ hệ thống vận hành ổn định chỉ với **\$1.720/tháng**, nằm an toàn trong trần cho phép \$5.000/tháng.
