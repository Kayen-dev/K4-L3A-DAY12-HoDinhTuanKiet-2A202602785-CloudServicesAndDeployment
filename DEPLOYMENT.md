# Thông Tin Deploy — Checkpoint 5

> Chỉ ghi tên biến môi trường. Không ghi giá trị API key hoặc secret vào repository.

## Thông Tin Học Viên

| Mục | Nội dung |
|---|---|
| Họ và tên | Hồ Đình Tuấn Kiệt |
| Mã học viên | 2A202602785 |
| Repo | https://github.com/Kayen-dev/K4-L3A-DAY12-HoDinhTuanKiet-2A202602785-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|---|---|
| Public URL | https://day12-agent-production-f111.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-28 |

Bản đang chạy trên Railway hiện trả `404 {"detail":"Not Found"}` tại URL gốc.
Mã nguồn local đã bổ sung service console tại `/`; cần push/redeploy để giao
diện này xuất hiện trên public URL. Healthcheck vẫn sử dụng `/health`.

## Biến Môi Trường Trên Cloud

| Biến | Trạng thái | Nguồn / ghi chú |
|---|---|---|
| `PORT` | ✅ | Railway tự gán; không hardcode trong dashboard |
| `AGENT_API_KEY` | ✅ | Secret đặt trong Railway Variables, không nằm trong repo |
| `REDIS_URL` | ⚠️ Cần kiểm tra | Phải là private connection URL của Redis service trên Railway; `/ready` hiện trả 500 |
| `RATE_LIMIT_PER_MINUTE` | ✅ | Cấu hình giới hạn request, không phải secret |
| `MONTHLY_BUDGET_USD` | ✅ | Cấu hình ngân sách, không phải secret |
| `LOG_LEVEL` | ✅ | Mức log của service, không phải secret |

## Lệnh Kiểm Tra

```bash
URL="https://day12-agent-production-f111.up.railway.app"

# Liveness
curl -i "$URL/health"

# Readiness
curl -i "$URL/ready"

# Không có API key
curl -i -X POST "$URL/ask" \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# Có API key; AGENT_API_KEY chỉ tồn tại trong shell/dashboard
curl -i -X POST "$URL/ask" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'
```

## Kết Quả Chạy Thật

Kiểm tra ngày 2026-09-28:

```text
GET  /          -> 404 {"detail":"Not Found"}             (revision cloud hiện tại)
GET  /health    -> 200 {"status":"ok","service":"day12-agent","version":"1.0.0"}
GET  /ready     -> 500 Internal Server Error               (cần sửa REDIS_URL)
POST /ask       -> 401 {"detail":"invalid or missing API key"} khi không có key
POST /ask + key -> chưa xác minh; DEPLOY_API_KEY local chưa được cấu hình
```

Trạng thái hiện tại: liveness và authentication đã hoạt động; readiness chưa đạt do kết nối Redis trên cloud.

## Kết Quả Chấm Thử

- CP1–CP4: **70/70 test pass**.
- CP5: **7/8 test bắt buộc pass**, còn fail `/ready` do Redis cloud.
- Tổng phần bắt buộc từ `grade.py --no-bonus`: **98.1/100**.
- Kiểm tra `/ask` có key đang bị skip vì `DEPLOY_API_KEY` trong `.env` local
  chưa có giá trị.
- Bonus CI/CD: chưa thực hiện; đây là phần không bắt buộc.

## Ảnh Minh Chứng

- [`screenshots/health.png`](screenshots/health.png) — kết quả thật của `GET /health` trên Railway.
- [`screenshots/ui-local.png`](screenshots/ui-local.png) — service console kiểm tra ở viewport desktop.
- [`screenshots/ui-mobile-local.png`](screenshots/ui-mobile-local.png) — service console kiểm tra responsive ở viewport 390 px.
- `screenshots/dashboard.png` — cần chụp từ Railway dashboard sau khi sửa `REDIS_URL`; file này chưa có trong repository.
