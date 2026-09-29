# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Đàm Việt Hưng |
| Mã học viên | 2A202602600 |
| Repo | https://github.com/viethwngg/K4-L3B-DAY12-DamVietHung-2A202602600-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-gndf.onrender.com |
| Platform | Render Blueprint — Docker web service + managed Valkey/Redis |
| Ngày deploy | 2026-09-29 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Render tự cấp cho web service |
| `AGENT_API_KEY` | ✅ | đặt trong Render dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | lấy từ managed `day12-redis` (Valkey) của Render Blueprint |
| `RATE_LIMIT_PER_MINUTE` | ✅ | khai báo trong `render.yaml`, giá trị 10 |
| `MONTHLY_BUDGET_USD` | ✅ | khai báo trong `render.yaml`, giá trị 10.0 |
| `LOG_LEVEL` | ✅ | khai báo trong `render.yaml`, giá trị INFO |

## Lệnh Kiểm Tra

Các lệnh dưới đây dùng URL Render công khai đã xác minh:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-agent-gndf.onrender.com/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-agent-gndf.onrender.com/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://day12-agent-gndf.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-agent-gndf.onrender.com/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-agent-gndf.onrender.com/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```text
GET  https://day12-agent-gndf.onrender.com/health -> 200
{"status":"ok","service":"day12-agent","version":"1.0.0"}

GET  https://day12-agent-gndf.onrender.com/ready -> 200
{"status":"ready","redis":true}

POST https://day12-agent-gndf.onrender.com/ask không có API key -> 401
{"detail":"invalid or missing API key"}

Render dashboard:
  day12-agent -> Deployed (Docker, Oregon)
  day12-redis -> Available (Valkey 8, Oregon)
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — Render Blueprint với `day12-agent` Deployed và `day12-redis` Available
- `screenshots/health.png` — Chrome chụp trực tiếp response thật của `/health`

