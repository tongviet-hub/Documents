# Lệnh cần chạy để chụp ảnh báo cáo SV2 (Ubuntu)

> Chạy **từng khối**, chụp ảnh cả **lệnh + kết quả** (thấy dấu nhắc `user@host`).
> Dùng `| cat` thay cho `--no-pager` để tránh lỗi `unrecognized option`.
> Đừng chụp mật khẩu, JWT secret, cột `passwordHash`.

---

## Phần 1 – Đã làm xong (C1–C6)

### Ảnh 1 – Thông tin máy
```bash
lsb_release -ds
uname -r
ip -4 addr show | grep inet
```

### Ảnh 2 – Nginx đã cài và chạy
```bash
nginx -v
systemctl status nginx | cat
```
✅ `Active: active (running)`

### Ảnh 3 – Thư mục nội dung và phân quyền
```bash
ls -l /var/www/school
namei -l /var/www/school/index.html
id svc_nginx
```
✅ Chủ `root:svc_nginx`, thư mục `drwxr-x---`, file `-rw-r-----`

### Ảnh 4 – Đặc quyền tối thiểu
```bash
sudo -u svc_nginx cat /var/www/school/index.html | head -3
sudo -u svc_nginx touch /var/www/school/hack.txt
sudo -u nobody cat /var/www/school/index.html
```
✅ Lệnh 1 đọc được · Lệnh 2 và 3 báo `Permission denied`

### Ảnh 5 – Tên miền nội bộ
```bash
cat /etc/hosts
getent hosts truong.test
ping -c 2 truong.test
```
✅ `127.0.0.1   truong.test`

### Ảnh 6 – Tường lửa
```bash
sudo ufw status numbered
```
✅ Có OpenSSH và cổng 80 cho dải LAN

### Ảnh 7 – Cấu hình Nginx
```bash
cat /etc/nginx/sites-available/school
grep -n '^user' /etc/nginx/nginx.conf
ls -l /etc/nginx/sites-enabled/
```
✅ `user svc_nginx;` · trong `sites-enabled` chỉ có `school`, không còn `default`

### Ảnh 8 – Kiểm tra cú pháp
```bash
sudo nginx -t
```
✅ `syntax is ok` và `test is successful`

### Ảnh 9 – Trạng thái dịch vụ sau khi restart
```bash
sudo systemctl restart nginx
systemctl status nginx | cat
```

### Ảnh 10 – Trình duyệt (chụp cả thanh địa chỉ)
- Mở `http://truong.test/`
- Mở `http://truong.test/tai-lieu/thong-bao-mau.txt`

---
viet@viet-BATTLE-AX-B760M-T-WIFI:~$ lsb_release -ds
Ubuntu 26.04.1 LTS
viet@viet-BATTLE-AX-B760M-T-WIFI:~$ ip -4 addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: enp5s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    altname enx8c322322c841
    inet 192.168.102.114/24 brd 192.168.102.255 scope global dynamic noprefixroute enp5s0
       valid_lft 6179sec preferred_lft 6179sec
4: tailscale0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1280 qdisc fq_codel state UNKNOWN group default qlen 500
    inet 100.83.219.103/32 scope global tailscale0
       valid_lft forever preferred_lft forever
viet@viet-BATTLE-AX-B760M-T-WIFI:~$ uname -r
7.0.0-42-generic
viet@viet-BATTLE-AX-B760M-T-WIFI:~$ nginx -v
nginx version: nginx/1.28.3 (Ubuntu)
viet@viet-BATTLE-AX-B760M-T-WIFI:~$ systemctl status nginx --no-paper
systemctl: unrecognized option '--no-paper'
viet@viet-BATTLE-AX-B760M-T-WIFI:~$ systemctl status nginx | cat
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-11-05 08:11:23 +07; 1 day 2h ago
 Invocation: 1836d047b6684462b3a587f4adaf9a96
       Docs: man:nginx(8)
   Main PID: 46587 (nginx)
      Tasks: 13 (limit: 37634)
     Memory: 11M (peak: 12.8M)
        CPU: 90ms
     CGroup: /system.slice/nginx.service
             ├─46587 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             ├─46588 "nginx: worker process"
             ├─46589 "nginx: worker process"
             ├─46590 "nginx: worker process"
             ├─46591 "nginx: worker process"
             ├─46592 "nginx: worker process"
             ├─46593 "nginx: worker process"
             ├─46594 "nginx: worker process"
             ├─46596 "nginx: worker process"
             ├─46597 "nginx: worker process"
             ├─46598 "nginx: worker process"
             ├─46599 "nginx: worker process"
             └─46600 "nginx: worker process"

