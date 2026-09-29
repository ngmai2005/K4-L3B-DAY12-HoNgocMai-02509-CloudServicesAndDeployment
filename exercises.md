# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay các dòng placeholder câu trả lời bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Hồ Ngọc Mai  Mã học viên: 02509

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để giá trị mặc định là `"changeme"`, khi deploy lên cloud mà người phát triển quên cấu hình biến môi trường `AGENT_API_KEY`, ứng dụng vẫn sẽ khởi động thành công và mở cổng ra Internet. Khi đó, bất kỳ bot quét mạng hoặc kẻ tấn công nào cũng có thể thử các giá trị mặc định phổ biến (như `"changeme"`) và gọi endpoint `/ask` miễn phí, làm cạn kiệt ngân sách hoặc gây phát sinh chi phí LLM khổng lồ mà ta chỉ phát hiện khi nhận hóa đơn vào cuối tháng. Ngược lại, với cơ chế "fail fast" (không đặt giá trị mặc định), Pydantic sẽ ném `ValidationError` ngay lúc khởi động, khiến container dừng ngay trong quá trình deploy/healthcheck khi lập trình viên vẫn đang theo dõi log deployment. Nhờ vậy, ta phát hiện và bổ sung secret ngay lập tức trước khi có bất kỳ traffic nguy hại nào tiếp cận hệ thống.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T02:30:48.123456+00:00", "user_id": "sv-123", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.00015}`

