# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng trả lời mẫu bằng câu trả lời thực tế.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đàm Việt Hưng  Mã học viên: 2A202602600

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Một tình huống cụ thể là tôi deploy lên Render nhưng quên khai báo
> `AGENT_API_KEY`. Nếu khóa có mặc định `"changeme"`, service vẫn lên và URL
> công khai có thể bị người khác gọi bằng khóa dễ đoán, làm phát sinh chi phí.
> Khi trường này bắt buộc, Pydantic dừng app ngay lúc khởi động và log deploy chỉ
> rõ biến còn thiếu, nên lỗi được phát hiện trước khi service nhận traffic.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng tôi lấy từ container sau khi gọi `/ask` là:
>
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:54:29.385486+00:00", "user_id": "exercise-log", "tokens_in": 6, "tokens_out": 44, "cost_usd": 2.73e-05}`
>
> Từ dòng này tôi có thể lọc và đếm số request theo `event`, `user_id` hoặc
> khoảng thời gian; đồng thời có thể cộng `cost_usd`, theo dõi token và tạo cảnh
> báo chi phí. Một chuỗi `print("đã trả lời xong")` không có trường cố định để
> máy truy vấn và cũng không cho biết user, token hay chi phí.

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
| 1 stage (bản đầu) | 1.7 GB (khoảng 1700 MB) |
| Multi-stage | 265 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build bản một stage từ image `python:3.12` đầy đủ và đo được 1.7 GB; image
> production multi-stage dùng `python:3.12-slim` là 265 MB. Phần chênh lệch chủ
> yếu là base image đầy đủ, công cụ build và các thành phần hệ thống không cần
> khi chạy. Runtime stage chỉ giữ Python slim, dependency đã cài và source cần
> thiết, nên giảm khoảng 1.4 GB và giảm bề mặt tấn công.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại, `COPY requirements.txt` và layer `pip install` nằm
> trước `COPY app`/`COPY utils`. Khi chỉ sửa một ký tự trong `app/main.py`, các
> layer base image, requirements và cài dependency được lấy lại từ cache; chỉ
> layer copy source và các layer sau nó phải chạy lại. Nếu đặt `COPY . .` trước
> `RUN pip install`, mọi thay đổi source làm checksum của layer copy đổi, khiến
> Docker cài lại toàn bộ thư viện dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code Python có lỗ hổng cho phép thực thi lệnh, kẻ tấn công trước hết có
> shell trong container. Container chạy root khiến lệnh đó có UID 0 bên trong;
> kết hợp volume nhạy cảm, Docker socket, capability thừa hoặc lỗ hổng kernel,
> kẻ tấn công có thể sửa file được mount hay leo sang host với quyền cao. Lệnh
> `USER app` cắt chuỗi ở bước thực thi: tiến trình bị giới hạn ở user thường,
> không có quyền sửa file hệ thống hoặc dùng tài nguyên chỉ dành cho root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa 20 request trong 2 giây: gửi 10 request ngay
> trước khi phút cũ kết thúc, ví dụ 10:00:59, rồi gửi thêm 10 request ngay sau
> khi bộ đếm reset ở 10:01:00. Sliding window 60 giây nhìn lại toàn bộ 60 giây
> gần nhất nên vẫn thấy nhóm cũ và chặn burst này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tốc độ/số request trong cửa sổ ngắn, còn cost guard giới
> hạn tổng số tiền theo user trong cả tháng. Ví dụ user chỉ gửi một request khi
> đã tiêu 9.999 USD trên ngân sách 10 USD: rate limit cho qua nhưng cost guard
> phải chặn nếu request ước tính làm vượt ngân sách. Ngược lại, một user gửi 11
> request rất rẻ trong vài giây khi ngân sách còn nhiều: cost guard vẫn cho qua
> về tiền nhưng rate limit chặn request thứ 11 bằng 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp hai endpoint và bắt health kiểm tra Redis, khi Redis mất kết nối thì
> cả ba container cùng trả health lỗi. Orchestrator hiểu nhầm cả ba process đã
> chết và restart đồng loạt. Redis vẫn chưa phục hồi nên các container mới lại
> fail health, tạo vòng restart; load balancer không còn instance ổn định và
> user nhận 502/503. Tách riêng giúp `/health` vẫn 200 vì process còn sống, còn
> `/ready` trả 503 để tạm ngừng traffic; khi Redis trở lại, các instance sẵn
> sàng ngay mà không cần restart hàng loạt.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Trong kiểm tra nhiều instance dùng chung Redis, request đầu trả
> `history_length=0`, request sau thấy hai message trước đó và trả 2; các lượt
> tiếp theo tăng 4, 6... bất kể instance nào xử lý. Nếu dùng một dict Python,
> mỗi container có bản nhớ riêng: load balancer chuyển request sang container
> khác sẽ làm số liệu nhảy như 0, 0, 2 hoặc quay lại 0, thay vì tăng đều. Khi
> container restart, phần lịch sử trong dict cũng mất hoàn toàn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Một lỗi tôi gặp khi build để deploy là:
> `failed to fetch oauth token: lookup auth.docker.io: no such host` (lần trước
> đó còn có `context deadline exceeded`). Tôi đọc log BuildKit và thấy lỗi xảy
> ra ở bước lấy metadata `python:3.12-slim`, trước khi Docker chạy lệnh trong
> Dockerfile, nên nguyên nhân là DNS/kết nối Docker Hub chứ không phải code.
> Tôi kiểm tra Docker Engine và image cache, đợi kết nối DNS hoạt động rồi build
> lại. Lần build sau thành công; image production đo được 265 MB, Compose báo
> `agent` và `redis` healthy, còn Render trả `/health` và `/ready` đều 200.