Nov 05 08:11:23 viet-BATTLE-AX-B760M-T-WIFI systemd[1]: Starting nginx.service - A high performance web server and a reverse proxy server...
Nov 05 08:11:23 viet-BATTLE-AX-B760M-T-WIFI systemd[1]: Started nginx.service - A high performance web server and a reverse proxy server.
viet@viet-BATTLE-AX-B760M-T-WIFI:~$ ls -l /var/www/school
namei -l /var/www/school/index.html
id svc_nginx
ls: cannot open directory '/var/www/school': Permission denied
f: /var/www/school/index.html
drwxr-xr-x root root      /
drwxr-xr-x root root      var
drwxr-xr-x root root      www
drwxr-x--- root svc_nginx school
                           index.html - Permission denied
uid=997(svc_nginx) gid=973(svc_nginx) groups=973(svc_nginx)
viet@viet-BATTLE-AX-B760M-T-WIFI:~$ sudo -u svc_nginx cat /var/www/school/index.html | head -3
sudo -u svc_nginx touch /var/www/school/hack.txt
sudo -u nobody cat /var/www/school/index.html
[sudo: authenticate] Password:  
<!doctype html>
<html lang="vi">
  <head>
touch: cannot touch '/var/www/school/hack.txt': Permission denied
cat: /var/www/school/index.html: Permission denied
viet@viet-BATTLE-AX-B760M-T-WIFI:~$ sudo -u nobody cat /var/www/school/index.html
cat: /var/www/school/index.html: Permission denied
viet@viet-BATTLE-AX-B760M-T-WIFI:~$ cat /etc/hosts
getent hosts truong.test
ping -c 2 truong.test
127.0.0.1 localhost
127.0.1.1 viet-BATTLE-AX-B760M-T-WIFI

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
127.0.0.1    truong.test
127.0.0.1       truong.test
PING truong.test (127.0.0.1) 56(84) bytes of data.
64 bytes from localhost (127.0.0.1): icmp_seq=1 ttl=64 time=0.019 ms
64 bytes from localhost (127.0.0.1): icmp_seq=2 ttl=64 time=0.079 ms

--- truong.test ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1039ms
rtt min/avg/max/mdev = 0.019/0.049/0.079/0.030 ms
viet@viet-BATTLE-AX-B760M-T-WIFI:~$ sudo ufw status numbered
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] OpenSSH                    ALLOW IN    Anywhere                  
[ 2] 80/tcp                     ALLOW IN    192.168.102.0/24          
[ 3] 80/tcp on tailscale0       ALLOW IN    Anywhere                  
[ 4] OpenSSH (v6)               ALLOW IN    Anywhere (v6)             
[ 5] 80/tcp (v6) on tailscale0  ALLOW IN    Anywhere (v6)             

viet@viet-BATTLE-AX-B760M-T-WIFI:~$ cat /etc/nginx/sites-available/school
grep -n '^user' /etc/nginx/nginx.conf
ls -l /etc/nginx/sites-enabled/
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
1:user svc_nginx;
total 0
lrwxrwxrwx 1 root root 33 Nov  5 08:06 school -> /etc/nginx/sites-available/school
viet@viet-BATTLE-AX-B760M-T-WIFI:~$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
viet@viet-BATTLE-AX-B760M-T-WIFI:~$ sudo systemctl restart nginx
systemctl status nginx | cat
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Fri 2026-11-06 10:59:22 +07; 12ms ago
 Invocation: 763a6b5272a2429c96f57d521a5708a9
       Docs: man:nginx(8)
    Process: 79586 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
    Process: 79589 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
   Main PID: 79591 (nginx)
      Tasks: 13 (limit: 37634)
     Memory: 12.2M (peak: 12.2M)
        CPU: 74ms
     CGroup: /system.slice/nginx.service
             ├─79591 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             ├─79592 "nginx: worker process"
             ├─79593 "nginx: worker process"
             ├─79594 "nginx: worker process"
             ├─79595 "nginx: worker process"
             ├─79596 "nginx: worker process"
             ├─79597 "nginx: worker process"
             ├─79598 "nginx: worker process"
             ├─79600 "nginx: worker process"
             ├─79601 "nginx: worker process"
             ├─79602 "nginx: worker process"
             ├─79603 "nginx: worker process"
             └─79604 "nginx: worker process"

