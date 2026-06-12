# W9 Challenge: GitOps, SLO Alerting và Automated Canary Abort

Repository: <https://github.com/Phuc776/gitops>

Lab này demo một quy trình triển khai theo GitOps sử dụng Argo CD, Argo
Rollouts, Prometheus và Alertmanager. Git là **source of truth** duy nhất cho
desired state của application.

## Checklist Nộp Bài

| Yêu cầu | Cách triển khai | Bằng chứng |
| --- | --- | --- |
| Thay đổi qua Git, Argo CD `Synced` và no drift, reproduce được từ Git | App-of-apps trong `argocd/`; bật automated sync, prune và self-heal | [Git là source of truth](#1-git-la-source-of-truth) |
| `git revert` rollback dưới 5 phút | Git history có revert commit; Argo CD tự động apply revision sau khi push | [Git revert rollback dưới 5 phút](#2-git-revert-rollback-duoi-5-phut) |
| Có 1 SLO và 1 alert firing về email cá nhân khi inject lỗi | `PrometheusRule` kết hợp với namespaced `AlertmanagerConfig` | [SLO và email alert](#3-slo-va-email-alert) |
| Canary bản lỗi tự abort về bản cũ | `Rollout` chạy `AnalysisTemplate` tại 25% và 50%; metric failure làm rollout abort | [Automatic canary abort](#4-automatic-canary-abort) |

## Kiến Trúc

```text
Git push
  -> Argo CD automated sync
  -> Argo Rollouts canary
  -> Flask /metrics
  -> Prometheus
       -> AnalysisTemplate: promote hoặc abort
       -> PrometheusRule: ApiHighErrorRate
       -> AlertmanagerConfig: gửi email cá nhân
```

Các manifest chính:

| File | Vai trò |
| --- | --- |
| `argocd/root.yaml` | Root Application theo mô hình app-of-apps |
| `argocd/apps/api.yaml` | Theo dõi thư mục `k8s-api/` từ Git |
| `k8s-api/api.yaml` | Canary Rollout và Service |
| `k8s-api/analysis-template.yaml` | Automated quality gate cho rollout |
| `k8s-api/servicemonitor.yaml` | Scrape Flask metrics |
| `k8s-api/prometheus-rule.yaml` | Định nghĩa SLO alert |
| `k8s-api/alertmanager-config.yaml` | Route alert tới email cá nhân |
| `app/app.py` | API hỗ trợ fault injection |

SMTP Secret được chủ động loại khỏi Git. Đây là bootstrap secret, không phải
application desired state. App password cần được rotate sau khi demo.

## Reproduce Từ Git

Điều kiện cần:

- Kubernetes cluster đã cài Argo CD.
- Docker image `w9-api:1` được build từ `app/Dockerfile` và có sẵn trên các
  cluster node.
- Secret `gmail-smtp-secret` được tạo trong namespace `demo`.
- Có continuous traffic tối thiểu 1 request/second tới endpoint `/` của API.

Chỉ cần bootstrap Root Application:

```bash
kubectl apply -f argocd/root.yaml
```

Sau bước bootstrap, mọi thay đổi workload và monitoring phải được commit và
push lên Git. Argo CD sẽ tự động apply desired state.

Kiểm tra Git revision và trạng thái no drift:

```bash
git rev-parse HEAD
kubectl get application api -n argocd \
  -o jsonpath='{.status.sync.status}{" | "}{.status.sync.revision}{" | "}{.status.operationState.phase}{"\n"}'
```

Kết quả mong đợi:

```text
Synced | <cùng Git SHA> | Succeeded
```

## SLO Và PromQL Query

SLO của API:

```text
Ít nhất 95% API request thành công.
5xx error rate phải thấp hơn 5%.
```

Rollout analysis và alert sử dụng cùng một PromQL ratio:

```promql
(
  sum(rate(flask_http_request_total{namespace="demo", status=~"5.."}[1m]))
  or vector(0)
)
/
clamp_min(
  sum(rate(flask_http_request_total{namespace="demo"}[1m])),
  1
)
```

Giải thích query:

- Numerator: số lượng 5xx request mỗi giây trong một phút gần nhất.
- Denominator: tổng request mỗi giây trong một phút gần nhất.
- `or vector(0)`: trả về `0` khi không có 5xx sample.
- `clamp_min(..., 1)`: tránh division by zero. Vì vậy demo tạo tối thiểu
  1 request/second để ratio có ý nghĩa.

Các threshold:

| Thành phần | Điều kiện | Hành vi |
| --- | --- | --- |
| `AnalysisTemplate` | `result[0] < 0.05` | Measurement từ 5% trở lên bị đánh giá Failed; hai measurement Failed làm rollout abort |
| `PrometheusRule` | Ratio `> 0.05` liên tục 1 phút | Alert `ApiHighErrorRate` firing và được route tới email |

Alert annotation có thêm measured value nên email tiếp theo sẽ hiển thị
`measured_value`. Nếu cần hiển thị thời điểm alert bắt đầu firing, cần custom
Alertmanager notification template và sử dụng trường `.StartsAt`.

## Bad Release Được Demo

Git revision được dùng để demo triển khai:

```yaml
- { name: ERROR_RATE, value: "0.25" }
- { name: VERSION, value: "v10" }
```

Với tổng cộng bốn replicas:

| Canary weight | Service-wide error rate dự kiến |
| --- | --- |
| 25%: một bad pod | Khoảng 6.25%, có thể dao động quanh quality gate 5% |
| 50%: hai bad pod | Khoảng 12.5%, đủ ổn định để fail quality gate 5% |

Kết quả thực tế trong lần chạy `v10`:

```text
25% analysis: Successful
50% analysis: Failed
Measurements: 13.8462%, sau đó 10.9375%
Kết quả: rollout abort và bad ReplicaSet bị scale down
```

## Bằng Chứng

### 1. Git Là Source Of Truth

Argo CD hiển thị API Application là `Synced` với Git branch `main`, trong khi
runtime health là `Degraded` vì bad canary đã bị abort.

Đây là hành vi đúng:

- `Synced` chứng minh cluster desired state khớp với Git và không có drift.
- `Degraded` chứng minh Argo Rollouts đã từ chối bad release do quality gate
  thất bại.

![Argo CD Synced với Git trong khi rollout Degraded](<image/Screenshot 2026-06-12 090648.png>)

App-of-apps view hiển thị Root Application, API, web, Argo Rollouts và
monitoring stack:

![Danh sách Argo CD Applications](<image/Screenshot 2026-06-12 084029.png>)

Lưu ý: ảnh overview cũ ghi nhận `argo-rollouts` đang OutOfSync. Khi nộp bài nên
chụp thêm một ảnh overview mới với toàn bộ Application đều `Synced` để chứng
minh no drift rõ ràng hơn.

### 2. Git Revert Rollback Dưới 5 Phút

Rollback cũng bắt buộc đi qua Git:

```bash
start=$(date +%s)
git revert <bad-release-commit>
git push origin main

# Chờ Argo CD hiển thị revert SHA mới ở trạng thái Synced/Succeeded.
kubectl get application api -n argocd -w
end=$(date +%s)
echo "rollback_seconds=$((end-start))"
```

Git history hiện có các revert commit chứng minh đã sử dụng `git revert`:

```text
4888b8c Revert "v6 good"
edf7845 Revert "2->4"
```

Bằng chứng còn cần bổ sung khi nộp: quay một clip liên tục hoặc chụp một cặp
ảnh thể hiện thời điểm push revert commit, Argo CD sync đúng revert SHA và tổng
thời gian nhỏ hơn 300 giây.

Các ảnh hiện tại chưa chứng minh được thời gian rollback dưới 5 phút, vì vậy
không nên tuyên bố tiêu chí này đã hoàn tất nếu chưa có capture bổ sung.

### 3. SLO Và Email Alert

Prometheus chuyển `ApiHighErrorRate` sang trạng thái `FIRING` khi measured error
rate vượt quá configured threshold:

![Prometheus ApiHighErrorRate firing](<image/Screenshot 2026-06-12 102233.png>)

Ảnh Prometheus này được chụp trong một lần chạy cũ với threshold 10%.
Repository hiện tại đã sử dụng threshold 5% thống nhất cho cả analysis và
alerting.

Alertmanager đã route thành công alert trong namespace `demo` tới Gmail cá
nhân:

![Email cá nhân nhận được từ Alertmanager](<image/Screenshot 2026-06-12 133417.png>)

Email chứng minh:

- Alert name là `ApiHighErrorRate`.
- Application là `api`.
- Namespace là `demo`.
- Severity là `warning`.
- Alert đã được gửi thành công tới email cá nhân.

### 4. Automatic Canary Abort

Failed AnalysisRun hiển thị cả hai event `MetricFailed` và
`AnalysisRunFailed`:

![AnalysisRun Failed vì Prometheus metric](<image/Screenshot 2026-06-12 090630.png>)

Argo CD vẫn `Synced`, Rollout chuyển sang `Degraded`, bad canary pods bị
terminate và stable ReplicaSet tiếp tục phục vụ request:

![Bad canary pods bị terminate và stable pods tiếp tục chạy](<image/Screenshot 2026-06-12 090655.png>)

Trạng thái cuối sau automatic abort: bad ReplicaSet không còn phục vụ traffic,
trong khi ReplicaSet cũ giữ đủ bốn running pods.

![Stable ReplicaSet được giữ lại sau automatic abort](<image/Screenshot 2026-06-12 090702.png>)

Các ảnh bổ sung thể hiện quá trình progressive rollout:

![Canary rollout bắt đầu](<image/Screenshot 2026-06-12 090527.png>)

![Canary được scale up theo từng bước](<image/Screenshot 2026-06-12 090555.png>)

![Argo CD vẫn Synced trong quá trình rollout](<image/Screenshot 2026-06-12 090605.png>)

## Demo Runbook

1. Đảm bảo continuous traffic đang chạy.
2. Commit và push một good release với `ERROR_RATE=0`.
3. Xác nhận Argo CD hiển thị đúng Git SHA ở trạng thái `Synced` và
   `Succeeded`.
4. Commit và push version mới với `ERROR_RATE=0.25`.
5. Theo dõi Rollout và AnalysisRun:

```bash
kubectl get rollout,analysisrun,pods -n demo -w
```

6. Xác nhận canary qua bước 25%, fail tại 50%, tự abort và quay về stable
   ReplicaSet.
7. Xác nhận `ApiHighErrorRate` firing và email cá nhân nhận được alert.
8. Chạy `git revert <bad-release-commit>`, push lên Git và quay lại bằng chứng
   Argo CD sync revert SHA trong thời gian dưới 5 phút.

## Kết Quả

- Git là source of truth; Argo CD automated sync, prune và self-heal đã được
  bật.
- SLO và alert threshold đều là 5%.
- Fault injection làm SLO alert firing và gửi thành công tới email cá nhân.
- Bad canary tự động abort trước khi đạt 100%.
- Bằng chứng còn cần capture thêm là timed `git revert` rollback dưới 5 phút và
  ảnh overview cuối cùng với toàn bộ Argo CD Application đều `Synced`.
