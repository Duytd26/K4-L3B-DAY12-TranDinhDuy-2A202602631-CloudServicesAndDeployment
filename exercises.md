# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Đình Duy  Mã học viên: 2A202602631

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để mặc định `changeme`, service vẫn khởi động được nhưng mọi request
đều có thể bị chấp nhận hoặc bị hiểu sai là cấu hình hợp lệ. Khi thiếu biến
thật, app dừng ngay từ lúc khởi động nên tôi phát hiện lỗi cấu hình trước khi
triển khai, thay vì để service chạy rồi mới bị lộ khóa hoặc bị người khác
gọi nhầm vào môi trường thật.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Một dòng log JSON điển hình cho tôi biết thời điểm, mức log, user nào gọi,
số token vào/ra và chi phí phát sinh. Từ đó tôi có thể lọc theo `user_id`,
đếm tần suất lỗi, và tự động gửi log sang dashboard hay hệ thống cảnh báo.
`print("đã trả lời xong")` chỉ cho thấy chương trình chạy tới đâu, không đủ
thông tin để phân tích hoặc truy vết sự cố.

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
| 1 stage (bản đầu) |1.7 GB  |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Sau khi tối ưu Dockerfile bằng multi-stage build, dung lượng Docker image giảm từ khoảng 1.7 GB xuống 271 MB, tương đương giảm khoảng 84%. Việc tách build dependencies khỏi runtime image giúp image triển khai nhỏ gọn hơn, giảm các thành phần không cần thiết trong production.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi sửa một ký tự trong `app/main.py`, các layer phía sau `COPY . .` phải build
lại, nhưng layer cài dependency vẫn được cache nếu `requirements.txt` không đổi.
Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi code đều làm mất cache
ở bước cài đặt, khiến build chậm hơn vì pip chạy lại không cần thiết.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Nếu container chạy root, một lỗ hổng cho phép thoát khỏi ứng dụng có thể dẫn
đến quyền root trong container và tăng khả năng chạm vào filesystem, socket
hoặc mount của host. `USER` chuyển tiến trình sang user không đặc quyền, nên
ngay cả khi ứng dụng bị chiếm quyền, kẻ tấn công vẫn bị giới hạn đáng kể.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Sliding window đếm 60 giây gần nhất nên mỗi request đều bị tính theo thời gian
thực, không phụ thuộc ranh giới phút. Nếu đếm theo phút đồng hồ, người dùng có
thể gửi 10 request ở cuối phút cũ và 10 request ngay đầu phút mới, tức 20
request trong 2 giây liên tiếp khi hạn mức chỉ là 10/phút.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit chặn theo tần suất request, còn cost guard chặn theo số tiền tích
lũy trong tháng. Ví dụ user gửi rất ít request nhưng mỗi request tốn chi phí
lớn thì rate limit cho qua, cost guard phải chặn. Ngược lại, user spam request
nhỏ và rẻ thì cost guard có thể chưa chặn nhưng rate limit phải chặn trước.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu gộp `/health` và `/ready`, khi Redis mất 30 giây thì toàn bộ instance sẽ
trả lỗi, load balancer coi cả cụm là chết và ngừng gửi traffic, dù bản thân web
server vẫn còn chạy được. Tách `/health` và `/ready` giúp hệ thống chỉ báo chết
khi tiến trình thật sự không sống, còn mất Redis thì chỉ đánh dấu chưa sẵn sàng.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Khi chạy nhiều replica với Redis, cùng một `X-User-Id` thì `history_length`
vẫn tăng đều vì lịch sử nằm chung một kho. Nếu dùng dict Python trong bộ nhớ,
mỗi container sẽ có lịch sử riêng, nên gọi qua nhiều replica sẽ thấy số này
nhảy không ổn định hoặc quay về 0 tùy instance nhận request.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Một lỗi tôi có đã gặp khi deploy là readiness trả 503 do `REDIS_URL` chưa
đúng hoặc Redis add-on chưa sẵn sàng. Tôi kiểm tra log platform, gọi trực tiếp
`/ready`, đối chiếu biến môi trường, rồi sửa URL kết nối hoặc tạo lại Redis
trước khi redeploy.
