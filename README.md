# GitOps Progressive Delivery & Observability

Hệ thống triển khai Canary tự động (Auto-Abort & Rollback) kết hợp giám sát SLO & Alerting qua Email bằng GitOps.

## 1. Thành phần kiến trúc
- **GitOps:** Tự động đồng bộ hóa các khai báo Kubernetes thông qua Argo CD.
- **Argo Rollouts (Canary):** Triển khai dạng Canary, tích hợp kiểm tra tự động bằng `AnalysisTemplate`.
- **Prometheus & Grafana:** Thu thập metric từ Flask API và giám sát các chỉ số SLO.
- **Alertmanager:** Gửi email cảnh báo trực tiếp về hòm thư khi SLO bị vi phạm.

---

## 2. Giải thích Metrics & Ngưỡng (Thresholds)

### 2.1. Cấu hình Canary Auto-Abort (`AnalysisTemplate`)
Chúng ta sử dụng metric `flask_http_request_total` do ứng dụng xuất ra để tính **Tỷ lệ thành công (Success Rate)**:

- **PromQL Query:**
  ```promql
  (sum(rate(flask_http_request_total{namespace="demo", status!~"5.*"}[1m])) or vector(1))
  /
  (sum(rate(flask_http_request_total{namespace="demo"}[1m])) or vector(1))
  ```
- **Giải thích:**
  - **Tử số:** Số lượng request thành công (không có status code dạng `5xx`) trong 1 phút.
  - **Mẫu số:** Tổng số request trong 1 phút.
  - Sử dụng `or vector(1)` để đảm bảo khi không có lưu lượng truy cập (no traffic), tỷ lệ thành công mặc định là `1` (100%), tránh làm hỏng quá trình cập nhật.
- **Ngưỡng (Threshold):** Tỷ lệ thành công phải **>= 95%** (`result[0] >= 0.95`). Nếu nhỏ hơn, quá trình Rollout sẽ tự động **Abort** và khôi phục về phiên bản cũ ngay lập tức.

### 2.2. Đo lường SLO & Alerting (`PrometheusRule`)
Cấu hình cảnh báo SLO cho ứng dụng:

- **Điều kiện vi phạm SLO:** Tỷ lệ lỗi >= 5% (tương đương Success Rate < 95%) kéo dài liên tục trên 1 phút.
- **PromQL Query:**
  ```promql
  (sum by (namespace) (rate(flask_http_request_total{namespace="demo", status="500"}[1m]))
  /
  sum by (namespace) (rate(flask_http_request_total{namespace="demo"}[1m]))) >= 0.05
  ```
- **Giải thích:**
  - **Tử số:** Tỷ lệ lỗi 500 phát sinh trong 1 phút.
  - **Mẫu số:** Tổng lượng request phát sinh trong 1 phút.
  - Sử dụng `sum by (namespace)` để giữ lại nhãn `namespace="demo"`, hỗ trợ định tuyến chính xác đến đúng cấu hình nhận alert của namespace đó.
- **Alertmanager Email:** Khi Alert `ApiHighErrorRate` được kích hoạt (`firing`), Alertmanager sẽ tự động định tuyến và gửi email cảnh báo về hòm thư của bạn qua kết nối SMTP Gmail.

---

## 3. Hướng dẫn thử nghiệm & Kiểm tra

### 3.1. Thử nghiệm Canary thành công (Bản tốt)
1. Đổi phiên bản trong `k8s-api/api.yaml` sang phiên bản mới ổn định (ví dụ `VERSION` sang `"v9"`, `ERROR_RATE` là `"0"`).
2. Commit & Push lên Git.
3. Quá trình Rollout sẽ tăng dần từ 25% -> 50% -> 100% thành công mà không bị ngắt quãng do chất lượng bản cập nhật tốt.

### 3.2. Thử nghiệm Canary Auto-Abort (Bản lỗi)
1. Cố tình cấu hình lỗi bằng cách đổi `ERROR_RATE` thành `"0.5"` (50% lỗi) trong `k8s-api/api.yaml`:
   ```yaml
   env:
   - name: ERROR_RATE
     value: "0.5"
   - name: VERSION
     value: "v8"
   ```
2. Commit & Push lên Git.
3. Khi Argo CD đồng bộ và Rollout phát hành 25% (Canary Pod đầu tiên nhận tải 50% lỗi):
   - Prometheus ghi nhận tỷ lệ lỗi vượt quá 5%.
   - `AnalysisTemplate` chạy định kỳ mỗi 10 giây phát hiện tỷ lệ thành công giảm sâu dưới 95%.
   - Argo Rollouts lập tức chuyển sang trạng thái **Aborted**, thu hồi bản lỗi và quay lại 100% bản cũ ổn định tự động.

---

## 4. Minh chứng hoạt động (Proof of Work)

*Dưới đây là các hình ảnh/video minh chứng các tính năng tự động hoạt động trên hệ thống:*

### 4.1. Minh chứng Canary Auto-Abort & Rollback
> [!TIP]
> Hãy thay thế hình ảnh/ảnh động dưới đây bằng file chụp màn hình trạng thái rollout bị Abort của bạn.

![Canary Auto-Abort & Rollback](./canary-abort-proof.png)

*(Mô tả: Hình ảnh trạng thái Rollout bị Aborted do AnalysisRun thất bại, tự động đưa replicas của phiên bản lỗi về 0 và đưa phiên bản cũ ổn định về lại đầy đủ replicas).*

### 4.2. Minh chứng Alert firing & Gửi Email thành công
> [!TIP]
> Hãy thay thế hình ảnh dưới đây bằng ảnh chụp màn hình email cảnh báo nhận được trong hộp thư của bạn.

![Alert Email Received](./alert-email-received.png)

*(Mô tả: Hình ảnh email cảnh báo từ Alertmanager thông báo vi phạm SLO Tỷ lệ lỗi > 5%).*
