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
