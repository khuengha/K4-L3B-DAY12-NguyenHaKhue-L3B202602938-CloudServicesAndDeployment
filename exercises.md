# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Hà Khuê  Mã học viên: 2A202602938

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Mình deploy lên Railway và quên set biến `AGENT_API_KEY` trong dashboard. Nếu
`agent_api_key` không có mặc định, app chết ngay lúc khởi động với lỗi
`ValidationError` hiện rõ trong log deploy. Nếu để mặc định `"changeme"`, app chạy
ngon như thể không có chuyện gì, mình cứ tưởng deploy thành công. Sau đó ai
đó quét được URL công khai, thử key `"changeme"` và gọi `/ask` miễn phí bằng
tiền OpenAI của mình, mình chỉ phát hiện ra khi nhìn hóa đơn cuối tháng.
"Chết sớm" biến một lỗi bảo mật âm thầm thành một lỗi thấy ngay lúc deploy.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log mình thu được khi gọi `/ask`:

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:29:35.724780+00:00", "user_id": "sv-ex4", "tokens_in": 4, "tokens_out": 38, "cost_usd": 2.34e-05}
```

Hai việc mà `print("đã trả lời xong")` không làm được:

1. **Lọc/truy vấn theo trường**: vì mỗi dòng là một JSON object có khóa rõ ràng,
   mình có thể lọc `user_id = "sv-ex4"` để xem một user đã hỏi gì, hoặc cộng
   dồn `cost_usd` để biết tổng tiền đã chi — chuỗi văn bản tự do thì không có
   "trường" nào để lọc cả.
2. **Cảnh báo tự động**: các hệ log trên cloud (Railway, Datadog...) parse được
   JSON nên mình đặt được alert kiểu "gửi Slack khi `level` = `error`" hoặc
   "báo động khi `cost_usd` > 0.01 trong một request". Với `print`, máy đọc
   không hiểu nội dung dòng chữ nên không thể tự động hóa được gì.

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
| 1 stage (bản đầu) | ~1,1 GB |
| Multi-stage | 271 MB |

Phần chênh lệch (~830 MB) là toàn bộ những thứ chỉ cần lúc **build** chứ không
cần lúc **chạy**: compiler/toolchain (`build-essential`, `gcc`... nếu có), cache
pip và các file tạm phát sinh trong lúc cài package. Bản 1-stage mang cả image
đầy đủ đó đi tới production. Bản multi-stage tách làm hai: stage `builder`
cài package vào `/install`, rồi stage `runtime` chỉ `COPY --from=builder
/install /usr/local` — tức là chỉ mang kết quả cài đặt (thư viện đã biên dịch
xong) sang, bỏ lại toàn bộ công cụ. Image nhỏ thì push/pull nhanh hơn (deploy
nhanh hơn) và giảm bề mặt tấn công vì ít phần mềm thừa trong container.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi mình sửa một ký tự trong `app/main.py` rồi build lại, Docker kiểm tra lần
lượt: các layer trước như `FROM`, `WORKDIR`, `COPY requirements.txt` và
`RUN pip install` vẫn dùng lại từ cache vì `requirements.txt` không đổi —
nhưng layer `COPY app ./app` thấy file `main.py` đã đổi nên phải chạy lại, và
mọi layer sau nó cũng chạy lại. Nhờ vậy build chỉ mất vài giây thay vì cài lại
toàn bộ thư viện.

Nếu đặt `COPY . .` lên trước `RUN pip install`: chỉ cần sửa một ký tự trong
`app/main.py` là layer `COPY . .` bị vô hiệu hóa, và vì Docker vô hiệu hóa
mọi layer sau một layer đã đổi nên `RUN pip install` cũng phải chạy lại —
nghĩa là mỗi lần sửa code dù nhỏ nhất cũng cài lại toàn bộ dependencies,
build chậm đi rất nhiều. Đó là lý do thứ tự đúng phải là copy
`requirements.txt` trước, `pip install`, rồi mới copy code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện: (1) code Python của mình có lỗ hổng, ví dụ cho phép thực thi
lệnh từ input của người dùng → (2) kẻ tấn công thực thi được lệnh **bên trong
container** → (3) container mặc định chạy bằng root nên lệnh đó cũng chạy với
quyền root trong container → (4) nếu có thêm một lỗi phân ly container
(container escape) trong Docker/kernel, kẻ tấn công leo từ root trong container
lên host và có quyền cao trên máy thật, nơi chạy cả database và các container
khác. Lệnh `USER appuser` (uid 10001) cắt đứt chuỗi ngay ở bước (3): dù bước
(2) xảy ra, kẻ tấn công chỉ là user thường — không ghi được file hệ thống,
không cài được gì, không sửa được cấu hình — nên bước (4) khó đi tới hơn nhiều.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa **20 request trong 2 giây** liên tiếp. Cách đạt: người dùng gửi 10
request lúc 10:00:59 — vẫn thuộc cửa sổ "phút 10:00", đúng hạn mức; chờ 2 giây
sang 10:01:01, đồng hồ vừa lật sang phút mới nên bộ đếm reset về 0, họ lại gửi
được 10 request nữa thuộc cửa sổ "phút 10:01". Nhìn vào từng phút thì ai cũng
chỉ dùng đúng 10/phút, không vi phạm gì, nhưng thực tế là 20 request chỉ trong
2 giây. Sliding window 60 giây không có kẽ hở này vì nó luôn đếm "số request
trong 60 giây gần nhất" liên tục — request thứ 11 trong bất kỳ khoảng 60 giây
nào đều bị từ chối, không có mốc reset để chui qua.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác nhau cốt lõi: rate limit đo **tần suất** — số request trong một cửa sổ
60 giây của từng user; cost guard đo **số tiền** — tổng `cost_usd` tích lũy
trong tháng so với ngân sách `monthly_budget_usd`. Một cái bảo vệ service khỏi
spam, cái kia bảo vệ túi tiền khỏi hóa đơn bất ngờ.

- Rate limit cho qua nhưng cost guard chặn: đầu phút mới, user `sv01` mới gửi
  request đầu tiên — còn 9 lượt nữa mới tới hạn — nhưng ngân sách tháng đã cạn
  ($ spent >= budget). Request hoàn toàn hợp lệ về tần suất nhưng bị cost guard
  trả 402 vì cứ xử lý là tốn thêm tiền.
- Ngược lại: đầu tháng, ngân sách còn đầy, user `sv02` gửi liên tục những
  request nhỏ rẻ. Cost guard không hề báo động vì tổng tiền còn rất ít, nhưng
  khi user gửi request thứ 11 trong cùng 60 giây thì rate limit chặn bằng 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện khi gộp hai endpoint làm một (và endpoint đó kiểm tra Redis):

1. Redis mất kết nối 30 giây → cả 3 container cùng trả 503 cho endpoint gộp.
2. Orchestrator dùng endpoint đó làm cả liveness lẫn readiness, nên nó hiểu
   503 là "process chết" → khởi động lại (restart) cả 3 container cùng lúc.
3. User đang giữa request thấy lỗi 500/502 hàng loạt dù bản thân code app
   không có gì hỏng — chỉ là Redis chập.
4. Container vừa restart thì Redis vẫn chưa về → container mới cũng trả 503 →
   lại bị restart → rơi vào crash loop.
5. Kết quả: một sự cố nhỏ của Redis (30 giây) leo thang thành sập toàn cụm.

Đó là lý do hai endpoint phải tách: `/health` (liveness) chỉ nói "process còn
sống", không đụng tới Redis — Redis chết thì container vẫn sống, không bị
restart oan; `/ready` (readiness) kiểm tra Redis để orchestrator tạm ngừng đẩy
traffic tới container, và khi Redis quay lại là nhận traffic ngay, không cần
restart gì cả.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với Redis làm store chung, gọi `/ask` 5 lần với cùng `X-User-Id`, dù request
rơi vào container nào (Docker tự chia round-robin giữa 3 container) thì
`history_length` vẫn tăng đều: 0, 2, 4, 6, 8 — vì cả 3 container cùng đọc/ghi
lịch sử vào đúng một chỗ là Redis. Container không giữ trạng thái gì trong bộ
nhớ của nó (stateless), nên có thể nhân bản thoải mái.

Nếu lịch sử lưu trong một dict Python trong process, thì mỗi container có dict
riêng. Request 1 rơi vào container A → A nhớ câu hỏi; request 2 rơi vào
container B → B không biết gì về câu 1, lịch sử trống. Nhìn từ ngoài,
`history_length` nhảy loạn kiểu 0, 2, 0, 2, 0 — không tăng đều nữa — và agent
"mất trí nhớ" ngẫu nhiên tùy container nào nhận request. Đó chính là lý do
phải externalize state ra Redis khi scale nhiều bản sao.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi thật mình gặp khi deploy lên Railway: sau lần chạy `railway up` đầu tiên,
service Redis của mình rơi vào trạng thái **crash loop** — container restart
liên tục, và endpoint `/ready` của app trả 503 vì không kết nối được Redis.

- **Thông báo lỗi**: trong log của service Redis (xem bằng `railway logs
  --service Redis` hoặc tab Deployments trên dashboard) lặp đi lặp lại dòng:
  `exec: docker-entrypoint.sh: not found`.
- **Cách tìm nguyên nhân**: log cho thấy Redis đang cố chạy script khởi động
  chuẩn của image Redis nhưng không tìm thấy. Mình nhận ra nguyên nhân: lần
  `railway up` đầu tiên mình gõ thiếu cờ `--service agent`, nên Railway đẩy
  code của mình vào service **Redis** — Dockerfile của app (image Python) ghi
  đè lên image `redis:8.2` gốc. Image Python đương nhiên không có file
  `docker-entrypoint.sh` nên Redis không thể khởi động, restart mãi.
- **Cách sửa**: xóa service Redis đã hỏng, tạo lại bằng `railway add
  --database redis` (lần này Railway dựng đúng từ image Redis gốc), rồi set
  lại biến `REDIS_URL=${{Redis.REDIS_URL}}` cho service agent và redeploy.
  Sau đó `/ready` trả 200 với `redis: true`. Bài học: luôn chỉ định
  `--service` khi `railway up`, và đọc log của đúng service bị lỗi thay vì
  đoán.
