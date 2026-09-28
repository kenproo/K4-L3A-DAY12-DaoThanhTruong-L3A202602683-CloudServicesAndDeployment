# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đào Thành Trường  Mã học viên: L3A202602683

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống cụ thể: Khi deploy ứng dụng lên nền tảng cloud (như Render hoặc Railway), nhà phát triển có thể sơ suất quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard.
- Nếu để giá trị mặc định là `"changeme"`, ứng dụng vẫn khởi động thành công và mở cổng ra Internet. Các bot quét tự động trên mạng có thể nhanh chóng dò ra API key mặc định `"changeme"` và gửi hàng ngàn request vào endpoint `/ask`, làm tiêu tốn toàn bộ ngân sách LLM thật. Thậm chí người dùng hợp lệ khi gửi API key của họ sẽ nhận mã lỗi 401 khó hiểu, trong khi dev tưởng rằng hệ thống vẫn đang vận hành bình thường.
- Ngược lại, nhờ cơ chế "fail fast" (không đặt giá trị mặc định), `pydantic-settings` sẽ ném lỗi `ValidationError` ngay lúc nạp cấu hình và tiến trình dừng lại lập tức. Nền tảng cloud sẽ đánh dấu quá trình build/deploy thất bại ngay tại chỗ, ghi rõ lỗi thiếu `AGENT_API_KEY` vào log. Dev sẽ phát hiện và bổ sung key ngay trên dashboard trước khi service nhận bất kỳ traffic công khai nào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T12:58:14.281934+00:00", "user_id": "sv-123", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.000142}`

Hai việc làm được với dòng log JSON này mà `print("đã trả lời xong")` không thể làm được:
1. **Phân tích và tổng hợp số liệu tự động (Aggregation & Querying):** Các công cụ gom log tập trung (như Datadog, Grafana Loki, CloudWatch) có thể tự động parse các trường có cấu trúc JSON để thực hiện tính toán: ví dụ tính tổng chi phí `SUM(cost_usd)` theo từng `user_id` trong ngày, đo lường lượng token tiêu thụ trung bình `AVG(tokens_in + tokens_out)`, hoặc thống kê tần suất gọi API theo thời gian mà không cần phải viết biểu thức chính quy (regex) để bóc tách chuỗi thô.
2. **Thiết lập cảnh báo ngưỡng tự động (Automated Alerting):** Có thể định nghĩa trực tiếp các quy tắc giám sát hệ thống như: "Cảnh báo nếu bất kỳ sự kiện nào có `cost_usd > 0.05`" hoặc "Kích hoạt cảnh báo khi tỷ lệ bản ghi có `level == 'error'` vượt quá 5% trong 5 phút". Với `print()` thông thường, máy tính không thể hiểu ngữ nghĩa của thông điệp để tự động kích hoạt cảnh báo chính xác.

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
| 1 stage (bản đầu) | 1025 MB |
| Multi-stage | 198 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~827 MB) bao gồm:
1. **Sự khác biệt giữa base image Debian đầy đủ và slim:** Bản đầu tiên sử dụng `python:3.11` đầy đủ chứa trình biên dịch GCC, G++, Make, Git, các header files C/C++ và các gói tiện ích hệ thống nặng nề. Bản multi-stage sử dụng `python:3.11-slim` chỉ giữ lại các gói thư viện tối thiểu cần thiết để Python runtime hoạt động.
2. **Loại bỏ bộ nhớ đệm và build artifacts:** Ở stage `builder`, quá trình biên dịch các wheel và cache tải về của pip (`pip cache`) bị cô lập. Sang stage `runtime`, ta chỉ sao chép thư mục kết quả `/install` sang `/usr/local`, hoàn toàn không mang theo compiler, header files hay cache trung gian vào image cuối cùng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

1. Khi sửa một ký tự trong `app/main.py`:
- Các layer được dùng lại từ cache (`CACHED`):
  + Stage builder: `FROM python:3.11-slim AS builder`, `WORKDIR /app`, `COPY requirements.txt .`, `RUN pip install ...`
  + Stage runtime: `FROM python:3.11-slim AS runtime`, `WORKDIR /app`, `COPY --from=builder /install /usr/local`, `RUN useradd ...`
- Các layer phải chạy lại:
  + Từ lệnh `COPY app ./app` trở đi (do nội dung mã nguồn trong thư mục `app` đã thay đổi), kéo theo `COPY utils ./utils`, `USER appuser`, `HEALTHCHECK`, và `CMD`.

2. Nếu đặt `COPY . .` lên trước `RUN pip install`:
Do Docker lưu trữ cache theo từng layer tuần tự, khi có bất kỳ thay đổi nào trong thư mục làm việc, layer `COPY . .` sẽ bị mất cache (cache miss). Khi đó toàn bộ các layer tiếp theo, đặc biệt là `RUN pip install`, sẽ bị buộc phải tải lại và cài đặt lại toàn bộ các gói thư viện từ đầu, làm tăng thời gian build từ vài giây lên vài phút trong mỗi lần code thay đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