Nov 06 10:59:22 viet-BATTLE-AX-B760M-T-WIFI systemd[1]: Starting nginx.service - A high performance web server and a reverse proxy server...
Nov 06 10:59:22 viet-BATTLE-AX-B760M-T-WIFI systemd[1]: Started nginx.service - A high performance web server and a reverse proxy server.

---
## Phần 2 – C7: bằng chứng "ít quyền" (quan trọng nhất)

### Ảnh 11 – Master = root, worker = svc_nginx
```bash
ps -eo pid,ppid,euser:20,comm | grep '[n]ginx'
```
✅ Một dòng `root`, các dòng còn lại `svc_nginx`

### Ảnh 12 – PID nào giữ cổng 80
```bash
sudo ss -ltnp 'sport = :80'
```

### Ảnh 13 – Cấu hình đang nạp đúng user
```bash
sudo nginx -T 2>/dev/null | grep -E '^\s*user '
```

### Ảnh 14 – CPU/RAM các tiến trình nginx
```bash
ps -C nginx -o pid,user,%cpu,%mem,rss,cmd
```

---

## Phần 3 – C8: mã HTTP và đo tải

### Ảnh 15 – Mã trả về
```bash
curl -I http://truong.test/
curl -I http://truong.test/tai-lieu/thong-bao-mau.txt
curl -I http://truong.test/tai-lieu/khong-co.pdf
curl -I http://truong.test/.git/config
```
✅ 200 · 200 · 404 · 403

### Ảnh 16 – 200 yêu cầu liên tiếp
```bash
for i in $(seq 1 200); do curl -s -o /dev/null -w "%{http_code}\n" http://truong.test/; done | sort | uniq -c
```

### Ảnh 17 – Benchmark (mở thêm 1 terminal chạy `top -u svc_nginx` rồi chụp cả hai)
```bash
ab -n 1000 -c 20 http://truong.test/
```
```bash
top -u svc_nginx
```

### Ảnh 18 – Log truy cập
```bash
sudo tail -n 20 /var/log/nginx/school.access.log
sudo awk -F'"' '{split($3,a," "); print a[1]}' /var/log/nginx/school.access.log | sort | uniq -c
```

---

## Phần 4 – KT2: hai lỗi "không vào được web" (nếu được yêu cầu)

### Lỗi 1 – DNS sai
```bash
sudo cp /etc/hosts /etc/hosts.bak
sudo sed -i 's/^127.0.0.1\s\+truong.test/10.255.255.1 truong.test/' /etc/hosts
getent hosts truong.test
ping -c 2 truong.test
curl -m 5 -v http://truong.test/
```
Chụp thêm trình duyệt (treo, `ERR_CONNECTION_TIMED_OUT`).

Khôi phục:
```bash
sudo cp /etc/hosts.bak /etc/hosts
getent hosts truong.test
```

### Lỗi 2 – Dừng dịch vụ
```bash
sudo systemctl stop nginx
getent hosts truong.test
ping -c 2 truong.test
curl -v http://truong.test/
sudo ss -ltnp 'sport = :80'
systemctl status nginx | cat
```
Chụp thêm trình duyệt (`ERR_CONNECTION_REFUSED`).

Khôi phục:
```bash
sudo systemctl start nginx
systemctl status nginx | cat
```

---

## Phần 5 – KT3: sao lưu và khôi phục

```bash
sudo rsync -a --delete /var/www/school/ /var/backups/school-demo/
ls -l /var/backups/school-demo/ | head

sudo rm /var/www/school/index.html
curl -I http://truong.test/            # chụp: lỗi 403/404

sudo rsync -a --delete /var/backups/school-demo/ /var/www/school/
curl -I http://truong.test/            # chụp: 200 OK trở lại
```

---

## Danh sách đối chiếu cuối cùng

- [ ] `sudo nginx -t` OK + trình duyệt mở `http://truong.test`
- [ ] `/etc/hosts` + `getent hosts truong.test`
- [ ] `ps` (root / svc_nginx) + `systemctl status nginx`
- [ ] `ls -l` + `namei -l` + thử ghi bằng `svc_nginx` bị từ chối
- [ ] `ss -ltnp` (PID giữ cổng 80)
- [ ] Tải được `/tai-lieu/thong-bao-mau.txt`
- [ ] Đoạn giải thích ngắn (mẫu ở mục C9 trong `HUONG_DAN_SV2_UBUNTU.md`)
