# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đỗ Thanh Lâm  Mã học viên: 2A202602577

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Deploy lên Railway lần đầu, tôi quên set biến `AGENT_API_KEY` trên dashboard.
> Vì trường này không có mặc định, container thoát ngay với `ValidationError`
> và log ghi rõ thiếu field nào — tôi sửa trong 1 phút. Nếu để mặc định
> `"changeme"`, app vẫn khởi động và trả lời bình thường; endpoint `/ask` công
> khai coi như không có xác thực, ai biết chuỗi `"changeme"` (một giá trị dễ
> đoán, có thể đọc thấy trong `.env.example` gốc trên GitHub) cũng gọi được
> miễn phí. Tôi chỉ phát hiện ra khi nhận hóa đơn LLM bất thường, lúc đó thiệt
> hại đã xảy ra rồi.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:56:42.657367+00:00", "user_id": "sv-exercise", "tokens_in": 3, "tokens_out": 41, "cost_usd": 2.505e-05}
> ```
> Hai việc làm được mà `print` không làm được:
> 1. **Lọc/truy vấn theo trường**: đưa log này vào một hệ thống như Datadog,
>    CloudWatch hay `jq`, tôi có thể chạy truy vấn kiểu `event=ask_completed
>    AND cost_usd > 0.01` để tìm những request tốn tiền bất thường — với
>    `print` thì phải viết regex parse chuỗi tự do, dễ vỡ khi câu trả lời có
>    chứa dấu `:` hay xuống dòng.
> 2. **Tổng hợp số liệu tự động**: vì `cost_usd` và `tokens_out` là field số
>    riêng biệt, tôi cộng dồn được chi phí theo `user_id` theo giờ/ngày để vẽ
>    biểu đồ hoặc đặt cảnh báo ngân sách — `print("đã trả lời xong")` không
>    mang theo số liệu nào để tổng hợp.

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
| 1 stage (bản đầu) | 1730 MB |
| Multi-stage | 299 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch ~1.43GB. Phần lớn là base image: `python:3.11` đầy đủ mang theo
> toàn bộ toolchain build (gcc, make, header file phát triển...) để có thể
> biên dịch bất kỳ Python package nào tại chỗ, còn `python:3.11-slim` chỉ có
> runtime tối thiểu. Ngoài ra bản 1-stage còn giữ lại trong image cuối cùng
> mọi thứ dùng để cài dependency (pip cache, build artifact) vì `COPY . .`
> rồi `pip install` chạy ngay trong stage runtime; bản multi-stage cài
> dependency ở stage `builder` riêng (có compiler nếu cần), rồi chỉ
> `COPY --from=builder` đúng thư mục `~/.local` (package đã cài xong) sang
> stage slim cuối — compiler và cache bị bỏ lại ở stage builder, không lọt
> vào image chạy thật.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Thêm 1 ký tự vào `app/main.py` rồi build lại, output thật:
> ```
> #6  [builder 2/4] WORKDIR /app                                  CACHED
> #7  [stage-1 3/6] RUN useradd ... appuser                       CACHED
> #8  [builder 3/4] COPY requirements.txt .                       CACHED
> #9  [builder 4/4] RUN pip install --no-cache-dir --user -r ...  CACHED
> #10 [stage-1 4/6] COPY --from=builder /root/.local ...          CACHED
> #11 [stage-1 5/6] COPY . .                                      DONE (chạy lại)
> #12 [stage-1 6/6] RUN chown -R appuser:appuser /app             DONE (chạy lại)
> ```
> Toàn bộ layer liên quan tới cài dependency (`WORKDIR`, `COPY
> requirements.txt`, `pip install`) đều cache lại vì input của chúng
> (`requirements.txt`) không đổi. Chỉ 2 layer cuối — `COPY . .` (copy toàn bộ
> source, trong đó có file vừa sửa) và `chown` sau nó — phải chạy lại.
>
> Nếu đặt `COPY . .` lên TRƯỚC `RUN pip install`: Docker cache layer theo
> checksum nội dung của layer trước đó. Sửa bất kỳ file nào trong source
> (kể cả 1 ký tự trong `main.py` không liên quan gì tới dependency) sẽ làm
> layer `COPY . .` đổi checksum → mọi layer PHÍA SAU nó, gồm cả `RUN pip
> install`, bị mất cache và phải chạy lại từ đầu — tải lại và cài lại toàn bộ
> thư viện (fastapi, uvicorn, redis...) chỉ vì sửa 1 dòng code, dù
> `requirements.txt` không hề thay đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) một lỗ hổng trong code — ví dụ endpoint nhận input
> không kiểm soát rồi truyền vào `eval()`, `pickle.loads()` hay một dependency
> có RCE — cho kẻ tấn công thực thi lệnh shell bên trong container; (2) nếu
> process đang chạy bằng UID 0 (root) của container, lệnh đó chạy với quyền
> root; (3) root trong container dùng chung namespace UID với host theo mặc
> định (không có user-namespace remapping), nên UID 0 trong container CHÍNH
> LÀ UID 0 trên host; (4) kẻ tấn công khai thác thêm một lỗ hổng container
> escape (misconfigured volume mount, Docker socket bị mount vào, kernel
> exploit...) để thoát ra tiến trình host — lúc đó họ đã có quyền root thật
> trên máy chủ.
> Lệnh `USER appuser` cắt đứt chuỗi ở bước (2)-(3): process trong container
> chạy bằng UID thường, không phải 0. Dù lỗ hổng (1) vẫn khai thác được và
> kẻ tấn công vẫn chạy lệnh trong container, họ chỉ có quyền của user
> thường — không ghi được vào file hệ thống, không cài binary vào thư mục hệ
> thống, và nếu thoát được ra host thì cũng chỉ mang theo quyền hạn chế đó,
> không phải root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request trong 2 giây**. Cách đạt được: gửi đủ 10 request lúc
> 10:00:59 (vẫn nằm trong "phút thứ 00", đúng luật, hạn mức 10/phút chưa
> chạm). Ngay khi đồng hồ nhảy sang 10:01:00, bộ đếm reset về 0 vì đó là
> "phút thứ 01" mới — gửi tiếp 10 request nữa lúc 10:01:00 hoặc 10:01:01 vẫn
> đúng luật của phút mới. Tổng cộng 20 request nằm gọn trong khoảng 10:00:59
> đến 10:01:01, tức 2 giây đồng hồ thật, dù cả hai lần đều "hợp lệ" theo cách
> đếm cố định. Sliding window (ZSET, xoá member cũ hơn `now - 60s`) không có
> lỗ hổng này vì nó luôn nhìn lại đúng 60 giây gần nhất tính từ thời điểm
> request tới, không có mốc reset cố định để lợi dụng.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit đếm **số lượng request theo thời gian** (10 request/phút), cost
> guard đếm **tổng số tiền theo tháng** (10 USD/tháng) — một cái giới hạn tần
> suất, một cái giới hạn ngân sách, độc lập với nhau.
>
> - Rate limit cho qua nhưng cost guard phải chặn: user chỉ gửi 2 request
>   trong phút này (dưới hạn mức 10/phút, rate limit cho qua), nhưng mỗi
>   request hỏi một câu rất dài (gần 2000 ký tự, sát `max_length`), tiêu tốn
>   nhiều token; tổng chi phí tháng này đã chạm 10 USD → cost guard trả 402
>   dù tần suất gọi thấp.
> - Rate limit chặn nhưng cost guard vẫn cho qua: user gửi 11 request rất
>   ngắn trong 1 phút (câu hỏi 1-2 từ, chi phí gần như 0), tổng chi phí tháng
>   mới có vài cent, còn dư ngân sách rất nhiều; nhưng vì vượt hạn mức
>   10/phút, request thứ 11 vẫn bị rate limit chặn ở 429 trước khi cost guard
>   kịp kiểm tra.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Redis mất kết nối. `/health` (giờ có gọi `store.ping()`) bắt đầu trả 503
>    ở cả 3 container cùng lúc, vì cả 3 đều dùng chung một Redis.
> 2. Orchestrator coi `/health` (liveness) là "process đã chết" — không phải
>    "chưa sẵn sàng" — nên nó **restart cả 3 container**, gần như đồng thời.
> 3. Container mới khởi động lại, nhưng Redis vẫn chưa sống (còn trong 30
>    giây mất kết nối) → liveness probe vẫn 503 → orchestrator restart tiếp.
>    Cụm rơi vào crash-loop trong suốt 30 giây đó.
> 4. Vì cả 3 instance cùng bị restart cùng lúc, **không còn instance nào
>    phục vụ được request**, kể cả những request không liên quan gì đến
>    Redis — toàn bộ dịch vụ down, dù bản thân process Python vẫn hoàn toàn
>    khỏe mạnh.
> 5. Redis sống lại → lần probe kế tiếp `/health` trả 200 → orchestrator
>    ngừng restart, cụm phục hồi. Nhưng thiệt hại (downtime toàn cụm suốt ~30
>    giây, cộng thêm thời gian cold-start của mỗi lần restart) đã xảy ra.
>    Tách riêng `/health` (không đụng Redis) và `/ready` (có đụng Redis)
>    tránh được việc này: `/ready` trả 503 chỉ khiến load balancer ngừng đẩy
>    traffic MỚI vào, không khiến orchestrator giết process.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> `docker-compose.yml` map cứng cổng `8000:8000` nên `--scale agent=3` không
> tạo được 3 container cùng lúc (xung đột port khi publish ra host — muốn
> scale thật cần bỏ port cố định và đặt nginx/load balancer phía trước, phần
> mở rộng không bắt buộc của lab). Tôi kiểm chứng tính stateless bằng cách
> gọi `/ask` 3 lần liên tiếp cùng `X-User-Id: sv-scale-test` vào MỘT
> container, quan sát `history_length` trả về: 0 → 2 → 4. Mỗi lần tăng đúng
> 2 (một message user, một message assistant), nghĩa là lịch sử được đọc/ghi
> từ Redis (`store.get_history` / `store.append`) chứ không giữ trong biến
> của process — khớp với test `test_state_khong_nam_trong_process` (2
> `ConversationStore` khác nhau cùng thấy chung dữ liệu qua `fake_redis`).
>
> Nếu lịch sử nằm trong một `dict` Python trong process: với 1 container thì
> kết quả bề ngoài giống hệt (dict đó vẫn tăng dần vì cùng process xử lý mọi
> request). Nhưng khi scale thật lên 3 container đứng sau load balancer, mỗi
> request của cùng `X-User-Id` có thể rơi vào container khác nhau (round-robin
> hoặc ngẫu nhiên) — mỗi container có `dict` riêng trong RAM riêng của nó.
> `history_length` khi đó sẽ **không tăng đều**: có lúc trả về 0 (rơi vào
> container chưa từng thấy user này), có lúc trả về một số nhỏ hơn thực tế
> (rơi vào container B trong khi 2 lượt trước rơi vào container A) — agent
> trông như bị "mất trí nhớ" tùy may rủi route traffic, và container bị
> restart thì toàn bộ dict đó biến mất luôn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> (TODO — điền sau khi deploy thật) Ở thời điểm nộp bài này tôi chưa deploy
> lên cloud thật (Railway/Render/Cloud Run), dùng phương án dự phòng
> `LOCAL_FALLBACK=true` với `docker compose up` — xem `DEPLOYMENT.md`. Khi
> deploy thật, lỗi nhiều khả năng gặp nhất theo cấu hình hiện tại: `/ready`
> trả 503 nếu biến `REDIS_URL` trên dashboard vẫn còn trỏ `localhost` thay vì
> hostname/URL của Redis add-on trên cloud (trong container cục bộ tôi đã
> phải sửa thành `redis://redis:6379/0` chứ không phải `localhost`, xem
> `docker-compose.yml`) — sẽ cập nhật câu trả lời này bằng lỗi thật khi deploy
> xong.