Hai việc làm được với dòng log JSON mà `print()` không làm được:
1. Truy vấn, tổng hợp và phân tích định lượng trên các hệ thống thu thập log tập trung (Datadog, Loki, CloudWatch): ví dụ lọc chính xác các log có `user_id = "sv-123"`, tính tổng chi phí `cost_usd` theo từng user trong ngày, hoặc thống kê số token tiêu thụ trung bình của từng request.
2. Thiết lập hệ thống cảnh báo tự động (alerting) dựa trên điều kiện giá trị trường: ví dụ kích hoạt cảnh báo tới Slack/PagerDuty khi `cost_usd > 0.05` ở một request đơn lẻ hoặc khi tỷ lệ log có `level == "error"` vượt quá 5% trong 5 phút. Với `print("đã trả lời xong")`, log không có cấu trúc nên không thể bóc tách trường hay tính toán tự động.

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
| 1 stage (bản đầu) | 1.02 GB |
| Multi-stage | 185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~835 MB) bao gồm:
1. Base image: Bản đầu dùng `python:3.11` đầy đủ chứa toàn bộ hệ điều hành Debian với các trình biên dịch (gcc, g++, make), công cụ build, file header C/C++ và các gói tiện ích không cần thiết cho runtime. Bản multi-stage dùng `python:3.11-slim` chỉ chứa Linux runtime tối thiểu.
2. Build artifacts & pip cache: Ở bản multi-stage, các công cụ biên dịch và thư mục cache tải về của pip (`~/.cache/pip`), file tạm thời chỉ nằm lại ở stage `builder` và bị loại bỏ; stage runtime cuối cùng chỉ copy kết quả các package Python đã cài đặt hoàn chỉnh từ `/install` sang `/usr/local`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại:
Các layer từ đầu cho đến trước lệnh `COPY app ./app` (bao gồm `FROM python:3.11-slim`, `COPY requirements.txt`, `RUN pip install`, `COPY --from=builder /install`) hoàn toàn không thay đổi nên Docker tái sử dụng cache (CACHED). Chỉ có layer `COPY app ./app` và các layer kế tiếp (`COPY utils`, `USER`, `CMD`) phải chạy lại, quá trình build chỉ mất 1-2 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`:
Khi sửa bất kỳ ký tự nào trong code, layer `COPY . .` sẽ bị mất cache, kéo theo toàn bộ các layer phía sau nó (bao gồm lệnh `RUN pip install`) bị vô hiệu hóa cache và phải chạy lại từ đầu. Docker sẽ phải tải và cài lại toàn bộ thư viện từ Internet, làm thời gian build kéo dài thêm vài chục giây đến vài phút mỗi lần sửa code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện:
1. Ứng dụng Python xuất hiện lỗ hổng bảo mật (ví dụ: Command Injection, Deserialization không an toàn, hoặc SSRF).
2. Kẻ tấn công khai thác lỗ hổng để thực thi mã shell tùy ý. Vì container chạy mặc định bằng root, tiến trình mã độc sở hữu UID 0 (root) trong container.
3. Kẻ tấn công lợi dụng các lỗ hổng container breakout (như kernel exploit, lỗi cấu hình mount volume nhạy cảm `/var/run/docker.sock` hoặc cgroups) để thoát ra ngoài máy host. Do UID 0 bên trong container thường tương ứng với UID 0 (root) trên Linux host, kẻ tấn công chiếm toàn quyền kiểm soát máy host.
- Lệnh `USER appuser` cắt đứt chuỗi ở bước 2:
Khi chuyển sang chạy dưới user không có đặc quyền (`appuser`, UID 10001), tiến trình bị giới hạn quyền truy cập hệ thống file, không thể tương tác với docker daemon socket hay sửa đổi file nhị phân của OS. Kẻ tấn công dù chạy được code cũng chỉ có quyền của `appuser`, không thể thực hiện các thao tác đòi hỏi đặc quyền để breakout ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa 20 request trong 2 giây liên tiếp.
Giải thích:
Với cơ chế đếm theo phút đồng hồ (fixed window reset tại giây 00):
- Ở giây 10:00:59 (cuối phút thứ nhất), người dùng gửi liên tiếp 10 request. Hệ thống ghi nhận 10 request trong phút 10:00 (vừa đủ hạn mức 10 req/phút).
- Ngay sau đó 1 giây, tại 10:01:00 (đầu phút thứ hai), bộ đếm được reset về 0. Người dùng gửi tiếp 10 request nữa. Hệ thống ghi nhận 10 request trong phút 10:01 (vẫn hợp lệ).
Tổng cộng từ 10:00:59 đến 10:01:01 (khoảng thời gian 2 giây), hệ thống phải hứng chịu tới 20 request (gấp đôi hạn mức). Thuật toán sliding window khắc phục được kẽ hở này nhờ tính tổng request trong đúng 60 giây gần nhất tính từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Khác biệt:
Rate limit bảo vệ tài nguyên hạ tầng và độ ổn định dịch vụ bằng cách giới hạn tần suất gọi API trong khoảng thời gian ngắn (ví dụ 10 request/phút) để chống nghẽn mạng và từ chối dịch vụ. Cost guard bảo vệ tài chính của chủ sở hữu bằng cách kiểm soát ngân sách tích lũy trong thời gian dài (ví dụ tối đa 10 USD/tháng) để tránh cạn kiệt ngân sách do token LLM.
- Rate limit cho qua nhưng Cost guard chặn:
Một user gửi chỉ 1 request trong 10 phút (rất thấp so với hạn mức 10 req/phút), nhưng tổng chi phí của user đó trong tháng đã chạm ngưỡng 10.0 USD. Rate limit cho phép request đi qua, nhưng Cost guard chặn lại với mã lỗi 402 Payment Required.
- Cost guard cho qua nhưng Rate limit chặn:
Một user mới đầu tháng có số dư nguyên vẹn 10.0 USD, nhưng chạy script gửi liên tục 15 request chỉ trong 3 giây. Mặc dù tổng chi phí ước tính còn rất xa mới chạm ngưỡng ngân sách tháng, Rate limit sẽ chặn từ request thứ 11 với mã lỗi 429 Too Many Requests để tránh nghẽn server.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Redis gặp sự cố mạng hoặc khởi động lại, tạm thời mất kết nối trong 30 giây.
2. Endpoint gộp kiểm tra thấy Redis không phản hồi nên đồng loạt trả về 503 cho cả 3 container.
3. Bộ điều phối (Docker Swarm/Kubernetes/Cloud platform) sử dụng endpoint này làm Liveness check, thấy 503 nên kết luận container đã hỏng và tiến hành restart đồng loạt cả 3 container.
4. Cả 3 container khởi động lại nhưng Redis vẫn đang trong 30 giây gián đoạn, khiến health check tiếp tục fail và orchestrator lại restart chúng tiếp (vòng lặp CrashLoopBackOff).
5. Toàn bộ cụm dịch vụ sập hoàn toàn (cascading failure), không còn container nào phục vụ traffic. Trong khi đó, nếu tách riêng: `/health` (liveness) vẫn trả 200 để giữ container sống, còn `/ready` (readiness) trả 503 để load balancer tạm thời không đẩy traffic vào, và ngay khi Redis hồi phục, hệ thống lập tức trở lại hoạt động bình thường mà không cần restart container.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lưu lịch sử trong một dict Python nội bộ của process (trong RAM):
Con số `history_length` trong response sẽ nhảy lộn xộn, không tăng liên tục và biến thiên ngẫu nhiên theo container nhận request. Ví dụ:
- Lần 1: request vào instance A -> `history_length` = 0.
- Lần 2: load balancer chuyển sang instance B -> `history_length` = 0 (vì RAM của B chưa có lịch sử).
- Lần 3: request vào instance C -> `history_length` = 0.
- Lần 4: request quay lại instance A -> `history_length` = 2.
Người dùng sẽ thấy câu trả lời của agent bị mất ngữ cảnh ("mất trí nhớ") tùy thuộc vào việc request rơi trúng container nào. Khi dùng Redis chung, tất cả instance đều truy xuất cùng một kho dữ liệu nên `history_length` tăng đều đặn 0 -> 2 -> 4 -> 6...

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Tên lỗi: Health check timeout do service không nhận biến môi trường `$PORT` của cloud platform.
- Thông báo lỗi: `Deploy failed: Container failed to respond to health checks on port 8000. Timed out waiting for application to start listening.`
- Cách tìm nguyên nhân: Kiểm tra runtime log trên dashboard của platform, nhận thấy uvicorn log dòng `Uvicorn running on http://0.0.0.0:8000`, trong khi platform cloud cấp phát một cổng động ngẫu nhiên qua biến môi trường `$PORT` (ví dụ `PORT=35421`). Vì Dockerfile ban đầu hardcode lệnh `["uvicorn", "app.main:app", "--port", "8000"]` nên app không lắng nghe trên cổng mà platform định tuyến tới.
- Cách sửa: Cập nhật lệnh `CMD` trong Dockerfile sang định dạng shell để lấy cổng động từ biến môi trường: `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`. Sau khi deploy lại, container đọc đúng biến `$PORT` được cấp phát và health check pass ngay lập tức.
