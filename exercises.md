# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay các dòng placeholder bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyen Minh Kiet  Mã học viên: 2A202602373

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để mặc định `changeme`, app vẫn khởi động khi deploy thiếu secret và người khác có thể đoán được khóa để gọi API. Với trường bắt buộc, Pydantic báo lỗi ngay lúc khởi động; lỗi được phát hiện trước khi service nhận traffic.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Log JSON có thể được hệ thống log lọc theo `event`, `level`, `timestamp` và `user_id`; đồng thời có thể thống kê chi phí qua `cost_usd`. `print("đã trả lời xong")` không có schema ổn định nên khó tìm theo user, khó tính tổng chi phí và khó tạo cảnh báo.

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
| 1 stage (bản đầu) | Chưa build trong phiên kiểm tra này |
| Multi-stage | Build thành công; test image xác nhận dưới 500 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Mình chưa có số đo single-stage từ phiên lab này nên không tự điền một con số giả. Multi-stage chỉ mang dependency runtime và source cần thiết vào stage cuối, còn compiler/cache của stage builder không đi theo nên image cuối nhỏ hơn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi chỉ sửa `app/main.py`, layer cài dependency trong builder vẫn được dùng lại vì `requirements.txt` không đổi. Các layer từ lúc copy source trở đi phải tạo lại và image runtime được export lại. Nếu `COPY . .` đặt trước `RUN pip install`, mọi thay đổi source sẽ làm layer trước đó thay đổi, khiến Docker phải cài dependency lại và build chậm hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Nếu Python app bị khai thác và container chạy bằng root, tiến trình bị khai thác có quyền cao trong container và có thể tìm cách tác động tới Docker socket hoặc dữ liệu được mount, làm tăng hậu quả trên host. `USER appuser` cắt chuỗi ở bước process trong container: app không còn chạy với quyền root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Với giới hạn 10 request/phút, cửa sổ trượt luôn xét 60 giây gần nhất. Cách đếm theo phút đồng hồ có thể cho phép 10 request lúc 10:00:59 và 10 request lúc 10:01:01, tức 20 request trong 2 giây. Sliding window xóa timestamp cũ, đếm request còn lại rồi mới ghi request mới.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn tần suất, còn cost guard giới hạn tổng tiền theo user và tháng UTC. User gửi một request khi ngân sách còn 0 có thể qua rate limit nhưng bị cost guard trả 402. Ngược lại, user còn ngân sách nhưng gửi request thứ 11 trong 60 giây sẽ bị rate limit trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu gộp hai endpoint và `/health` kiểm tra Redis, khi Redis mất 30 giây cả ba container có thể bị đánh dấu unhealthy và restart dù process agent vẫn sống. Tách ra giúp `/health` tiếp tục trả liveness, còn `/ready` trả 503 để ngừng gửi traffic trong thời gian Redis lỗi.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với Redis List dùng chung, mọi container đọc cùng `history:<user_id>` nên `history_length` tăng nhất quán dù request đi vào instance nào. Nếu dùng dict Python, mỗi container có một bản riêng; request luân phiên giữa ba instance sẽ thấy lịch sử ngắn hơn hoặc khác nhau.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Khi deploy Railway, `/ready` ban đầu trả 500. Log cho thấy `redis.from_url()` nhận URL không có scheme hợp lệ vì biến tham chiếu nhầm tới `day12-redis.DATABASE_URL`, trong khi Redis service cung cấp `REDIS_URL`. Mình đổi reference thành `${{day12-redis.REDIS_URL}}`, Railway redeploy, rồi `/ready` trả 200 với `redis: true`.
