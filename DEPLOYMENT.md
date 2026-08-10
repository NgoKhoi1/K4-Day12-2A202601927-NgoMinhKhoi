# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị token vào đây.**
> Repo này công khai — dán token vào là mất token.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Ngô Minh Khôi |
| Mã học viên | 2A202601927 |
| Repo | https://github.com/NgoKhoi1/K4-Day12-2A202601927-NgoMinhKhoi |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://k4-day12-2a202601927-ngominhkhoi.onrender.com |
| Platform | Render |
| Ngày deploy | 10/08/2026 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `API_TOKEN` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Render Redis add-on (`day12-chat-redis`), gắn tự động qua `fromService` trong `render.yaml` |
| `BUCKET_CAPACITY` | ✅ | 10 |
| `REFILL_PER_MINUTE` | ✅ | 10 |
| `DAILY_BUDGET_USD` | ✅ | 1.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

```bash
URL=https://k4-day12-2a202601927-ngominhkhoi.onrender.com
export $(grep -E '^DEPLOY_API_TOKEN=' .env | xargs)   # token của chính service đã deploy

# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i "$URL/healthz"

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i "$URL/readyz"

# 3. Không có token — mong đợi 401 kèm header WWW-Authenticate
curl -i -X POST "$URL/chat" \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'

# 4. Có token — mong đợi 200 kèm câu trả lời
curl -i -X POST "$URL/chat" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $DEPLOY_API_TOKEN" \
  -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy la gi?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST "$URL/chat" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $DEPLOY_API_TOKEN" \
    -H "X-Client-Id: sv-test" \
    -d '{"message":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Chạy lúc 2026-08-10 10:50 UTC:

```
=== 1. healthz ===
HTTP/1.1 200 OK
{"status":"ok","service":"day12-chat-service","version":"1.0.0"}

=== 2. readyz ===
HTTP/1.1 503 Service Unavailable
{"status":"not ready","redis":false}

=== 3. chat khong token ===
HTTP/1.1 401 Unauthorized
www-authenticate: Bearer
{"detail":"invalid or missing bearer token"}

=== 4. chat co token ===
HTTP/1.1 500 Internal Server Error
Internal Server Error

=== 5. rate limit x15 ===
500 500 500 500 500 500 500 500 500 500 500 500 500 500 500
```

**Vấn đề đang biết:** `/readyz` và `/chat` (khi có token) đang lỗi vì service
`day12-chat` chưa kết nối được tới Redis trên Render — log server báo
`redis.exceptions.ConnectionError: Error 111 connecting to localhost:6379.
Connection refused.`, tức là biến `REDIS_URL` thật (gắn tự động qua
`fromService` trong `render.yaml`) chưa được process đang chạy đọc đúng, dù
service Redis `day12-chat-redis` báo trạng thái bình thường. Đã thử: xoá biến
`REDIS_URL` bị set tay đè lên bằng `redis://localhost:6379/0`, và chạy Manual
Sync từ Blueprint — vẫn chưa hết lỗi. Các phần khác (`/healthz`, xác thực
Bearer 401) hoạt động đúng. Sẽ tiếp tục xử lý sau khi nộp.

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/healthz.png` — kết quả gọi `/healthz` từ trình duyệt hoặc curl

---

## Nếu Dùng Phương Án Dự Phòng

Không đăng ký được tài khoản cloud? Vẫn nộp được bài, nhưng CP5 tối đa 60% điểm:

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`
4. Chạy `pytest tests/test_cp5.py -v` — bộ test sẽ tự chuyển sang kiểm tra
   `http://localhost:8000`
5. Ghi rõ lý do không deploy được vào phần dưới đây:

Không áp dụng — đã deploy thật lên Render (xem mục Service ở trên), không
dùng phương án dự phòng LOCAL_FALLBACK.
