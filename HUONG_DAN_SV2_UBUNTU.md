# Đề tài 05 – Website trường học: hướng dẫn SV2 (bản **Ubuntu**)

> **Cậu là SV2 – phần việc: "Cài Web, cấu hình hệ điều hành".** Bản này làm **trên Ubuntu Server/Desktop 22.04 hoặc 24.04** (máy thật, máy ảo VirtualBox/VMware/Hyper-V hoặc VPS đều được).
> Đề bài, sơ đồ 5 bước, ví dụ "thư viện" giống hệt bản Windows – xem phần **A** trong [HUONG_DAN_SV2.md](HUONG_DAN_SV2.md). File này chỉ thay **cách làm (phần C)** sang Ubuntu, kèm bảng đối chiếu Windows ↔ Ubuntu để cậu bảo vệ cho chắc.
> Muốn làm thêm cho điểm cao (HTTPS, benchmark, hardening, tự động hóa…) → xem [HUONG_DAN_SV2_ADVANCE.md](HUONG_DAN_SV2_ADVANCE.md).

---

## Mục lục
- [A. Tóm tắt đề & đối chiếu Windows ↔ Ubuntu](#a-tóm-tắt-đề--đối-chiếu-windows--ubuntu)
- [B. SV2 phải nộp gì](#b-sv2-phải-nộp-gì)
- [C. Hướng dẫn từng bước trên Ubuntu](#c-hướng-dẫn-từng-bước-trên-ubuntu)
- [D. KT1–KT3 và câu hỏi bảo vệ](#d-kt1kt3-và-câu-hỏi-bảo-vệ)
- [E. Lỗi thường gặp](#e-lỗi-thường-gặp)
- [F. Từ điển lệnh nhanh](#f-từ-điển-lệnh-nhanh)
- [G. Backend Auth và PostgreSQL (tùy chọn)](#g-backend-auth-và-postgresql-tùy-chọn)
- [H. Phụ lục: TOÀN BỘ LỆNH theo thứ tự (copy-paste)](#h-phụ-lục-toàn-bộ-lệnh-theo-thứ-tự-copy-paste)

---

# A. Tóm tắt đề & đối chiếu Windows ↔ Ubuntu

**Đề một câu:** dựng website trường trên máy của nhóm, **chứng minh hiểu** đường đi của một lần truy cập web, và web **an toàn + có sao lưu**.
Website React đã có sẵn trong repo → cậu chỉ **build ra file tĩnh rồi đưa lên Nginx**.

**Đường đi một lần truy cập (nhớ 5 bước):**
```
Trình duyệt ─1─► DNS/hosts (truong.test → IP)
            ─2─► mạng tới IP, cổng 80 ─► tường lửa (ufw) cho phép?
            ─3─► tiến trình đang "nghe" cổng 80 (Nginx)  ← hệ điều hành cấp CPU/RAM, kiểm tra quyền
            ─4─► Nginx dò file trong /var/www/school   (hệ thống tập tin ext4)
            ─5─► trả mã 200/404/403 + nội dung; ghi 1 dòng vào access.log
```

**Bảng đối chiếu** (rất hay được hỏi: "cùng một việc, hai hệ điều hành làm khác nhau thế nào?"):

| Việc | Windows (bản gốc) | Ubuntu (bản này) |
|---|---|---|
| Cài Nginx | tải `.zip` về `C:\nginx` | `sudo apt install nginx` |
| Thư mục nội dung | `C:\web\school` | `/var/www/school` |
| File cấu hình | `C:\nginx\conf\nginx.conf` | `/etc/nginx/nginx.conf` + `/etc/nginx/sites-available/school` |
| "Danh bạ" tên miền cục bộ | `C:\Windows\System32\drivers\etc\hosts` | `/etc/hosts` |
| Chạy như dịch vụ | NSSM → dịch vụ `nginx-school` | **systemd** → dịch vụ `nginx` (có sẵn) |
| Tài khoản ít quyền | `svc_nginx` (net user) | `svc_nginx` (useradd, không đăng nhập được) |
| Phân quyền thư mục | `icacls` | `chown` / `chmod` (rwx cho owner/group/other) |
| Tường lửa | Windows Defender Firewall | `ufw` (bọc ngoài nftables/iptables) |
| Xem tiến trình / PID | `tasklist`, `Get-Process` | `ps`, `pgrep`, `top`/`htop` |
| Ai đang giữ cổng 80 | `Get-NetTCPConnection` | `ss -ltnp` |
| Xem log dịch vụ | file log, Event Viewer | file log + `journalctl -u nginx` |
| Sao lưu thư mục | `robocopy /MIR` | `rsync -a --delete`, `tar` |

> 💡 **Khác biệt đáng nhớ về tài khoản chạy dịch vụ:** trên Linux, tiến trình **master** của Nginx chạy bằng **root** (vì chỉ root mới được mở cổng < 1024), rồi **sinh ra các tiến trình worker chạy bằng tài khoản ít quyền** (`svc_nginx`). Worker là bên đụng vào dữ liệu người dùng nên nếu bị tấn công cũng không có quyền root. Cậu sẽ **chụp bằng chứng** việc này ở mục C7.

---

# B. SV2 phải nộp gì

Nhiệm vụ: *cài web server, cấu hình hệ điều hành để web chạy bằng tên miền nội bộ, đúng thư mục nội dung, và chạy bằng tiến trình ít quyền.*

6 việc: (1) build web → `dist` · (2) tạo thư mục nội dung `/var/www/school` · (3) cài Nginx · (4) tên miền `truong.test` + mở cổng 80 · (5) worker chạy dưới `svc_nginx` · (6) chụp bằng chứng + viết giải thích.

**Phần giáp ranh** (hỏi nhau cho rõ):

| Bạn | Cậu đưa | Bạn đưa |
|---|---|---|
| SV1 (DNS) | IP máy chủ, tên miền đang dùng | Sơ đồ luồng truy cập, tên miền thống nhất |
| SV3 (quyền, log, sao lưu) | File cấu hình Nginx, vị trí log + thư mục nội dung | Cấu hình khóa khu vực quản trị, script backup |
| SV4 (đo tài nguyên) | Máy chủ chạy sẵn + PID Nginx | Số liệu CPU/RAM khi nhiều request |

**Hồ sơ cá nhân:**
- [ ] File cấu hình Nginx đang dùng (có chú thích): `/etc/nginx/sites-available/school` và dòng `user` trong `nginx.conf`.
- [ ] Ảnh `sudo nginx -t` báo OK + trình duyệt mở `http://truong.test`.
- [ ] Ảnh: nội dung `/etc/hosts` + kết quả `getent hosts truong.test`.
- [ ] Ảnh: `ps -eo pid,ppid,euser:20,comm | grep '[n]ginx'` (master = root, worker = svc_nginx) + `systemctl status nginx`.
- [ ] Ảnh: `ls -l /var/www/school` + `namei -l` (quyền) + thử ghi bằng `svc_nginx` bị từ chối.
- [ ] Ảnh: `sudo ss -ltnp 'sport = :80'` (PID nào giữ cổng 80).
- [ ] Ảnh: tải được `/tai-lieu/thong-bao-mau.txt`.
- [ ] 1 đoạn giải thích ngắn (mẫu ở C9).

---

# C. Hướng dẫn từng bước trên Ubuntu

> Mở **Terminal** (Ctrl+Alt+T). Lệnh nào cần quyền quản trị đã có `sudo` sẵn – nhập mật khẩu tài khoản Ubuntu của cậu khi được hỏi (gõ không thấy ký tự là bình thường).
> Dòng bắt đầu bằng `#` là chú thích, không cần gõ.

## C0. Cài đặt ban đầu cho máy Ubuntu mới tinh

Máy vừa cài xong gần như **chưa có gì**. Làm lần lượt các khối dưới đây; cài một lần là đủ cho cả file này và file nâng cao.

**0.1 – Xem phiên bản và cập nhật hệ thống**
```bash
lsb_release -ds            # phiên bản Ubuntu (ghi lại để đưa vào báo cáo)
uname -r                   # phiên bản nhân Linux
ip -4 addr show            # IP máy (tìm dòng "inet 192.168.x.x")
sudo apt update && sudo apt full-upgrade -y
```
> Trong lúc cài, Ubuntu có thể hiện hộp thoại **"Daemons using outdated libraries / Which services should be restarted?"** – cứ nhấn **Enter** (chọn mặc định) là được.
> Nếu `full-upgrade` báo cần khởi động lại (`*** System restart required ***`), chạy `sudo reboot` rồi làm tiếp.

**0.2 – Gói bắt buộc cho phần cơ bản (mục C)**
```bash
sudo apt install -y nginx rsync curl ca-certificates git openssh-server ufw apache2-utils jq openssl htop tree
```
| Gói | Dùng để |
|---|---|
| `nginx` | Web server (tự chạy ngay sau khi cài, giữ cổng 80) |
| `rsync` | Chép/đồng bộ thư mục web, giữ nguyên quyền |
| `curl` | Gọi thử web, xem mã HTTP (`curl -I`) |
| `ca-certificates`, `git` | Tải file qua HTTPS, `git clone` repo |
| `openssh-server` | Để chép file từ Windows sang bằng `scp`, hoặc điều khiển bằng SSH |
| `ufw` | Tường lửa (Ubuntu có sẵn nhưng **tắt**) |
| `apache2-utils` | Có `ab` (bắn nhiều yêu cầu) và `htpasswd` |
| `jq` | Đọc JSON khi thử API (mục G) |
| `openssl` | Sinh mật khẩu/khóa ngẫu nhiên |
| `htop`, `tree` | Xem tiến trình, xem cây thư mục (tiện lợi) |

> `apache2-utils` **không** cài web server Apache, chỉ cài công cụ `ab`/`htpasswd`. Đừng cài gói `apache2`, nó sẽ tranh cổng 80 với Nginx.

**0.3 – Node.js (để build web ở mục C1 cách 2 và chạy backend ở mục G)**
Project cần **Node.js ≥ 22**. Xem kho Ubuntu có bản đủ mới chưa:
```bash
apt-cache policy nodejs | head -n 3        # nhìn dòng "Candidate:"
```
- Nếu `Candidate` là **22 trở lên**: `sudo apt install -y nodejs npm`
- Nếu cũ hơn (hoặc trống): cài từ NodeSource
  ```bash
  curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
  sudo apt install -y nodejs
  ```
```bash
node -v && npm -v                          # ✅ node ≥ v22
```
> Nếu NodeSource chưa hỗ trợ Ubuntu bản mới nhất của cậu (script báo lỗi), báo tớ kèm kết quả `lsb_release -a`, tớ chỉ cách cài bằng `nvm` thay thế.
> Nếu cậu **build `dist` trên Windows rồi `scp` sang** (mục C1 cách 1) thì phần cơ bản **không cần Node** trên Ubuntu. Chỉ cần Node khi làm mục G (backend).

**0.4 – PostgreSQL (chỉ khi làm mục G – hướng có backend)**
```bash
sudo apt install -y postgresql
psql --version
systemctl is-active postgresql             # ✅ active
```

**0.5 – Công cụ cho file nâng cao ([HUONG_DAN_SV2_ADVANCE.md](HUONG_DAN_SV2_ADVANCE.md))**
```bash
sudo apt install -y libcap2-bin apparmor apparmor-utils
```
| Gói | Dùng để |
|---|---|
| `libcap2-bin` | Lệnh `capsh` giải mã quyền capability (Cấp 2) |
| `apparmor`, `apparmor-utils` | Kiểm soát truy cập bắt buộc: `aa-status`, `aa-complain`, `aa-enforce`, `aa-logprof` (Cấp 4) |

(Các công cụ còn lại dùng ở file nâng cao như `systemd-analyze`, `setpriv`, `mountpoint`, `findmnt`, `nsenter`, `ss`, `ps` đã có sẵn trong Ubuntu.)

**0.6 – Kiểm tra mọi thứ đã sẵn sàng**
```bash
nginx -v ; git --version ; curl --version | head -n 1 ; rsync --version | head -n 1
node -v ; npm -v                           # nếu đã cài Node
psql --version                             # nếu đã cài PostgreSQL
systemctl is-active nginx ssh              # ✅ active active
sudo ufw status                            # Status: inactive (bình thường – ta bật ở C4)
sudo --version | head -n 1
```
> **Về Ubuntu bản mới:** một số bản mới dùng `sudo-rs` (sudo viết bằng Rust) và các lệnh cơ bản (`ls`, `cp`…) bản Rust. Các lệnh trong file này tớ đã viết theo cách chạy được ở cả hai. Nếu lệnh nào báo `unrecognized option` hoặc cho kết quả lạ, **chụp lỗi gửi cho tớ**, đừng bỏ qua.
> Tớ không biết chính xác phiên bản Ubuntu của cậu; một số đường dẫn (ví dụ `/etc/postgresql/<số>/main`) và số phiên bản Nginx/PostgreSQL có thể khác ví dụ, nhưng cách làm giữ nguyên.

## C1. Lấy file website (`dist`)

Website là ứng dụng React; "build" biến code thành **file tĩnh** (HTML/CSS/JS) để Nginx chỉ việc gửi đi. Có 2 cách – **cách 1 dễ hơn**.

### Cách 1 (khuyên dùng): build trên Windows, chép sang Ubuntu
Trên Windows (PowerShell):
```powershell
cd "C:\Users\ADMIN\OneDrive\Documents\KEY_PROJECT\School Portal\frontend"
npm install
npm run build
```
Rồi chép thư mục `dist` sang Ubuntu (thay `ten_user` và IP máy Ubuntu):
```powershell
scp -r dist ten_user@192.168.1.10:/home/ten_user/dist
```
(Máy ảo thì cũng có thể dùng thư mục chia sẻ hoặc kéo-thả. Cần SSH: trên Ubuntu `sudo apt install -y openssh-server`.)

### Cách 2: build ngay trên Ubuntu
Project dùng Vite bản mới nên cần **Node.js ≥ 22**:
```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs git
node -v        # phải ≥ v22

git clone <link-repo-của-nhóm> ~/school-portal
cd ~/school-portal/frontend
npm ci
npm run build   # ra thư mục frontend/dist
```
> ⚠️ **Không chép `node_modules` từ Windows sang Ubuntu** (chứa file nhị phân của Windows, sẽ lỗi). Luôn `npm ci` lại trên Ubuntu.

> 💡 Tài liệu tải về nằm ở `frontend/public/tai-lieu/` (đã có `thong-bao-mau.txt` để test KT1). Thêm tài liệu thật thì bỏ vào đó rồi build lại.

Từ giờ gọi thư mục vừa có là `~/dist` (cách 1) hoặc `~/school-portal/frontend/dist` (cách 2). **Sửa đường dẫn cho đúng ở các lệnh dưới.**

## C2. Cài Nginx

```bash
sudo apt install -y nginx
nginx -v                        # nginx version: nginx/1.x
systemctl status nginx --no-pager
```
Cài xong Nginx **tự chạy** ngay. Mở `http://localhost` trong trình duyệt Ubuntu sẽ thấy trang "Welcome to nginx!" – nghĩa là cài ổn.

## C3. Tạo thư mục nội dung & tài khoản dịch vụ ít quyền

**Vì sao `/var/www`?** Đây là nơi chuẩn của Linux cho nội dung web, tách khỏi thư mục cá nhân (`/home`) để gán quyền gọn và dễ giải thích.

```bash
# 1) Tài khoản hệ thống: không có thư mục home, KHÔNG đăng nhập được
sudo useradd --system --no-create-home --shell /usr/sbin/nologin svc_nginx

# 2) Thư mục nội dung + chép web vào (dấu / ở cuối ~/dist/ quan trọng: chép "ruột" thư mục)
sudo mkdir -p /var/www/school
sudo rsync -a --delete ~/dist/ /var/www/school/

# 3) Phân quyền theo nguyên tắc đặc quyền tối thiểu:
#    chủ = root (chỉ root sửa được), nhóm svc_nginx chỉ ĐỌC, người khác không đụng được
sudo chown -R root:svc_nginx /var/www/school
sudo find /var/www/school -type d -exec chmod 750 {} \;
sudo find /var/www/school -type f -exec chmod 640 {} \;
```
Đọc kết quả:
```bash
ls -l /var/www/school
namei -l /var/www/school/index.html
```
Giải thích số quyền: `750` = chủ **rwx**, nhóm **r-x**, người khác **---**. `640` = chủ rw-, nhóm r--, người khác ---.
(Thư mục cần `x` để "đi vào"; file chỉ cần `r` để đọc.)

> Mỗi lần sửa web: build lại → chạy lại 3 lệnh `rsync` + `chown` + `chmod` ở trên.
> `--delete` làm thư mục đích **giống hệt** nguồn (file thừa bị xóa) – chỉ dùng cho thư mục này.

Chứng minh "đặc quyền tối thiểu" (chụp ảnh để nộp):
```bash
sudo -u svc_nginx cat /var/www/school/index.html | head -3     # ĐỌC được
sudo -u svc_nginx touch /var/www/school/hack.txt               # GHI → "Permission denied"
sudo -u nobody cat /var/www/school/index.html                  # người lạ → "Permission denied"
```

## C4. Tên miền nội bộ `truong.test` + tường lửa

Chưa có DNS thật nên dùng **file `/etc/hosts`** ("danh bạ" nằm trong máy, được hỏi **trước** DNS).
> Dùng đuôi `.test` vì đây là đuôi **dành riêng cho thử nghiệm**. Tránh `.local` (xung đột với mDNS/Avahi trên Ubuntu).

```bash
echo "127.0.0.1    truong.test" | sudo tee -a /etc/hosts
getent hosts truong.test        # phải ra: 127.0.0.1   truong.test
ping -c 2 truong.test
```
> `getent` đi qua **đúng thứ tự tra cứu của hệ điều hành** (hosts trước, DNS sau) nên là phép kiểm chính xác nhất. `nslookup`/`dig` chỉ hỏi DNS, **không đọc `/etc/hosts`** → đừng dùng chúng để kiểm hosts.

**Cho máy khách khác trong cùng mạng (để KT1 đúng nghĩa):**
1. Trên **máy chủ Ubuntu** lấy IP LAN: `ip -4 addr show` → ví dụ `192.168.1.10`. (Máy ảo: đặt card mạng chế độ **Bridged** để cùng dải mạng với máy khách.)
2. Trên **máy khách** thêm vào file hosts: `192.168.1.10    truong.test`
   - Windows: mở Notepad bằng admin → `C:\Windows\System32\drivers\etc\hosts`
   - Ubuntu/macOS: `echo "192.168.1.10 truong.test" | sudo tee -a /etc/hosts`
3. Mở cổng 80 trên tường lửa máy chủ:
```bash
sudo ufw status                                   # mặc định "inactive" = chưa có tường lửa
sudo ufw allow OpenSSH                            # ⚠️ LÀM TRƯỚC khi bật, kẻo tự khóa mất SSH của mình
sudo ufw allow from 192.168.1.0/24 to any port 80 proto tcp   # chỉ cho mạng LAN vào cổng 80
sudo ufw enable
sudo ufw status numbered
```
(Đổi `192.168.1.0/24` đúng dải mạng của cậu. Sau demo nếu không cần: `sudo ufw delete <số>` hoặc `sudo ufw disable`.)

## C5. Viết cấu hình Nginx

Ubuntu chia cấu hình thành nhiều file nhỏ: `sites-available/` (kho cấu hình) và `sites-enabled/` (cái nào được **bật** thì tạo symlink sang đó).

**Bước 1 – Định dạng log (đặt ở mức `http`, file riêng):**
```bash
sudo tee /etc/nginx/conf.d/school-log.conf >/dev/null <<'EOF'
# Định dạng log: ai (IP), lúc nào, xin gì, mã trả về, bao nhiêu byte, mất bao lâu
log_format school '$remote_addr [$time_local] "$request" '
                  '$status $body_bytes_sent ${request_time}s "$http_user_agent"';
EOF
```

**Bước 2 – Khai báo website:**
```bash
sudo tee /etc/nginx/sites-available/school >/dev/null <<'EOF'
# ===== Website trường học (Đề tài 05) =====
server {
    listen      80;                          # cổng dịch vụ
    server_name truong.test;                 # tên miền nội bộ

    root  /var/www/school;                   # THƯ MỤC NỘI DUNG
    index index.html;

    access_log /var/log/nginx/school.access.log school;   # log truy cập
    error_log  /var/log/nginx/school.error.log  warn;     # log lỗi

    server_tokens off;                       # ẩn số phiên bản Nginx (bảo mật cơ bản)

    gzip on;                                 # nén bớt dữ liệu cho nhẹ
    gzip_types text/css application/javascript application/json image/svg+xml;

    # Ứng dụng React: đường dẫn không phải file thật → trả index.html để React tự xử lý (/grades, /schedule...)
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

    # Chặn file ẩn (.git, .env...)
    location ~ /\. {
        deny all;
    }
}
EOF
```

**Bước 3 – Bật site, tắt site mặc định, đổi tài khoản chạy worker:**
```bash
sudo ln -s /etc/nginx/sites-available/school /etc/nginx/sites-enabled/school
sudo rm /etc/nginx/sites-enabled/default          # chỉ xóa cái symlink, file gốc vẫn còn trong sites-available
sudo sed -i 's/^user .*;/user svc_nginx;/' /etc/nginx/nginx.conf
sudo chown -R svc_nginx:svc_nginx /var/lib/nginx   # thư mục tạm của Nginx, để worker ghi được
grep -n '^user' /etc/nginx/nginx.conf              # phải thấy: user svc_nginx;
```

**Bước 4 – Kiểm tra cú pháp (luôn làm trước khi nạp):**
```bash
sudo nginx -t
```
Phải thấy `syntax is ok` và `test is successful`.

## C6. Chạy & điều khiển dịch vụ

Trên Ubuntu, Nginx đã là **dịch vụ systemd** – không cần cài thêm gì (khác Windows phải dùng NSSM).

```bash
sudo systemctl restart nginx       # nạp lại tài khoản worker mới
systemctl status nginx --no-pager  # Active: active (running)
```
Mở **http://truong.test/** → thấy cổng thông tin đỏ trắng; thử **http://truong.test/tai-lieu/thong-bao-mau.txt**.

| Việc | Lệnh |
|---|---|
| Nạp lại cấu hình, **không ngắt** web | `sudo systemctl reload nginx` |
| Khởi động lại (ngắt vài giây) | `sudo systemctl restart nginx` |
| Dừng / bật | `sudo systemctl stop nginx` / `start` |
| Tự chạy khi bật máy | `sudo systemctl enable nginx` (thường đã bật sẵn) |
| Kiểm tra cấu hình | `sudo nginx -t` |
| Xem log dịch vụ | `journalctl -u nginx -n 50 --no-pager` |
| Xem log truy cập trực tiếp | `sudo tail -f /var/log/nginx/school.access.log` |

## C7. Kiểm tra "ít quyền" – bằng chứng quan trọng nhất

```bash
# 1) Danh sách tiến trình Nginx: master = root, worker = svc_nginx
#    (dùng euser:20 vì cột "user" mặc định bị cắt thành "svc_ngi+")
ps -eo pid,ppid,euser:20,comm | grep '[n]ginx'

# 2) PID nào đang giữ cổng 80
sudo ss -ltnp 'sport = :80'

# 3) Cấu hình đang nạp có đúng user không
sudo nginx -T 2>/dev/null | grep -E '^\s*user '

# 4) Trạng thái dịch vụ + cây tiến trình
systemctl status nginx --no-pager

# 5) CPU/RAM của các tiến trình nginx
ps -C nginx -o pid,user,%cpu,%mem,rss,cmd
```
Kết quả mong đợi (ví dụ):
```
  PID  PPID EUSER                COMMAND
  812     1 root       nginx        ← master (mở cổng 80, đọc cấu hình)
  813   812 svc_nginx  nginx        ← worker (xử lý yêu cầu, đọc file web)
```
> 🧠 **Ba câu ghép lại là "hệ điều hành kiểm soát dịch vụ web":** *cổng 80 do PID nào nghe → tiến trình đó chạy bằng tài khoản nào → tài khoản đó có quyền gì trên thư mục web.*

## C8. "Bắn" nhiều yêu cầu & xem mã HTTP (hỗ trợ SV4)

```bash
curl -I http://truong.test/                              # 200 OK
curl -I http://truong.test/tai-lieu/thong-bao-mau.txt    # 200 OK
curl -I http://truong.test/tai-lieu/khong-co.pdf         # 404 Not Found
curl -I http://truong.test/.git/config                   # 403 Forbidden (bị chặn)
```
200 yêu cầu liên tiếp, đếm theo mã trả về:
```bash
for i in $(seq 1 200); do curl -s -o /dev/null -w "%{http_code}\n" http://truong.test/; done | sort | uniq -c
```
Hoặc dùng công cụ benchmark `ab` (chuẩn của Apache):
```bash
sudo apt install -y apache2-utils
ab -n 1000 -c 20 http://truong.test/        # 1000 yêu cầu, 20 yêu cầu cùng lúc
```
Vừa bắn vừa quan sát ở **một cửa sổ terminal khác**:
```bash
top -u svc_nginx          # CPU/RAM của worker (nhấn q để thoát)
sudo tail -f /var/log/nginx/school.access.log
```
Đếm mã trạng thái trong log:
```bash
sudo awk -F'"' '{split($3,a," "); print a[1]}' /var/log/nginx/school.access.log | sort | uniq -c
```

## C9. Mẫu đoạn giải thích cho hồ sơ (sửa lại bằng lời của cậu)

> **Phần thực hiện của SV2 – Cài Web & cấu hình hệ điều hành (Ubuntu)**
> 1. Em dùng Nginx trên Ubuntu làm HTTP server, lắng nghe cổng 80, phục vụ nội dung tĩnh của website React đã build, đặt tại `/var/www/school`.
> 2. Tên miền nội bộ `truong.test` được ánh xạ về IP máy chủ bằng `/etc/hosts`; hệ điều hành tra hosts trước khi hỏi DNS nên không cần máy chủ DNS thật.
> 3. Nginx chạy dưới systemd. Tiến trình master chạy root (chỉ để mở cổng 80), còn **các worker chạy bằng tài khoản `svc_nginx`** – tài khoản hệ thống, không đăng nhập được. Thư mục web thuộc `root:svc_nginx`, quyền 750/640 nên worker chỉ **đọc** được, không sửa được; người dùng khác không đọc được.
> 4. Hệ điều hành: giữ cổng 80 cho đúng PID, cấp CPU/RAM cho tiến trình, kiểm tra quyền file theo UID/GID của worker, còn `ufw` quyết định gói tin nào được vào cổng 80.
> 5. Bằng chứng: ảnh `nginx -t`, `systemctl status`, `ps -eo pid,user,comm`, `ss -ltnp`, `ls -l`, trình duyệt mở `http://truong.test`.

---

# D. KT1–KT3 và câu hỏi bảo vệ

## KT1 – Truy cập bằng tên miền từ máy khách
Từ máy khách (đã thêm dòng hosts ở C4) mở `http://truong.test/` và `http://truong.test/tai-lieu/thong-bao-mau.txt`. Chụp ảnh **có thanh địa chỉ**.

## KT2 – Hai lỗi "không vào được web", hai bằng chứng khác nhau

| | **Lỗi 1: DNS sai** | **Lỗi 2: dừng dịch vụ** |
|---|---|---|
| Cách tạo | Sửa dòng trong hosts thành `10.255.255.1 truong.test` (hoặc xóa dòng) | `sudo systemctl stop nginx` |
| `getent hosts truong.test` | IP **sai** (hoặc **không ra gì** nếu xóa dòng) | IP **đúng** |
| `ping` | Không phân giải được tên / không tới | Tới được (ping không phụ thuộc web) |
| Trình duyệt | `ERR_NAME_NOT_RESOLVED` (xóa dòng) hoặc treo → `ERR_CONNECTION_TIMED_OUT` (IP sai) | `ERR_CONNECTION_REFUSED` (từ chối ngay) |
| `curl -v http://truong.test/` | `Could not resolve host` hoặc timeout | `Failed to connect ... Connection refused` |
| Cổng 80 trên máy chủ | Vẫn đang nghe | **Không ai nghe** (`sudo ss -ltnp 'sport = :80'` trống) |
| `school.access.log` | **Không có dòng mới** (yêu cầu chưa tới server) | **Không có dòng mới** – nhưng `systemctl status nginx` = `inactive (dead)` |

**Cách nhớ:** DNS sai = *"không biết nhà ở đâu"* (hỏng ở bước 1). Dừng dịch vụ = *"biết nhà, nhưng không ai mở cửa"* (hỏng ở bước 3).
Muốn chứng minh rõ: chụp **cả `getent hosts` lẫn `systemctl status`/`ss`** cho từng lỗi.

> **Bonus – lỗi 3: tường lửa chặn** (`sudo ufw delete allow ...` hoặc `sudo ufw deny 80`): DNS đúng, ping được, dịch vụ **đang chạy**, nhưng máy khách **treo → timeout** (khác "refused" của lỗi 2!). Đây là cách phân biệt *"cửa đóng"* với *"có bảo vệ chặn ngoài cổng"*.

> Xong KT2 nhớ **sửa lại hosts**, `sudo systemctl start nginx`, trả lại luật `ufw`.

## KT3 – Khôi phục từ bản sao (SV3 làm chính)
Web "sống" ở `/var/www/school`. Hỏng/xóa → chép lại bản sao vào đúng thư mục đó (hoặc `rsync` lại từ `dist`).
```bash
sudo rsync -a --delete /var/www/school/ /var/backups/school-demo/        # sao lưu
sudo rm /var/www/school/index.html                                       # phá thử → web trả 404/403
sudo rsync -a --delete /var/backups/school-demo/ /var/www/school/        # khôi phục → web vào lại
```
(`rsync -a` giữ nguyên chủ sở hữu/quyền khi chạy bằng sudo, nên `svc_nginx` vẫn đọc được. Nếu bị 403 sau khi khôi phục, chạy lại 2 lệnh `chown` + `chmod` ở C3.) Rồi F5 trang để xác nhận.

## HTTPS (chỉ cần trình bày phương án)
- HTTP gửi **chữ rõ**, ai nghe lén đều đọc được. **HTTPS = HTTP + mã hóa (TLS)** + xác thực "đúng là trang của trường".
- Triển khai thật: tên miền thật → chứng chỉ miễn phí **Let's Encrypt** (`sudo apt install certbot python3-certbot-nginx` rồi `sudo certbot --nginx`) → Nginx `listen 443 ssl;` + chuyển hướng HTTP→HTTPS.
- Môi trường `.test` không xin được chứng chỉ thật → dùng `mkcert` để minh họa. **Cách làm chi tiết: xem file nâng cao.**

## Câu hỏi chuẩn bị: *"Ping thành công nhưng Website không mở được thì kiểm tra gì tiếp?"*
Ping chỉ chứng minh **mạng tới được máy chủ**. Kiểm tra **theo thứ tự**:
1. **Dịch vụ web có chạy không?** `systemctl status nginx`, `pgrep -a nginx`.
2. **Cổng 80 có ai nghe không?** `sudo ss -ltnp 'sport = :80'`. Không có → chưa chạy/nghe sai cổng.
3. **Tường lửa có chặn không?** Thử trên chính máy chủ (`curl -I http://127.0.0.1/`) rồi từ máy khác. Máy chủ OK mà máy khách không → nghi `ufw`/mạng (`sudo ufw status`).
4. **Gõ đúng chưa?** `http://` hay `https://`, đúng cổng?
5. **Đọc log lỗi:** `sudo tail /var/log/nginx/school.error.log`, `journalctl -u nginx`.
6. **Cấu hình + quyền:** `sudo nginx -t`; đường dẫn `root`; quyền đọc của `svc_nginx` (`namei -l` → lỗi 403).
7. **Tên miền có trỏ đúng IP không?** `getent hosts truong.test` (ping bằng **IP** ok nhưng bằng **tên** ra IP khác = vào nhầm máy). Xóa cache DNS: `sudo resolvectl flush-caches`.

---

# E. Lỗi thường gặp

| Triệu chứng | Nguyên nhân hay gặp | Cách xử lý |
|---|---|---|
| `bind() to 0.0.0.0:80 failed (98: Address already in use)` | Cổng 80 đã bị chương trình khác giữ (Apache, nginx chạy 2 lần…) | `sudo ss -ltnp 'sport = :80'` lấy PID/tên → `sudo systemctl stop apache2` (hoặc tắt nó), hoặc đổi `listen 8080;` |
| Mở `truong.test` ra trang "Welcome to nginx" | Quên bỏ site mặc định hoặc chưa `reload` | `ls /etc/nginx/sites-enabled/` (không còn `default`) → `sudo systemctl reload nginx` |
| Mở `truong.test` không thấy gì / không phân giải | Quên lưu hosts, sai chính tả, cache | `getent hosts truong.test`; `sudo resolvectl flush-caches` |
| **403 Forbidden** | `svc_nginx` không đọc được thư mục (sai chủ/quyền, thiếu `x` ở thư mục cha) | `namei -l /var/www/school/index.html` → chạy lại các lệnh `chown`/`chmod` ở C3 |
| **404** khi F5 ở trang `/grades` | Thiếu `try_files ... /index.html` | Kiểm tra khối `location /` rồi `sudo nginx -t && sudo systemctl reload nginx` |
| Trang trắng, F12 báo lỗi tải `assets/...` | Chép thiếu thư mục `assets` | Build lại + `rsync` lại |
| Sửa web xong vẫn thấy bản cũ | Trình duyệt cache | `Ctrl + F5` |
| `nginx -t` báo `unknown directive` / `unexpected "}"` | Gõ thiếu dấu `;` hoặc `}` | Đọc dòng báo lỗi (có số dòng) và sửa file đó |
| `sudo systemctl start nginx` thất bại | Cấu hình lỗi hoặc `user svc_nginx` chưa tạo | `journalctl -xeu nginx --no-pager -n 30`; `id svc_nginx` |
| Máy khách không vào được | `ufw` chặn / máy ảo dùng NAT / khác mạng | Kiểm tra C4: `ufw status`, card mạng **Bridged**, cùng dải IP |
| `scp`/SSH bị từ chối | Chưa cài/bật SSH | `sudo apt install -y openssh-server && sudo systemctl enable --now ssh` |
| `ufw enable` xong mất kết nối SSH | Quên `ufw allow OpenSSH` trước | Vào bằng màn hình máy ảo/máy thật → `sudo ufw allow OpenSSH` |

---

# F. Từ điển lệnh nhanh

| Lệnh | Ý nghĩa |
|---|---|
| `sudo` | chạy một lệnh với quyền quản trị (root) |
| `systemctl status/start/stop/reload nginx` | điều khiển dịch vụ (systemd) |
| `journalctl -u nginx` | xem log của dịch vụ do systemd thu thập |
| `ss -ltnp` | liệt kê cổng đang nghe (**l**isten, **t**cp, **n**umeric, **p**rocess) |
| `ps -eo pid,user,comm` | liệt kê tiến trình kèm PID và tài khoản chạy |
| `chown owner:group` / `chmod 750` | đổi chủ sở hữu / đổi quyền |
| `namei -l <đường dẫn>` | in quyền của **từng thư mục dọc đường** tới file (tìm nguyên nhân 403) |
| `getent hosts <tên>` | tra tên miền theo đúng thứ tự của hệ điều hành (hosts → DNS) |
| `rsync -a --delete A/ B/` | làm B giống hệt A (sao lưu/triển khai) |
| `ufw allow/deny/status` | quản lý tường lửa |
| `tail -f <file>` | xem file log chạy trực tiếp (Ctrl+C để thoát) |

**Mã trạng thái HTTP:** 200 thành công · 301/302 chuyển hướng · 304 dùng bản đã lưu · 403 bị cấm · 404 không có · 500 server lỗi · 502/503 phía sau/dịch vụ lỗi.
**Đặc quyền tối thiểu:** chỉ cấp đúng quyền cần dùng, không hơn.

---

# G. Backend Auth và PostgreSQL (tùy chọn)

> **Chỉ làm khi cả nhóm đã thống nhất "hướng B"** (có backend đăng nhập + database). Nếu nhóm bám đề (web tĩnh) thì **bỏ qua cả mục G** – phần A–F ở trên là đủ.
> Repo có sẵn `backend/` (NestJS + TypeORM + PostgreSQL, đăng nhập JWT, vai trò admin/teacher/student). Mục này hướng dẫn **đưa nó lên chạy thật ở mức cơ bản**.

## G0. Ai làm gì (tránh làm thay phần của người khác)

| Việc | Của ai |
|---|---|
| Viết/sửa code backend, **nối frontend gọi API** (hiện frontend đang dùng đăng nhập giả bằng `localStorage`) | Người viết backend / frontend |
| **Cài PostgreSQL, chạy backend như dịch vụ ít quyền, Nginx chuyển tiếp `/api`, giữ bí mật cấu hình** | **SV2 (cậu)** |
| Phân quyền theo vai trò, đọc log đăng nhập, sao lưu `pg_dump` | SV3 (dùng thứ cậu dựng) |

**Kiến trúc sau khi xong** – chỉ Nginx lộ ra ngoài, backend và DB chỉ nói chuyện nội bộ:
```
Trình duyệt ──80──► Nginx ──/──────► file tĩnh /var/www/school/current
                      │
                      └──/api/──► 127.0.0.1:3000  NestJS (tài khoản svc_api) ──► 127.0.0.1:5432  PostgreSQL
```
Đường đi 5 bước ở mục A vẫn đúng, chỉ thêm một chặng: Nginx **chuyển tiếp** (reverse proxy) yêu cầu `/api/...` cho backend thay vì tự trả file.

## G1. Cài Node.js (nếu chưa có) và PostgreSQL, tạo database

Node.js ≥ 22 đã cài ở mục C1 (cách 2). Cài PostgreSQL:
```bash
sudo apt install -y postgresql jq openssl
sudo -u postgres psql -Atc "SHOW listen_addresses"       # → localhost  (chỉ nghe nội bộ, đúng ý ta)
sudo ss -ltnp 'sport = :5432'                            # → 127.0.0.1:5432
```
Tạo tài khoản và database **với mật khẩu ngẫu nhiên, không hiện ra màn hình**:
```bash
DBPW=$(openssl rand -hex 16)                             # giữ biến này đến hết G4 (đừng đóng terminal)
echo "CREATE ROLE school LOGIN PASSWORD '$DBPW';" | sudo -u postgres psql
sudo -u postgres createdb -O school school_portal
PGPASSWORD="$DBPW" psql -h 127.0.0.1 -U school -d school_portal -c '\conninfo'     # đăng nhập thử bằng mật khẩu
```
> Cổng PostgreSQL trên Ubuntu là **5432**. Con số `5440` trong `.env.example` chỉ là cổng của bản DB dev trên máy Windows.

## G2. Tài khoản chạy backend và cài code

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin svc_api
```
Build ngay trên Ubuntu (cần bản clone repo ở mục C1, cách 2):
```bash
cd ~/school-portal/backend
npm ci                      # cài cả gói build
npm run build               # ra thư mục dist/
```
Đưa vào `/opt/school-api` (không chứa `.env`, chỉ chứa code đã build + gói chạy thật):
```bash
sudo install -d -o root -g svc_api -m 750 /opt/school-api
sudo rm -rf /opt/school-api/dist
sudo cp -r dist package.json package-lock.json /opt/school-api/
cd /opt/school-api && sudo npm ci --omit=dev --ignore-scripts      # chỉ gói chạy, không chạy script cài đặt bằng root
sudo chown -R root:svc_api /opt/school-api
sudo find /opt/school-api -type d -exec chmod 750 {} +
sudo find /opt/school-api -type f -exec chmod 640 {} +
sudo -u svc_api node -e "console.log('node chạy được dưới svc_api')"
```
(Quyền giống thư mục web: **root là chủ, `svc_api` chỉ đọc** – nếu backend bị chiếm, kẻ xấu không sửa được code của chính nó.)

## G3. File cấu hình bí mật – nằm **ngoài** thư mục code

Mật khẩu DB và khóa ký JWT **không bao giờ** nằm trong repo hay `/opt/school-api`:
```bash
sudo install -d -o root -g svc_api -m 750 /etc/school-api
{
  echo "NODE_ENV=production"
  echo "PORT=3000"
  echo "CORS_ORIGINS=http://truong.test"
  echo "DB_HOST=127.0.0.1"
  echo "DB_PORT=5432"
  echo "DB_USER=school"
  echo "DB_PASSWORD=$DBPW"
  echo "DB_NAME=school_portal"
  echo "DB_SYNCHRONIZE=false"
  echo "JWT_ACCESS_SECRET=$(openssl rand -hex 48)"
  echo "JWT_REFRESH_SECRET=$(openssl rand -hex 48)"
  echo "JWT_ACCESS_TTL_SECONDS=900"
  echo "JWT_REFRESH_TTL_SECONDS=604800"
} | sudo tee /etc/school-api/api.env >/dev/null
sudo chown root:svc_api /etc/school-api/api.env && sudo chmod 640 /etc/school-api/api.env
sudo ls -l /etc/school-api/api.env                       # -rw-r----- root svc_api
```
Khóa JWT sinh ngẫu nhiên đủ dài (≥ 32 ký tự, backend bắt buộc). Chỉ `root` và nhóm `svc_api` đọc được file này.

## G4. Khởi tạo bảng và tài khoản demo (làm **một lần**)

Backend ở chế độ `production` **không tự tạo bảng** (`synchronize` bị tắt) và lệnh `seed` **từ chối chạy ở production** – repo chưa có migration. Nên lần đầu phải chạy khởi tạo ở chế độ `development` **tạm thời**, rồi dịch vụ chính thức chạy `production`:
```bash
cd /opt/school-api
SEEDPW="$(openssl rand -hex 6)Aa1!"
sudo env SEEDPW="$SEEDPW" bash -c '
  set -a; . /etc/school-api/api.env; set +a
  NODE_ENV=development DB_SYNCHRONIZE=true SEED_PASSWORD="$SEEDPW" \
  exec setpriv --reuid=svc_api --regid=svc_api --clear-groups node dist/database/seed.js'
echo "Mật khẩu 3 tài khoản demo: $SEEDPW   ← LƯU Ở NƠI AN TOÀN, KHÔNG ĐƯA LÊN GITHUB"
sudo -u postgres psql -d school_portal -c '\dt'          # phải thấy bảng "users"
unset DBPW
```
> Đừng dùng mật khẩu mặc định `Password123!` của README: ai đọc repo cũng biết, mà trong đó có tài khoản **admin**. Chạy lại lệnh này chỉ tạo tài khoản chưa có, không đổi mật khẩu của tài khoản đã có.

## G5. Dịch vụ systemd cho backend

```bash
sudo tee /etc/systemd/system/school-api.service >/dev/null <<'EOF'
[Unit]
Description=School Portal API (NestJS)
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=simple
User=svc_api
Group=svc_api
WorkingDirectory=/opt/school-api
EnvironmentFile=/etc/school-api/api.env
ExecStart=/usr/bin/node dist/main.js
Restart=on-failure
RestartSec=3
NoNewPrivileges=yes
PrivateTmp=yes
ProtectHome=yes
UMask=0027

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now school-api
systemctl status school-api --no-pager                   # active (running)
journalctl -u school-api -n 20 --no-pager                # log của backend nằm ở journald
curl -s http://127.0.0.1:3000/api/health                 # {"status":"ok"}  (chạm cả DB: SELECT 1)
```
Kiểm tra backend đang nghe ở đâu:
```bash
sudo ss -ltnp 'sport = :3000'
```
⚠️ Nest mặc định nghe **mọi giao diện mạng** (`*:3000`), chứ không chỉ `127.0.0.1`. Nghĩa là nếu tường lửa tắt, máy khác gọi thẳng `http://192.168.1.10:3000` được và **bỏ qua Nginx**. Với `ufw` đã bật ở C4 (chỉ mở cổng 80), cổng 3000 bị chặn. Kiểm tra từ máy khác: `curl -m 5 http://192.168.1.10:3000/api/health` phải **treo/timeout**. (Cách chặn chắc hơn: Cấp 8 trong file nâng cao.)

## G6. Nginx chuyển tiếp `/api` cho backend

Trong khối `server { ... }` của `/etc/nginx/sites-available/school`, thêm **trước** `location / {`:
```nginx
    # Mọi yêu cầu /api/... chuyển cho backend (backend vốn đã có tiền tố /api)
    location /api/ {
        proxy_pass http://127.0.0.1:3000;                 # KHÔNG có "/" ở cuối → giữ nguyên đường dẫn /api/...
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $remote_addr;  # ghi đè, không nối thêm → client không giả được IP
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 30s;
    }
```
```bash
sudo nginx -t && sudo systemctl reload nginx
curl -s http://truong.test/api/health                    # {"status":"ok"}  ← đi qua Nginx rồi tới backend và DB
```
Từ giờ frontend và API **cùng một origin** (`http://truong.test`), không dính lỗi CORS. Backend đã được đặt `CORS_ORIGINS=http://truong.test` cho chắc.

## G7. Kiểm tra bằng `curl` (chứng minh đăng nhập và phân quyền chạy thật)

Đăng nhập admin và lấy token (đổi mật khẩu cho đúng `$SEEDPW` cậu đã lưu):
```bash
login() { curl -s -X POST http://truong.test/api/auth/login -H 'Content-Type: application/json' \
          -d "{\"email\":\"$1\",\"password\":\"$2\"}"; }
code()  { curl -s -o /dev/null -w '%{http_code}\n' "$@"; }

ADMIN=$(login admin@adu.edu.vn "$SEEDPW" | jq -r .accessToken)
STUDENT=$(login minhanh@sv.adu.edu.vn "$SEEDPW" | jq -r .accessToken)

code http://truong.test/api/users                                              # 401  chưa đăng nhập
code -H "Authorization: Bearer $ADMIN"   http://truong.test/api/auth/me        # 200
code -H "Authorization: Bearer $ADMIN"   http://truong.test/api/users          # 200  admin xem được danh sách
code -H "Authorization: Bearer $STUDENT" http://truong.test/api/users          # 403  sinh viên bị cấm
code -X POST http://truong.test/api/auth/login -H 'Content-Type: application/json' \
     -d '{"email":"admin@adu.edu.vn","password":"sai-mat-khau"}'               # 401  sai mật khẩu
```
Ba mã **401 / 403 / 200** là bằng chứng ngắn gọn nhất cho "phân quyền hoạt động". (Đừng thử nhiều hơn 10 lần đăng nhập trong 1 phút: backend giới hạn và trả `429`.)

Bằng chứng ở tầng hệ điều hành (chụp ảnh):
```bash
ps -C node -o pid,ppid,euser:20,cmd                      # node chạy bằng svc_api, không phải root
sudo ss -ltnp 'sport = :3000 or sport = :5432 or sport = :80'
sudo -u postgres psql -d school_portal -c 'SELECT email, role, "isActive" FROM users;'     # xem dữ liệu thật trong DB
```
(Tên cột viết hoa-thường kiểu camelCase nên phải đặt trong dấu nháy kép như trên. Xem đủ cột bằng `\d users`. Cột `passwordHash` là mật khẩu đã băm bcrypt – **đừng** chụp hay chép nó vào báo cáo.)

## G8. Lỗi thường gặp riêng của mục G

| Triệu chứng | Nguyên nhân | Cách xử lý |
|---|---|---|
| Backend thoát ngay, log: `Cấu hình .env không hợp lệ: ...` | Thiếu/sai biến trong `api.env` | `journalctl -u school-api`, đối chiếu tên biến với `.env.example`; JWT secret phải ≥ 32 ký tự |
| `password authentication failed for user "school"` | Mật khẩu trong `api.env` khác mật khẩu trong PostgreSQL | `echo "ALTER ROLE school PASSWORD '<mk-mới>';" \| sudo -u postgres psql` rồi sửa `api.env` cho khớp, `sudo systemctl restart school-api` |
| `ECONNREFUSED 127.0.0.1:5440` | Quên đổi cổng | `DB_PORT=5432` |
| Đăng nhập/gọi API báo 500, log: `relation "users" does not exist` | Chưa chạy bước G4 (production không tự tạo bảng) | Chạy lại G4 |
| Nginx trả **502 Bad Gateway** | Backend không chạy | `systemctl status school-api`; `curl http://127.0.0.1:3000/api/health` |
| Gọi `http://127.0.0.1:3000/api/health` được, qua Nginx lại **404** | `proxy_pass` có `/` ở cuối hoặc `location` sai | Viết đúng `location /api/ { proxy_pass http://127.0.0.1:3000; ... }` |
| `Cannot find module '/opt/school-api/dist/main'` | Thiếu đuôi `.js` (dự án chạy kiểu ESM) | `ExecStart=/usr/bin/node dist/main.js` |
| Nhiều người cùng bị `429 Too Many Requests` khi đăng nhập | Qua Nginx, backend thấy **mọi người cùng IP `127.0.0.1`** nên dùng chung hạn mức 10 lần/phút | Xem "Báo người viết backend" dưới đây |
| `EACCES` khi backend khởi động | Sai quyền thư mục/`api.env` | `namei -l /opt/school-api/dist/main.js`; `ls -l /etc/school-api/api.env` (phải `root:svc_api`, 640) |

## G9. Báo người viết backend/frontend (đừng sửa thay họ)

1. **`trust proxy`:** thêm trong `app.setup.ts` một dòng để Express tin `X-Forwarded-For` từ Nginx (ví dụ `app.set('trust proxy', 'loopback')` với `NestExpressApplication`). Nếu không, hạn mức đăng nhập 10 lần/phút bị dùng chung cho **toàn bộ người dùng**.
2. **Migration:** hiện production không có cách tạo bảng ngoài việc chạy tạm ở `development` như G4. Nên có migration của TypeORM.
3. **Frontend gọi API:** `frontend/src/api` và `lib/auth.tsx` vẫn là dữ liệu giả. Họ cần đổi sang gọi `/api/auth/login`, `/api/auth/me`… (cùng origin nên chỉ cần đường dẫn `/api/...`).
4. **Nghe `127.0.0.1`:** nếu muốn backend không nghe mọi giao diện, `app.listen(port, '127.0.0.1')`.

**Hồ sơ cá nhân thêm cho mục G:** ảnh `systemctl status school-api` + `ps` (node chạy `svc_api`); `ls -l /etc/school-api/api.env`; `ss -ltnp` (3000, 5432); ảnh `curl` 401/200/403; ảnh `curl` cổng 3000 từ máy khác bị chặn; 3–4 câu giải thích vì sao backend và DB không lộ ra ngoài.

---

# H. Phụ lục: TOÀN BỘ LỆNH theo thứ tự (copy-paste)

> Mục này gom **mọi lệnh** ở A–G thành các khối chạy từ trên xuống, để cậu mở file trên máy Ubuntu và dán lần lượt.
> **Quy ước:** `<...>` là chỗ cậu phải điền. Dán **từng khối một**, đọc kết quả rồi mới sang khối sau. Giải thích "vì sao" nằm ở mục C và G.
> Mỗi khối có dòng `# ✅ Mong đợi:` để cậu biết đã đúng chưa.

## H1. Phần cơ bản (mục C)

**H1.1 – Chuẩn bị & công cụ**
```bash
lsb_release -ds && ip -4 addr show | grep inet
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y nginx rsync curl ca-certificates git openssh-server ufw apache2-utils jq openssl htop tree
nginx -v                                           # ✅ nginx version: nginx/1.x
# Node.js ≥ 22 (chỉ cần nếu tự build trên Ubuntu hoặc làm backend) – xem C0.3:
apt-cache policy nodejs | head -n 3
#   Candidate ≥ 22:  sudo apt install -y nodejs npm
#   cũ hơn:          curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash - && sudo apt install -y nodejs
# Phần nâng cao (file Advance):  sudo apt install -y libcap2-bin apparmor apparmor-utils
```

**H1.2 – Lấy thư mục `dist`** (chọn 1)
```powershell
# (A) Trên WINDOWS (PowerShell): build rồi chép sang Ubuntu
cd "C:\Users\ADMIN\OneDrive\Documents\KEY_PROJECT\School Portal\frontend"
npm install
npm run build
scp -r dist <ten_user>@<IP-Ubuntu>:/home/<ten_user>/dist
```
```bash
# (B) Trên UBUNTU: tự build (cần Node ≥ 22)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs git
node -v                                            # ✅ ≥ v22
git clone <link-repo-cua-nhom> ~/school-portal
cd ~/school-portal/frontend && npm ci && npm run build
cp -r dist ~/dist
```

**H1.3 – Thư mục nội dung, tài khoản ít quyền, phân quyền**
```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin svc_nginx
sudo mkdir -p /var/www/school
sudo rsync -a --delete ~/dist/ /var/www/school/
sudo chown -R root:svc_nginx /var/www/school
sudo find /var/www/school -type d -exec chmod 750 {} \;
sudo find /var/www/school -type f -exec chmod 640 {} \;
ls -l /var/www/school && namei -l /var/www/school/index.html
sudo -u svc_nginx cat /var/www/school/index.html | head -3        # ✅ đọc được
sudo -u svc_nginx touch /var/www/school/hack.txt                  # ✅ Permission denied
sudo -u nobody cat /var/www/school/index.html                     # ✅ Permission denied
```

**H1.4 – Tên miền nội bộ + tường lửa**
```bash
echo "127.0.0.1    truong.test" | sudo tee -a /etc/hosts
getent hosts truong.test                           # ✅ 127.0.0.1   truong.test

sudo ufw allow OpenSSH                             # ⚠️ làm TRƯỚC khi bật
sudo ufw allow from <dải-LAN, ví dụ 192.168.1.0/24> to any port 80 proto tcp
sudo ufw enable
sudo ufw status numbered
```
Trên **máy khách** (để KT1): thêm `<IP-máy-chủ>    truong.test` vào file hosts của máy đó.

**H1.5 – Cấu hình Nginx**
```bash
sudo tee /etc/nginx/conf.d/school-log.conf >/dev/null <<'EOF'
log_format school '$remote_addr [$time_local] "$request" '
                  '$status $body_bytes_sent ${request_time}s "$http_user_agent"';
EOF

sudo tee /etc/nginx/sites-available/school >/dev/null <<'EOF'
server {
    listen      80;
    server_name truong.test;

    root  /var/www/school;
    index index.html;

    access_log /var/log/nginx/school.access.log school;
    error_log  /var/log/nginx/school.error.log  warn;

    server_tokens off;

    gzip on;
    gzip_types text/css application/javascript application/json image/svg+xml;

    location / {
        try_files $uri $uri/ /index.html;
    }
    location /assets/ {
        expires 7d;
    }
    location /tai-lieu/ {
        try_files $uri =404;
    }
    location ~ /\. {
        deny all;
    }
}
EOF

sudo ln -s /etc/nginx/sites-available/school /etc/nginx/sites-enabled/school
sudo rm /etc/nginx/sites-enabled/default
sudo sed -i 's/^user .*;/user svc_nginx;/' /etc/nginx/nginx.conf
sudo chown -R svc_nginx:svc_nginx /var/lib/nginx
grep -n '^user' /etc/nginx/nginx.conf              # ✅ user svc_nginx;
sudo nginx -t                                      # ✅ syntax is ok / test is successful
```

**H1.6 – Chạy & kiểm tra**
```bash
sudo systemctl restart nginx
systemctl status nginx --no-pager                  # ✅ active (running)
curl -I http://truong.test/                                # ✅ 200
curl -I http://truong.test/tai-lieu/thong-bao-mau.txt      # ✅ 200
curl -I http://truong.test/tai-lieu/khong-co.pdf           # ✅ 404
curl -I http://truong.test/.git/config                     # ✅ 403
```

**H1.7 – Bằng chứng "ít quyền" (chụp ảnh các lệnh này)**
```bash
ps -eo pid,ppid,euser:20,comm | grep '[n]ginx'     # ✅ master = root, worker = svc_nginx
sudo ss -ltnp 'sport = :80'                        # ✅ PID nào giữ cổng 80
sudo nginx -T 2>/dev/null | grep -E '^\s*user '
systemctl status nginx --no-pager
ps -C nginx -o pid,euser:20,%cpu,%mem,rss,cmd
```

**H1.8 – Bắn thử nhiều yêu cầu (hỗ trợ SV4)**
```bash
for i in $(seq 1 200); do curl -s -o /dev/null -w "%{http_code}\n" http://truong.test/; done | sort | uniq -c
ab -n 1000 -c 20 http://truong.test/
sudo awk -F'"' '{split($3,a," "); print a[1]}' /var/log/nginx/school.access.log | sort | uniq -c
```

**H1.9 – KT2: tạo lỗi rồi nhớ TRẢ LẠI**
```bash
# Lỗi 1: DNS sai
sudo cp /etc/hosts /etc/hosts.bak
sudo sed -i 's/^127.0.0.1\s\+truong.test/10.255.255.1    truong.test/' /etc/hosts
getent hosts truong.test; curl -m 5 -v http://truong.test/ 2>&1 | tail -n 3
sudo mv /etc/hosts.bak /etc/hosts                  # ← TRẢ LẠI

# Lỗi 2: dừng dịch vụ
sudo systemctl stop nginx
getent hosts truong.test; curl -v http://truong.test/ 2>&1 | tail -n 3      # ✅ Connection refused
sudo ss -ltnp 'sport = :80' || echo "không ai nghe cổng 80"
sudo systemctl start nginx                         # ← TRẢ LẠI

# (Bonus) Lỗi 3: tường lửa chặn  → treo/timeout, khác "refused"
sudo ufw deny 80 && curl -m 5 http://<IP-máy-chủ>/ ; sudo ufw delete deny 80
```

**H1.10 – KT3: sao lưu → phá → khôi phục**
```bash
sudo rsync -a --delete /var/www/school/ /var/backups/school-demo/
sudo rm /var/www/school/index.html
curl -s -o /dev/null -w "%{http_code}\n" http://truong.test/               # ✅ không còn 200
sudo rsync -a --delete /var/backups/school-demo/ /var/www/school/
curl -s -o /dev/null -w "%{http_code}\n" http://truong.test/               # ✅ 200
```

## H2. Phần Backend + PostgreSQL (mục G) – chỉ khi nhóm chọn hướng B

**H2.1 – PostgreSQL + role + database**
```bash
sudo apt install -y postgresql jq openssl
DBPW=$(openssl rand -hex 16)                       # ⚠️ giữ terminal này mở đến hết H2.4
echo "CREATE ROLE school LOGIN PASSWORD '$DBPW';" | sudo -u postgres psql
sudo -u postgres createdb -O school school_portal
PGPASSWORD="$DBPW" psql -h 127.0.0.1 -U school -d school_portal -c '\conninfo'
sudo ss -ltnp 'sport = :5432'                      # ✅ chỉ 127.0.0.1
```

**H2.2 – Tài khoản + build + cài backend**
```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin svc_api
cd ~/school-portal/backend && npm ci && npm run build

sudo install -d -o root -g svc_api -m 750 /opt/school-api
sudo rm -rf /opt/school-api/dist
sudo cp -r dist package.json package-lock.json /opt/school-api/
cd /opt/school-api && sudo npm ci --omit=dev --ignore-scripts
sudo chown -R root:svc_api /opt/school-api
sudo find /opt/school-api -type d -exec chmod 750 {} +
sudo find /opt/school-api -type f -exec chmod 640 {} +
sudo -u svc_api node -e "console.log('node chạy được dưới svc_api')"
```

**H2.3 – File bí mật**
```bash
sudo install -d -o root -g svc_api -m 750 /etc/school-api
{
  echo "NODE_ENV=production"
  echo "PORT=3000"
  echo "CORS_ORIGINS=http://truong.test"
  echo "DB_HOST=127.0.0.1"
  echo "DB_PORT=5432"
  echo "DB_USER=school"
  echo "DB_PASSWORD=$DBPW"
  echo "DB_NAME=school_portal"
  echo "DB_SYNCHRONIZE=false"
  echo "JWT_ACCESS_SECRET=$(openssl rand -hex 48)"
  echo "JWT_REFRESH_SECRET=$(openssl rand -hex 48)"
  echo "JWT_ACCESS_TTL_SECONDS=900"
  echo "JWT_REFRESH_TTL_SECONDS=604800"
} | sudo tee /etc/school-api/api.env >/dev/null
sudo chown root:svc_api /etc/school-api/api.env && sudo chmod 640 /etc/school-api/api.env
sudo ls -l /etc/school-api/api.env                 # ✅ -rw-r----- root svc_api
```

**H2.4 – Khởi tạo bảng + tài khoản demo (một lần)**
```bash
cd /opt/school-api
SEEDPW="$(openssl rand -hex 6)Aa1!"
sudo env SEEDPW="$SEEDPW" bash -c '
  set -a; . /etc/school-api/api.env; set +a
  NODE_ENV=development DB_SYNCHRONIZE=true SEED_PASSWORD="$SEEDPW" \
  exec setpriv --reuid=svc_api --regid=svc_api --clear-groups node dist/database/seed.js'
echo "Mật khẩu 3 tài khoản demo: $SEEDPW   ← LƯU LẠI, KHÔNG ĐƯA LÊN GITHUB"
sudo -u postgres psql -d school_portal -c '\dt'    # ✅ có bảng users
unset DBPW
```

**H2.5 – Dịch vụ `school-api`**
```bash
sudo tee /etc/systemd/system/school-api.service >/dev/null <<'EOF'
[Unit]
Description=School Portal API (NestJS)
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=simple
User=svc_api
Group=svc_api
WorkingDirectory=/opt/school-api
EnvironmentFile=/etc/school-api/api.env
ExecStart=/usr/bin/node dist/main.js
Restart=on-failure
RestartSec=3
NoNewPrivileges=yes
PrivateTmp=yes
ProtectHome=yes
UMask=0027

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now school-api
systemctl status school-api --no-pager             # ✅ active (running)
curl -s http://127.0.0.1:3000/api/health           # ✅ {"status":"ok"}
sudo ss -ltnp 'sport = :3000'
```

**H2.6 – Nginx chuyển tiếp `/api`**
Mở `sudo nano /etc/nginx/sites-available/school`, thêm **trước** `location / {`:
```nginx
    location /api/ {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 30s;
    }
```
```bash
sudo nginx -t && sudo systemctl reload nginx
curl -s http://truong.test/api/health              # ✅ {"status":"ok"}
```

**H2.7 – Kiểm tra đăng nhập & phân quyền**
```bash
login() { curl -s -X POST http://truong.test/api/auth/login -H 'Content-Type: application/json' \
          -d "{\"email\":\"$1\",\"password\":\"$2\"}"; }
code()  { curl -s -o /dev/null -w '%{http_code}\n' "$@"; }
ADMIN=$(login admin@adu.edu.vn "$SEEDPW" | jq -r .accessToken)
STUDENT=$(login minhanh@sv.adu.edu.vn "$SEEDPW" | jq -r .accessToken)

code http://truong.test/api/users                                              # ✅ 401
code -H "Authorization: Bearer $ADMIN"   http://truong.test/api/auth/me        # ✅ 200
code -H "Authorization: Bearer $ADMIN"   http://truong.test/api/users          # ✅ 200
code -H "Authorization: Bearer $STUDENT" http://truong.test/api/users          # ✅ 403
code -X POST http://truong.test/api/auth/login -H 'Content-Type: application/json' \
     -d '{"email":"admin@adu.edu.vn","password":"sai"}'                        # ✅ 401

ps -C node -o pid,ppid,euser:20,cmd                # ✅ svc_api
sudo ss -ltnp 'sport = :3000 or sport = :5432 or sport = :80'
sudo -u postgres psql -d school_portal -c 'SELECT email, role, "isActive" FROM users;'
```
Từ **máy khác**: `curl -m 5 http://<IP-máy-chủ>:3000/api/health` ✅ phải treo/timeout.

## H3. Khi sửa web / dọn dẹp sau demo

```bash
# Cập nhật web sau khi build lại dist
sudo rsync -a --delete ~/dist/ /var/www/school/
sudo chown -R root:svc_nginx /var/www/school
sudo find /var/www/school -type d -exec chmod 750 {} \;
sudo find /var/www/school -type f -exec chmod 640 {} \;

# Xem log / trạng thái nhanh
sudo tail -f /var/log/nginx/school.access.log
journalctl -u nginx -n 50 --no-pager
journalctl -u school-api -n 50 --no-pager

# Sau demo: gỡ luật tường lửa không cần nữa
sudo ufw status numbered     # rồi: sudo ufw delete <số>
```

---

> ⚠️ **Ghi chú trung thực:** các lệnh được viết theo cách Ubuntu 22.04/24.04 và Nginx từ kho `apt` hoạt động, nhưng **chưa được chạy thử trên máy cậu** (tớ chưa cài Nginx, đổi hosts hay bật tường lửa thay cậu). Nếu một lệnh báo lỗi, đừng bỏ qua – chụp màn hình lỗi, đối chiếu mục E; rất nhiều khi lỗi chính là một "bằng chứng" có thể đưa vào hồ sơ. Đừng đưa mật khẩu/khóa riêng lên GitHub.
