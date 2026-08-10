# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: ..........................  Mã học viên: ..........................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> nếu có giá trị mặc định, người dùng có thể gửi request mà không cần token, khi đó app tự động thêm api mặc định vào, khiến ai cũng có thể truy cập tài nguyên.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> HTTP/1.1 401 Unauthorized
>Date: Mon, 10 Aug 2026 10:21:32 GMT
>Content-Type: application/json
>Transfer-Encoding: chunked
>Connection: keep-alive
>rndr-id: 5992a498-448b-444e
>Server: cloudflare
>vary: Accept-Encoding
>www-authenticate: Bearer
>x-render-origin-server: uvicorn
>cf-cache-status: DYNAMIC
>CF-RAY: a28e46365e63cdee-SIN
>alt-svc: h3=":443"; ma=86400

>{"detail":"invalid or missing bearer token"}

>Trong log trả về có thể biết request đang lỗi gì, thông qua mã lỗi, details lỗi có thể biết được server bị lỗi nếu có mã lỗi 500, server thực hiện load balance: cloudflare

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 446 MB |
| Multi-stage | 70.9 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản 1 stage chứa cả base image Python đầy đủ và toàn bộ phần filesystem không cần cho runtime. Bản multi-stage dùng `python:3.11-slim`, chỉ copy virtualenv đã cài dependency cùng source cần chạy; các file build và phần dư của base image builder không đi vào image cuối. Vì vậy image nhỏ hơn, tải nhanh hơn và giảm bề mặt tấn công.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, layer `COPY requirements.txt` và layer `pip install` vẫn được dùng lại từ cache. Các layer copy source phía sau phải chạy lại, rồi image runtime được tạo lại. Nếu đặt `COPY . .` trước `RUN pip install`, thay đổi một dòng code cũng làm layer copy bị đổi, khiến Docker cài lại toàn bộ dependency và build lâu hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng cho phép kẻ tấn công thực thi mã trong process Python. Nếu process chạy bằng root, mã đó có thể đọc/ghi nhiều file, cài thêm công cụ hoặc khai thác tiếp các quyền của container. `USER app` chuyển process sang user thường, nên dù ứng dụng bị chiếm quyền, kẻ tấn công chỉ có quyền giới hạn trong phạm vi được cấp cho app.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> `WWW-Authenticate: Bearer` là header chuẩn HTTP cho biết client phải gửi token theo scheme Bearer khi thử lại. Dùng cùng một lỗi cho thiếu header, sai scheme và sai token giúp tránh tiết lộ token có đúng hay không, khiến việc dò từng phần của thông tin xác thực khó hơn và giữ response nhất quán.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Xô chỉ chứa tối đa 10 token, nên sau 10 phút im lặng client vẫn chỉ có 10 token. Nó gửi được 10 request liên tiếp; request thứ 11 nhận 429. Nếu bỏ `min(capacity, ...)`, 10 phút sẽ nạp thêm 100 token (10 token/phút), cho phép khoảng 100 request sau một lần im lặng, phá vỡ giới hạn burst đã đặt ra.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Hạn mức 30 USD/tháng cho phép sự cố tiêu gần hết 30 USD trước khi bị chặn và chỉ hồi phục khi sang tháng mới. Hạn mức 1 USD/ngày giới hạn thiệt hại của ngày đó ở khoảng 1 USD và tự hồi phục lúc bắt đầu ngày UTC tiếp theo. Hạn mức ngày bảo vệ tốt hơn trước các sự cố gọi LLM liên tục.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Khi Redis mất kết nối, cả ba container đều làm health check thất bại nếu endpoint chung kiểm tra Redis. Load balancer đánh dấu cả ba instance không khỏe, rồi orchestrator lần lượt restart chúng. Trong lúc Redis vẫn hỏng, các instance mới cũng tiếp tục fail và tạo restart loop. Tách hai endpoint giúp `/healthz` chỉ kiểm tra process còn sống, còn `/readyz` trả 503 để ngừng nhận traffic khi Redis lỗi.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Em đã để biến redis url là link localhost, khi chạy thật, post 1 api /chat nhận về mã lỗi 500 do không tồn tại 1 redis database thật sự nào cả. Em đã thêm link redis add-on của render vào.
