# GitOps Progressive Delivery & Observability

Hệ thống triển khai Canary tự động (Auto-Abort) kết hợp giám sát SLO & Alerting qua Email.

## 1. Thành phần kiến trúc
- **GitOps:** Tự động đồng bộ hóa các khai báo K8s thông qua Argo CD.
- **Argo Rollouts (Canary):** Triển khai dạng Canary, tích hợp kiểm tra tự động `AnalysisTemplate`.
- **Prometheus & Grafana:** Thu thập metric từ Flask API và giám sát các chỉ số SLO.
- **Alertmanager:** Gửi email cảnh báo khi SLO bị vi phạm.

---

## 2. Giải thích Metrics & Ngưỡng (Thresholds)

### 2.1. Cấu hình Canary Auto-Abort (`AnalysisTemplate`)
Chúng ta sử dụng metric `flask_http_request_total` do ứng dụng xuất ra để tính **Tỉ lệ thành công (Success Rate)**:

- **PromQL Query:**
  ```promql
  (sum(rate(flask_http_request_total{namespace="demo", status!~"5.*"}[1m])) or vector(1))
  /
  (sum(rate(flask_http_request_total{namespace="demo"}[1m])) or vector(1))
  ```
- **Giải thích:**
  - Tử số: Số lượng request thành công (không có status code dạng `5xx`) trong 1 phút.
  - Mẫu số: Tổng số request trong 1 phút.
  - Sử dụng `or vector(1)` để đảm bảo khi không có lưu lượng truy cập (no traffic), tỉ lệ thành công mặc định là `1` (100%), tránh làm hỏng quá trình cập nhật.
- **Ngưỡng:** Tỉ lệ thành công phải **>= 95%** (`result[0] >= 0.95`). Nếu nhỏ hơn, quá trình Rollout sẽ tự động **Abort** và khôi phục về phiên bản cũ ngay lập tức.

### 2.2. Đo lường SLO & Alerting (`PrometheusRule`)
Cấu hình cảnh báo SLO cho ứng dụng:

- **Điều kiện vi phạm SLO:** Tỉ lệ lỗi >= 5% (tương đương Success Rate < 95%) kéo dài liên tục trên 1 phút.
- **PromQL Query:**
  ```promql
  ((sum(rate(flask_http_request_total{namespace="demo", status="500"}[1m])) or vector(0))
  /
  (sum(rate(flask_http_request_total{namespace="demo"}[1m])) or vector(1))) >= 0.05
  ```
- **Alertmanager Email:** Khi Alert `ApiHighErrorRate` được kích hoạt, nó sẽ được gửi tới Email cấu hình trong `AlertmanagerConfig` qua giao thức SMTP.

---

## 3. Hướng dẫn thử nghiệm & Kiểm tra

### 3.1. Thử nghiệm Canary thành công (Bản tốt)
1. Đảm bảo ứng dụng load generator vẫn đang gửi traffic:
   ```bash
   kubectl -n demo get pods
   # pod/load ở trạng thái Running
   ```
2. Đổi phiên bản sang `v3` trong `k8s-api/api.yaml` (với `ERROR_RATE` giữ nguyên là `"0"`).
3. Commit & Push lên Git.
4. Quá trình Rollout sẽ tăng dần từ 25% -> 50% -> 100% thành công mà không bị ngắt quãng do chất lượng bản cập nhật tốt.

### 3.2. Thử nghiệm Canary Auto-Abort (Bản lỗi)
1. Cố tình cấu hình lỗi bằng cách đổi `ERROR_RATE` thành `"0.5"` (50% lỗi) trong `k8s-api/api.yaml`:
   ```yaml
   env:
   - name: ERROR_RATE
     value: "0.5"
   - name: VERSION
     value: "v4"
   ```
2. Commit & Push lên Git.
3. Khi Argo CD đồng bộ và Rollout phát hành 25% (Canary Pod đầu tiên nhận tải 50% lỗi):
   - Prometheus ghi nhận tỉ lệ lỗi vượt quá 5%.
   - `AnalysisTemplate` chạy định kỳ mỗi 10 giây phát hiện tỉ lệ thành công giảm sâu dưới 95%.
   - Argo Rollouts lập tức chuyển sang trạng thái **Aborted**, thu hồi bản `v4` và quay lại 100% bản `v3` cũ một cách tự động.
