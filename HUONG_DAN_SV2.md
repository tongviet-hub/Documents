# Đề tài 05 – Website trường học: phân tích đề & hướng dẫn cho SV2

> **Cậu là SV2 – phần việc: "Cài Web, cấu hình hệ điều hành".**
> File này gồm 3 phần: **(A)** hiểu đề bằng lời dễ hiểu, **(B)** hiểu phần của SV2, **(C)** làm từng bước trên repo này (Windows).
> Cậu không cần biết trước gì về mạng hay server – cứ đọc từ trên xuống.

---

## Mục lục
- [A. Hiểu đề bài](#a-hiểu-đề-bài)
- [B. SV2 phải làm gì và nộp gì](#b-sv2-phải-làm-gì-và-nộp-gì)
- [C. Hướng dẫn thực hiện từng bước](#c-hướng-dẫn-thực-hiện-từng-bước)
- [D. Chuẩn bị cho KT1–KT3 và câu hỏi bảo vệ](#d-chuẩn-bị-cho-kt1kt3-và-câu-hỏi-bảo-vệ)
- [E. Lỗi thường gặp](#e-lỗi-thường-gặp)
- [F. Từ điển nhanh](#f-từ-điển-nhanh)

---

# A. Hiểu đề bài

## A1. Đề bài nói gì, một câu thôi

> "Hãy tự dựng một website cho trường trên máy của nhóm, rồi **chứng minh được là cậu hiểu** khi mình gõ địa chỉ web thì chuyện gì xảy ra phía sau, và web đó an toàn – có sao lưu – khi gặp sự cố."

Đề **không** bắt làm web phức tạp. Web chỉ cần có **thông báo** và **tài liệu tải về**. Phần "ăn điểm" là hiểu và chứng minh cách hệ thống chạy.
Repo này đã có sẵn website (School Portal, React) nên cả nhóm **khỏi phải code web**, chỉ cần đưa nó lên một web server.

## A2. Ví dụ đời thường: website giống một thư viện

| Trong đời thường (thư viện) | Trong đề bài (website) |
|---|---|
| Tên thư viện: "Thư viện Ánh Dương" | **Tên miền**: `truong.test` |
| Tra danh bạ để biết thư viện ở số nhà nào | **DNS**: đổi tên miền → địa chỉ IP |
| Số nhà / địa chỉ | **Địa chỉ IP**: ví dụ `192.168.1.10` |
| Thư viện có nhiều cửa, cửa số 80 là cửa đón khách | **Cổng (port)**: web thường dùng cổng 80 (HTTP) và 443 (HTTPS) |
| Thủ thư ngồi quầy, nhận yêu cầu và đưa sách | **HTTP server** (Nginx): nhận yêu cầu, trả file |
| Kệ sách trong kho | **Hệ thống tập tin**: thư mục chứa file web |
| Tòa nhà + điện nước + bảo vệ + quy định ai được vào kho | **Hệ điều hành** (Windows/Linux): cấp CPU, RAM, ổ đĩa, kiểm soát quyền |
| Câu thủ thư trả lời: "có đây" / "không có" / "cậu không được vào" | **Mã trạng thái HTTP**: 200 / 404 / 403 |
| Sổ ghi ai vào mượn gì, lúc nào | **Log truy cập** |
| Photo sách quý cất chỗ khác | **Sao lưu (backup)** |
| Khóa cửa phòng quản lý | **Giới hạn truy cập quản trị** |

## A3. Đường đi của một lần truy cập web (đề yêu cầu cậu vẽ cái này)

Khi cậu gõ `http://truong.test/` rồi nhấn Enter:

```
 Trình duyệt (máy khách)
        │ 1. "truong.test là ở đâu?"
        ▼
   DNS  (hoặc file hosts)  ──►  trả lời: "192.168.1.10"
        │ 2. Mở kết nối tới IP 192.168.1.10, cổng 80
        ▼
  Card mạng / Tường lửa của máy chủ   (tường lửa phải cho phép cổng 80)
        │ 3. Gói tin được giao cho tiến trình đang "nghe" ở cổng 80
        ▼
  HTTP server (Nginx)   ◄── hệ điều hành cấp CPU/RAM, kiểm tra quyền
        │ 4. Đọc yêu cầu: "GET /tai-lieu/thong-bao-mau.txt"
        │    Dò trong thư mục nội dung (C:\web\school)
        ▼
  Hệ thống tập tin (ổ đĩa)  ──►  đọc file
        │ 5. Trả về: mã trạng thái (200/404...) + nội dung file
        ▼
 Trình duyệt hiển thị trang / tải file
        (và Nginx ghi 1 dòng vào log truy cập)
```

**Nhớ 5 bước này** – hầu hết câu hỏi bảo vệ đều xoay quanh chúng. Mỗi lỗi sẽ "gãy" ở một bước khác nhau (xem KT2 ở phần D).

## A4. Bốn yêu cầu bắt buộc – giải thích dễ hiểu

| # | Đề viết | Nghĩa là | Ai chính |
|---|---|---|---|
| 1 | Vẽ đường đi yêu cầu từ trình duyệt… đến hệ thống tập tin | Vẽ sơ đồ như mục A3 của nhóm, có ghi chú từng bước | SV1 |
| 2 | Triển khai Web trên Linux/Windows; tên miền nội bộ, thư mục nội dung, quyền tiến trình | **Cài web server, đặt tên miền giả (`truong.test`), chọn thư mục chứa web, cho server chạy bằng tài khoản ít quyền** | **SV2 (cậu)** |
| 3 | Quan sát mã HTTP, log, PID, CPU/RAM khi có nhiều yêu cầu | Bắn nhiều request vào web rồi xem log, số tiến trình, mức CPU/RAM; giải thích HĐH làm gì | SV4 (+ log: SV3) |
| 4 | Giới hạn truy cập quản trị, sao lưu & phục hồi, phương án HTTPS | Khóa khu vực quản trị, copy dự phòng nội dung rồi thử xóa–khôi phục, trình bày cách bật HTTPS | SV3 |

## A5. Ba bài kiểm thử nghiệm thu

- **KT1 – Chạy được:** từ **máy khác** (hoặc ít nhất trình duyệt), vào `http://truong.test/` thấy trang và tải được tài liệu.
- **KT2 – Phân biệt 2 loại lỗi:** (a) cố tình làm **DNS sai**, (b) **tắt dịch vụ** web. Hai lỗi đều "không vào được web" nhưng **bằng chứng khác nhau** → phải chỉ ra được.
- **KT3 – Phục hồi:** xóa/hỏng nội dung → khôi phục từ bản sao lưu → web vào lại bình thường.

---

# B. SV2 phải làm gì và nộp gì

**Nhiệm vụ của cậu:** *cài web server, cấu hình hệ điều hành để web chạy được bằng tên miền nội bộ, đúng thư mục nội dung, và chạy bằng tiến trình ít quyền.*

Cậu sẽ làm 6 việc:

1. Build website từ repo → có thư mục `dist`.
2. Tạo **thư mục nội dung** riêng cho web (`C:\web\school`).
3. Cài **Nginx** (web server).
4. Tạo **tên miền nội bộ** `truong.test` (file hosts) + mở cổng 80 ở tường lửa.
5. Cho Nginx chạy như **dịch vụ** dưới **tài khoản ít quyền** (`svc_nginx`).
6. Chụp bằng chứng + viết giải thích.

**Phần giáp ranh với bạn khác** (hỏi nhau cho rõ, tránh làm trùng):

| Bạn | Cậu đưa gì cho bạn | Bạn đưa gì cho cậu |
|---|---|---|
| SV1 (DNS) | IP máy chủ, tên miền đang dùng | Sơ đồ DNS/luồng truy cập, tên miền thống nhất |
| SV3 (quyền, log, sao lưu) | File `nginx.conf`, vị trí log và thư mục nội dung | Cấu hình khóa khu vực quản trị, script backup |
| SV4 (đo tài nguyên) | Máy chủ chạy sẵn + tên PID/tiến trình `nginx.exe` | Số liệu CPU/RAM khi có nhiều request |

**Hồ sơ cá nhân cậu cần nộp** ("file hoặc số liệu kèm giải thích"):
- [ ] File `nginx.conf` đang dùng (có chú thích).
- [ ] Ảnh chụp `nginx -t` báo OK + trang web mở bằng `http://truong.test`.
- [ ] Ảnh/số liệu: nội dung file `hosts`, kết quả `Resolve-DnsName truong.test`.
- [ ] Ảnh/số liệu: `Get-Service`/`tasklist /v` cho thấy Nginx chạy dưới `svc_nginx`.
- [ ] Ảnh/số liệu: `icacls C:\web\school` cho thấy quyền.
- [ ] Ảnh: tải được `/tai-lieu/thong-bao-mau.txt`.
- [ ] 1 đoạn giải thích ngắn (mẫu ở cuối mục C9).

---

# C. Hướng dẫn thực hiện từng bước

> Máy cậu: **Windows 11**. Dùng **PowerShell mở bằng "Run as administrator"** cho các bước có ghi 🔒.
> Mở PowerShell admin: nhấn nút Start, gõ `PowerShell`, chuột phải → **Run as administrator**.

## C0. Chuẩn bị

Kiểm tra đã có Node.js (đã có sẵn nếu cậu build được project này):
```powershell
node -v
npm -v
```

## C1. Build website → ra thư mục `dist`

Website là ứng dụng React. "Build" là biến code thành **file tĩnh** (HTML/CSS/JS) mà web server chỉ việc gửi đi.

```powershell
cd "C:\Users\ADMIN\OneDrive\Documents\KEY_PROJECT\School Portal\frontend"
npm install
npm run build
```
Xong sẽ có thư mục `frontend\dist` (chứa `index.html`, thư mục `assets`, thư mục `tai-lieu`).

> 💡 Thư mục `frontend\public\tai-lieu\` là nơi để **tài liệu tải về**. Muốn thêm tài liệu thật (PDF, Word…) thì bỏ vào đó rồi build lại. Tớ đã tạo sẵn file mẫu `thong-bao-mau.txt` để cậu test KT1.

## C2. Tạo thư mục nội dung và chép web vào

**Vì sao không để web trong thư mục project (OneDrive)?** OneDrive đồng bộ liên tục, có thể khóa file/đổi quyền làm server lỗi lặt vặt. Thư mục `C:\web\school` gọn, dễ gán quyền, dễ giải thích.

🔒
```powershell
New-Item -ItemType Directory -Force C:\web\school | Out-Null
robocopy "C:\Users\ADMIN\OneDrive\Documents\KEY_PROJECT\School Portal\frontend\dist" C:\web\school /MIR
```
- `/MIR` = làm cho `C:\web\school` **giống hệt** `dist` (file thừa bên kia bị xóa). Chỉ dùng cho thư mục này thôi nhé.
- Mỗi lần sửa web: `npm run build` rồi chạy lại lệnh `robocopy` này.

Kiểm tra:
```powershell
Get-ChildItem C:\web\school
```
Phải thấy `index.html`, `assets`, `tai-lieu`.

## C3. Cài Nginx

1. Vào https://nginx.org/en/download.html → mục **Stable version** → tải bản **nginx/Windows-x.y.z** (file `.zip`).
2. Giải nén vào `C:\nginx` (sao cho có file `C:\nginx\nginx.exe`).

Kiểm tra:
```powershell
cd C:\nginx
.\nginx.exe -v
```
Thấy `nginx version: nginx/...` là được.

## C4. Tên miền nội bộ `truong.test`

Chưa có DNS thật, nên tạm dùng **file hosts** (một "danh bạ" nằm ngay trong máy: tên miền → IP). Máy sẽ hỏi file này **trước** khi hỏi DNS.

> Dùng đuôi `.test` vì đây là đuôi **dành riêng cho thử nghiệm**, không đụng tên miền thật. Tránh dùng `.local` (Windows hay chậm vì cơ chế mDNS).

🔒 Mở file bằng quyền admin:
```powershell
notepad C:\Windows\System32\drivers\etc\hosts
```
Thêm **một dòng** ở cuối rồi lưu:
```
127.0.0.1    truong.test
```
Kiểm tra:
```powershell
Resolve-DnsName truong.test
ping truong.test
```
Phải thấy IP `127.0.0.1`. (Nếu ping/Resolve không thấy, xem mục E.)

**Cho máy khách khác trong cùng mạng (để làm KT1 đúng nghĩa):**
1. Trên máy cậu (máy chủ) lấy IP mạng LAN: `ipconfig` → dòng `IPv4 Address` (ví dụ `192.168.1.10`).
2. Trên **máy khách**, mở file hosts (quyền admin) và thêm: `192.168.1.10    truong.test`
3. Mở cổng 80 ở tường lửa máy chủ 🔒:
   ```powershell
   New-NetFirewallRule -DisplayName "Web truong.test (TCP 80)" -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow -Profile Private
   ```
   (Mạng Wi-Fi phải được đặt là **Private network**. Hãy nhớ **xóa/tắt rule** này sau khi demo nếu không cần.)

## C5. Viết cấu hình Nginx

Mở `C:\nginx\conf\nginx.conf`, **xóa hết nội dung cũ**, dán nguyên đoạn sau:

```nginx
# ===== Cấu hình Nginx cho Website trường học (Đề tài 05) =====
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;                 # bảng "đuôi file → loại nội dung"
    default_type  application/octet-stream;
    server_tokens off;                        # ẩn số phiên bản Nginx (bảo mật cơ bản)

    # Định dạng log: ai (IP), lúc nào, xin gì, trả mã gì, bao nhiêu byte, mất bao lâu
    log_format school '$remote_addr [$time_local] "$request" '
                      '$status $body_bytes_sent ${request_time}s "$http_user_agent"';

    sendfile        on;
    keepalive_timeout  65;

    gzip on;                                  # nén bớt dữ liệu cho nhẹ
    gzip_types text/css application/javascript application/json image/svg+xml;

    server {
        listen       80;                      # cổng dịch vụ
        server_name  truong.test;             # tên miền nội bộ

        root   C:/web/school;                 # THƯ MỤC NỘI DUNG (dùng dấu / trên Windows)
        index  index.html;

        access_log  logs/access.log  school;  # log truy cập
        error_log   logs/error.log   warn;    # log lỗi

        # Trang web (ứng dụng React): đường dẫn nào không phải file thật
        # thì trả index.html để React tự xử lý (/grades, /schedule...)
        location / {
            try_files $uri $uri/ /index.html;
        }

        # File giao diện: cho trình duyệt nhớ 7 ngày
        location /assets/ {
            expires 7d;
        }

        # Tài liệu tải về: không có file thì trả 404 thật (để quan sát mã HTTP)
        location /tai-lieu/ {
            try_files $uri =404;
        }

        # Chặn truy cập file ẩn (.git, .env...)
        location ~ /\. {
            deny all;
        }
    }
}
```

Kiểm tra cú pháp (luôn làm bước này **trước khi** chạy):
```powershell
cd C:\nginx
.\nginx.exe -t
```
Phải thấy `syntax is ok` và `test is successful`.

## C6. Chạy thử (kiểu thủ công)

```powershell
cd C:\nginx
start .\nginx.exe          # bật
```
Mở trình duyệt: **http://truong.test/** → thấy cổng thông tin màu đỏ trắng.
Thử tải tài liệu: **http://truong.test/tai-lieu/thong-bao-mau.txt**.

Các lệnh hay dùng (chạy trong `C:\nginx`):
| Việc | Lệnh |
|---|---|
| Nạp lại cấu hình không ngắt web | `.\nginx.exe -s reload` |
| Tắt êm | `.\nginx.exe -s quit` |
| Tắt ngay | `.\nginx.exe -s stop` |
| Kiểm tra cấu hình | `.\nginx.exe -t` |

Xem log trực tiếp (mở trình duyệt, nhấn F5 và nhìn log chạy):
```powershell
Get-Content C:\nginx\logs\access.log -Wait -Tail 10
```

## C7. Cho Nginx chạy như dịch vụ, dưới tài khoản ít quyền 🔒

**Vì sao?** Nếu web server chạy bằng tài khoản Administrator mà bị tấn công, kẻ xấu có luôn quyền admin. Nguyên tắc: **dịch vụ chỉ nên có đúng quyền cần thiết** (đọc thư mục web, ghi log) – gọi là *nguyên tắc đặc quyền tối thiểu*.

**Bước 1 – Tạo tài khoản dịch vụ** (thay `MatKhauManh_123` bằng mật khẩu của cậu, **đừng đưa mật khẩu lên GitHub**):
```powershell
net user svc_nginx "MatKhauManh_123" /add
```

**Bước 2 – Gán quyền** (đọc thư mục web, ghi log):
```powershell
# Thư mục web: chỉ đọc. Bỏ quyền kế thừa mặc định (mặc định ai cũng sửa được), rồi cấp lại đúng người cần.
icacls C:\web\school /inheritance:r
icacls C:\web\school /grant "SYSTEM:(OI)(CI)F" "Administrators:(OI)(CI)F" "svc_nginx:(OI)(CI)RX"

# Nginx cần đọc cấu hình và ghi log/tạm
icacls C:\nginx /grant "svc_nginx:(OI)(CI)RX"
icacls C:\nginx\logs /grant "svc_nginx:(OI)(CI)M"
icacls C:\nginx\temp /grant "svc_nginx:(OI)(CI)M"
```
(`RX` = đọc + chạy, `M` = sửa, `F` = toàn quyền. Tài khoản `svc_nginx` **không** được ghi vào thư mục web → web không tự bị sửa nếu server bị chiếm.)

**Bước 3 – Đăng ký thành dịch vụ bằng NSSM** (công cụ biến chương trình bất kỳ thành dịch vụ Windows):
1. Tải NSSM tại https://nssm.cc/download (hoặc thử `winget install nssm`), giải nén, lấy `nssm.exe` bản `win64`.
2. Tắt Nginx đang chạy tay: `cd C:\nginx; .\nginx.exe -s stop`
3. Chạy:
```powershell
nssm install nginx-school "C:\nginx\nginx.exe"
nssm set nginx-school AppDirectory "C:\nginx"
nssm set nginx-school ObjectName ".\svc_nginx" "MatKhauManh_123"
nssm set nginx-school Start SERVICE_AUTO_START
nssm start nginx-school
```
> Nếu báo lỗi *"logon failure"/"không có quyền Log on as a service"*: mở `services.msc` → chuột phải `nginx-school` → Properties → tab **Log On** → nhập lại tài khoản `.\svc_nginx` và mật khẩu → OK (Windows sẽ tự cấp quyền) → Start.

**Bước 4 – Kiểm tra:**
```powershell
Get-Service nginx-school                          # Status = Running
tasklist /v /fi "imagename eq nginx.exe"          # cột "User Name" phải là ...\svc_nginx
Get-Process nginx | Select-Object Id, ProcessName, CPU, WS   # PID của các tiến trình nginx
Get-NetTCPConnection -LocalPort 80 -State Listen | Select-Object LocalAddress, LocalPort, OwningProcess   # PID nào đang giữ cổng 80
icacls C:\web\school
```
> 🧠 Điểm giải thích hay: **"cổng 80 đang do PID số mấy nghe, tiến trình đó chạy bằng tài khoản nào, và tài khoản đó có những quyền gì?"** – ba câu này ghép lại là "hệ điều hành kiểm soát dịch vụ web".

## C8. "Bắn" thử nhiều yêu cầu & xem mã HTTP (hỗ trợ SV4)

Xem mã trạng thái của từng loại yêu cầu:
```powershell
curl.exe -I http://truong.test/                              # 200 OK
curl.exe -I http://truong.test/tai-lieu/thong-bao-mau.txt    # 200 OK
curl.exe -I http://truong.test/tai-lieu/khong-co.pdf         # 404 Not Found
curl.exe -I http://truong.test/.git/config                   # 403 Forbidden (bị chặn)
```
Tạo 200 yêu cầu liên tiếp để log và tài nguyên có số liệu:
```powershell
1..200 | ForEach-Object { curl.exe -s -o NUL -w "%{http_code} " http://truong.test/ }
```
Sau đó nhìn `access.log` – sẽ thấy 200 dòng mới, mỗi dòng có IP, thời gian, mã, kích thước, thời gian xử lý.

## C9. Viết phần giải thích cho hồ sơ (mẫu để cậu sửa lại bằng lời của mình)

> **Phần thực hiện của SV2 – Cài Web & cấu hình hệ điều hành**
> 1. Em dùng Nginx (bản Windows) làm HTTP server, lắng nghe cổng 80, phục vụ nội dung tĩnh của website React đã build, đặt trong `C:\web\school`.
> 2. Tên miền nội bộ `truong.test` được ánh xạ về IP máy chủ bằng file hosts; máy khách hỏi file hosts trước khi hỏi DNS nên không cần máy chủ DNS thật.
> 3. Nginx chạy như dịch vụ Windows `nginx-school` bằng tài khoản `svc_nginx` (không phải Administrator). Tài khoản này chỉ có quyền **đọc** thư mục web và **ghi** thư mục log, nên nếu dịch vụ bị lỗi/bị chiếm thì thiệt hại bị giới hạn.
> 4. Hệ điều hành đóng vai trò: giữ cổng 80 cho đúng tiến trình (PID …), cấp CPU/RAM cho tiến trình, kiểm tra quyền đọc/ghi file theo tài khoản chạy dịch vụ, và tường lửa quyết định gói tin nào được vào.
> 5. Bằng chứng: ảnh `nginx -t`, `Get-Service`, `tasklist /v`, `icacls`, ảnh trình duyệt mở `http://truong.test`.

---

# D. Chuẩn bị cho KT1–KT3 và câu hỏi bảo vệ

## KT1 – Truy cập bằng tên miền từ máy khách
Từ máy khách (đã thêm dòng hosts như C4): mở `http://truong.test/` và `http://truong.test/tai-lieu/thong-bao-mau.txt`. Chụp ảnh có thanh địa chỉ.

## KT2 – Hai lỗi, hai bằng chứng khác nhau
Cậu hỗ trợ SV4 làm mẫu như sau:

| | **Lỗi 1: DNS sai** (sửa/xóa dòng trong hosts) | **Lỗi 2: dừng dịch vụ** (`nginx-school` stop) |
|---|---|---|
| Cách tạo | Đổi dòng hosts thành `10.255.255.1  truong.test` (hoặc xóa dòng đó) | `nssm stop nginx-school` (hoặc `Stop-Service nginx-school`) |
| `Resolve-DnsName truong.test` | IP **sai** (hoặc **lỗi không tìm thấy** nếu đã xóa dòng) | IP **đúng** |
| `ping` | Không tới / không phân giải được tên | Tới được (ping không phụ thuộc web) |
| Trình duyệt | `ERR_NAME_NOT_RESOLVED` (xóa dòng) hoặc treo rồi `ERR_CONNECTION_TIMED_OUT` (IP sai) | `ERR_CONNECTION_REFUSED` (bị từ chối ngay) |
| `curl.exe -v http://truong.test/` | `Could not resolve host` hoặc timeout | `Failed to connect ... Connection refused` |
| Cổng 80 trên máy chủ | Vẫn đang nghe | **Không còn ai nghe** (`Get-NetTCPConnection -LocalPort 80` trống) |
| `access.log` | **Không có dòng mới** (yêu cầu chưa hề tới server) | **Không có dòng mới** (server đang tắt) – nhưng `error.log`/trạng thái dịch vụ cho thấy server dừng |

**Cách nhớ:** DNS sai = *"không biết nhà ở đâu"* (hỏng ở bước 1). Dừng dịch vụ = *"biết nhà ở đâu, nhưng không ai mở cửa"* (hỏng ở bước 3–4).
Muốn chứng minh rõ, hãy chụp **cả `Resolve-DnsName` lẫn `Get-Service`/cổng 80** cho từng lỗi.

> Xong KT2 nhớ **sửa lại hosts** và **`Start-Service nginx-school`**.

## KT3 – Khôi phục từ bản sao (SV3 làm chính)
Cậu chỉ cần biết: web "sống" ở `C:\web\school`. Nếu nội dung bị xóa, khôi phục = chép lại bản sao vào đúng thư mục đó (hoặc chạy lại `robocopy` từ `dist`). Sau đó F5 trang để xác nhận.
Ví dụ minh họa nhanh:
```powershell
robocopy C:\web\school C:\web\backup-demo /MIR          # sao lưu
Remove-Item C:\web\school\index.html                     # phá thử → mở web sẽ thấy 403/404
robocopy C:\web\backup-demo C:\web\school /MIR           # khôi phục → web vào lại
```
> Lưu ý: sau `icacls ... /inheritance:r` ở C7, chạy các lệnh này bằng PowerShell **admin**. Nếu `svc_nginx` mất quyền đọc sau khi khôi phục, chạy lại lệnh `icacls ... "svc_nginx:(OI)(CI)RX"` ở C7.

## HTTPS (đề bảo trình bày phương án – chỉ cần nói, không bắt buộc cài)
- HTTP gửi dữ liệu **dạng chữ rõ**, ai nghe lén trên mạng đều đọc được. **HTTPS = HTTP + mã hóa (TLS)**, thêm xác thực "đúng là trang của trường".
- Triển khai thật: đăng ký **tên miền thật**, xin **chứng chỉ miễn phí Let's Encrypt** (công cụ certbot/win-acme), cấu hình Nginx `listen 443 ssl;` + `ssl_certificate`, `ssl_certificate_key`, và chuyển hướng HTTP→HTTPS.
- Trong môi trường thử `.test`, không xin được chứng chỉ thật; có thể dùng công cụ `mkcert` để tạo chứng chỉ tự ký chỉ để minh họa.

## Câu hỏi chuẩn bị: *"Ping thành công nhưng Website không mở được thì kiểm tra gì tiếp?"*
Ping chỉ chứng minh **mạng tới được máy chủ** (bước 2). Web còn phụ thuộc các bước sau. Kiểm tra **theo thứ tự**:
1. **Dịch vụ web có chạy không?** `Get-Service nginx-school`, `tasklist | findstr nginx`.
2. **Cổng 80 có ai nghe không?** `Get-NetTCPConnection -LocalPort 80 -State Listen` (hoặc `netstat -ano | findstr :80`). Không có → web server chưa chạy/đang nghe sai cổng.
3. **Tường lửa có chặn cổng 80 không?** Thử từ chính máy chủ (`curl.exe -I http://127.0.0.1/`) rồi thử từ máy khác. Máy chủ OK mà máy khách không → nghi tường lửa/mạng.
4. **Gõ đúng tên miền/cổng/giao thức chưa?** `http://` hay `https://`? có đúng cổng?
5. **Đọc log lỗi:** `C:\nginx\logs\error.log` và `access.log` (có yêu cầu tới không? mã trả về là gì?).
6. **Cấu hình đúng chưa?** `nginx -t`; đường dẫn `root`; **quyền đọc thư mục web** của tài khoản chạy dịch vụ (lỗi 403).
7. **Tên miền có trỏ đúng IP không?** (`Resolve-DnsName`) – ping bằng **IP** thành công nhưng ping bằng **tên** phân giải ra IP khác thì web vào nhầm máy; cũng cần xóa cache DNS: `ipconfig /flushdns`.

---

# E. Lỗi thường gặp

| Triệu chứng | Nguyên nhân hay gặp | Cách xử lý |
|---|---|---|
| `nginx: [emerg] bind() to 0.0.0.0:80 failed (10013/10048)` | Cổng 80 đã bị chương trình khác dùng (IIS, Skype, Apache, XAMPP…) | `netstat -ano \| findstr ":80 "` để lấy PID → `tasklist /fi "pid eq <PID>"` xem là ai → tắt nó, hoặc đổi `listen 8080;` |
| Mở `truong.test` ra trang lạ / không thấy gì | Hosts chưa lưu (cần quyền admin) hoặc sai chính tả; cache DNS | Mở Notepad bằng admin, kiểm tra lại; `ipconfig /flushdns` |
| **403 Forbidden** | Tài khoản dịch vụ không đọc được `C:\web\school` | `icacls C:\web\school` và cấp lại `svc_nginx:(OI)(CI)RX` |
| **404** khi F5 ở trang `/grades` | Thiếu `try_files ... /index.html` | Kiểm tra khối `location /` trong `nginx.conf` |
| Trang trắng, F12 thấy lỗi tải `assets/...` | Chép thiếu thư mục `assets` | Chạy lại `npm run build` và `robocopy` |
| Sửa web xong nhưng vẫn thấy bản cũ | Trình duyệt cache | `Ctrl + F5` |
| Dịch vụ `nginx-school` không start | Sai mật khẩu `svc_nginx` / thiếu quyền ghi `logs`, `temp` | Xem `C:\nginx\logs\error.log`, kiểm tra lại C7 |
| Máy khách không vào được | Tường lửa / mạng đặt Public / khác mạng | Kiểm tra C4 (rule cổng 80, profile Private, cùng Wi-Fi) |

---

# F. Từ điển nhanh

- **Tên miền (domain):** tên dễ nhớ của website, ví dụ `truong.test`.
- **DNS:** "danh bạ" của Internet, đổi tên miền thành địa chỉ IP. **hosts:** danh bạ tạm nằm trong máy, được hỏi trước DNS.
- **IP:** địa chỉ của một máy trong mạng.
- **Cổng (port):** "cửa" của máy; mỗi dịch vụ nghe ở một cổng. HTTP: 80, HTTPS: 443.
- **HTTP server:** chương trình nhận yêu cầu web và trả file (Nginx, Apache, IIS).
- **Mã trạng thái HTTP:** 200 thành công · 301/302 chuyển hướng · 304 dùng bản trình duyệt đã lưu · 403 bị cấm · 404 không có · 500 server lỗi · 502/503 phía sau/ dịch vụ lỗi.
- **PID:** số định danh của một tiến trình đang chạy.
- **Tiến trình / dịch vụ:** chương trình đang chạy / chương trình chạy nền do hệ điều hành quản lý.
- **Đặc quyền tối thiểu:** chỉ cấp đúng quyền cần dùng, không hơn.
- **Backup / Restore:** sao lưu / khôi phục.
- **HTTPS / TLS:** HTTP có mã hóa và xác thực.

---

### Mở rộng tự chọn (nếu còn thời gian, có thể lấy điểm cộng)
Thay **Nginx bằng IIS** (hoặc Apache) cho cùng nội dung, rồi so sánh bằng **cùng tiêu chí**: thời gian cài đặt, độ dài cấu hình, RAM/CPU khi chạy 200 request (lệnh ở C8), mã trạng thái, cách tách quyền chạy dịch vụ. Lập bảng số liệu hai cột – đó chính là "so sánh bằng số liệu" mà đề yêu cầu.

> ⚠️ **Ghi chú trung thực:** các lệnh trên được viết theo cách các công cụ này hoạt động trên Windows 11 nhưng **chưa được chạy thử trên máy cậu** (tớ chưa cài Nginx hay đổi hosts/tường lửa thay cậu). Nếu một lệnh báo lỗi, đừng bỏ qua – chụp màn hình lỗi, đối chiếu mục E; rất nhiều khi lỗi chính là một "bằng chứng" có thể đưa vào hồ sơ.
