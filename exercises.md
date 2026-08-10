# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng giữ chỗ bên dưới bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Đức Chung  Mã học viên: 2A202601705

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Một tình huống cụ thể là lúc deploy lên cloud nhưng tôi quên đặt
> `AGENT_API_KEY`. Trong bản hiện tại, `agent_api_key` không có mặc định,
> không chấp nhận chuỗi rỗng và cấu hình được nạp ngay trong lifespan, nên
> container dừng lúc khởi động với `ValidationError`; log chỉ thẳng biến còn
> thiếu trước khi service nhận traffic. Nếu dùng mặc định `changeme`, service
> vẫn có thể báo healthy và người biết khóa mặc định có thể gọi `/ask`, dùng
> tài nguyên và ngân sách của tôi. Fail fast biến lỗi cấu hình thành lỗi deploy
> rõ ràng thay vì một lỗ hổng âm thầm trên public URL.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON tôi thu được khi gọi `/ask` cục bộ là:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T04:11:11.684641+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}`.
> Với log này tôi có thể: (1) lọc/đếm `ask_completed` theo `user_id`, `level`
> và khoảng thời gian để tìm lưu lượng bất thường; (2) cộng `cost_usd`,
> `tokens_in`, `tokens_out` để theo dõi chi phí và đặt cảnh báo ngân sách.
> `print("đã trả lời xong")` không có các trường ổn định để truy vấn, nhóm hay
> tính tổng như vậy.

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
| 1 stage (bản đầu) | 1.730 MB (Docker hiển thị 1,73 GB) |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build thật hai tag `day12-agent:single` và `day12-agent:prod`. Bản mới
> giảm khoảng 1,46 GB, tương đương 84,4%. Phần chênh lệch chủ yếu đến từ việc
> bản đầu dùng image đầy đủ `python:3.11`, giữ cả công cụ/hệ thống không cần cho
> runtime và cache cài gói trong cùng image. Bản mới dùng `python:3.11-slim`,
> tắt pip cache và chỉ copy các package đã cài từ builder sang runtime; source
> được copy có chọn lọc nên image không mang theo test, tài liệu, `.git` hay
> môi trường ảo trên máy phát triển.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi tôi sửa code trong `app/` rồi build lại, output BuildKit cho thấy toàn bộ
> stage `builder` vẫn lấy cache: base image, `WORKDIR`, `COPY requirements.txt`
> và `RUN pip install` đều không chạy lại. Ở runtime, base image,
> `WORKDIR` và `COPY --from=builder /install /usr/local` cũng dùng cache;
> cache bị mất từ `COPY app ./app`, sau đó `COPY utils` và lệnh tạo/chown user
> được dựng lại. Nếu đặt `COPY . .` trước `RUN pip install`, chỉ cần đổi một ký
> tự source là layer `COPY` đổi và kéo theo việc cài lại toàn bộ dependency,
> làm build chậm hơn dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi rủi ro là: lỗ hổng Python cho phép thực thi lệnh, lệnh chạy với user
> hiện tại của container; nếu đó là root thì kẻ tấn công có UID 0 trong
> container, có thể sửa file/process hệ thống và lợi dụng volume đặc quyền,
> Docker socket hoặc lỗ hổng container-runtime/kernel để leo sang host. Root
> trong container không tự động là root trên host, nhưng làm hậu quả và bước
> thoát container nguy hiểm hơn nhiều. `USER appuser` (UID 10001) cắt chuỗi
> ngay sau bước thực thi mã: mã độc chỉ có quyền user thường, không làm được
> các thao tác cần root. Đây là giảm thiểu tác động, không phải bảo vệ tuyệt đối.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa là **20 request trong 2 giây**: gửi 10 request ở cuối phút, ví dụ
> 10:00:59.x, rồi bộ đếm reset lúc 10:01:00 và gửi thêm 10 request ở đầu phút
> mới. Mỗi nhóm thuộc một fixed window khác nhau nên đều hợp lệ. Sliding window
> hiện tại vẫn nhìn thấy 10 request cũ trong 60 giây gần nhất và chặn nhóm sau,
> nên không có burst 20 request này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số request trong 60 giây gần nhất và trả 429; cost guard
> giới hạn tổng USD theo từng user trong tháng UTC và trả 402. Rate limit có
> thể cho qua nhưng cost guard chặn khi đây là request đầu tiên trong phút,
> trong khi user đã tiêu quá ngân sách 10 USD. Chiều ngược lại, user mới tiêu
> rất ít nhưng gửi request thứ 11 trong cùng 60 giây: cost guard vẫn cho qua
> nhưng rate limiter chặn. Hai cơ chế bảo vệ hai tài nguyên khác nhau: tốc độ
> và tiền.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp `/health` với `/ready` và cho probe đó kiểm tra Redis: (1) Redis mất
> kết nối; (2) cả ba container cùng trả 503; (3) sau ngưỡng retry, orchestrator
> hiểu nhầm cả ba process đã chết và restart chúng; (4) container mới vẫn chưa
> nối được Redis nên tiếp tục 503, tạo vòng lặp restart đồng loạt; (5) request
> đang xử lý bị cắt và cụm không còn replica khỏe. Với thiết kế hiện tại,
> `/health` vẫn 200 để process không bị restart, còn `/ready` trả 503 để load
> balancer tạm ngừng gửi traffic. Redis trở lại thì `/ready` tự về 200 và cụm
> nhận traffic mà không phải restart.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Tôi đã chạy `docker-compose up -d --scale agent=3`: cả ba agent, Redis và
> Nginx đều ở trạng thái healthy. Năm lần gọi `/ask` qua Nginx với cùng
> `X-User-Id` trả `history_length` lần lượt **0, 2, 4, 6, 8**, vì mỗi lượt ghi
> hai message vào Redis chung. Nếu mỗi container giữ một dict riêng và Nginx
> chia vòng tròn, kết quả sẽ thành 0, 0, 0, 2, 2, 2, ...; nếu phân phối không
> đều thì số còn có thể nhảy tới/lùi, và restart container sẽ làm mất lịch sử.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Deployment Railway đầu tiên build image thành công nhưng healthcheck thất bại.
> Trong deployment log tôi thấy Uvicorn lặp lại lỗi
> `Error: Invalid value for '--port': '$PORT' is not a valid integer.` Tôi đối
> chiếu log với `railway.toml` và phát hiện `startCommand` đang truyền `$PORT`
> theo exec form nên biến không được shell mở rộng. Tôi xóa override này để
> Railway dùng `CMD` trong Dockerfile; lệnh đó chạy qua `sh -c` và dùng
> `${PORT:-8000}`, vì vậy nhận đúng cổng Railway cấp rồi healthcheck `/health`
> có thể kết nối.
