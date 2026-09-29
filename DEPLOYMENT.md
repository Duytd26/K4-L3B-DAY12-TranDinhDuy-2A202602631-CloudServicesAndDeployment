# Thông Tin Deploy — Checkpoint 5

> Mục tiêu: deploy bằng Render Blueprint, dùng Redis managed của Render. Không
> đưa API key hay Deploy Hook URL vào repository.
>
> Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Trần Đình Duy |
| Mã học viên | 2A202602631 |
| Repo | K4-L3B-DAY12-TranDinhDuy-2A202602631-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-trandinhduy.onrender.com |
| Platform | Render Blueprint + Render Key Value (Redis compatible) |
| Ngày deploy | 2026-09-29 |

## Biến Môi Trường Cần Set Trên Render

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | Render tự gán | Không tự khai báo giá trị |
| `AGENT_API_KEY` | `generateValue` trong `render.yaml` | Secret do Render tự tạo, không nằm trong repo |
| `REDIS_URL` | `fromService` trong `render.yaml` | Lấy internal connection string của `day12-redis-sg` cùng region Singapore |
| `RATE_LIMIT_PER_MINUTE` | `render.yaml` | 10 |
| `MONTHLY_BUDGET_USD` | `render.yaml` | 10.0 |
| `LOG_LEVEL` | `render.yaml` | INFO |

## Lệnh Kiểm Tra

Các lệnh kiểm tra chạy với Public URL HTTPS ở bảng Service:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-agent-trandinhduy.onrender.com/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-agent-trandinhduy.onrender.com/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://day12-agent-trandinhduy.onrender.com/ask ^
  -H "Content-Type: application/json" ^
  -d "{\"question\":\"Hello\"}"

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-agent-trandinhduy.onrender.com/ask ^
  -H "Content-Type: application/json" ^
  -H "X-API-Key: %AGENT_API_KEY%" ^
  -H "X-User-Id: sv-test" ^
  -d "{\"question\":\"Deploy là gì?\"}"

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for /l %i in (1,1,15) do @curl -s -o NUL -w "%{http_code} " -X POST https://day12-agent-trandinhduy.onrender.com/ask ^
  -H "Content-Type: application/json" ^
  -H "X-API-Key: %AGENT_API_KEY%" ^
  -H "X-User-Id: sv-test" ^
  -d "{\"question\":\"test\"}"
```

## Kết Quả Chạy Thật

Kết quả kiểm tra ngày 2026-09-29:

```text
GET /health
HTTP/1.1 200 OK
{"status":"ok","service":"day12-agent","version":"1.0.0"}

GET /ready
HTTP/1.1 500 Internal Server Error
Internal Server Error

Ghi chú: /ready hiện chưa đạt vì service Render chưa kết nối được Redis/Key Value.
Đã sửa render.yaml để dùng type: keyvalue và REDIS_URL fromService connectionString.
Cần sync/deploy lại Blueprint trên Render rồi chạy lại pytest tests/test_cp5.py -v.
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

---

## Các bước triển khai Render

1. Push commit này lên nhánh `main` của repository public.
2. Trên Render, chọn **New +** → **Blueprint**, kết nối repository này và chọn
   `render.yaml`. Render sẽ tạo web service cùng Key Value store.
3. Blueprint tự tạo `AGENT_API_KEY` bằng `generateValue`. Không dùng token
   GitHub/Render và không commit khóa này.
4. Chờ deploy lần đầu hoàn tất, sao chép Public URL HTTPS vào bảng Service,
   chạy các lệnh kiểm tra ở trên và chụp lại dashboard/health.
5. Sau mỗi lần đổi `render.yaml`, vào Blueprint trên Render và chọn **Sync
   Blueprint**. Việc này tạo Key Value `day12-redis-sg` tại Singapore và cập nhật
   `REDIS_URL` sang web service;
   chỉ bấm Manual Deploy sẽ không cập nhật biến tham chiếu.
   Key Value và web service phải cùng region **Singapore** thì internal URL mới
   kết nối được. Instance `day12-redis` cũ ở region khác (nếu có) không còn được
   dùng và có thể xoá sau khi CP5 đã pass.
6. Render dùng `autoDeployTrigger: checksPass`, vì vậy các commit lên `main`
   chỉ được deploy sau khi GitHub Actions CI xanh. Job Deploy trong workflow vẫn
   hỗ trợ Deploy Hook tùy chọn qua GitHub Secret `RENDER_DEPLOY_HOOK_URL`.
7. Chạy `pytest tests/test_cp5.py -v` với `LOCAL_FALLBACK=false` để xác nhận
   endpoint công khai trước khi nộp.

## Nếu Dùng Phương Án Dự Phòng

Không đăng ký được tài khoản cloud? Vẫn nộp được bài, nhưng CP5 tối đa 60% điểm:

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`
4. Chạy `pytest tests/test_cp5.py -v` — bộ test sẽ tự chuyển sang kiểm tra
   `http://localhost:8000`
5. Ghi rõ lý do không deploy được vào phần dưới đây:

```
Không dùng cloud ở môi trường hiện tại vì đang dùng phương án triển khai cục bộ
qua Docker Compose để hoàn tất bài lab và xác minh chức năng trước khi nộp.
```
