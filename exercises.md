# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay mỗi dòng placeholder bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đinh Tuấn Long  Mã học viên: 2A202602620

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tôi đã để `agent_api_key` là trường bắt buộc không default (`app/config.py:L44`), nên nếu quên set `AGENT_API_KEY` trên Railway, app ném `ValidationError` ngay lúc startup và bản deploy báo đỏ khi tôi còn đang nhìn màn hình — sửa bằng một biến môi trường là xong. Nếu để default `"changeme"`, app vẫn khởi động "xanh giả", endpoint `/ask` mở cho cả Internet dùng chùa, và tôi chỉ phát hiện khi hóa đơn LLM về. Với tôi, fail fast biến lỗi cấu hình âm thầm thành lỗi deploy ồn ào. Trade-off duy nhất: chạy local bắt buộc phải có `.env`, nhưng đó đúng là cái giá của an toàn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Một dòng log thật tôi thu được khi gọi `/ask` (từ `log_event` ở `app/logging_utils.py:L37-L45`):

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T05:33:20.703004+00:00", "user_id": "sv01", "tokens_in": 8, "tokens_out": 40, "cost_usd": 2.52e-05}
```

Hai việc tôi làm được với nó mà `print("đã trả lời xong")` không làm được: (1) cộng tổng `cost_usd` theo từng `user_id` để biết ai tiêu nhiều tiền nhất — text tự do không parse được field; (2) đếm tỷ lệ `level=error` theo phút để gắn cảnh báo tự động trên cloud. Vì mỗi event là một dòng JSON độc lập trên stdout nên Docker/Railway gom đúng một bản ghi một dòng, không vỡ log.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | chưa đo riêng (base `python:3.11` full thuộc lớp ~1GB) |
| Multi-stage | 271 MB (đo bằng `docker images`) |

Giải thích: tôi chỉ đo bản multi-stage đang dùng (271MB cho cả hai tag `day12-agent:cp2-test` và image compose); bản 1 stage tôi không build riêng nên không bịa số. Phần chênh lệch nằm ở: base full mang theo compiler và gói hệ thống mà bản slim (`Dockerfile:L2`, `L9`) không có; stage `builder` cài xong chỉ copy kết quả sang (`L17`), vứt lại pip cache và tool build; cộng thêm `--no-cache-dir` ở `L7`. Nói ngắn gọn: multi-stage chỉ đóng gói "món chín", còn single-stage bê cả "căn bếp" theo.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Trong Dockerfile của tôi, `COPY requirements.txt` (`L6`) đứng trước `RUN pip install` (`L7`), còn code (`COPY app`, `COPY utils` ở `L18-L19`) đứng sau. Sửa một ký tự trong `app/main.py` thì Docker dùng lại cache từ layer `pip install` trở về trước, chỉ chạy lại các layer copy code và runtime. Nếu đặt `COPY . .` lên trước `pip install`, mọi lần sửa code đều làm đổi layer copy → hủy cache từ đó trở đi → cài lại toàn bộ thư viện mỗi lần build. Bài học của tôi: xếp lệnh ít đổi nhất lên trước, code hay đổi nhất xuống cuối.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện: lỗ hổng trong code Python (ví dụ injection qua `question`) cho kẻ tấn công thực thi shell trong container → vì process chạy bằng root nên shell đó cũng root → lợi dụng thêm lỗ hổng escape là có quyền cao trên host. Lệnh `USER appuser` của tôi (`Dockerfile:L21-L24`, kèm `chown /app`) cắt đứt ngay mắt xích đầu: dù chiếm được shell thì cũng chỉ là user thường — không cài gói, không bind cổng đặc quyền, không đọc/ghi file của root. Nó không vá được lỗ hổng code, nhưng biến "mất cả máy" thành "kẹt trong một phòng khóa".

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa 20 request trong 2 giây: gửi 10 request lúc 10:00:59 rồi 10 request nữa lúc 10:01:01 — mỗi phút đồng hồ đều "đúng luật" 10/phút nhưng thực tế là 20 request trong 2 giây. Sliding window 60 giây của tôi (`WINDOW_SECONDS = 60`, xóa cũ bằng `zremrangebyscore` ở `app/rate_limiter.py:L41` rồi đếm) luôn nhìn 60 giây gần nhất nên request thứ 11 trong cửa sổ bị chặn 429. Trên Railway tôi đã kiểm chứng limit 10/phút: request 1–10 về 200, request 11–12 về 429.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Hai cơ chế khác nhau bốn điểm: mục tiêu (rate limit chặn tần suất, cost guard chặn tiền), đơn vị (số request/60s trượt vs USD/tháng), cửa sổ (60 giây vs tháng `cost:<user>:<YYYY-MM>`), và từ chối (429 kèm `Retry-After` vs 402). Tình huống rate cho qua nhưng cost chặn: user gửi ít request nhưng mỗi request hàng chục nghìn token — dưới 10/phút nhưng vượt ngân sách, `spent + estimated > budget` (`app/cost_guard.py:L56-L60`) trả 402. Ngược lại: bot spam hàng chục request tiny-token — tốn vài xu nhưng vượt 10/phút nên ăn 429 trong khi ngân sách còn nguyên. Vì vậy hai lớp không thay thế nhau.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện: (1) Redis mất kết nối → endpoint gộp báo 503 trên cả 3 container; (2) orchestrator hiểu cả 3 đều "unhealthy" và restart đồng loạt; (3) khi Redis quay lại thì không còn container nào phục vụ — sự cố 30 giây thành outage toàn cụm. Vì vậy tôi tách `/health` chỉ hỏi "process còn sống không", không chạm Redis (`app/main.py:L90-L92`), còn `/ready` được phép kiểm tra Redis và trả 503 `{"status":"not ready","redis":false}` (`L110`) để load balancer chỉ rút traffic chứ không restart. Trên Railway bản deploy của tôi `/ready` trả 200 `redis=true`, chứng tỏ dependency đã nối đúng.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Tôi đã chạy 3 replicas và gọi cùng `X-User-Id`: request đầu vào replica 1 trả `history_length=0`, request hai vào replica 2 trả `history_length=2` — replica 2 đọc được đúng 2 message replica 1 đã ghi, vì cả hai cùng đọc key `history:<user>` trên Redis (`ConversationStore.get_history` ở `app/store.py:L83-L84`, ghi bằng `append` ở `L72-L75`). Nếu lịch sử nằm trong dict Python, mỗi container có RAM riêng: request hai rơi vào replica khác sẽ thấy `history_length=0` trở lại (mất trí nhớ ngẫu nhiên theo load balancer), và restart một replica là mất sạch. Tôi còn restart riêng một replica và history vẫn nguyên — đó là bằng chứng state sống ngoài process.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi của tôi: chạy `docker compose up -d --scale agent=3` thì hai replica chết ngay, log báo đại ý không bind được host port 8000 — vì cả ba cùng khai `ports: "8000:8000"` nên giành nhau một cổng host. Tôi tìm ra bằng `docker compose ps` (thấy container Exited) rồi đọc log từng service. Cách sửa: chuyển sang dynamic host ports (chỉ khai cổng container, để Docker tự gán 61793–61795), sau đó 3 agent và Redis đều healthy. Bài học: cổng container và cổng host là hai thứ khác nhau — scale là nhân container, nên cổng host phải động; trên Railway vấn đề này biến mất vì platform tự gán `$PORT` cho từng instance.