1. **Chuỗi sự kiện leo thang:**
- Bước 1: Ứng dụng Python tồn tại một lỗ hổng thực thi mã từ xa (RCE) — ví dụ qua việc deserialize dữ liệu không an toàn (`pickle`), gọi hàm `eval()`, hoặc lỗ hổng buffer overflow trong một thư viện C-extension.
- Bước 2: Kẻ tấn công gửi payload khai thác thành công và chiếm được quyền điều khiển shell trong container.
- Bước 3: Do container chạy mặc định bằng user `root` (UID 0), tiến trình bị chiếm giữ sở hữu toàn bộ quyền quản trị tối cao bên trong container namespace.
- Bước 4: Kẻ tấn công lợi dụng các lỗ hổng kernel Linux chưa được vá, lỗ hổng container breakout trong container runtime (như CVE-2024-21626 / runc escape) hoặc tìm thấy socket docker (`/var/run/docker.sock`) bị mount nhầm để thoát ra ngoài không gian container.
- Bước 5: Khi đã thoát ra ngoài host, do tiến trình ban đầu mang UID 0 (root), kẻ tấn công trở thành `root` trực tiếp trên hệ điều hành của máy host và kiểm soát toàn bộ máy chủ vật lý/máy ảo.

2. **Lệnh `USER appuser` cắt đứt chuỗi ở đâu:**
Lệnh `USER appuser` cắt đứt chuỗi ngay tại **Bước 3**. Khi kẻ tấn công khai thác được mã Python, tiến trình shell chỉ chạy với quyền của một người dùng thông thường không có đặc quyền (`UID 10001`). Người dùng này không có quyền can thiệp vào các tệp tin hệ điều hành, không có quyền mount thiết bị, và không sở hữu các Linux capabilities nguy hiểm (như `CAP_SYS_ADMIN`), do đó các kỹ thuật container breakout thông thường sẽ bị vô hiệu hóa hoàn toàn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Con số tối đa có thể gửi được là: **20 request** trong 2 giây liên tiếp.
- Cách đạt được:
  1. Người dùng chờ đến giây thứ 59 của phút hiện tại (ví dụ 10:00:59) và gửi liên tiếp 10 request. Vì trong phút 10:00 người dùng chưa tiêu hết hạn mức nên cả 10 request này đều được hệ thống chấp thuận hợp lệ.
  2. Ngay 1 giây sau đó, đồng hồ chuyển sang phút tiếp theo (10:01:00), bộ đếm cố định của hệ thống bị reset về 0.
  3. Tại giây 10:01:01, người dùng gửi tiếp tục 10 request nữa. Hệ thống nhận diện đây là 10 request thuộc phút mới 10:01 nên tiếp tục cho qua toàn bộ.
  => Kết quả: Người dùng đã thực hiện thành công 20 request chỉ trong khoảng thời gian vỏn vẹn 2 giây (từ 10:00:59 đến 10:01:01), gây áp lực tải tăng gấp đôi lên hệ thống. Thuật toán cửa sổ trượt (sliding window) khắc phục triệt để lỗ hổng này bằng cách tính tổng số request trong đúng 60 giây trôi qua tính từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

1. **Điểm khác biệt:**
- **Rate limit:** Kiểm soát **tần suất và lưu lượng truy cập** trong một khoảng thời gian ngắn (ví dụ số request mỗi phút) để bảo vệ hạ tầng máy chủ khỏi tình trạng quá tải, nghẽn mạng hoặc tấn công DDoS. Rate limit không quan tâm chi phí tài chính của request là bao nhiêu.
- **Cost guard:** Kiểm soát **ngân sách tài chính thực tế** (tính theo USD hoặc lượng token lũy kế) trong chu kỳ dài hạn (theo tháng) để ngăn ngừa việc cạn kiệt ngân sách hoặc hóa đơn đám mây tăng vọt ngoài tầm kiểm soát.

2. **Tình huống Rate limit cho qua nhưng Cost guard phải chặn:**
Một người dùng chỉ gửi 1 request duy nhất trong vòng 15 phút (tần suất cực kỳ thấp, hoàn toàn dưới hạn mức 10 request/phút của Rate limit). Tuy nhiên, người dùng này đã tiêu hết 10.0 USD ngân sách tháng trong các ngày trước đó, hoặc câu hỏi có prompt quá dài kèm tài liệu lớn khiến chi phí ước tính vượt quá hạn mức còn lại. Khi đó, Rate limit cho qua nhưng Cost guard sẽ chặn lại và trả về mã lỗi `402 Payment Required`.

