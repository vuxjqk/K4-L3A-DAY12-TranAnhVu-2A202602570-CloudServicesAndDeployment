# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder `*Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Anh Vũ  Mã học viên: 2A202602570

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Giả sử tôi deploy lên Railway nhưng quên set `AGENT_API_KEY`. Nếu có mặc
> định `"changeme"`, app vẫn khởi động, health check xanh, và `/ask` được bảo vệ
> bằng một khóa ai cũng đoán được — bot quét Internet gọi thoải mái, tôi chỉ
> biết khi nhìn hóa đơn LLM. Không có mặc định thì `Settings()` ném
> `ValidationError` ngay lúc start, deploy fail, log báo thiếu biến — tôi sửa
> trong vài phút khi vẫn đang nhìn dashboard, trước khi có request nào lọt vào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật khi chạy bằng `docker compose`:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:21:32.127914+00:00", "user_id": "sv01", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}
> ```
>
> 1. **Lọc và tổng hợp theo trường**: ví dụ cộng `cost_usd` theo `user_id` để
>    biết ai tiêu nhiều nhất hôm nay, hoặc đếm số `ask_completed` mỗi phút.
> 2. **Đặt cảnh báo tự động**: tạo alert khi `level == "error"` tăng đột biến
>    hoặc `tokens_in` vượt ngưỡng. Với `print("đã trả lời xong")` chỉ có chuỗi
>    tự do — không biết của user nào, lúc nào, tốn bao nhiêu, máy không parse được.

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
| 1 stage (bản đầu) | 1.73 GB (~1730 MB) |
| Multi-stage | 247 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Đo thật bằng `docker image ls`: bản 1 stage 1.73 GB, bản multi-stage 247 MB
> — nhỏ hơn khoảng 7 lần. Phần chênh lệch ~1.5 GB gồm:
>
> - **Base image đầy đủ `python:3.11`** (gần như toàn bộ chênh lệch): kèm
>   compiler `gcc`, header, thư viện `-dev`, git, curl... để build C extension.
>   Tôi kiểm tra trong image 1 stage vẫn có `/usr/bin/gcc`. App lúc chạy không
>   cần những thứ đó; `python:3.11-slim` bỏ hết.
> - **Cache của pip**: bản đầu không dùng `--no-cache-dir` nên còn ~17 MB ở
>   `/root/.cache/pip`.
> - **File thừa từ `COPY . .`**: `grade.py`, `nginx/`, `railway.toml`,
>   `render.yaml`... (với `.dockerignore` ban đầu chỉ có `.git` thì còn cả
>   `tests/`, tài liệu và nguy hiểm nhất là `.env`).
>
> Bản multi-stage chỉ giữ slim base + thư viện đã cài (layer
> `COPY --from=builder /install` ~65 MB) + `app/` và `utils/`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Tôi thêm một ký tự vào `app/main.py` rồi `docker build --progress=plain`:
>
> - **Dùng lại cache** (`CACHED`): `WORKDIR /build`, `COPY requirements.txt`,
>   `RUN pip install ...` ở stage builder; `COPY --from=builder /install`,
>   `RUN useradd`, `WORKDIR /app` ở stage runtime.
> - **Chạy lại**: chỉ `COPY app ./app` và `COPY utils ./utils` (mỗi lệnh ~0,1 s).
>   Cả lần build chỉ mất vài giây.
>
> Nếu đặt `COPY . .` trước `RUN pip install`, sửa một ký tự bất kỳ cũng làm
> layer `COPY` đổi checksum → mọi layer phía sau mất cache → pip tải và cài lại
> toàn bộ thư viện. Với mạng chậm như máy tôi (pip từng timeout sau 200+ giây)
> thì mỗi lần sửa code là mất vài phút, thay vì vài giây.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: code Python có lỗ hổng (ví dụ RCE qua thư viện dính CVE)
> → kẻ tấn công chạy lệnh shell trong container **với uid 0** → root trong
> container có quyền ghi mọi file, cài tool, đọc secret; nếu gặp thêm một lỗi
> cấu hình hoặc lỗ hổng kernel/runtime (volume mount thư mục host, Docker socket,
> container `--privileged`) thì root trong container thoát ra thành root trên
> host. `USER appuser` (uid 10001) cắt chuỗi ở bước thứ hai: lệnh của kẻ tấn công
> chỉ chạy với quyền user thường — không cài được gói, không ghi được file hệ
> thống, và thoát khỏi container thì cũng chỉ là một uid không đặc quyền.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request trong 2 giây**. Cách đạt: gửi 10 request lúc 10:00:59
> (hết quota của phút 10:00), đến 10:01:00 bộ đếm reset về 0, gửi tiếp 10
> request lúc 10:01:00–10:01:01. Cả hai lô đều "đúng luật" vì thuộc hai phút
> đồng hồ khác nhau. Với sliding window, request lúc 10:01:01 vẫn thấy 10
> request trong 60 giây gần nhất nên bị 429.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn **số request trên thời gian** (10/phút) để chống spam
> và bảo vệ tài nguyên; cost guard giới hạn **tổng tiền trên tháng** ($10/user)
> để bảo vệ ngân sách. Trả về cũng khác: 429 và 402.
>
> - Rate limit cho qua, cost guard chặn: user gửi đều 5 request/phút, mỗi
>   request prompt rất dài tốn nhiều token; không lần nào quá nhanh nhưng vài
>   ngày là cộng dồn vượt $10 → 402.
> - Cost guard cho qua, rate limit chặn: user mới, gần như chưa tiêu gì, nhưng
>   script lỗi bắn 15 request trong 1 giây → từ request thứ 11 bị 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Redis mất kết nối, cả 3 container cùng lúc thấy endpoint trả 503.
> 2. Orchestrator coi đó là liveness fail → sau vài lần retry thì **restart cả 3**.
> 3. Trong lúc restart không còn instance nào phục vụ — kể cả những request
>    không cần Redis — user thấy 502/503 toàn bộ.
> 4. Container khởi động lại, Redis vẫn chưa về → lại fail → vòng lặp restart.
> 5. Redis quay lại sau 30 giây, nhưng hệ thống còn phải chờ cả 3 container
>    start xong và qua health check mới phục vụ lại.
>
> Tách ra thì `/health` vẫn 200 (không restart), chỉ `/ready` 503 → load
> balancer tạm ngừng gửi traffic, Redis về là cả 3 nhận lại ngay.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Tôi chạy 3 instance (cổng 8000, 8001, 8002) và gọi lần lượt với cùng
> `X-User-Id`. Kết quả thật: `history_length` = 0 → 2 → 4 → 6 → 8 → 10, tăng đều
> dù mỗi request vào một container khác, vì cả 3 đọc/ghi chung Redis.
>
> Nếu lưu trong dict Python, mỗi container có dict riêng nên con số nhảy lung
> tung theo container nhận request, ví dụ 0 → 0 → 0 → 2 → 2 → 2: agent "quên"
> hội thoại mỗi khi đổi instance, và mất sạch khi container restart.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi gặp khi build image (cùng Dockerfile dùng để deploy):
> `ERROR: ResolutionImpossible` ở bước `RUN pip install`, sau hơn 200 giây.
>
> Cách tìm nguyên nhân: cùng `requirements.txt` cài được trong venv Python 3.11
> ở máy, nên không phải xung đột phiên bản thật. Tôi build lại với
> `--progress=plain` để xem log đầy đủ và thấy `Read timed out` tới pypi.org;
> đo bằng `curl` thì mỗi request tới PyPI/Docker Hub mất ~17 giây. Pip timeout
> khi tải metadata nên tưởng không có phiên bản nào thỏa mãn.
>
> Cách sửa: thêm `--default-timeout=120 --retries 10` cho `pip install` trong
> stage builder; build lại thành công, và lần `railway up` build ngay lần đầu.
> Ngoài ra tôi bỏ `startCommand` trong `railway.toml` để Railway dùng `CMD` của
> Dockerfile (`sh -c "exec uvicorn ... --port ${PORT:-8000}"`), tránh rủi ro
> `$PORT` không được nội suy. Kiểm tra lại bằng `/health`, `/ready` đều 200.
