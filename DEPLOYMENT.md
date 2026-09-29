# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Đinh Tuấn Long |
| Mã học viên | 2A202602620 |
| Repo | https://github.com/brxcewxyne/K4-L3B-DAY12-DinhTuanLong-2A202602620-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-production-62c1.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-29 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong Railway Variables, không nằm trong repo |
| `REDIS_URL` | ✅ | tham chiếu từ Railway service `day12-redis` |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Public URL đã điền sẵn trong các lệnh dưới đây:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-agent-production-62c1.up.railway.app/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-agent-production-62c1.up.railway.app/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://day12-agent-production-62c1.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-agent-production-62c1.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-agent-production-62c1.up.railway.app/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Kết quả do học viên trực tiếp kiểm chứng trên Railway
(Public URL: https://day12-agent-production-62c1.up.railway.app):

```
1. Railway Deployment
   Platform: Railway — agent chạy trên public HTTPS, Redis đã kết nối.

2. Health & Readiness
   GET /health → 200 {"status":"ok","service":"day12-agent","version":"1.0.0"}
   GET /ready → 200 {"status":"ready","redis":true}

3. Authentication
   POST /ask không API key → 401 {"detail":"invalid or missing API key"}
   POST /ask có API key hợp lệ → 200, gồm answer, user_id,
     history_length, cost_usd và tokens

4. Rate Limiting (10 requests/phút, kiểm chứng live)
   Requests 1–10 → 200; requests 11–12 → 429

5. Redis Conversation History (user long-cloud-test)
   Request đầu → history_length=0
   Request thứ hai cùng user → history_length=2 (shared history)

6. Cost Guard
   pytest tests/test_cp3.py -k "cost or budget" → 7 passed
     (gồm unit test HTTP 402)
   Live HTTP 402 trên Railway: chưa kiểm chứng

7. CP5 Automated Tests (với DEPLOY_API_KEY hợp lệ)
   tests/test_cp5.py → 9 passed, 4 skipped
   (4 skip thuộc Local Fallback — không áp dụng vì đã deploy Railway)

8. Full Regression
   Full pytest → 79 passed, 4 skipped, 1 failed, 12 errors;
   failure và errors đều thuộc tests/test_bonus_cicd.py
   (Bonus CI/CD chưa triển khai) — không tuyên bố suite PASS
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

> Chưa có ảnh — cần bổ sung trước khi nộp bài.

---

## Nếu Dùng Phương Án Dự Phòng

Không áp dụng — đã deploy Railway thành công.