3. **Tình huống Cost guard cho qua nhưng Rate limit phải chặn:**
Vào ngày đầu tiên của tháng mới, người dùng có đủ 10.0 USD ngân sách và chưa tiêu bất kỳ đồng nào. Người dùng viết một đoạn mã gửi liên tục 15 câu hỏi cực ngắn (ví dụ "Hi", "Hello", chi phí mỗi request chỉ khoảng 0.00001 USD, tổng cộng chưa tới 0.001 USD — quá nhỏ so với 10 USD ngân sách). Dù không lo ngại về chi phí tài chính, nhưng vì tần suất vượt quá 10 request/phút nên Rate limit sẽ chặn lại từ request thứ 11 và trả về mã lỗi `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra khi gộp chung và kiểm tra Redis:
1. Redis gặp sự cố mạng hoặc khởi động lại tạm thời trong vòng 30 giây.
2. Endpoint `/health` (vốn được dùng làm liveness probe) của cả 3 container agent kiểm tra kết nối tới Redis và đều nhận lỗi, dẫn tới trả về HTTP 503.
3. Container orchestrator (Docker/Kubernetes/Cloud platform) thăm dò định kỳ và thấy liveness probe của cả 3 container liên tục thất bại quá số lần cho phép (`retries`).
4. Do hiểu nhầm rằng tiến trình ứng dụng bên trong các container đã bị treo hoặc chết hoàn toàn, orchestrator lập tức gửi lệnh SIGKILL/restart để khởi động lại cả 3 container cùng lúc.
5. Khi 3 container mới khởi động lại, Redis vẫn chưa hoàn tất 30 giây phục hồi, nên các container mới vừa khởi động xong lại tiếp tục kiểm tra Redis thất bại và tiếp tục bị restart lại (rơi vào chu kỳ CrashLoopBackOff).
6. Toàn bộ cụm dịch vụ bị sập hoàn toàn (downtime 100%), mọi request đang được xử lý bị ngắt quãng và người dùng nhận lỗi 502 Bad Gateway.
*Bài học:* `/health` (liveness) chỉ kiểm tra tiến trình app còn chạy hay không; còn `/ready` (readiness) mới kiểm tra dependency để load balancer tạm thời không chuyển traffic tới mà không giết container.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu bằng **Redis (Stateless)**: Cả 3 container đều truy vấn chung một cơ sở dữ liệu Redis tập trung. Do đó, dù mỗi lượt request được load balancer chuyển ngẫu nhiên tới container 1, container 2 hay container 3, `history_length` luôn tăng dần đều và nhất quán theo chuỗi: 0, 2, 4, 6, 8...
- Nếu lưu trong một **dict Python (Stateful)**: Mỗi container sở hữu một tiến trình và vùng nhớ RAM độc lập, không chia sẻ bộ nhớ cho nhau:
  + Giả sử Request 1 vào container A: dict của A lưu câu hỏi và câu trả lời đầu tiên (`history_length` = 0).
  + Request 2 của cùng user bị load balancer định tuyến sang container B: do dict của B hoàn toàn rỗng, B không tìm thấy ngữ cảnh nào và phản hồi với `history_length` = 0 (agent bị mất trí nhớ hoàn toàn).
  + Request 3 sang container C: tiếp tục nhận `history_length` = 0.
  + Request 4 ngẫu nhiên quay lại container A: A tìm thấy dữ liệu cũ của Request 1 nên trả về `history_length` = 2.
  => Kết quả: Người dùng sẽ thấy `history_length` nhảy một cách ngẫu nhiên và khó lường (0, 0, 0, 2, 0, 2...), khiến trải nghiệm hội thoại đa lượt bị vỡ vụn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi gặp phải:**
  `Health check failed on port 8000: connection refused / timeout. Container failed to start and pass healthcheck within 60s.`
- **Cách tìm ra nguyên nhân:**
  Mở phần Runtime Logs trên giao diện Cloud Dashboard (Render/Railway). Tôi nhận thấy nền tảng tự động cấp phát một cổng ngẫu nhiên thông qua biến môi trường `$PORT` (ví dụ `PORT=10000`), tuy nhiên file cấu hình khởi chạy ban đầu trong Dockerfile lại cố định cứng cổng lắng nghe của uvicorn là `--port 8000`. Kết quả là uvicorn mở cổng 8000 trong khi Cloud Router lại gửi probe kiểm tra vào cổng `$PORT` do họ chỉ định, gây ra timeout.
- **Cách sửa:**
  Sửa lại lệnh `CMD` trong `Dockerfile` sang dạng shell string để mở rộng biến môi trường `$PORT`, có fallback về 8000 khi chạy local:
  `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
  Đồng thời đảm bảo `--host 0.0.0.0` để uvicorn lắng nghe trên tất cả các network interfaces của container. Sau khi commit và push lại, nền tảng nhận diện cổng chính xác và health check trả về HTTP 200 OK ngay lập tức.
