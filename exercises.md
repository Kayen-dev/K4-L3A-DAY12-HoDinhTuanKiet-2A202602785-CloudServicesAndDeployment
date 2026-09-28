# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng giữ chỗ bên dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Hồ Đình Tuấn Kiệt  Mã học viên: 2A202602785

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Railway, nếu tôi quên khai báo `AGENT_API_KEY`, `Settings`
> phát sinh `ValidationError` ngay lúc process khởi động và healthcheck không thể
> pass. Nhờ vậy tôi biết cấu hình cloud đang thiếu secret trước khi nhận
> request. Nếu dùng mặc định `"changeme"`, service vẫn báo healthy và người
> ngoài có thể đoán khóa này để gọi API, làm lộ dữ liệu hoặc tiêu tốn
> ngân sách.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật tôi thu được:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:01:44.696828+00:00", "user_id": "sv01", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}`.
> Với log có cấu trúc, hệ thống có thể (1) lọc/tìm tất cả sự kiện
> `ask_completed` của `sv01`, và (2) cộng `cost_usd` theo user hoặc theo khoảng
> thời gian để làm dashboard/cảnh báo. Chuỗi `print("đã trả lời xong")`
> không mang các trường định danh và số liệu đó.

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
| 1 stage (bản đầu) | 1.73 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build lại Dockerfile ban đầu từ commit `306b897` và đo bằng
> `docker images`: bản một stage là 1.73 GB, bản multi-stage hiện tại là
> 271 MB. Phần chênh lệch chủ yếu đến từ base image `python:3.11` đầy đủ
> (có nhiều gói hệ thống/công cụ build) so với `python:3.11-slim`. Runtime
> multi-stage chỉ nhận dependency đã cài từ builder và mã cần chạy, không mang
> toàn bộ build context và công cụ không cần thiết sang production.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Sau khi chỉ sửa `app/main.py`, các layer base image, `COPY requirements.txt`,
> `pip install`, tạo user và copy dependency từ builder đều dùng cache; Docker chỉ
> chạy lại layer `COPY app ./app` và các layer runtime phía sau bị ảnh hưởng.
> Nếu đặt `COPY . .` trước `RUN pip install`, bất kỳ thay đổi nào trong
> source cũng làm mất cache của layer cài dependency, khiến pip phải chạy lại
> dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi tấn công có thể là: lỗ hổng Python cho phép thực thi mã từ xa
> → kẻ tấn công mở shell trong container → nếu process chạy root thì shell cũng
> có quyền root trong container → có thể sửa file hệ thống/volume được mount,
> và tiếp tục khai thác lỗ hổng kernel hoặc socket Docker nếu chúng bị lộ để
> chiếm host. `USER 10001:10001` cắt chuỗi ngay sau bước RCE: shell chỉ có
> quyền của user thường, giảm đáng kể phạm vi ghi file và khả năng leo thang.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa là 20 request trong 2 giây: gửi 10 request ở giây 59 của phút
> trước, sau đó gửi thêm 10 request ở giây 00 của phút sau. Bộ đếm theo
> phút đã reset tại ranh giới nên cả hai nhóm đều hợp lệ, trong khi sliding
> window 60 giây vẫn nhìn thấy nhóm đầu và sẽ chặn nhóm sau.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tần suất trong một cửa sổ thời gian, còn cost guard giới
> hạn tổng tiền theo user trong tháng. Ví dụ, 5 request rất dài, mỗi request
> dùng hàng chục nghìn token: rate limit 10/phút cho qua nhưng cost guard phải
> chặn khi chạm ngân sách. Ngược lại, 11 request rất ngắn gửi liên tiếp có
> chi phí thấp hơn nhiều so với ngân sách, nhưng request thứ 11 bị rate limit chặn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Trình tự khi gộp hai probe là: Redis mất kết nối → probe của cả ba
> container đều thất bại → orchestrator hiểu nhầm process bị hỏng và restart
> cả ba container → Redis vẫn chưa phục hồi nên các container mới lại fail,
> tạo restart storm → không còn instance nào phục vụ. Khi tách ra, `/health`
> vẫn 200 nên process không bị restart; `/ready` trả 503 để load balancer tạm
> rút instance khỏi traffic cho đến khi Redis hoạt động lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Compose hiện map cố định `8000:8000`, nên thử scale trực tiếp ba replica
> trên cùng host sẽ xung đột cổng; cần Nginx hoặc bỏ host port của từng
> replica để quan sát load balancing thật. Tôi đã xác minh tính stateless qua
> test CP4: hai `ConversationStore` khác nhau dùng chung Redis vẫn đọc cùng lịch sử.
> Vì mỗi lượt `/ask` ghi hai message, `history_length` trước khi ghi sẽ tăng
> 0, 2, 4, 6... bất kể instance nào nhận request. Nếu dùng dict riêng trong
> từng process, số này sẽ nhảy hoặc lặp lại theo replica, ví dụ 0, 0, 2,
> 0, 2, thay vì tăng liên tục.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi tôi gặp là Railway báo `Healthcheck failure` trong bước Network. Tôi
> kiểm tra `/health`, log deploy, commit mà Railway đang build và so sánh với code
> local; nguyên nhân là bản CP1–CP4 hoàn chỉnh chưa được commit/push, nên cloud
> vẫn build code khung cũ. Tôi commit/push bản sửa (`884b44d`), bảo đảm Uvicorn
> bind `0.0.0.0`, đọc `${PORT:-8000}`, và khai báo `AGENT_API_KEY` trên Railway.
> Sau khi redeploy, `/health` trả 200. Hiện còn một lỗi riêng cần sửa:
> `/ready` trả 500 do `REDIS_URL` của service agent trên Railway chưa đúng.
