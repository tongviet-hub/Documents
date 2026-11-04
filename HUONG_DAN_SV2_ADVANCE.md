# Đề tài 05 – SV2 **Nâng cao (MAX)**: chạy web không root, sandbox, MAC, tinh chỉnh HĐH

> **Chỉ dành cho phần việc của SV2** – *"cài web server, cấu hình hệ điều hành, tên miền nội bộ, thư mục nội dung, quyền tiến trình"* – nhưng đào **đến tận đáy**.
> Điều kiện: đã làm xong [HUONG_DAN_SV2_UBUNTU.md](HUONG_DAN_SV2_UBUNTU.md) (Nginx chạy được `http://truong.test/` trên Ubuntu, có `svc_nginx`, có `/var/www/school`).
> Bản Windows xem [HUONG_DAN_SV2.md](HUONG_DAN_SV2.md); bảng quy đổi sang Windows ở [mục 11](#11-nếu-cậu-vẫn-ở-windows).
> Máy Ubuntu nên là **máy ảo thật** (VirtualBox/Hyper-V/VMware) hoặc VPS. WSL2 **không** đủ (thiếu AppArmor, `mount`, `ufw` đầy đủ).

---

## 0. Ranh giới: cái gì của SV2, cái gì KHÔNG

File này **không** đụng vào phần của bạn khác. Ai làm phần nào theo đúng đề:

| Việc | Của ai | Trong file này |
|---|---|---|
| DNS, sơ đồ luồng truy cập | **SV1** | ❌ Không làm. Chỉ dùng tên miền SV1 chốt (`truong.test`, `www.truong.test`) |
| Khóa khu vực quản trị, **sao lưu & phục hồi**, **phương án HTTPS**, xoay vòng log | **SV3** | ❌ Không làm. Chỉ báo cho SV3 *đường dẫn mới* (mục 12) |
| Đo mã HTTP/CPU/RAM khi tải, benchmark, đọc & phân tích log, chẩn đoán KT2 | **SV4** | ❌ Không làm. Chỉ đưa SV4 *tên dịch vụ/PID* và cảnh báo giới hạn tài nguyên (mục 12) |
| Code backend, nối frontend với API (hiện frontend đang đăng nhập giả) | Người viết backend/frontend | ❌ Không làm, chỉ *báo* (mục G9 file Ubuntu) |
| Phân quyền theo vai trò ở tầng ứng dụng, đọc log đăng nhập | SV3 | ❌ Không làm. Cậu chỉ dựng nền để SV3 dùng |
| Web server, HĐH, tên miền nội bộ, thư mục nội dung, **quyền tiến trình** (gồm dịch vụ backend và PostgreSQL nếu nhóm chọn hướng B) | **SV2 (cậu)** | ✅ Toàn bộ file này |

### Bản đồ độ khó

| Cấp | Chủ đề | Độ khó | Ăn điểm ở đâu |
|---|---|---|---|
| 1 | Triển khai chuyên nghiệp: tách vai trò, atomic, manifest, rollback | ★★★ | Thư mục nội dung, quyền |
| 2 | **Chạy TOÀN BỘ Nginx không root** (capability, unit riêng) | ★★★★ | Quyền tiến trình |
| 3 | Sandbox systemd tối đa + cgroup | ★★★★ | Quyền tiến trình, HĐH cấp tài nguyên |
| 4 | **AppArmor** – kiểm soát truy cập bắt buộc (MAC) | ★★★★★ | Quyền tiến trình |
| 5 | Tinh chỉnh HĐH: giới hạn file, backlog, mount chỉ-đọc, tín hiệu | ★★★★★ | Cấu hình hệ điều hành |
| 6 | Tự chấm điểm phần SV2 bằng script | ★★★ | Bằng chứng nộp |
| 7 | Phòng lab lỗi (gây lỗi → chẩn đoán → sửa) | ★★★★ | Bảo vệ |
| 8 | **Backend Auth + PostgreSQL**: vận hành an toàn (chỉ khi nhóm chọn hướng B, cần làm mục G file Ubuntu trước) | ★★★★★ | Quyền tiến trình, cô lập dịch vụ |
| + | Bonus: so sánh Nginx ↔ Apache theo tiêu chí của SV2 | ★★ | Mở rộng của đề |

> Làm **lần lượt từng cấp**; xong cấp nào chạy `verify-sv2.sh` (Cấp 6) và chụp ảnh ngay. Cấp sau xây trên cấp trước.

**Quy ước:** IP máy chủ `192.168.1.10`; tài khoản chạy dịch vụ `svc_nginx`; nội dung `/var/www/school`; dịch vụ mới `nginx-school`. Đổi theo máy của cậu.

## Cài thêm gói cho file này (máy Ubuntu mới)

Phần cơ bản (Nginx, rsync, curl, git, ufw, jq, openssl…) đã cài ở **mục C0 của file Ubuntu**. Ở đây chỉ cần thêm:
```bash
sudo apt update
sudo apt install -y libcap2-bin apparmor apparmor-utils
# Chỉ khi làm Cấp 8 (backend):  sudo apt install -y postgresql   và Node.js ≥ 22 (xem C0.3)
# Chỉ khi làm Bonus so sánh Apache: sudo apt install -y apache2  (cài xong tắt ngay: sudo systemctl disable --now apache2)

# Kiểm tra
capsh --print | head -n 1 ; aa-status | head -n 2 ; systemd-analyze --version | head -n 1
```
Các công cụ khác của file này (`systemd-analyze`, `setpriv`, `mountpoint`, `findmnt`, `nsenter`, `ss`, `ps`, `sysctl`) đã có sẵn trong Ubuntu.
> **Ubuntu bản mới:** nếu `sudo --version` hiện `sudo-rs`, một vài tùy chọn hiếm có thể khác bản sudo cũ. Lệnh trong file này đã chọn cách chạy được cả hai; gặp lỗi `unrecognized option` thì chụp gửi tớ.

---

# Cấp 1 – Triển khai chuyên nghiệp (★★★)

**Vấn đề của cách cơ bản** (`rsync --delete` thẳng vào thư mục đang chạy):
1. Vài giây "nửa cũ nửa mới" → khách gặp trang vỡ.
2. Hỏng thì **không quay lại được**.
3. Ai cũng cần `sudo` toàn quyền để cập nhật web → sai nguyên tắc đặc quyền tối thiểu.
4. Không biết file có bị **sửa lén** hay không.

**Giải pháp:** mỗi bản build ở một thư mục riêng, đổi con trỏ `current` nguyên tử; người đưa web lên (`deploy`) **khác** người chạy web (`svc_nginx`) **khác** root; kèm sổ băm (manifest) để kiểm chứng toàn vẹn.

```
/var/www/school/                    root:svc_nginx  750
├── releases/                       root:svc_nginx  750
│   ├── 20261104-091500/            root:svc_nginx  750 (file 640)
│   └── 20261104-103000/
├── manifests/                      root:root       750   ← sổ băm, ngoài tầm với của web
│   └── 20261104-103000.sha256
└── current ──► releases/20261104-103000                  ← Nginx trỏ root vào đây
```

## 1.1 Tách vai trò

```bash
sudo useradd -m -s /bin/bash deploy                 # đại diện "người đưa web lên" (hoặc dùng luôn tài khoản của cậu)
sudo install -d -o root -g svc_nginx -m 750 /var/www/school /var/www/school/releases
sudo install -d -o root -g root      -m 750 /var/www/school/manifests
sudo apt install -y rsync curl
```

**Bảng quyền** (cậu phải điền/giải thích được bảng này khi bảo vệ):

| Đối tượng | Đọc web | Sửa web | Đổi `current` / reload | Đọc cấu hình | Ghi log |
|---|---|---|---|---|---|
| `root` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `deploy` | ❌ | ❌ (chỉ qua script) | ❌ (chỉ qua script) | ❌ | ❌ |
| `svc_nginx` (chạy web) | ✅ | ❌ | ❌ | ✅ | ✅ (thư mục log riêng) |
| Người khác | ❌ | ❌ | ❌ | ❌ | ❌ |

## 1.2 Script triển khai

Lưu `school-deploy.sh` rồi cài vào hệ thống (**chủ root, deploy không sửa được**):
```bash
sudo install -o root -g root -m 755 school-deploy.sh /usr/local/sbin/school-deploy.sh
```
```bash
#!/usr/bin/env bash
# /usr/local/sbin/school-deploy.sh
# Dùng:  sudo school-deploy.sh deploy <thư-mục-dist>
#        sudo school-deploy.sh verify                 (bản đang chạy có bị sửa lén không?)
#        sudo school-deploy.sh rollback [tên-bản]     (quay về bản trước, hoặc bản chỉ định)
set -euo pipefail
export LC_ALL=C
BASE=/var/www/school; REL_DIR=$BASE/releases; MAN_DIR=$BASE/manifests; KEEP=5
URL=${SCHOOL_URL:-http://truong.test/}
# Tự nhận biết đang dùng dịch vụ nào (bản cơ bản = nginx, sau Cấp 2 = nginx-school)
if systemctl is-active --quiet nginx-school; then SVC=nginx-school; else SVC=nginx; fi

die()    { echo "LỖI: $*" >&2; exit 1; }
cur()    { readlink -f "$BASE/current" 2>/dev/null || true; }
swap()   { ln -sfn "$1" "$BASE/current.tmp"; mv -T "$BASE/current.tmp" "$BASE/current"; systemctl reload "$SVC"; }
# Cấp 5: nếu thư mục web đang bị khóa chỉ-đọc, mở khóa tạm thời rồi khóa lại khi thoát
unlock() { mountpoint -q "$BASE" && mount -o remount,bind,rw "$BASE" || true; }
lock()   { mountpoint -q "$BASE" && mount -o remount,bind,ro,nosuid,nodev,noexec "$BASE" || true; }

deploy() {
  local SRC=${1:?thiếu thư mục dist}
  [ -f "$SRC/index.html" ] || die "$SRC không phải bản build hợp lệ (thiếu index.html)"
  # Script chạy bằng root nhưng đường dẫn do deploy đưa vào → không tin: từ chối symlink
  if find "$SRC" -type l | grep -q .; then die "dist chứa symlink – từ chối"; fi

  unlock; trap lock EXIT
  local NEW="$REL_DIR/$(date +%Y%m%d-%H%M%S)" PREV; PREV=$(cur)
  install -d -o root -g svc_nginx -m 750 "$REL_DIR" "$NEW"
  install -d -o root -g root      -m 750 "$MAN_DIR"

  rsync -a --chown=root:svc_nginx --chmod=D750,F640 "$SRC"/ "$NEW"/

  # Sổ băm lấy từ NGUỒN, rồi kiểm tra bản đã chép == nguồn (bắt lỗi chép hỏng giữa chừng)
  local M; M=$(mktemp)
  ( cd "$SRC" && find . -type f -print0 | sort -z | xargs -0 sha256sum ) > "$M"
  ( cd "$NEW" && sha256sum -c --quiet "$M" ) || die "bản chép KHÁC nguồn → hủy, chưa đổi bản"
  install -o root -g root -m 640 "$M" "$MAN_DIR/$(basename "$NEW").sha256"; rm -f "$M"

  swap "$NEW"

  # Kiểm tra sức khỏe; hỏng thì tự quay về bản trước
  local code; code=$(curl -s -o /dev/null -m 5 -w '%{http_code}' "$URL" || true)
  if [ "$code" != "200" ]; then
    [ -n "$PREV" ] && swap "$PREV"
    die "health-check $URL = '$code' → ĐÃ ROLLBACK về ${PREV:-<không có>}"
  fi
  ls -1dt "$REL_DIR"/*/ | tail -n +$((KEEP + 1)) | xargs -r rm -rf --     # chỉ giữ $KEEP bản gần nhất
  echo "OK: đang chạy $NEW (bản trước: ${PREV:-không có}) – dịch vụ: $SVC"
}

verify() {
  local C M; C=$(cur); [ -n "$C" ] || die "chưa có bản nào"
  M="$MAN_DIR/$(basename "$C").sha256"; [ -f "$M" ] || die "không có sổ băm cho $C"
  ( cd "$C" && sha256sum -c --quiet "$M" ) || { echo "❌ CÓ FILE BỊ SỬA/MẤT trong $C"; exit 2; }
  local extra; extra=$( cd "$C" && find . -type f | sort | comm -13 <(awk '{print $2}' "$M" | sort) - )
  [ -z "$extra" ] || { echo "❌ CÓ FILE LẠ (không có trong sổ băm):"; echo "$extra"; exit 2; }
  echo "✅ $(basename "$C") nguyên vẹn"
}

rollback() {
  local T=${1:-} P
  if [ -n "$T" ]; then P="$REL_DIR/$T"; else P=$(ls -1dt "$REL_DIR"/*/ | sed -n 2p); P=${P%/}; fi
  [ -d "$P" ] || die "không có bản để quay về"
  unlock; trap lock EXIT; swap "$P"; echo "Đã quay về: $P"
}

case "${1:-}" in
  deploy)   shift; deploy "$@" ;;
  verify)   verify ;;
  rollback) shift; rollback "$@" ;;
  *) sed -n '2,6p' "$0"; exit 1 ;;
esac
```

## 1.3 Cho `deploy` đúng **một** quyền (sudoers tối thiểu)
```bash
sudo tee /etc/sudoers.d/school-deploy >/dev/null <<'EOF'
# deploy chỉ được chạy ĐÚNG script này bằng root – không shell, không lệnh nào khác
deploy ALL=(root) NOPASSWD: /usr/local/sbin/school-deploy.sh
EOF
sudo chmod 440 /etc/sudoers.d/school-deploy
sudo visudo -cf /etc/sudoers.d/school-deploy         # phải: "parsed OK" (sai cú pháp sudoers có thể khóa mất sudo!)
```
Đổi `root` trong site Nginx cơ bản sang `current`:
```bash
sudo sed -i -E 's#^(\s*root\s+)/var/www/school;#\1/var/www/school/current;#' /etc/nginx/sites-available/school
```
Chuyển `dist` cho `deploy` rồi triển khai **bằng tài khoản deploy**:
```bash
sudo cp -r ~/dist /home/deploy/dist && sudo chown -R deploy: /home/deploy/dist
sudo -u deploy sudo -n /usr/local/sbin/school-deploy.sh deploy /home/deploy/dist
sudo nginx -t && sudo systemctl reload nginx
curl -I http://truong.test/                                  # 200
```

## 1.4 Chứng minh (chụp ảnh)
```bash
sudo -l -U deploy                                            # chỉ liệt kê ĐÚNG 1 lệnh được phép
sudo -u deploy sudo -n systemctl stop nginx                  # ❌ "a password is required"  ← deploy không dừng được dịch vụ
sudo -u deploy sudo -n cat /etc/shadow                       # ❌ bị từ chối
readlink -f /var/www/school/current                          # đang chạy bản nào
sudo school-deploy.sh verify                                 # ✅ nguyên vẹn

# Sửa lén 1 file → verify phải phát hiện
sudo sh -c 'echo "<!-- hacked -->" >> /var/www/school/current/index.html'
sudo school-deploy.sh verify                                 # ❌ CÓ FILE BỊ SỬA
#   (rollback cần ≥ 2 bản trong releases/: nếu mới deploy 1 lần, hãy deploy thêm 1 lần nữa trước khi sửa lén)
sudo school-deploy.sh rollback && sudo school-deploy.sh verify   # quay về bản sạch → ✅

# Health-check tự rollback: cố tình cho kiểm tra vào URL không tồn tại
sudo SCHOOL_URL=http://truong.test/tai-lieu/khong-co.pdf school-deploy.sh deploy /home/deploy/dist
# → "health-check ... = '404' → ĐÃ ROLLBACK", web vẫn 200

# Không gián đoạn: chạy ở terminal 2 trong lúc deploy, KHÔNG được có mã nào khác 200
while true; do curl -s -o /dev/null -w '%{http_code} ' http://truong.test/; sleep 0.05; done
```

> 🧠 **Điểm ăn tiền:** `mv -T` (đổi tên đè) là thao tác **nguyên tử** của hệ thống tập tin – tại mỗi thời điểm `current` trỏ vào *hoặc* bản cũ *hoặc* bản mới, không bao giờ ở giữa. Không cần khóa, không cần dừng dịch vụ.

---

# Cấp 2 – Chạy TOÀN BỘ Nginx không root (★★★★)

**Hiện trạng:** master chạy **root**, chỉ worker là `svc_nginx`. Nghĩa là *một* tiến trình quyền lực vẫn nằm trong dịch vụ internet. Cấp này **bỏ hẳn root**: mọi tiến trình Nginx đều là `svc_nginx`, và để mở được cổng 80 ta cấp **đúng một "siêu năng lực" nhỏ** (capability) thay vì cả quyền root.

> 🧠 **Capability:** Linux chia quyền root thành ~40 mảnh nhỏ. `CAP_NET_BIND_SERVICE` = "được mở cổng < 1024", *chỉ thế thôi* – không đọc được file của người khác, không tải module, không đổi chủ file.

## 2.1 Cấu hình riêng cho bản chạy không root

Tách hẳn khỏi `/etc/nginx` của gói để không ảnh hưởng bản cơ bản (rollback dễ):
```bash
sudo install -d -o root -g svc_nginx -m 750 /etc/nginx-school
sudo tee /etc/nginx-school/nginx.conf >/dev/null <<'EOF'
# ===== Nginx chạy KHÔNG root – Website trường học =====
# Không có chỉ thị "user": cả master lẫn worker đã là svc_nginx do systemd quyết định.
worker_processes  auto;
worker_rlimit_nofile 8192;                     # (Cấp 5) trần file mở của mỗi worker
pid        /run/nginx-school/nginx.pid;
error_log  /var/log/nginx-school/error.log warn;

events {
    worker_connections 4096;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    server_tokens off;

    # Mọi thư mục tạm nằm trong vùng dịch vụ được phép ghi
    client_body_temp_path /var/lib/nginx-school/body;
    proxy_temp_path       /var/lib/nginx-school/proxy;
    fastcgi_temp_path     /var/lib/nginx-school/fastcgi;
    uwsgi_temp_path       /var/lib/nginx-school/uwsgi;
    scgi_temp_path        /var/lib/nginx-school/scgi;

    log_format school '$remote_addr [$time_local] "$request" '
                      '$status $body_bytes_sent ${request_time}s "$http_user_agent"';
    access_log /var/log/nginx-school/access.log school;

    sendfile on;
    keepalive_timeout 30;
    gzip on;
    gzip_types text/css application/javascript application/json image/svg+xml;

    # (1) BẮT-HẾT: truy cập bằng IP hoặc tên lạ → đóng kết nối, không lộ site nào
    server {
        listen 80 default_server backlog=1024;
        server_name _;
        return 444;                              # mã đặc biệt của Nginx: đóng kết nối không trả lời
    }

    # (2) Site thật
    server {
        listen 80;
        server_name truong.test www.truong.test;

        root  /var/www/school/current;
        index index.html;
        autoindex off;                           # không liệt kê thư mục
        disable_symlinks if_not_owner from=$document_root;   # không đi theo symlink lạ dưới root

        # index.html + mọi đường dẫn React: luôn hỏi lại server (nhờ ETag, thường chỉ tốn 304)
        location / {
            add_header Cache-Control "no-cache" always;
            try_files $uri $uri/ /index.html;
        }

        # File build có băm tên (assets/index-abc123.js) → đổi nội dung là đổi tên → cache 1 năm an toàn
        location /assets/ {
            add_header Cache-Control "public, max-age=31536000, immutable" always;
            try_files $uri =404;
        }

        # Tài liệu tải về: 404 thật nếu không có
        location /tai-lieu/ {
            add_header Cache-Control "no-cache" always;
            try_files $uri =404;
        }

        # Chặn file ẩn (.git, .env…)
        location ~ /\. { deny all; }
    }
}
EOF
sudo chown root:svc_nginx /etc/nginx-school/nginx.conf && sudo chmod 640 /etc/nginx-school/nginx.conf
```
> **Vì sao cache khác nhau?** React/Vite đặt tên file build kèm mã băm nội dung. `assets/*` không bao giờ đổi nội dung ⇒ cho trình duyệt nhớ 1 năm. `index.html` thì **phải luôn kiểm tra lại**, nếu không khách kẹt bản cũ sau khi deploy.

## 2.2 Dịch vụ systemd riêng

```bash
sudo tee /etc/systemd/system/nginx-school.service >/dev/null <<'EOF'
[Unit]
Description=Nginx - School Portal (chạy KHÔNG root)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=svc_nginx
Group=svc_nginx
# systemd tự tạo & gán chủ các thư mục này cho svc_nginx:
RuntimeDirectory=nginx-school      # /run/nginx-school      (pid)
StateDirectory=nginx-school        # /var/lib/nginx-school  (thư mục tạm)
LogsDirectory=nginx-school         # /var/log/nginx-school  (log)

ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx-school/nginx.conf
ExecStart=/usr/sbin/nginx -g "daemon off;" -c /etc/nginx-school/nginx.conf
ExecReload=/usr/sbin/nginx -t -c /etc/nginx-school/nginx.conf
ExecReload=/bin/kill -HUP $MAINPID
KillSignal=SIGQUIT                 # tắt "êm": xử lý nốt yêu cầu đang dở
KillMode=mixed
TimeoutStopSec=15
Restart=on-failure
RestartSec=2s
LimitNOFILE=16384                  # (Cấp 5) trần file mở

# Quyền: ĐÚNG MỘT capability – được mở cổng 80
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
NoNewPrivileges=yes
UMask=0027

[Install]
WantedBy=multi-user.target
EOF
```
Chuyển sang bản mới (phải **tắt bản cũ** vì cùng cổng 80):
```bash
sudo systemctl disable --now nginx            # tắt Nginx của gói (cấu hình vẫn giữ nguyên để rollback)
sudo systemctl daemon-reload
sudo systemctl enable --now nginx-school
systemctl status nginx-school --no-pager      # active (running)
curl -I http://truong.test/                   # 200
```
> ⚠️ **Kiểm tra cú pháp từ nay dùng `sudo -u svc_nginx nginx -t -c /etc/nginx-school/nginx.conf`** (không phải `sudo nginx -t`). Chạy bằng root sẽ **tạo file log/thư mục tạm thuộc root**, lần sau `svc_nginx` không ghi được → dịch vụ lỗi `Permission denied`. (`systemctl start` đã tự chạy `-t` bằng đúng `svc_nginx`.)
> Trên Nginx 1.18 (Ubuntu 22.04) có thể thấy cảnh báo `[alert] could not open error log file ... /var/log/nginx/error.log` lúc khởi động – vô hại, do đường dẫn log mặc định biên dịch sẵn chưa kịp bị cấu hình thay thế. Ubuntu 24.04 (Nginx 1.24) thêm `-e /var/log/nginx-school/error.log` vào `ExecStart`/`ExecStartPre` là hết.

## 2.3 Chứng minh (bằng chứng quan trọng nhất của cả file)

```bash
# 1) TẤT CẢ tiến trình Nginx đều là svc_nginx – KỂ CẢ MASTER (trước đây master là root)
ps -C nginx -o pid,ppid,euser:20,cmd
#   (dùng euser:20 vì cột "user" mặc định bị cắt thành "svc_ngi+")

# 2) Cổng 80 do PID nào giữ
sudo ss -ltnp 'sport = :80'

# 3) Tiến trình chỉ còn ĐÚNG 1 capability
sudo apt install -y libcap2-bin
PID=$(systemctl show -p MainPID --value nginx-school)
grep -E '^Cap(Prm|Eff|Amb|Bnd)' /proc/$PID/status
capsh --decode=0000000000000400                  # → 0x0000000000000400=cap_net_bind_service
grep NoNewPrivs /proc/$PID/status                # NoNewPrivs: 1

# 4) ĐỐI CHỨNG: không có capability thì mở cổng 80 thất bại
sudo systemctl stop nginx-school
sudo setpriv --reuid=svc_nginx --regid=svc_nginx --clear-groups \
     /usr/sbin/nginx -g 'daemon off;' -c /etc/nginx-school/nginx.conf 2>&1 | grep -i bind
#   → bind() to 0.0.0.0:80 failed (13: Permission denied)      ← chính là lý do Nginx thường phải chạy master bằng root
sudo systemctl start nginx-school

# 5) Đăng nhập thử tài khoản dịch vụ: không được
sudo su - svc_nginx                              # → "This account is currently not available."
```
**Câu giải thích hoàn chỉnh khi bảo vệ:** *"Cổng 80 < 1024 nên mặc định cần root. Em không cấp root mà cấp riêng capability `CAP_NET_BIND_SERVICE`, nên chiếm được Nginx cũng chỉ mở được cổng, không đọc được file người khác, không cài driver, không đổi quyền."*

---

# Cấp 3 – Sandbox systemd tối đa + cgroup (★★★★)

Cấp 2 hạ quyền **tài khoản**. Cấp 3 **giam luôn môi trường**: dịch vụ chỉ thấy phần hệ thống nó cần, chỉ được gọi những lời gọi hệ thống nó cần, và có trần CPU/RAM (cgroup).

## 3.1 Chấm điểm "trước"
```bash
sudo apt install -y libcap2-bin
systemd-analyze security nginx-school.service --no-pager | tail -n 1
#   → "Overall exposure level for nginx-school.service: 6.x MEDIUM"  → CHỤP ẢNH
```

## 3.2 Thêm lớp sandbox bằng drop-in riêng
```bash
sudo mkdir -p /etc/systemd/system/nginx-school.service.d
sudo tee /etc/systemd/system/nginx-school.service.d/10-sandbox.conf >/dev/null <<'EOF'
[Service]
# --- Nhìn thấy gì: hệ thống tập tin gần như chỉ-đọc ---
ProtectSystem=strict               # toàn bộ / chỉ-đọc (trừ Runtime/State/LogsDirectory ở unit chính)
ProtectHome=yes                    # không thấy /home
PrivateTmp=yes                     # /tmp riêng
PrivateDevices=yes                 # không thấy thiết bị thật (/dev chỉ có null, zero, random…)
InaccessiblePaths=-/root -/boot -/mnt -/media -/srv -/opt     # dấu "-" = bỏ qua nếu thư mục không tồn tại
ProtectProc=invisible              # không thấy tiến trình của người khác trong /proc
ProcSubset=pid
# --- Không đụng được nhân ---
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectKernelLogs=yes
ProtectControlGroups=yes
ProtectClock=yes
ProtectHostname=yes
# --- Hạn chế hành vi ---
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX    # chỉ dùng mạng IPv4/IPv6/socket nội bộ
RestrictNamespaces=yes
RestrictRealtime=yes
RestrictSUIDSGID=yes
LockPersonality=yes
MemoryDenyWriteExecute=yes         # không cho vùng nhớ vừa ghi vừa chạy (chặn nhiều kiểu tiêm mã)
SystemCallArchitectures=native
SystemCallFilter=@system-service   # chỉ cho nhóm syscall "dịch vụ bình thường"
SystemCallErrorNumber=EPERM
# --- Trần tài nguyên (cgroup) ---
MemoryMax=300M
CPUQuota=80%
TasksMax=200
EOF
sudo systemctl daemon-reload && sudo systemctl restart nginx-school
systemctl status nginx-school --no-pager && curl -I http://truong.test/
```
> Dịch vụ không lên? `journalctl -xeu nginx-school -n 30`. Nghi phạm: `MemoryDenyWriteExecute` (chỉ vỡ nếu bật `pcre_jit on;`), `SystemCallFilter` (log có `SIGSYS`/`status=31` → bớt/ thêm nhóm syscall). Gỡ 1 dòng bằng cách chú thích nó (`#`) trong `10-sandbox.conf`.

## 3.3 Chấm điểm "sau" + chứng minh từng lớp
```bash
systemd-analyze security nginx-school.service --no-pager | tail -n 1     # điểm GIẢM rõ rệt → CHỤP ẢNH đặt cạnh ảnh "trước"
PID=$(systemctl show -p MainPID --value nginx-school)
grep -E 'NoNewPrivs|Seccomp' /proc/$PID/status                           # Seccomp: 2 = đang lọc syscall
systemctl show nginx-school -p MemoryMax -p CPUQuotaPerSecUSec -p TasksMax
cat /sys/fs/cgroup/system.slice/nginx-school.service/memory.max          # file kernel đọc để ép trần (≈ 314572800)
sudo ls -l /proc/$PID/root/home 2>&1 | head -2                           # dịch vụ KHÔNG thấy /home (rỗng/không truy cập được)
sudo nsenter -t $PID -m ls /home                                         # vào "góc nhìn" của dịch vụ: /home trống

# Đòn kiểm chứng: dịch vụ giả lập cùng cơ chế thử ghi vào hệ thống
sudo systemd-run --wait --pipe -p User=svc_nginx -p ProtectSystem=strict touch /usr/local/probe 2>&1 | tail -n 1
#   → "touch: cannot touch '/usr/local/probe': Read-only file system"
```
> 🧠 **Ngôn ngữ chuẩn khi bảo vệ – "phòng thủ nhiều lớp":** (1) tài khoản không root → (2) quyền file 750/640 → (3) chỉ 1 capability → (4) hệ thống tập tin chỉ-đọc + tách /tmp, /home, /proc → (5) lọc syscall (seccomp) → (6) cgroup ép trần → (7) tường lửa. Kẻ chiếm được Nginx phải *vượt cả 7 lớp*.

---

# Cấp 4 – AppArmor: kiểm soát truy cập BẮT BUỘC (★★★★★)

**DAC vs MAC – phân biệt này rất hay bị hỏi:**
- **DAC** (*Discretionary*): quyền theo **chủ sở hữu** (rwx, tài khoản). Cậu đã làm ở trên. Điểm yếu: mọi thứ "world-readable" (như `/etc/passwd`) thì tiến trình nào cũng đọc được.
- **MAC** (*Mandatory*): **hệ điều hành** áp một *chính sách riêng cho từng chương trình*, bất kể quyền file. "Nginx chỉ được đọc đúng ba chỗ này, còn lại cấm."

```bash
sudo apt install -y apparmor apparmor-utils
sudo aa-status | head -3                          # xem AppArmor đang bật
```

## 4.1 Viết profile cho Nginx
```bash
sudo tee /etc/apparmor.d/usr.sbin.nginx >/dev/null <<'EOF'
#include <tunables/global>

/usr/sbin/nginx flags=(attach_disconnected) {
  #include <abstractions/base>
  #include <abstractions/nameservice>

  capability net_bind_service,

  network inet stream,
  network inet6 stream,
  network unix stream,

  /usr/sbin/nginx                  mr,
  /etc/nginx-school/**             r,
  /etc/nginx/mime.types            r,
  /var/www/school/**               r,        # nội dung web: CHỈ ĐỌC
  /var/log/nginx-school/**         rw,
  /var/lib/nginx-school/**         rwk,
  /run/nginx-school/**             rwk,
  /proc/sys/net/core/somaxconn     r,
  /sys/devices/system/cpu/         r,
  /sys/devices/system/cpu/**       r,
}
EOF
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx
sudo aa-complain /usr/sbin/nginx                  # CHẾ ĐỘ HỌC: chưa chặn, chỉ ghi lại vi phạm
```

## 4.2 Học – tinh chỉnh – bật chặn
```bash
sudo systemctl restart nginx-school
curl -s -o /dev/null http://truong.test/ ; curl -s -o /dev/null http://truong.test/tai-lieu/thong-bao-mau.txt
sudo journalctl -k --since "5 min ago" | grep -i 'apparmor="ALLOWED"' | head     # những thứ profile CHƯA khai báo
sudo aa-logprof                                   # trả lời từng câu hỏi: Allow/Deny → tự bổ sung profile
sudo aa-enforce /usr/sbin/nginx                   # BẬT CHẶN THẬT
sudo systemctl restart nginx-school && curl -I http://truong.test/               # vẫn 200
sudo aa-status | grep -A2 nginx                   # profile ở "enforce mode"
```

## 4.3 Thí nghiệm DAC vs MAC (bằng chứng đắt giá)
Tạo file **ai cũng đọc được** và một đường dẫn "lỡ tay" trỏ tới nó:
```bash
echo "bí-mật-giả-lập" | sudo tee /etc/school-probe.txt && sudo chmod 644 /etc/school-probe.txt
sudo sed -i '/location ~ \/\\\./i\        location = /probe { alias /etc/school-probe.txt; }' /etc/nginx-school/nginx.conf
sudo systemctl reload nginx-school
```
| Chế độ | `curl -s -o /dev/null -w "%{http_code}\n" http://truong.test/probe` | Vì sao |
|---|---|---|
| Profile ở **complain** (`sudo aa-complain /usr/sbin/nginx`, restart) | **200** | DAC cho phép (file 644 ai cũng đọc), AppArmor chỉ ghi nhận |
| Profile ở **enforce** (`sudo aa-enforce /usr/sbin/nginx`, restart) | **403** | **MAC chặn**: profile không cho Nginx đọc `/etc/school-probe.txt` |

Xem vết bị chặn: `sudo journalctl -k | grep -i 'apparmor="DENIED"' | tail -n 2` → thấy `profile="/usr/sbin/nginx"`, `name="/etc/school-probe.txt"`, `requested_mask="r"`.
**Dọn dẹp:** xóa dòng `location = /probe ...` khỏi `nginx.conf`, `sudo rm /etc/school-probe.txt`, reload.

> 🧠 *"Quyền tài khoản (DAC) trả lời 'ai được đọc'. AppArmor (MAC) trả lời 'chương trình này được đọc những gì' – dù tài khoản có quyền. Hai lớp độc lập nhau: một lớp lỗi cấu hình, lớp kia vẫn chặn."*

---

# Cấp 5 – Tinh chỉnh hệ điều hành cho web server (★★★★★)

*Tinh chỉnh* (cấu hình) là việc của SV2; *đo hiệu quả dưới tải* là của SV4 – nên ở đây chỉ **kiểm chứng cấu hình đã có hiệu lực** (đọc `/proc`, `ss`), không benchmark.

## 5.1 Giới hạn file mở (file descriptor)
Mỗi kết nối = 1 "file" (socket). Mặc định mỗi tiến trình chỉ 1024 → hết là từ chối kết nối (`Too many open files`).
Đã đặt `LimitNOFILE=16384` (unit) và `worker_rlimit_nofile 8192` (nginx). Kiểm chứng:
```bash
for p in $(pgrep -f 'nginx: worker'); do grep 'Max open files' /proc/$p/limits; done     # → 8192 8192 (hoặc 16384)
PID=$(systemctl show -p MainPID --value nginx-school); grep 'Max open files' /proc/$PID/limits   # master: 16384
```
> Quy tắc: `worker_connections × (1 hoặc 2 tùy proxy)` ≤ `worker_rlimit_nofile` ≤ `LimitNOFILE`. Ở đây 4096 ≤ 8192 ≤ 16384 ✅.

## 5.2 Hàng đợi kết nối (backlog)
Khi yêu cầu đến nhanh hơn Nginx nhận, kernel xếp chúng vào hàng đợi; hàng đầy → khách bị từ chối/timeout.
```bash
sudo tee /etc/sysctl.d/90-school.conf >/dev/null <<'EOF'
net.core.somaxconn = 4096            # trần hàng đợi mỗi socket nghe
net.ipv4.tcp_max_syn_backlog = 4096  # trần hàng đợi kết nối đang bắt tay
EOF
sudo sysctl --system | grep -E 'somaxconn|tcp_max_syn'
ss -ltn 'sport = :80'                # cột Send-Q của socket LISTEN = backlog đang áp dụng = min(1024 ở listen, somaxconn)
```
Đọc kết quả: `Send-Q = 1024` ⇒ `listen ... backlog=1024` (Cấp 2) được kernel chấp nhận vì ≤ `somaxconn`. Thử hạ `somaxconn` xuống 128 rồi restart → `Send-Q` tụt về 128: **nhân HĐH áp trần lên cấu hình ứng dụng**.

## 5.3 Khóa nội dung web ở mức **mount** (chỉ-đọc kể cả root)
Mount chỉ-đọc + `noexec,nosuid,nodev` cho thư mục web: dù lỗ hổng nào đó cho kẻ xấu quyền root, **cũng không ghi được** vào nội dung (trừ khi remount) và không chạy được chương trình từ đó.
```bash
sudo mount --bind /var/www/school /var/www/school                          # biến nó thành điểm mount riêng
sudo mount -o remount,bind,ro,nosuid,nodev,noexec /var/www/school         # khóa
findmnt /var/www/school                                                    # OPTIONS có: ro,nosuid,nodev,noexec
sudo touch /var/www/school/test                                            # ❌ "Read-only file system" – KỂ CẢ ROOT
curl -I http://truong.test/                                                # vẫn 200 (đọc không ảnh hưởng)
```
Script `school-deploy.sh` ở Cấp 1 đã có hàm `unlock`/`lock`: tự mở khóa khi deploy rồi khóa lại (kể cả khi lỗi, nhờ `trap`). Thử: `sudo -u deploy sudo -n /usr/local/sbin/school-deploy.sh deploy /home/deploy/dist` → vẫn thành công; sau đó `findmnt` vẫn `ro`.
> ⚠️ Cách này **mất khi khởi động lại máy** (chỉ để demo). Làm vĩnh viễn = tạo `.mount` unit của systemd – coi là **thử thách tự làm** (gợi ý: `systemctl edit --force --full var-www-school.mount`, `What=/var/www/school`, `Where=/var/www/school`, `Type=none`, `Options=bind,ro,nosuid,nodev,noexec`).

## 5.4 Tiến trình & tín hiệu: master điều khiển worker

| Tín hiệu gửi master | Tác dụng |
|---|---|
| `HUP` | **Reload**: nạp cấu hình mới, sinh worker mới, worker cũ làm nốt rồi thoát (không rớt kết nối) |
| `QUIT` | Tắt **êm** (làm nốt yêu cầu dở rồi thoát) – systemd dùng cái này (`KillSignal=SIGQUIT`) |
| `TERM`/`INT` | Tắt **ngay** |
| `USR1` | **Mở lại file log** (sau khi log bị xoay vòng/đổi tên) |
| `USR2` | **Nâng cấp binary nóng**: chạy master mới song song master cũ, không ngắt web |
| `WINCH` | Tắt êm các worker nhưng giữ master |

Chứng minh "reload không ngắt, chỉ thay worker":
```bash
ps -C nginx -o pid,ppid,euser:20,lstart,cmd                  # GHI LẠI PID master và worker
while true; do curl -s -o /dev/null -w '%{http_code} ' http://truong.test/; sleep 0.05; done &   # tải nhẹ liên tục
sudo systemctl reload nginx-school
ps -C nginx -o pid,ppid,euser:20,lstart,cmd                  # master: PID GIỮ NGUYÊN; worker: PID & thời gian khởi động MỚI
kill %1                                                      # dừng vòng curl; chuỗi in ra phải toàn 200
```
Mô tả được sự khác `reload` ↔ `restart` bằng **PID thật** là điểm cộng lớn.

---

# Cấp 6 – Tự chấm điểm phần SV2 (★★★)

Một script chạy một lần, in **PASS/FAIL** từng yêu cầu của phần SV2 → bằng chứng nộp + phát hiện sót trước khi thầy cô phát hiện. Lưu `verify-sv2.sh`, chạy `sudo bash verify-sv2.sh`:
```bash
#!/usr/bin/env bash
# verify-sv2.sh – tự chấm phần việc SV2.  Chạy: sudo bash verify-sv2.sh
SVC=${SVC:-nginx-school}; HOST=${HOST:-truong.test}
WEB=/var/www/school/current; CONF=/etc/nginx-school/nginx.conf
pass=0; fail=0
chk() { local d=$1; shift
  if "$@" >/dev/null 2>&1; then printf '  \033[32m[PASS]\033[0m %s\n' "$d"; pass=$((pass+1))
  else printf '  \033[31m[FAIL]\033[0m %s\n' "$d"; fail=$((fail+1)); fi; }
http()   { [ "$(curl -s -o /dev/null -m 5 -w '%{http_code}' "${@:2}")" = "$1" ]; }   # http <mã> <curl-args…>
users()  { ps -C nginx -o euser:32= | awk '{print $1}' | sort -u; }
capeff() { awk '/^CapEff/{print $2}' "/proc/$(systemctl show -p MainPID --value "$SVC")/status"; }
prop()   { [ "$(systemctl show "$SVC" -p "$1" --value)" = "$2" ]; }

echo "== Tên miền nội bộ =="
chk "/etc/hosts có $HOST"                    grep -Eq "^[0-9.]+[[:space:]]+.*\b$HOST\b" /etc/hosts
chk "hệ điều hành phân giải được $HOST"      getent hosts "$HOST"

echo "== Dịch vụ & tiến trình =="
chk "$SVC đang chạy"                          systemctl is-active --quiet "$SVC"
chk "$SVC tự chạy khi bật máy"               systemctl is-enabled --quiet "$SVC"
chk "cú pháp cấu hình hợp lệ"                sudo -u svc_nginx nginx -t -c "$CONF"
chk "cổng 80 do nginx giữ"                    bash -c "ss -ltnp 'sport = :80' | grep -q nginx"
chk "MỌI tiến trình nginx chạy bằng svc_nginx (không root)"   test "$(users)" = "svc_nginx"
chk "tiến trình chỉ có capability bind-port (0x400)"          test "$(capeff)" = "0000000000000400"
chk "NoNewPrivileges=yes"                     prop NoNewPrivileges yes

echo "== Quyền thư mục nội dung =="
chk "svc_nginx ĐỌC được index.html"           sudo -u svc_nginx test -r "$WEB/index.html"
chk "svc_nginx KHÔNG ghi được thư mục web"   bash -c "! sudo -u svc_nginx test -w $WEB"
chk "người khác KHÔNG đọc được index.html"   bash -c "! sudo -u nobody test -r $WEB/index.html"
chk "deploy KHÔNG đọc được thư mục web"      bash -c "! sudo -u deploy test -r $WEB/index.html"
chk "bản đang chạy nguyên vẹn (manifest)"    /usr/local/sbin/school-deploy.sh verify

echo "== Hành vi HTTP đúng thiết kế =="
chk "/ → 200"                                 http 200 "http://$HOST/"
chk "/schedule (route React) → 200"          http 200 "http://$HOST/schedule"
chk "/tai-lieu/thong-bao-mau.txt → 200"      http 200 "http://$HOST/tai-lieu/thong-bao-mau.txt"
chk "/tai-lieu/khong-co.pdf → 404"           http 404 "http://$HOST/tai-lieu/khong-co.pdf"
chk "/.git/config → 403 (file ẩn bị chặn)"   http 403 "http://$HOST/.git/config"
chk "truy cập bằng IP/tên lạ → bị cắt (000)" http 000 -H "Host: la.example" "http://127.0.0.1/"
chk "index.html: Cache-Control no-cache"     bash -c "curl -sI http://$HOST/ | grep -i '^cache-control:.*no-cache'"
A=$(ls "$WEB/assets" 2>/dev/null | head -1)
chk "assets: cache 1 năm + immutable"        bash -c "curl -sI http://$HOST/assets/$A | grep -i '^cache-control:.*immutable'"
chk "không lộ phiên bản Nginx"               bash -c "! curl -sI http://$HOST/ | grep -i '^server:.*[0-9]\\.[0-9]'"

echo "== Sandbox & tài nguyên =="
chk "ProtectSystem=strict"                    prop ProtectSystem strict
chk "MemoryMax đã đặt"                        bash -c "[ \"\$(systemctl show $SVC -p MemoryMax --value)\" != infinity ]"
chk "seccomp đang lọc syscall"                bash -c "grep -q 'Seccomp:.2' /proc/\$(systemctl show -p MainPID --value $SVC)/status"
chk "AppArmor: nginx ở enforce"               bash -c "aa-status 2>/dev/null | sed -n '/in enforce mode/,/in complain mode/p' | grep -q '/usr/sbin/nginx'"
if findmnt /var/www/school >/dev/null 2>&1; then
  chk "thư mục web mount chỉ-đọc (ro)"       bash -c "findmnt -no OPTIONS /var/www/school | grep -qw ro"
fi
echo; echo "Điểm bảo mật systemd:"; systemd-analyze security "$SVC" --no-pager | tail -n 1
echo; echo "KẾT QUẢ: $pass PASS, $fail FAIL"; [ "$fail" -eq 0 ]
```
Chụp ảnh kết quả cuối ("… 0 FAIL") đặt vào hồ sơ. Mục nào FAIL → đối chiếu cấp tương ứng.

**Thử thách tự động hóa (nâng thêm):** gộp mọi bước từ Cấp 1 → Cấp 3 thành **một** script `provision-sv2.sh` chạy được nhiều lần liên tiếp mà kết quả không đổi (*idempotent*: `id svc_nginx || useradd …`, `install -d`, `tee` file cấu hình, `systemctl enable --now`) rồi kết thúc bằng `verify-sv2.sh`. Dựng máy Ubuntu mới tinh, chạy 1 lệnh, thấy 0 FAIL = *"hạ tầng dưới dạng mã"* – tái lập được phần SV2 trong vài phút.

---

# Cấp 7 – Phòng lab lỗi (★★★★)

Mỗi lab: **gây lỗi → đọc triệu chứng → chẩn đoán bằng công cụ → sửa**. Chỉ gồm lỗi thuộc phạm vi cấu hình/HĐH của SV2 (không phải lỗi DNS/tải/log). Làm xong từng lab nhớ trả về trạng thái tốt và chạy lại `verify-sv2.sh`.

| # | Gây lỗi | Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|---|---|
| 1 | `sudo chmod 700 /var/www/school` (mất quyền `x` của nhóm) | Web trả **403** | `sudo tail /var/log/nginx-school/error.log` → `(13: Permission denied)`; `namei -l /var/www/school/current/index.html` → thấy thư mục nào thiếu `x` | `sudo chmod 750 /var/www/school` |
| 2 | Chép web bằng tay `sudo cp -r ~/dist/* /var/www/school/releases/x/` (file thuộc root, 600) | **403** ở một số file | `ls -l`, `namei -l`; so với release chuẩn | Dùng `school-deploy.sh` (tự `chown/chmod`) hoặc `chown -R root:svc_nginx` + `chmod` D750/F640 |
| 3 | `sudo systemctl start nginx` khi `nginx-school` đang chạy | `nginx` **không lên**: `Address already in use` | `systemctl status nginx`; `sudo ss -ltnp 'sport = :80'` → PID thuộc `nginx-school` | `sudo systemctl disable --now nginx` (chỉ 1 dịch vụ giữ cổng 80) |
| 4 | `sudo systemctl edit nginx-school --drop-in=99-lab` rồi ghi `[Service]` + `AmbientCapabilities=` (rỗng) | `nginx-school` **failed** | `journalctl -xeu nginx-school` → `bind() to 0.0.0.0:80 failed (13: Permission denied)` | Xóa `/etc/systemd/system/nginx-school.service.d/99-lab.conf` → `daemon-reload` → restart |
| 5 | Trong `nginx.conf` đổi `access_log /var/log/nginx-school/access.log` thành `/var/log/nginx/lab.log` | Dịch vụ **không lên** | `journalctl -xeu nginx-school` → **`(30: Read-only file system)`**, *không phải* `13` | Đưa log về `LogsDirectory`. **Bài học:** `13 EACCES` = thiếu quyền (DAC); `30 EROFS` = bị sandbox/mount chặn |
| 6 | Thêm `location = /probe { alias /etc/school-probe.txt; }` khi AppArmor **enforce** | **403** dù file 644 | `sudo journalctl -k \| grep 'apparmor="DENIED"'` | Cố ý chặn thì để nguyên; muốn cho phép → thêm dòng vào profile + `apparmor_parser -r` |
| 7 | Khi thư mục web đang `ro` (Cấp 5.3): `sudo rsync -a ~/dist/ /var/www/school/releases/x/` | `Read-only file system` kể cả root | `findmnt /var/www/school` → `ro` | Dùng `school-deploy.sh` (tự unlock/lock) hoặc `sudo mount -o remount,bind,rw /var/www/school` |
| 8 | Chạy **`sudo nginx -t -c /etc/nginx-school/nginx.conf`** bằng root *trước khi* dịch vụ chạy lần đầu | Sau đó `nginx-school` **không ghi được log/thư mục tạm** | `ls -l /var/log/nginx-school /var/lib/nginx-school` → file **thuộc root** | `sudo chown -R svc_nginx: /var/log/nginx-school /var/lib/nginx-school`; từ nay `-t` bằng `sudo -u svc_nginx` |

> Với mỗi lab, ghi 3 dòng vào hồ sơ: *Triệu chứng nhìn thấy – lệnh/dòng log chứng minh nguyên nhân – cách sửa*. Đây chính là phần "giải thích bằng bằng chứng" thầy cô cần.

---

# Cấp 8 – Backend Auth + PostgreSQL: vận hành an toàn (★★★★★)

> **Chỉ làm khi nhóm chọn hướng B** và cậu đã làm xong **mục G của [HUONG_DAN_SV2_UBUNTU.md](HUONG_DAN_SV2_UBUNTU.md)** (dịch vụ `school-api` chạy dưới `svc_api`, DB `school_portal`, Nginx chuyển tiếp `/api`, `curl` ra 401/200/403).
> Cấp này nâng "chạy được" thành "**vận hành an toàn**": giam dịch vụ, khóa mạng, tách quyền ở tầng DB, giữ bí mật, đổi bản có rollback. Vẫn nằm trong phạm vi SV2 – *quyền tiến trình, cấu hình hệ điều hành*. Phần phân quyền theo vai trò trong ứng dụng và đọc log đăng nhập là của SV3.

Các cấu hình dưới đây **áp lên backend thật** chứ không phải bản demo, nên có hai khác biệt so với Nginx:
- Backend là **Node.js**: V8 cần vùng nhớ vừa ghi vừa chạy cho JIT ⇒ **không được** bật `MemoryDenyWriteExecute` (khác Nginx ở Cấp 3).
- Backend **không có "reload êm"**: đổi bản = `restart` (mất vài giây). Nginx thì `reload` không rớt kết nối.

## 8.1 Giam dịch vụ `school-api` + khóa mạng về localhost

Backend mặc định nghe `*:3000` (mọi giao diện). Thay vì nhờ người viết code sửa hay chỉ dựa vào `ufw`, ta để **systemd** ép: dịch vụ này chỉ được nhận/gửi gói tin tới `localhost` (Nginx và PostgreSQL đều ở localhost).

```bash
sudo systemd-analyze security school-api.service --no-pager | tail -n 1       # điểm TRƯỚC → chụp ảnh
sudo mkdir -p /etc/systemd/system/school-api.service.d
sudo tee /etc/systemd/system/school-api.service.d/10-sandbox.conf >/dev/null <<'EOF'
[Service]
# --- Chỉ nói chuyện với localhost (Nginx → backend, backend → PostgreSQL) ---
IPAddressDeny=any
IPAddressAllow=localhost
# --- Hệ thống tập tin gần như chỉ-đọc ---
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
PrivateDevices=yes
InaccessiblePaths=-/root -/boot -/mnt -/media -/srv
ProtectProc=invisible
ProcSubset=pid
# --- Không đụng được nhân ---
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectKernelLogs=yes
ProtectControlGroups=yes
ProtectClock=yes
ProtectHostname=yes
# --- Hạn chế hành vi (KHÔNG bật MemoryDenyWriteExecute: Node/V8 sẽ chết) ---
CapabilityBoundingSet=
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
RestrictNamespaces=yes
RestrictRealtime=yes
RestrictSUIDSGID=yes
LockPersonality=yes
SystemCallArchitectures=native
SystemCallFilter=@system-service
SystemCallErrorNumber=EPERM
# --- Trần tài nguyên (cgroup) ---
MemoryMax=400M
CPUQuota=100%
TasksMax=200
EOF
sudo systemctl daemon-reload && sudo systemctl restart school-api
systemctl status school-api --no-pager && curl -s http://truong.test/api/health
sudo systemd-analyze security school-api.service --no-pager | tail -n 1       # điểm SAU → giảm rõ
```
`CapabilityBoundingSet=` để **trống** nghĩa là bỏ hết capability (backend chỉ nghe cổng 3000 nên không cần cái nào). Dịch vụ không lên: `journalctl -xeu school-api -n 30`; chú thích bớt dòng nghi ngờ (thường là `SystemCallFilter` hoặc `PrivateDevices`).

**Chứng minh khóa mạng thật sự có hiệu lực** (chụp ảnh, **không cần `ufw` hay máy thứ hai**):
```bash
IP=$(hostname -I | awk '{print $1}')                     # IP LAN của máy chủ, ví dụ 192.168.1.10
curl -s -m 5 http://127.0.0.1:3000/api/health            # {"status":"ok"}      ← qua localhost: được
curl -s -m 5 http://$IP:3000/api/health || echo "BỊ CHẶN (timeout)"   # ← qua IP LAN: bị chặn, dù backend đang nghe *:3000
sudo ss -ltnp 'sport = :3000'                            # vẫn thấy *:3000 – nhưng gói tin từ ngoài localhost bị systemd loại
```
> 🧠 Đây là "phòng thủ nhiều lớp" cho **cùng một mục tiêu** (không cho đi vòng qua Nginx): (1) `ufw` mặc định chặn cổng 3000, (2) `IPAddressAllow=localhost` của systemd, (3) nếu người code sửa `listen(port, '127.0.0.1')` thì thêm lớp thứ 3.

## 8.2 Cô lập PostgreSQL: tài khoản ứng dụng ≠ chủ database (⚠️ thống nhất với SV3 trước)

Hiện backend kết nối bằng `school` – **chủ** database, có quyền `DROP TABLE`. Nếu kẻ xấu khai thác được lỗi (ví dụ SQL injection), nó xóa được cả bảng. Cách xử lý: tách hai vai trò.

| Vai trò | Quyền | Dùng khi |
|---|---|---|
| `school` (chủ) | Tạo/sửa/xóa bảng (DDL) | Chỉ lúc **khởi tạo / đổi cấu trúc** (bước G4) |
| `school_app` | Chỉ `SELECT / INSERT / UPDATE / DELETE` trên dữ liệu | **Backend chạy hằng ngày** |

> Phân quyền tầng database là chỗ SV3 quan tâm (họ nói cần DB để "phân quyền"). Báo SV3 là cậu làm bước tách này, để hai bên không viết trùng và cùng dùng một bảng kết quả.

```bash
APPPW=$(openssl rand -hex 16)
sudo -u postgres psql -d school_portal -v pw="$APPPW" -f - <<'SQL'
CREATE ROLE school_app LOGIN PASSWORD :'pw' NOSUPERUSER NOCREATEDB NOCREATEROLE;
REVOKE ALL ON DATABASE school_portal FROM PUBLIC;
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
GRANT CONNECT ON DATABASE school_portal TO school_app;
GRANT USAGE ON SCHEMA public TO school_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO school_app;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO school_app;
-- Bảng tạo sau này bởi "school" cũng tự cấp đúng quyền đó cho school_app:
ALTER DEFAULT PRIVILEGES FOR ROLE school IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO school_app;
ALTER DEFAULT PRIVILEGES FOR ROLE school IN SCHEMA public GRANT USAGE, SELECT ON SEQUENCES TO school_app;
SQL
# Chuyển backend sang dùng school_app
sudo sed -i -e 's/^DB_USER=.*/DB_USER=school_app/' -e "s/^DB_PASSWORD=.*/DB_PASSWORD=$APPPW/" /etc/school-api/api.env
sudo systemctl restart school-api && curl -s http://truong.test/api/health        # vẫn ok
```
**Chứng minh (chụp ảnh):**
```bash
export PGPASSWORD="$APPPW"
psql -h 127.0.0.1 -U school_app -d school_portal -c 'SELECT count(*) FROM users;'            # ✅ đọc được
psql -h 127.0.0.1 -U school_app -d school_portal -c 'DROP TABLE users;'                      # ❌ must be owner of table users
psql -h 127.0.0.1 -U school_app -d school_portal -c 'CREATE TABLE hack(x int);'              # ❌ permission denied for schema public
unset PGPASSWORD APPPW
sudo -u postgres psql -Atc "SELECT rolname, rolsuper, rolcreatedb FROM pg_roles WHERE rolname LIKE 'school%';"   # cả hai: f|f
```
Muốn đổi cấu trúc bảng sau này (hoặc chạy lại G4) thì tạm đổi `DB_USER`/`DB_PASSWORD` về `school` cho lần chạy khởi tạo, rồi trả lại `school_app`.

Ba kiểm tra "bề mặt" của DB (SV2 chịu trách nhiệm cấu hình, cứ chụp):
```bash
sudo -u postgres psql -Atc "SHOW listen_addresses"                         # localhost
sudo grep -vE '^\s*(#|$)' /etc/postgresql/*/main/pg_hba.conf               # local = peer; host 127.0.0.1/::1 = scram-sha-256 → mọi kết nối đều bắt mật khẩu
sudo ufw status | grep -E '5432|3000' || echo "tường lửa KHÔNG mở 5432/3000 ra ngoài ✅"
```

## 8.3 Giữ bí mật cấu hình

```bash
sudo ls -l /etc/school-api/api.env                       # -rw-r----- root svc_api  (640)
sudo find /opt/school-api /home -name '.env*' 2>/dev/null        # KHÔNG được có file .env nào chứa bí mật (nếu thấy: xóa/di chuyển)
PID=$(systemctl show -p MainPID --value school-api)
sudo cat /proc/$PID/environ | tr '\0' '\n' | grep -c JWT_   # root đọc được: bí mật nằm trong môi trường tiến trình
cat /proc/$PID/environ                                   # ❌ "Permission denied" – người khác không đọc được
sudo -u nobody cat /proc/$PID/environ                    # ❌
```
**Thay (rotate) khóa JWT – bằng chứng cho "hệ thống chịu được lộ khóa":** tạo khóa mới, restart, mọi phiên cũ mất hiệu lực.
```bash
ADMIN=$(login admin@adu.edu.vn "$SEEDPW" | jq -r .accessToken)           # (hàm login ở mục G7 file Ubuntu)
code -H "Authorization: Bearer $ADMIN" http://truong.test/api/auth/me     # 200
sudo sed -i "s/^JWT_ACCESS_SECRET=.*/JWT_ACCESS_SECRET=$(openssl rand -hex 48)/" /etc/school-api/api.env
sudo systemctl restart school-api
code -H "Authorization: Bearer $ADMIN" http://truong.test/api/auth/me     # 401  ← token cũ vô hiệu ngay
```
> Chính vì thế **phải sinh khóa ngẫu nhiên trên máy chủ** và không bao giờ dùng `change-me-...` trong `.env.example`: ai đọc repo cũng ký được token giả có quyền admin.

## 8.4 Triển khai backend không "mất bản cũ": `releases/` + `current` + rollback

Cùng ý tưởng Cấp 1, áp cho backend. Cấu trúc: `/opt/school-api/releases/<thời-gian>/` (mỗi bản có `dist/` và `node_modules/` riêng) và `current` là symlink.

Chuyển dịch vụ sang dùng `current`:
```bash
sudo mkdir -p /opt/school-api/releases
sudo tee /etc/systemd/system/school-api.service.d/20-layout.conf >/dev/null <<'EOF'
[Service]
WorkingDirectory=/opt/school-api/current
EOF
sudo systemctl daemon-reload
```
Script `school-api-deploy.sh` (cài vào `/usr/local/sbin`, chủ root, `chmod 755`):
```bash
#!/usr/bin/env bash
# Dùng:  sudo school-api-deploy.sh deploy <thư-mục-backend-đã-build>   (phải có dist/main.js, package.json, package-lock.json)
#        sudo school-api-deploy.sh rollback [tên-bản]
set -euo pipefail
BASE=/opt/school-api; REL_DIR=$BASE/releases; SVC=school-api; KEEP=5
HEALTH=http://127.0.0.1:3000/api/health
die()  { echo "LỖI: $*" >&2; exit 1; }
cur()  { readlink -f "$BASE/current" 2>/dev/null || true; }
swap() { ln -sfn "$1" "$BASE/current.tmp"; mv -T "$BASE/current.tmp" "$BASE/current"; systemctl restart "$SVC"; }
healthy() { local c=; for _ in $(seq 1 20); do
              c=$(curl -s -o /dev/null -m 2 -w '%{http_code}' "$HEALTH" || true); [ "$c" = 200 ] && return 0; sleep 1; done; return 1; }

deploy() {
  local SRC=${1:?thiếu thư mục backend đã build}
  [ -f "$SRC/dist/main.js" ] && [ -f "$SRC/package-lock.json" ] || die "$SRC thiếu dist/main.js hoặc package-lock.json (đã npm run build chưa?)"
  if find "$SRC/dist" -type l | grep -q .; then die "dist chứa symlink – từ chối"; fi
  local NEW="$REL_DIR/$(date +%Y%m%d-%H%M%S)" PREV; PREV=$(cur)
  install -d -o root -g svc_api -m 750 "$REL_DIR" "$NEW"
  cp -r "$SRC/dist" "$SRC/package.json" "$SRC/package-lock.json" "$NEW/"
  ( cd "$NEW" && npm ci --omit=dev --ignore-scripts --no-audit --no-fund )     # gói chạy đúng theo lockfile
  chown -R root:svc_api "$NEW"; find "$NEW" -type d -exec chmod 750 {} +; find "$NEW" -type f -exec chmod 640 {} +
  swap "$NEW"
  if ! healthy; then
    [ -n "$PREV" ] && swap "$PREV" && healthy
    die "health-check thất bại → ĐÃ ROLLBACK về ${PREV:-<không có>}  (xem: journalctl -u $SVC -n 30)"
  fi
  ls -1dt "$REL_DIR"/*/ | tail -n +$((KEEP + 1)) | xargs -r rm -rf --
  echo "OK: đang chạy $NEW (bản trước: ${PREV:-không có})"
}
rollback() {
  local T=${1:-} P
  if [ -n "$T" ]; then P="$REL_DIR/$T"; else P=$(ls -1dt "$REL_DIR"/*/ | sed -n 2p); P=${P%/}; fi
  [ -d "$P" ] || die "không có bản để quay về"
  swap "$P"; healthy || die "quay về $P nhưng health-check vẫn lỗi"; echo "Đã quay về: $P"
}
case "${1:-}" in deploy) shift; deploy "$@" ;; rollback) shift; rollback "$@" ;; *) sed -n '2,3p' "$0"; exit 1 ;; esac
```
Dùng và chứng minh:
```bash
cd ~/school-portal/backend && npm ci && npm run build          # build ra dist/
sudo school-api-deploy.sh deploy ~/school-portal/backend       # bản 1
sudo school-api-deploy.sh deploy ~/school-portal/backend       # bản 2
readlink -f /opt/school-api/current
sudo school-api-deploy.sh rollback                              # quay về bản 1 → health-check OK

# Bản hỏng → tự rollback: cố ý làm hỏng bản mới rồi deploy
mkdir -p /tmp/bad && cp -r ~/school-portal/backend/{dist,package.json,package-lock.json} /tmp/bad/
echo "throw new Error('lỗi cố ý')" > /tmp/bad/dist/main.js
sudo school-api-deploy.sh deploy /tmp/bad                       # → "health-check thất bại → ĐÃ ROLLBACK"
curl -s http://truong.test/api/health                           # vẫn {"status":"ok"} – dịch vụ sống bằng bản cũ
```

## 8.5 Tự chấm điểm phần backend/DB

Lưu `verify-api.sh`, chạy `sudo bash verify-api.sh`:
```bash
#!/usr/bin/env bash
# verify-api.sh – tự chấm phần backend + PostgreSQL của SV2.   Chạy: sudo bash verify-api.sh
SVC=school-api; HOST=${HOST:-truong.test}; IP=${IP:-$(hostname -I | awk '{print $1}')}
pass=0; fail=0
chk()  { local d=$1; shift
  if "$@" >/dev/null 2>&1; then printf '  \033[32m[PASS]\033[0m %s\n' "$d"; pass=$((pass+1))
  else printf '  \033[31m[FAIL]\033[0m %s\n' "$d"; fail=$((fail+1)); fi; }
code() { curl -s -o /dev/null -m 4 -w '%{http_code}' "$@"; }
PID=$(systemctl show -p MainPID --value "$SVC")
pg()   { sudo -u postgres psql -Atc "$1"; }

echo "== Dịch vụ & tiến trình =="
chk "$SVC đang chạy, tự chạy khi bật máy"     bash -c "systemctl is-active --quiet $SVC && systemctl is-enabled --quiet $SVC"
chk "node chạy bằng svc_api (không root)"     test "$(ps -o euser:32= -p "$PID" | awk '{print $1}')" = svc_api
chk "NODE_ENV=production"                     bash -c "tr '\0' '\n' < /proc/$PID/environ | grep -qx NODE_ENV=production"
chk "ProtectSystem=strict"                    test "$(systemctl show $SVC -p ProtectSystem --value)" = strict

echo "== Bí mật =="
chk "api.env quyền 640 root:svc_api"          test "$(stat -c '%a %U:%G' /etc/school-api/api.env)" = "640 root:svc_api"
chk "không có file .env trong /opt/school-api" bash -c "! find /opt/school-api -name '.env*' | grep -q ."
chk "người khác không đọc được environ"       bash -c "! sudo -u nobody cat /proc/$PID/environ"

echo "== Mạng: chỉ Nginx lộ ra ngoài =="
chk "backend nhận qua localhost"              test "$(code http://127.0.0.1:3000/api/health)" = 200
chk "backend KHÔNG nhận qua IP LAN ($IP:3000)" bash -c "! curl -s -m 3 http://$IP:3000/api/health"
chk "PostgreSQL chỉ nghe localhost"           test "$(pg 'SHOW listen_addresses')" = localhost
chk "Swagger tắt ở production (/docs → 404)"  test "$(code http://127.0.0.1:3000/docs)" = 404

echo "== PostgreSQL: đặc quyền tối thiểu =="
chk "school_app không phải superuser"         test "$(pg "SELECT rolsuper FROM pg_roles WHERE rolname='school_app'")" = f
chk "school_app không sở hữu bảng users"      test "$(pg "SELECT tableowner FROM pg_tables WHERE tablename='users'")" != school_app
chk "backend đang dùng school_app"            bash -c "sudo grep -qx 'DB_USER=school_app' /etc/school-api/api.env"

echo "== Qua Nginx =="
chk "/api/health → 200"                       test "$(code http://$HOST/api/health)" = 200
chk "/api/users khi chưa đăng nhập → 401"     test "$(code http://$HOST/api/users)" = 401
echo; echo "Điểm bảo mật systemd:"; systemd-analyze security "$SVC" --no-pager | tail -n 1
echo; echo "KẾT QUẢ: $pass PASS, $fail FAIL"; [ "$fail" -eq 0 ]
```

## 8.6 Phòng lab lỗi riêng cho backend

| # | Gây lỗi | Triệu chứng | Chẩn đoán | Sửa |
|---|---|---|---|---|
| B1 | Sửa `DB_PASSWORD` trong `api.env` thành sai, `restart` | Dịch vụ lên rồi chết liên tục; Nginx **502** | `journalctl -u school-api -n 20` → `password authentication failed` | Trả mật khẩu đúng |
| B2 | Đổi `DB_USER=school_app` rồi chạy lại **G4** (khởi tạo) | Lỗi `permission denied for schema public` | Log của lệnh seed; `\dp` | Chạy khởi tạo bằng `school`, rồi đổi lại |
| B3 | `sudo chmod 600 /etc/school-api/api.env` | Dịch vụ không đọc được cấu hình, `status=1`/`EACCES` | `ls -l`, `journalctl` | `chown root:svc_api` + `chmod 640` |
| B4 | Thêm `MemoryDenyWriteExecute=yes` vào drop-in | Node **chết ngay** khi khởi động (SIGSEGV/`status=11`) | `journalctl -xeu school-api` | Xóa dòng đó. **Bài học:** hardening không áp mù quáng; kiểm tra từng tùy chọn với loại chương trình |
| B5 | Bỏ `IPAddressAllow=localhost` (chỉ để `IPAddressDeny=any`) | Backend **không kết nối nổi DB** và Nginx 502 | `journalctl`: `ECONNREFUSED`/timeout tới 127.0.0.1 | Thêm lại `IPAddressAllow=localhost` |
| B6 | Gọi login 11 lần trong 1 phút | **429** | Response + log | Đúng thiết kế (chống dò mật khẩu). Đợi 1 phút |

## 8.7 Câu hỏi bảo vệ cho phần backend

**1. Vì sao backend nghe `*:3000` mà vẫn "không lộ ra ngoài"?**
Do ba lớp độc lập: `ufw` chặn cổng 3000 từ ngoài, `IPAddressAllow=localhost` của systemd loại mọi gói tin không đến từ localhost, và (nếu được sửa code) `listen(port, '127.0.0.1')`. Em chứng minh bằng `curl` tới `127.0.0.1:3000` (được) và tới IP LAN (timeout).

**2. Vì sao backend dùng tài khoản `school_app` thay vì `school`?**
Đặc quyền tối thiểu ở tầng dữ liệu: `school_app` không phải chủ bảng, không `DROP/CREATE`. Nếu có SQL injection thì kẻ xấu đọc/ghi dữ liệu được nhưng không phá cấu trúc. Em chứng minh bằng lệnh `DROP TABLE users` bị từ chối.

**3. Vì sao `NODE_ENV=production` thì không có bảng và Swagger?**
Production tắt `synchronize` (tự sửa bảng theo code là nguy hiểm trên dữ liệu thật) và tắt Swagger (không lộ danh sách API). Vì repo chưa có migration nên lần đầu phải khởi tạo ở `development` tạm thời – đó là chỗ nên cải thiện bằng migration của TypeORM.

**4. Qua Nginx, vì sao hạn mức 10 lần đăng nhập/phút có thể bị dùng chung cho mọi người?**
Backend nhìn thấy kết nối từ Nginx nên IP nguồn là `127.0.0.1` cho **tất cả**. Cần `trust proxy` để Express lấy IP thật từ `X-Forwarded-For`. Nginx cũng phải **ghi đè** header đó bằng `$remote_addr` (không nối thêm) để client không giả được IP nhằm né giới hạn.

**5. Vì sao `restart` backend mất vài giây còn `reload` Nginx thì không?**
Nginx có cơ chế master/worker nên nạp cấu hình mới mà worker cũ làm nốt việc. Node chạy một tiến trình duy nhất, đổi bản = dừng rồi chạy lại. Muốn không gián đoạn phải chạy hai bản song song sau Nginx (nâng cấp xanh–lục), ngoài phạm vi bài này.

**6. Vì sao không bật `MemoryDenyWriteExecute` cho Node dù Nginx bật được?**
V8 biên dịch JavaScript thành mã máy lúc chạy (JIT) nên cần vùng nhớ vừa ghi vừa thực thi. Cấm điều đó làm Node sập. Hardening phải thử với từng loại chương trình, không áp mù quáng.

**7. Đổi khóa `JWT_ACCESS_SECRET` thì chuyện gì xảy ra?**
Token đã cấp bằng khóa cũ không còn xác thực được ⇒ mọi người phải đăng nhập lại. Đó là cách thu hồi hàng loạt khi nghi lộ khóa. Em đã chạy thử: `/api/auth/me` từ 200 thành 401.

---

# Bonus – So sánh Nginx ↔ Apache theo tiêu chí của SV2

Đề cho phép "mở rộng: thay HTTP server rồi so sánh bằng cùng tiêu chí". Ở đây chỉ so các tiêu chí **thuộc SV2** (cài đặt, cấu hình, mô hình tiến trình, tách quyền, sandbox). Số liệu CPU/RAM/độ trễ dưới tải là **việc đo của SV4** – cần bảng đầy đủ thì xin số từ SV4, không tự đo lại.

```bash
sudo apt install -y apache2 && sudo systemctl disable --now apache2     # chỉ để so sánh, không chạy
systemd-analyze security apache2.service --no-pager | tail -n 1         # điểm hardening mặc định của Apache
systemd-analyze security nginx-school.service --no-pager | tail -n 1    # điểm của bản đã hardening
wc -l /etc/apache2/apache2.conf /etc/apache2/sites-available/000-default.conf /etc/nginx-school/nginx.conf
```
| Tiêu chí (tự điền, đo thật) | Nginx (`nginx-school`) | Apache (mặc định gói) |
|---|---|---|
| Thời gian cài + cấu hình tới khi chạy | | |
| Số dòng cấu hình cho site này | | |
| Master chạy bằng tài khoản nào | `svc_nginx` | `root` |
| Worker chạy bằng tài khoản nào | `svc_nginx` | `www-data` |
| Điểm `systemd-analyze security` | | |
| Cách làm một việc phổ biến (SPA fallback) | `try_files … /index.html` | `FallbackResource /index.html` |
| Reload không ngắt? | `HUP` | `graceful` |

---

# 11. Nếu cậu vẫn ở Windows

Nền tảng thiết kế như nhau, công cụ khác. Cấp 4 (AppArmor) và 5.3 (mount) **không có tương đương trực tiếp**.

| Cấp | Linux | Windows |
|---|---|---|
| 1 Tách vai trò & atomic | `deploy`, sudoers, `mv -T` | Tài khoản `deploy` + `icacls` (cho phép ghi thư mục releases, **không** cho ghi cấu hình); `mklink /J C:\web\school\current <bản>` rồi `nginx -s reload`; băm bằng `Get-FileHash -Algorithm SHA256` |
| 2 Không root | capability + unit | NSSM chạy `svc_nginx` (đã là tài khoản thường). Thu hẹp thêm: `sc.exe privs nginx-school SeChangeNotifyPrivilege` (chỉ giữ đặc quyền cần thiết); `sc.exe sidtype nginx-school restricted` (SID dịch vụ hạn chế) |
| 3 Sandbox & cgroup | systemd + cgroup | Giới hạn tiến trình bằng **Job Object** (công cụ của Sysinternals / NSSM `AppPriority`, `AppAffinity`); kiểm tra bằng Process Explorer |
| 4 MAC | AppArmor | **AppLocker** / Windows Defender Application Control: chỉ cho phép `C:\nginx\nginx.exe` chạy |
| 5 Tinh chỉnh HĐH | `ulimit`, sysctl | `worker_connections` (Windows dùng `select`, trần ~1024 kết nối/worker); ngăn ghi thư mục web bằng `icacls /deny` cho mọi tài khoản trừ SYSTEM |
| 6 Tự chấm điểm | `verify-sv2.sh` | Viết `verify-sv2.ps1` dùng `Get-Service`, `Get-NetTCPConnection`, `icacls`, `Invoke-WebRequest` (`$r.StatusCode`) |

---

# 12. Bàn giao cho bạn khác (không làm thay)

Báo cho từng bạn **đúng thông tin cần**, để họ tự làm phần của họ:

| Bạn | Cậu báo | Lưu ý quan trọng |
|---|---|---|
| **SV1** | IP máy chủ; tên miền đang dùng (`truong.test`, `www.truong.test`); web trả `444` nếu gọi bằng tên khác | Cấu hình chỉ phục vụ đúng 2 tên đó – SV1 thêm tên miền mới thì báo cậu thêm vào `server_name` |
| **SV3** | Đường dẫn cần **sao lưu**: `/var/www/school` (gồm `releases/`, `manifests/`) **và** `/etc/nginx-school/`, `/etc/systemd/system/nginx-school.service*`, `/etc/apparmor.d/usr.sbin.nginx`. Log ở **`/var/log/nginx-school/`** (đã đổi chỗ; cần xoay vòng) | Thư mục web có thể đang `ro` (Cấp 5.3) – phục hồi phải dùng `school-deploy.sh` hoặc remount. Khóa riêng TLS/HTTPS (nếu SV3 làm) nên đặt ngoài `/var/www`, quyền `600` |
| **SV4** | Tên dịch vụ `nginx-school`; PID: `systemctl show -p MainPID --value nginx-school`; thống kê cgroup: `systemctl show nginx-school -p MemoryCurrent -p CPUUsageNSec -p TasksCurrent`; log: `/var/log/nginx-school/access.log` | ⚠️ Cậu **đã đặt trần** `MemoryMax=300M`, `CPUQuota=80%`, `TasksMax=200` – sẽ ảnh hưởng số liệu khi tải. Nói rõ với SV4 để họ ghi vào điều kiện đo. Truy cập bằng **IP** sẽ bị cắt (444): SV4 phải dùng `truong.test` hoặc `-H "Host: truong.test"` |

---

**Bổ sung nếu nhóm chọn hướng B (có backend + DB):**

| Bạn | Cậu báo thêm | Lưu ý |
|---|---|---|
| **SV3** | Tên dịch vụ `school-api`; log ứng dụng ở **journald** (`journalctl -u school-api`), **không** nằm trong `/var/log/nginx-school/`; DB `school_portal`, hai vai trò `school` (chủ) và `school_app` (ứng dụng); cần sao lưu thêm `pg_dump` (cậu **không** làm giúp) và `/etc/school-api/` (chứa bí mật, bản sao lưu phải được bảo vệ) | Phối hợp tách quyền ở DB (mục 8.2) để SV3 không viết trùng. Phân quyền theo vai trò (admin/teacher/student) nằm trong code backend – SV3 dùng thử bằng token của từng vai trò |
| **SV4** | `school-api` cũng có cgroup riêng (`MemoryMax=400M`, `CPUQuota=100%`); PID bằng `systemctl show -p MainPID --value school-api`; PostgreSQL là dịch vụ `postgresql` | Khi đo tải qua `/api`, cậu đã đặt trần tài nguyên và hạn mức đăng nhập 10 lần/phút (429) – báo để họ ghi vào điều kiện đo |
| **Người viết backend/frontend** | 4 việc ở mục G9 file Ubuntu (`trust proxy`, migration, frontend gọi `/api`, `listen 127.0.0.1`) | Gửi dạng đề nghị, không tự sửa code của họ |

---

# 13. Câu hỏi bảo vệ nâng cao (có đáp án)

**1. Vì sao Nginx thường để master chạy root, còn cậu chạy cả master không root vẫn mở được cổng 80?**
Cổng < 1024 đòi quyền `CAP_NET_BIND_SERVICE`; root có sẵn, nên gói Nginx dùng root rồi hạ quyền worker. Em thay root bằng đúng capability đó (`AmbientCapabilities`), nên không tiến trình nào có root. Bằng chứng: `ps` toàn `svc_nginx`, `CapEff = 0x400`.

**2. DAC và MAC khác nhau thế nào? Cậu có ví dụ chạy thật?**
DAC = chủ sở hữu quyết định (rwx). MAC = hệ điều hành áp chính sách riêng cho từng chương trình bất kể quyền file. Thí nghiệm `/probe` (Cấp 4.3): file 644 ai cũng đọc được, complain mode trả 200, enforce mode trả 403 kèm dòng `apparmor="DENIED"`.

**3. `13 EACCES` khác `30 EROFS` thế nào, gặp ở đâu?**
EACCES = thiếu quyền (DAC/MAC từ chối). EROFS = hệ thống tập tin **chỉ-đọc** (do `ProtectSystem`/mount `ro`) – kể cả root cũng không ghi. Phân biệt để chẩn đoán đúng tầng: Lab 5 so với Lab 1.

**4. Vì sao đổi bản bằng symlink + `mv -T` chứ không `rsync --delete` thẳng?**
`mv -T` đổi tên là thao tác nguyên tử của hệ thống tập tin: `current` luôn trỏ vào một bản trọn vẹn. `rsync --delete` có khoảng "nửa cũ nửa mới" và không quay lại được. Bản cũ còn trong `releases/` nên rollback = đổi con trỏ.

**5. Sổ băm (manifest) giúp gì? Nằm trong thư mục web được không?**
Phát hiện file bị sửa/mất/thêm lạ (`verify`) và bắt lỗi chép hỏng khi deploy. Không để trong thư mục web vì sẽ lộ danh sách file cho người ngoài và `svc_nginx` đọc được; để ở `manifests/` thuộc root.

**5b. Khi deploy xong, người đang mở trang cũ bấm chuyển trang bị lỗi tải file – vì sao?**
File build có băm tên; bản mới đổi tên (`index-abc.js` → `index-xyz.js`), file cũ biến mất sau khi đổi `current`; trang cũ trong tab vẫn xin tên cũ → 404. Cách giảm: giữ lại asset của 1–2 bản trước khi deploy (gộp thư mục `assets`) hoặc cho React tự tải lại trang khi phát hiện bản mới.

**6. `reload` khác `restart`? Cậu chứng minh thế nào?**
`reload` (`HUP`): master nạp cấu hình mới, **PID master giữ nguyên**, sinh worker mới, worker cũ làm nốt rồi thoát. `restart`: tắt hẳn – PID đổi, kết nối rớt. Em đặt `ps` trước/sau và chạy vòng `curl` cho thấy toàn 200.

**7. `systemd-analyze security` cho điểm gì? Điểm thấp có nghĩa là an toàn tuyệt đối?**
Nó cộng "mức phơi nhiễm" theo số tùy chọn sandbox đã bật. Điểm thấp = bề mặt tấn công hẹp hơn, **không** đảm bảo không có lỗi. Em dùng như số đo trước/sau để chứng minh hardening có tác dụng.

**8. `noexec,nosuid,nodev,ro` trên thư mục web ngăn được gì?**
`ro`: không ghi được (kể cả root, trừ khi remount). `noexec`: không chạy chương trình từ đó. `nosuid`: file setuid không có hiệu lực. `nodev`: không dùng file thiết bị giả. Một kẻ upload được file thực thi vào web cũng không chạy được nó.

**9. Vì sao `index.html` để `no-cache` còn `assets/` cho cache 1 năm `immutable`?**
`assets/*` đặt tên theo mã băm nội dung → đổi nội dung là đổi tên, cache lâu không sợ cũ. `index.html` giữ tên cố định và trỏ tới các asset mới, nên phải được kiểm tra lại mỗi lần, nếu không khách kẹt bản cũ.

**10. Vì sao `deploy` có sudo mà vẫn "an toàn"? Script chạy root nhận đường dẫn từ người khác có nguy cơ gì?**
sudoers chỉ cho **đúng một** lệnh. Nguy cơ: người đó đưa vào thư mục chứa symlink trỏ tới file nhạy cảm (ví dụ `/etc/shadow`) để root chép giúp ⇒ script **từ chối** mọi symlink, kiểm tra có `index.html`, và đặt lại chủ file thành `root:svc_nginx` (không giữ chủ cũ). Nguyên tắc: *script đặc quyền không bao giờ tin tham số.*

---

# 14. Checklist nộp bổ sung

Ghi rõ **cấp nào đã làm thật** (đừng nộp cấp chưa chạy):

- [ ] **Cấp 1:** `sudo -l -U deploy`; ảnh `deploy` bị từ chối `systemctl stop`; `readlink -f current`; ảnh `verify` ✅ → ❌ (sửa lén) → ✅; ảnh health-check tự rollback; chuỗi `200 200 200…` khi deploy.
- [ ] **Cấp 2:** ảnh `ps -C nginx -o pid,ppid,euser:20,cmd` (**không còn root**); `capsh --decode`; ảnh đối chứng `bind() … (13: Permission denied)` khi thiếu capability.
- [ ] **Cấp 3:** hai ảnh điểm `systemd-analyze security` **trước/sau**; `Seccomp: 2`; `memory.max`; ảnh `Read-only file system`.
- [ ] **Cấp 4:** `aa-status` (enforce); bảng 200 ↔ 403 của `/probe`; dòng `apparmor="DENIED"`.
- [ ] **Cấp 5:** `/proc/PID/limits`; `ss -ltn` (Send-Q); `findmnt` (ro…); ảnh `ps` trước/sau `reload` (PID master giữ nguyên).
- [ ] **Cấp 6:** ảnh kết quả `verify-sv2.sh` (… 0 FAIL).
- [ ] **Cấp 7:** mỗi lab đã làm: triệu chứng – bằng chứng – cách sửa (3 dòng).
- [ ] **Cấp 8 (chỉ nếu hướng B):** hai ảnh điểm `systemd-analyze security school-api` trước/sau; ảnh `curl` qua `127.0.0.1:3000` (được) và qua IP LAN (bị chặn); ảnh `DROP TABLE` bị từ chối với `school_app`; ảnh token cũ 200 → 401 sau khi đổi khóa JWT; ảnh deploy backend tự rollback; ảnh `verify-api.sh` (… 0 FAIL).
- [ ] **Bàn giao (mục 12):** đã gửi thông tin cho SV1/SV3/SV4.
- [ ] Giải thích 5–8 câu **bằng lời của cậu** cho từng cấp đã làm.

# 15. Phụ lục: bản đồ tệp và LỆNH theo thứ tự (copy-paste)

> Nội dung các script/cấu hình dài nằm **trong từng Cấp ở trên** (để khỏi chép trùng). Phụ lục này cho cậu 2 thứ: **(A)** danh sách tệp phải tạo, lấy nội dung ở đâu, đặt quyền gì; **(B)** các lệnh còn lại theo đúng thứ tự làm.
> `<...>` là chỗ cậu điền. Dán từng khối, đọc kết quả rồi mới sang khối sau. Dòng `# ✅` là kết quả mong đợi.

## 15A. Danh sách tệp cần tạo

Cách tạo một tệp từ nội dung trong file này: `sudo nano <đường-dẫn>` → dán → Ctrl+O, Enter, Ctrl+X. (Hoặc dùng `sudo tee <đường-dẫn> >/dev/null <<'EOF' … EOF` như trong từng mục.)

| # | Đường dẫn | Lấy nội dung ở | Chủ / quyền | Sau khi tạo |
|---|---|---|---|---|
| 1 | `/usr/local/sbin/school-deploy.sh` | Cấp 1.2 | `root:root` 755 | `sudo install -o root -g root -m 755 school-deploy.sh /usr/local/sbin/` |
| 2 | `/etc/sudoers.d/school-deploy` | Cấp 1.3 | `root:root` 440 | `sudo visudo -cf /etc/sudoers.d/school-deploy` |
| 3 | `/etc/nginx-school/nginx.conf` | Cấp 2.1 | `root:svc_nginx` 640 | `sudo -u svc_nginx nginx -t -c /etc/nginx-school/nginx.conf` |
| 4 | `/etc/systemd/system/nginx-school.service` | Cấp 2.2 | `root:root` 644 | `sudo systemctl daemon-reload` |
| 5 | `/etc/systemd/system/nginx-school.service.d/10-sandbox.conf` | Cấp 3.2 | `root:root` 644 | `daemon-reload` + `restart nginx-school` |
| 6 | `/etc/apparmor.d/usr.sbin.nginx` | Cấp 4.1 | `root:root` 644 | `sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx` |
| 7 | `/etc/sysctl.d/90-school.conf` | Cấp 5.2 | `root:root` 644 | `sudo sysctl --system` |
| 8 | `~/verify-sv2.sh` | Cấp 6 | tài khoản của cậu | `sudo bash ~/verify-sv2.sh` |
| 9 | `/etc/systemd/system/school-api.service.d/10-sandbox.conf` *(hướng B)* | Cấp 8.1 | `root:root` 644 | `daemon-reload` + `restart school-api` |
| 10 | `/etc/systemd/system/school-api.service.d/20-layout.conf` *(hướng B)* | Cấp 8.4 | `root:root` 644 | `daemon-reload` |
| 11 | `/usr/local/sbin/school-api-deploy.sh` *(hướng B)* | Cấp 8.4 | `root:root` 755 | `sudo chmod 755 …` |
| 12 | `~/verify-api.sh` *(hướng B)* | Cấp 8.5 | tài khoản của cậu | `sudo bash ~/verify-api.sh` |

Mọi tệp trong `/etc` và `/usr/local/sbin` đều thuộc `root`; **không** để `svc_nginx`/`svc_api`/`deploy` sửa được chúng.

## 15B. Lệnh theo thứ tự

**Cấp 1 – Triển khai chuyên nghiệp**
```bash
sudo apt install -y rsync curl
sudo useradd -m -s /bin/bash deploy
sudo install -d -o root -g svc_nginx -m 750 /var/www/school /var/www/school/releases
sudo install -d -o root -g root      -m 750 /var/www/school/manifests

# (tạo tệp #1 và #2 ở bảng 15A)
sudo chmod 440 /etc/sudoers.d/school-deploy && sudo visudo -cf /etc/sudoers.d/school-deploy     # ✅ parsed OK

# Trỏ site cơ bản vào "current" (nếu chưa làm Cấp 2)
sudo sed -i -E 's#^(\s*root\s+)/var/www/school;#\1/var/www/school/current;#' /etc/nginx/sites-available/school

# Triển khai bằng tài khoản deploy
sudo cp -r ~/dist /home/deploy/dist && sudo chown -R deploy: /home/deploy/dist
sudo -u deploy sudo -n /usr/local/sbin/school-deploy.sh deploy /home/deploy/dist
sudo nginx -t && sudo systemctl reload nginx
curl -I http://truong.test/                                           # ✅ 200

# Chứng minh
sudo -l -U deploy
sudo -u deploy sudo -n systemctl stop nginx                           # ✅ bị từ chối
readlink -f /var/www/school/current
sudo school-deploy.sh verify                                          # ✅ nguyên vẹn
sudo sh -c 'echo "<!-- hacked -->" >> /var/www/school/current/index.html'
sudo school-deploy.sh verify                                          # ✅ báo FILE BỊ SỬA
sudo -u deploy sudo -n /usr/local/sbin/school-deploy.sh deploy /home/deploy/dist   # bản mới
sudo school-deploy.sh rollback && sudo school-deploy.sh verify
sudo SCHOOL_URL=http://truong.test/tai-lieu/khong-co.pdf school-deploy.sh deploy /home/deploy/dist   # ✅ tự rollback
while true; do curl -s -o /dev/null -w '%{http_code} ' http://truong.test/; sleep 0.05; done          # Ctrl+C để dừng
```

**Cấp 2 – Chạy toàn bộ Nginx không root**
```bash
sudo install -d -o root -g svc_nginx -m 750 /etc/nginx-school
# (tạo tệp #3 và #4)
sudo chown root:svc_nginx /etc/nginx-school/nginx.conf && sudo chmod 640 /etc/nginx-school/nginx.conf
sudo systemctl disable --now nginx
sudo systemctl daemon-reload
sudo systemctl enable --now nginx-school
systemctl status nginx-school --no-pager                              # ✅ active (running)
curl -I http://truong.test/                                           # ✅ 200

ps -C nginx -o pid,ppid,euser:20,cmd                                  # ✅ TẤT CẢ là svc_nginx
sudo ss -ltnp 'sport = :80'
sudo apt install -y libcap2-bin
PID=$(systemctl show -p MainPID --value nginx-school)
grep -E '^Cap(Prm|Eff|Amb|Bnd)' /proc/$PID/status
capsh --decode=0000000000000400                                       # ✅ cap_net_bind_service
grep NoNewPrivs /proc/$PID/status

# Đối chứng: thiếu capability thì không mở được cổng 80
sudo systemctl stop nginx-school
sudo setpriv --reuid=svc_nginx --regid=svc_nginx --clear-groups \
     /usr/sbin/nginx -g 'daemon off;' -c /etc/nginx-school/nginx.conf 2>&1 | grep -i bind    # ✅ (13: Permission denied)
sudo systemctl start nginx-school
sudo su - svc_nginx                                                   # ✅ "This account is currently not available."
```

**Cấp 3 – Sandbox + cgroup**
```bash
sudo systemd-analyze security nginx-school.service --no-pager | tail -n 1     # điểm TRƯỚC
sudo mkdir -p /etc/systemd/system/nginx-school.service.d
# (tạo tệp #5)
sudo systemctl daemon-reload && sudo systemctl restart nginx-school
systemctl status nginx-school --no-pager && curl -I http://truong.test/
sudo systemd-analyze security nginx-school.service --no-pager | tail -n 1     # điểm SAU (giảm)

PID=$(systemctl show -p MainPID --value nginx-school)
grep -E 'NoNewPrivs|Seccomp' /proc/$PID/status
systemctl show nginx-school -p MemoryMax -p CPUQuotaPerSecUSec -p TasksMax
cat /sys/fs/cgroup/system.slice/nginx-school.service/memory.max
sudo nsenter -t $PID -m ls /home
sudo systemd-run --wait --pipe -p User=svc_nginx -p ProtectSystem=strict touch /usr/local/probe 2>&1 | tail -n 1    # ✅ Read-only file system
```

**Cấp 4 – AppArmor**
```bash
sudo apt install -y apparmor apparmor-utils
sudo aa-status | head -3
# (tạo tệp #6)
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx
sudo aa-complain /usr/sbin/nginx
sudo systemctl restart nginx-school
curl -s -o /dev/null http://truong.test/ ; curl -s -o /dev/null http://truong.test/tai-lieu/thong-bao-mau.txt
sudo journalctl -k --since "5 min ago" | grep -i 'apparmor="ALLOWED"' | head
sudo aa-logprof
sudo aa-enforce /usr/sbin/nginx
sudo systemctl restart nginx-school && curl -I http://truong.test/    # ✅ vẫn 200
sudo aa-status | grep -A2 nginx

# Thí nghiệm DAC vs MAC (/probe)
echo "bí-mật-giả-lập" | sudo tee /etc/school-probe.txt && sudo chmod 644 /etc/school-probe.txt
sudo sed -i '/location ~ \/\\\./i\        location = /probe { alias /etc/school-probe.txt; }' /etc/nginx-school/nginx.conf
sudo systemctl reload nginx-school
sudo aa-complain /usr/sbin/nginx && sudo systemctl restart nginx-school
curl -s -o /dev/null -w "%{http_code}\n" http://truong.test/probe     # ✅ 200
sudo aa-enforce /usr/sbin/nginx && sudo systemctl restart nginx-school
curl -s -o /dev/null -w "%{http_code}\n" http://truong.test/probe     # ✅ 403
sudo journalctl -k | grep -i 'apparmor="DENIED"' | tail -n 2
# Dọn dẹp: xóa dòng "location = /probe" khỏi nginx.conf, rồi:
sudo rm /etc/school-probe.txt && sudo systemctl reload nginx-school
```

**Cấp 5 – Tinh chỉnh HĐH**
```bash
for p in $(pgrep -f 'nginx: worker'); do grep 'Max open files' /proc/$p/limits; done
PID=$(systemctl show -p MainPID --value nginx-school); grep 'Max open files' /proc/$PID/limits
# (tạo tệp #7)
sudo sysctl --system | grep -E 'somaxconn|tcp_max_syn'
ss -ltn 'sport = :80'

# Mount chỉ-đọc (mất khi khởi động lại)
sudo mount --bind /var/www/school /var/www/school
sudo mount -o remount,bind,ro,nosuid,nodev,noexec /var/www/school
findmnt /var/www/school
sudo touch /var/www/school/test                                       # ✅ Read-only file system
curl -I http://truong.test/                                           # ✅ 200
sudo -u deploy sudo -n /usr/local/sbin/school-deploy.sh deploy /home/deploy/dist    # ✅ vẫn thành công (tự unlock/lock)

# Reload không ngắt
ps -C nginx -o pid,ppid,euser:20,lstart,cmd
while true; do curl -s -o /dev/null -w '%{http_code} ' http://truong.test/; sleep 0.05; done &
sudo systemctl reload nginx-school
ps -C nginx -o pid,ppid,euser:20,lstart,cmd                           # ✅ master giữ PID, worker PID mới
kill %1
```

**Cấp 6 – Tự chấm điểm**
```bash
# (tạo tệp #8)
sudo bash ~/verify-sv2.sh                                             # ✅ KẾT QUẢ: N PASS, 0 FAIL
```

**Cấp 8 – Backend + PostgreSQL (hướng B; đã làm xong mục G file Ubuntu)**
```bash
# 8.1 Sandbox + khóa mạng
sudo systemd-analyze security school-api.service --no-pager | tail -n 1
sudo mkdir -p /etc/systemd/system/school-api.service.d
# (tạo tệp #9)
sudo systemctl daemon-reload && sudo systemctl restart school-api
curl -s http://truong.test/api/health
sudo systemd-analyze security school-api.service --no-pager | tail -n 1
IP=$(hostname -I | awk '{print $1}')
curl -s -m 5 http://127.0.0.1:3000/api/health                                    # ✅ ok
curl -s -m 5 http://$IP:3000/api/health || echo "BỊ CHẶN (timeout)"              # ✅ bị chặn

# 8.2 Tách vai trò DB
APPPW=$(openssl rand -hex 16)
sudo -u postgres psql -d school_portal -v pw="$APPPW" -f - <<'SQL'
CREATE ROLE school_app LOGIN PASSWORD :'pw' NOSUPERUSER NOCREATEDB NOCREATEROLE;
REVOKE ALL ON DATABASE school_portal FROM PUBLIC;
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
GRANT CONNECT ON DATABASE school_portal TO school_app;
GRANT USAGE ON SCHEMA public TO school_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO school_app;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO school_app;
ALTER DEFAULT PRIVILEGES FOR ROLE school IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO school_app;
ALTER DEFAULT PRIVILEGES FOR ROLE school IN SCHEMA public GRANT USAGE, SELECT ON SEQUENCES TO school_app;
SQL
sudo sed -i -e 's/^DB_USER=.*/DB_USER=school_app/' -e "s/^DB_PASSWORD=.*/DB_PASSWORD=$APPPW/" /etc/school-api/api.env
sudo systemctl restart school-api && curl -s http://truong.test/api/health
export PGPASSWORD="$APPPW"
psql -h 127.0.0.1 -U school_app -d school_portal -c 'SELECT count(*) FROM users;'     # ✅ đọc được
psql -h 127.0.0.1 -U school_app -d school_portal -c 'DROP TABLE users;'               # ✅ must be owner
psql -h 127.0.0.1 -U school_app -d school_portal -c 'CREATE TABLE hack(x int);'       # ✅ permission denied
unset PGPASSWORD APPPW
sudo -u postgres psql -Atc "SELECT rolname, rolsuper, rolcreatedb FROM pg_roles WHERE rolname LIKE 'school%';"
sudo -u postgres psql -Atc "SHOW listen_addresses"
sudo grep -vE '^\s*(#|$)' /etc/postgresql/*/main/pg_hba.conf

# 8.3 Bí mật + đổi khóa JWT
sudo ls -l /etc/school-api/api.env
PID=$(systemctl show -p MainPID --value school-api)
cat /proc/$PID/environ                                                # ✅ Permission denied
sudo -u nobody cat /proc/$PID/environ                                 # ✅ Permission denied
login() { curl -s -X POST http://truong.test/api/auth/login -H 'Content-Type: application/json' \
          -d "{\"email\":\"$1\",\"password\":\"$2\"}"; }
code()  { curl -s -o /dev/null -w '%{http_code}\n' "$@"; }
ADMIN=$(login admin@adu.edu.vn "<mật-khẩu-seed>" | jq -r .accessToken)
code -H "Authorization: Bearer $ADMIN" http://truong.test/api/auth/me          # ✅ 200
sudo sed -i "s/^JWT_ACCESS_SECRET=.*/JWT_ACCESS_SECRET=$(openssl rand -hex 48)/" /etc/school-api/api.env
sudo systemctl restart school-api
code -H "Authorization: Bearer $ADMIN" http://truong.test/api/auth/me          # ✅ 401

# 8.4 Deploy backend có rollback
sudo mkdir -p /opt/school-api/releases
# (tạo tệp #10 và #11)
sudo systemctl daemon-reload
cd ~/school-portal/backend && npm ci && npm run build
sudo school-api-deploy.sh deploy ~/school-portal/backend
sudo school-api-deploy.sh deploy ~/school-portal/backend
readlink -f /opt/school-api/current
sudo school-api-deploy.sh rollback
mkdir -p /tmp/bad && cp -r ~/school-portal/backend/{dist,package.json,package-lock.json} /tmp/bad/
echo "throw new Error('lỗi cố ý')" > /tmp/bad/dist/main.js
sudo school-api-deploy.sh deploy /tmp/bad                             # ✅ tự rollback
curl -s http://truong.test/api/health                                 # ✅ vẫn ok

# 8.5 Tự chấm
# (tạo tệp #12)
sudo bash ~/verify-api.sh                                             # ✅ 0 FAIL
```

> Lưu ý thứ tự ở Cấp 8.4: dịch vụ `school-api` chỉ chạy được sau khi có bản triển khai đầu tiên (`current` phải tồn tại). Nếu `systemctl status school-api` báo lỗi `WorkingDirectory` **trước** lần `deploy` đầu thì đó là bình thường.

---

> ⚠️ **Ghi chú trung thực:** lệnh và script ở đây được viết theo cách Ubuntu 22.04/24.04, systemd ≥ 249, Nginx 1.18–1.24 và AppArmor hoạt động, nhưng **chưa được chạy thử trên máy cậu**. Chỗ dễ vấp nhất: `SystemCallFilter`/`MemoryDenyWriteExecute` (Cấp 3), profile AppArmor (Cấp 4 – luôn học bằng `complain` trước khi `enforce`), `mount` chỉ-đọc kết hợp deploy (Cấp 5). Nếu dịch vụ không lên: `journalctl -xeu nginx-school -n 30`, chú thích bớt đúng dòng nghi ngờ, restart. Mỗi lỗi gặp + cách sửa **đưa vào hồ sơ** là bằng chứng quá trình làm. Làm **từng cấp một**, xong chụp ảnh rồi mới sang cấp sau. Đừng đưa mật khẩu hay khóa riêng lên GitHub.
