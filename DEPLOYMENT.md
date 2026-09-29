# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Đỗ Thanh Lâm |
| Mã học viên | 2A202602577|
| Repo | https://github.com/lamm45lamm/K4-L3A-DoThanhLam-2A202602577-Cloud-Service-And-Deployment.git |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | http://localhost:8000  |
| Platform | Không deploy được lên Railway |
| Ngày deploy | 2026-09-29 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | 8000, đọc trong container qua `${PORT:-8000}` |
| `AGENT_API_KEY` | ✅ | đặt trong `.env` cục bộ, không nằm trong repo |
| `REDIS_URL` | ✅ | `redis://redis:6379/0` — service `redis` trong docker-compose.yml |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

```
$ docker compose ps
NAME                                           IMAGE                                        STATUS
k4-l3a-cloud-service-and-deployment-agent-1    k4-l3a-cloud-service-and-deployment-agent    Up (healthy)
k4-l3a-cloud-service-and-deployment-redis-1    redis:7-alpine                               Up (healthy)

$ curl -i http://localhost:8000/health
HTTP/1.1 200 OK
{"status":"ok","service":"day12-agent","version":"1.0.0"}

$ curl -i http://localhost:8000/ready
HTTP/1.1 200 OK
{"status":"ready","redis":true}

$ curl -i -X POST http://localhost:8000/ask (không API key)
HTTP/1.1 401 Unauthorized
{"detail":"invalid or missing API key"}
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

---

## Nếu Dùng Phương Án Dự Phòng

Không đăng ký được tài khoản cloud? Vẫn nộp được bài, nhưng CP5 tối đa 60% điểm:

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`
4. Chạy `pytest tests/test_cp5.py -v` — bộ test sẽ tự chuyển sang kiểm tra
   `http://localhost:8000`
5. Ghi rõ lý do không deploy được vào phần dưới đây:

```
Không dùng phương án dự phòng cloud lần này — tài khoản trên
Railway bị tính phí.
```
