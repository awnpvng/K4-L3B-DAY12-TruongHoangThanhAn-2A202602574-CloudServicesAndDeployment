# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng đánh dấu "chưa trả lời" ở mỗi câu bằng câu trả lời
> thật của bạn. `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: ..........................  Mã học viên: ..........................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi mình cấu hình `render.yaml`, biến `AGENT_API_KEY` khai báo `sync: false`
> nghĩa là Render bắt buộc mình phải tự nhập giá trị lúc tạo Blueprint — nếu
> quên nhập, vì `agent_api_key: str` không có default, Pydantic ném
> `ValidationError` ngay lúc container khởi động, service không bao giờ lên
> trạng thái "Live" và mình biết ngay có chuyện. Nếu field này có default
> `"changeme"`, container vẫn khởi động bình thường, `/health` vẫn trả 200,
> nhưng `/ask` sẽ chấp nhận bất kỳ ai gửi đúng chuỗi `"changeme"` — một public
> URL với khóa mặc định ai cũng đoán được thì coi như không có xác thực. Mình
> chỉ phát hiện ra khi xem log thấy user lạ gọi `/ask` liên tục, tức là đã bị
> lợi dụng một khoảng thời gian trước đó.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật lấy từ `docker compose logs agent` khi mình test rate limit:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T02:47:14.427112+00:00", "user_id": "sv02", "tokens_in": 302, "tokens_out": 43, "cost_usd": 7.11e-05}
> ```
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
> 1. **Lọc/tổng hợp theo trường**: vì mỗi dòng là JSON có key rõ ràng, mình
>    dùng `jq` hoặc query trên Datadog để tính tổng `cost_usd` theo `user_id`
>    trong ngày — trả lời câu "user nào tốn tiền nhất". Với `print` thì phải
>    tự viết regex đoán vị trí số tiền trong câu chữ tự do.
> 2. **Cảnh báo tự động theo điều kiện**: đặt alert kiểu "nếu `level: error`
>    xuất hiện quá 10 lần trong 5 phút thì bắn Slack" — máy đọc được field
>    `level` trực tiếp. `print` không có field, chỉ có chuỗi, máy không phân
>    biệt được "báo lỗi" với "câu trong nội dung trả lời" nếu tình cờ có chữ
>    "error".

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
| 1 stage (bản đầu) | 1730 MB (1.73GB) |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Số đo thật từ `docker images`: bản 1 stage dùng `python:3.11` (bản đầy đủ,
> có sẵn compiler, header file, các thư viện dựng sẵn cho nhiều mục đích) và
> `COPY . .` rồi `pip install` ngay trên đó — mọi cache pip, mọi file nguồn
> `.git` không bị loại trừ đều nằm lại trong image cuối. Bản multi-stage dùng
> `python:3.11-slim` (đã nhỏ hơn nhiều) cho cả hai stage, và quan trọng nhất
> là stage `builder` (chứa toàn bộ quá trình `pip install`, cache pip, các gói
> build phụ trợ nếu có) bị **vứt bỏ hoàn toàn** sau khi `COPY --from=builder
> /install /usr/local` chỉ mang đúng thư viện Python đã cài đặt xong sang
> stage `runtime`. Gần 1.46GB chênh lệch chủ yếu là: base image đầy đủ so với
> slim, và toàn bộ "rác build" (pip cache, apt cache nếu có, các layer trung
> gian của quá trình cài đặt) không đi theo vào image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Mình thêm một dòng comment vào cuối `app/main.py` rồi `docker build` lại,
> output thật:
>
> ```
> [builder 2/4] WORKDIR /app               CACHED
> [builder 3/4] COPY requirements.txt .    CACHED
> [builder 4/4] RUN pip install ...        CACHED
> [runtime 3/7] COPY --from=builder ...    CACHED
> [runtime 4/7] RUN useradd ...            CACHED
> [runtime 5/7] COPY app ./app             (chạy lại, không CACHED)
> [runtime 6/7] COPY utils ./utils         (chạy lại)
> [runtime 7/7] RUN chown -R appuser ...   (chạy lại)
> ```
>
> Toàn bộ layer cài dependency (cả stage `builder`) được giữ cache — build lại
> chỉ mất vài giây thay vì gần 100 giây cài lại `pip install`. Nếu đảo ngược,
> đặt `COPY . .` lên trước `RUN pip install -r requirements.txt`, thì mọi thay
> đổi trong `app/` (kể cả một dấu phẩy) sẽ làm Docker coi layer `COPY . .` đã
> đổi → hủy cache từ đó trở đi → `pip install` phải chạy lại từ đầu mỗi lần,
> dù `requirements.txt` không hề thay đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) kẻ tấn công tìm ra một lỗ hổng trong app (ví dụ dependency
> có RCE, hoặc một thư viện xử lý input không an toàn) và chạy được lệnh tùy ý
> bên trong container; (2) nếu process FastAPI/uvicorn đang chạy bằng root,
> lệnh tùy ý đó cũng chạy với quyền root **bên trong container**; (3) nếu
> container engine có lỗ hổng escape (container breakout) hoặc container được
> mount volume/socket nhạy cảm (ví dụ `/var/run/docker.sock`), quyền root
> trong container có thể leo thang thành quyền root **trên máy host** — từ đó
> kẻ tấn công kiểm soát toàn bộ máy chủ, không chỉ riêng service của mình.
>
> Dòng `RUN useradd --create-home --uid 10001 appuser` và `USER appuser` trong
> `Dockerfile` cắt đứt chuỗi này ở bước (2): ngay cả khi kẻ tấn công chạy được
> lệnh tùy ý bên trong container, lệnh đó chỉ có quyền của `appuser` (UID
> 10001, không có sudo, không ghi được vào hầu hết hệ thống file), nên dù có
> lỗ hổng escape ở tầng container engine thì thiệt hại tối đa cũng chỉ bằng
> quyền một user thường, không phải root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request trong 2 giây**. Cách đạt được: gửi đúng 10 request vào
> lúc 10:00:59 (vẫn tính vào "phút 10:00", chưa vượt hạn mức 10/phút của phút
> đó), rồi gửi tiếp 10 request vào lúc 10:01:01 (bộ đếm đã reset về 0 vì sang
> "phút 10:01" mới, nên 10 request này cũng hợp lệ). Tổng cộng 20 request chỉ
> trong khoảng 2 giây đồng hồ thực tế, nhưng theo cách đếm "phút đồng hồ" thì
> cả hai đợt đều "đúng luật" vì chúng rơi vào hai cửa sổ đếm khác nhau. Sliding
> window 60 giây mà mình cài ở `rate_limiter.py` không có kẽ hở này vì nó luôn
> nhìn lại đúng 60 giây gần nhất tính từ thời điểm request đến, không neo theo
> mốc giờ cố định.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn **số lượng** request trong một khoảng thời gian ngắn
> (60 giây), không quan tâm mỗi request tốn bao nhiêu tiền. Cost guard giới
> hạn **tổng số tiền** trong cả tháng, không quan tâm request đến nhanh hay
> chậm.
>
> - Rate limit cho qua nhưng cost guard chặn: user gửi đúng 5 request/phút
>   (dưới hạn mức 10/phút nên rate limit không chặn), nhưng mỗi câu hỏi rất
>   dài (nhiều token) khiến `cost_usd` cộng dồn nhanh; đến giữa tháng tổng chi
>   đã vượt `MONTHLY_BUDGET_USD` → `guard.check()` trả 402 dù tần suất gọi vẫn
>   bình thường.
> - Rate limit chặn nhưng cost guard vẫn còn dư ngân sách: user gửi 15 request
>   liên tiếp trong vài giây, mỗi câu hỏi rất ngắn và rẻ (như lúc mình test
>   thật ở trên, `cost_usd` chỉ khoảng 0.00002–0.00008 USD/lần) — tổng chi phí
>   còn rất xa ngân sách 10 USD/tháng, nhưng `limiter.check()` vẫn trả 429 ở
>   request thứ 11 vì vượt tần suất cho phép, bất kể tiền còn nhiều.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện nếu `/health` cũng gọi `store.ping()`:
> 1. Redis mất kết nối → `ping()` (như mình cài ở `store.py`, bọc try/except)
>    trả `False` ở cả 3 container cùng lúc, vì cả 3 đều trỏ chung một
>    `REDIS_URL`.
> 2. Endpoint gộp trả 503 ở cả 3 container — vì đây là `/health` (liveness),
>    orchestrator (Docker/Render) hiểu 503 nghĩa là "process hỏng, cần khởi
>    động lại", chứ không phải "tạm thời đừng gửi traffic vào".
> 3. Orchestrator **restart cả 3 container** gần như đồng thời.
> 4. Trong lúc cả 3 đang khởi động lại (mất vài giây đến vài chục giây), cụm
>    hoàn toàn không có instance nào chạy để phục vụ request — kể cả những
>    request không liên quan gì đến Redis.
> 5. Nếu Redis quay lại đúng lúc 3 container vẫn đang restart, khi chúng lên
>    lại thì mọi thứ ổn — nhưng nếu Redis còn chưa kịp hồi phục, `/health`
>    (đã gộp) tiếp tục 503 → bị restart tiếp → vòng lặp restart liên tục.
>
> Đây chính là lý do CP4 tách riêng `/ready` (được phép kiểm tra Redis, trả
> 503 chỉ khiến load balancer rút instance ra khỏi vòng xoay, không restart)
> khỏi `/health` (không đụng dependency, chỉ trả lời "process còn sống").

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Vì compose ánh xạ cổng cố định `8000:8000` nên không scale ra ngoài qua cùng
> một cổng được, mình test bằng cách gọi trực tiếp vào IP nội bộ của từng
> container (`docker inspect` lấy IP, gọi qua network của compose) — tương
> đương việc load balancer phân phối request vào các container khác nhau.
> Kết quả thật với cùng `X-User-Id: scale-test`:
>
> ```
> container 172.19.0.4 → history_length: 0
> container 172.19.0.3 → history_length: 2
> container 172.19.0.5 → history_length: 4
> ```
>
> `history_length` tăng dần liên tục (0 → 2 → 4) dù mỗi lần gọi vào một
> container khác nhau — chứng minh cả 3 container cùng đọc/ghi một nguồn dữ
> liệu chung (Redis), không phải bộ nhớ riêng của từng process.
>
> Nếu lịch sử được lưu trong một `dict` Python trong process thay vì Redis,
> mỗi container sẽ có dict riêng trong RAM của nó. Khi đó `history_length` sẽ
> **không tăng liên tục** mà nhảy lộn xộn theo container nào xử lý request:
> ví dụ có thể thấy 0 → 1 → 0 → 1 (mỗi container tự đếm từ đầu), tức là agent
> "quên" mất các lượt hỏi trước đó mỗi khi load balancer đổi sang container
> khác — đúng như phần lý thuyết CP4 mô tả.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Trên dashboard Render, deploy đầu tiên ("Correct submission naming to L3B",
> commit `a790628`) chạy 1m07s rồi kết thúc bằng dấu ❌ (xem
> `screenshots/dashboard.png`). Nguyên nhân: lúc đó mình tạo Blueprint và bấm
> deploy trên Render **trước khi push code CP1–CP4 lên GitHub** — Render kéo
> đúng commit `a790628` (bản còn nguyên các `raise NotImplementedError(...)`
> và Dockerfile 1-stage cũ), nên build/khởi động thất bại vì code chưa hoàn
> chỉnh với nhánh `main` trên GitHub, không phải lỗi cấu hình Render.
>
> Cách mình tìm ra: so sánh mã commit hiển thị trên dashboard (`a790628`) với
> lịch sử commit thật trong repo cục bộ, thấy đó là commit rất cũ, trước khi
> mình implement xong các CP — tức Render đang build code cũ do GitHub chưa
> có bản mới.
>
> Cách sửa: `git add -A && git commit && git push` code đã hoàn thiện lên
> nhánh `main` trên GitHub trước, sau đó bấm **Manual Deploy** lại trên
> Render để nó kéo đúng commit mới nhất (`5cd62e4`) — lần này build 36.7s và
> lên trạng thái ✅ Live. Bài học: **thứ tự bắt buộc là push trước, deploy
> sau** — Blueprint của Render chỉ đọc được những gì đã có trên GitHub, không
> đọc code đang nằm trên máy mình.
