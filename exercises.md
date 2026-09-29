# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder `*Câu trả lời của bạn*` (có dấu `>` ở đầu) bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Anh Duy  Mã học viên: 2A202602723

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Không có giá trị mặc định thì app báo lỗi ngay lúc khởi động, trước khi có request nào cần dùng API key. Còn nếu có giá trị mặc định mà giá trị đó không dùng được, thì phải đến lúc thật sự dùng mới lộ lỗi. Ví dụ: deploy lên Render mà quên set `AGENT_API_KEY`. Không có mặc định thì container không chạy được, log hiện `ValidationError` và mình sửa được ngay. Còn để mặc định `"changeme"` thì app vẫn chạy bình thường, lỗi chỉ lộ ra sau khi deploy nên sửa tốn kém hơn, và ai đoán được khóa `changeme` là gọi được `/ask`.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thu được: `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:57:52.720529+00:00", "user_id": "sv01", "tokens_in": 2, "tokens_out": 34, "cost_usd": 2.07e-05}`. Vì log có cấu trúc nên ta dễ theo dõi hơn, và có thể tìm log theo từng trường, ví dụ tìm tất cả log có `user_id` là `sv01`. Với `print("đã trả lời xong")` thì không làm được việc này, vì chuỗi đó không có trường nào để máy tách ra lọc.

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
| 1 stage (bản đầu) | 1.73 GB (~1770 MB) |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần chênh lệch chủ yếu đến từ base image: bản `python:3.11` đầy đủ cài sẵn các công cụ build như `gcc`, `make` và các gói `apt` (thư viện dev, header...), riêng phần này đã hơn 1 GB. Bản `python:3.11-slim` không có những thứ đó, và lúc chạy app cũng không cần chúng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Hai layer `COPY requirements.txt` và `RUN pip install` được lấy từ cache, vì chúng nằm trước `COPY app` và `requirements.txt` không đổi. `main.py` nằm trong `app`, nên Docker phải build lại từ layer `COPY app` trở về sau. Vì vậy build lại rất nhanh (đo được khoảng 2 s). Còn nếu đưa `COPY . .` lên trước `pip install`, thì sửa `main.py` làm layer `COPY . .` thay đổi, kéo theo `pip install` phải chạy lại và tải lại toàn bộ thư viện, rất tốn thời gian (đo được khoảng 82 s, riêng `pip install` mất 75.8 s).

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Container chạy bằng root thì có toàn quyền ghi file và thực thi lệnh. Nếu code Python có lỗ hổng, kẻ tấn công chiếm được app cũng sẽ có quyền root trong container, và từ đó dễ gây hại cho hệ thống hoặc thoát ra máy host. Lệnh `USER` cho app chạy bằng user thường, chỉ có đúng những quyền cần thiết (least privilege). Nhờ vậy, kể cả khi bị tấn công, kẻ tấn công cũng không đủ quyền để gây hại cho hệ thống hay thoát ra khỏi container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request**. Xét hai phút đồng hồ liền nhau: L1 = giây [0, 59] và L2 = giây [60, 119], bộ đếm reset ở đầu mỗi phút. Người dùng gửi 10 request ở giây 59 (cuối L1, vẫn trong hạn mức của L1), rồi 10 request ở giây 60 (đầu L2, bộ đếm vừa reset). Như vậy trong 2 giây liên tiếp đã có 20 request, gấp đôi hạn mức. Sliding window thì luôn đếm đúng 60 giây gần nhất, nên request thứ 11 sẽ bị chặn.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn **số request** trong một khoảng thời gian, để chặn việc gọi dồn dập vượt quá khả năng xử lý của server. Cost guard giới hạn **số tiền** mỗi người dùng được tiêu trong tháng. Ví dụ: người dùng đã hết ngân sách tháng, hôm nay họ chưa gửi request nào nên vẫn qua được rate limit, nhưng bị cost guard chặn (402).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> `/health` kiểm tra process có còn sống hay không, phải nhanh, nhẹ và không gọi Redis. `/ready` kiểm tra instance đã sẵn sàng phục vụ chưa: các dependency đã khởi động xong chưa, hoặc nếu từng bị down thì đã hồi phục chưa. Khi Redis mất kết nối, cả 3 container vẫn trả `/health` là ok vì process vẫn chạy bình thường, còn `/ready` trả "not ready" (503) vì phát hiện Redis chưa sẵn sàng.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Chạy 3 container agent, gọi `/ask` 6 lần với cùng `X-User-Id: q9user`, xoay vòng agent-1 -> agent-2 -> agent-3 -> agent-1 -> agent-2 -> agent-3, `history_length` lần lượt là 0, 2, 4, 6, 8, 10, và sau 6 lượt Redis `LLEN history:q9user` = 12. `history_length` tăng đều 2 sau mỗi lượt (mỗi lần `/ask` ghi 2 message: câu hỏi và câu trả lời) dù mỗi request rơi vào một container khác nhau, vì cả 3 container cùng đọc/ghi lịch sử vào một Redis chung. Nếu lưu bằng dict Python, mỗi container có một dict riêng trong RAM và chỉ nhớ những lượt chính nó xử lý: request 1–3 đều thấy `history_length = 0` (mỗi container gặp user lần đầu), request 4–6 đều thấy 2, thay vì 6, 8, 10, nên người dùng thấy agent "lúc nhớ lúc quên". Thực tế còn tệ hơn vì load balancer không chia đều nên con số nhảy lung tung, và mỗi lần container restart hay deploy lại thì lịch sử mất sạch. Vì vậy state phải nằm ngoài process (Redis) thì mới scale được.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Ngay sau khi deploy lên Render, gọi `curl -i <URL>/health` và `/ready` thì lúc được 200, lúc lại trả `HTTP/1.1 404 Not Found` với body `Not Found` (text thường, không phải JSON của FastAPI). Để tìm nguyên nhân, tôi xem header của response và thấy `x-render-routing: no-server`, trong khi `<URL>/openapi.json` vẫn liệt kê đủ `/health`, `/ready`, `/ask`, tức là code có route, lỗi 404 do tầng routing của Render trả về chứ không phải app. Nghĩa là lúc đó Render chưa có instance nào sẵn sàng để chuyển request vào (service vừa deploy/redeploy xong, hoặc gói free vừa "thức dậy" sau khi ngủ). Cách xử lý: không sửa code, mà chờ instance khởi động xong và kiểm tra lại bằng cách gọi `/health` lặp lại vài lần cho tới khi ổn định 200; `healthCheckPath: /health` trong `render.yaml` giúp Render chỉ chuyển traffic vào instance khi `/health` đã trả 200. Sau đó `/health` và `/ready` đều trả 200 ổn định (`{"status":"ready","redis":true}`).
