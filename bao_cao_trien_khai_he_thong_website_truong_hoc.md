# BÁO CÁO ĐỒ ÁN / BÀI TẬP LỚN
## ĐỀ TÀI 05: PHÂN TÍCH VÀ TRIỂN KHAI HỆ THỐNG WEBSITE CHO TRƯỜNG HỌC

---

### THÔNG TIN CHUNG
* **Môn học:** Mạng máy tính & Hệ điều hành / Quản trị hệ thống
* **Lớp:** ................................................................
* **Nhóm:** ...............................................................
* **Hạn nộp:** ............................................................
* **Danh sách thành viên:**
  1. **SV1:** .................................................... (Mã SV: ....................)
  2. **SV2:** .................................................... (Mã SV: ....................)
  3. **SV3:** .................................................... (Mã SV: ....................)
  4. **SV4:** .................................................... (Mã SV: ....................)
  5. **SV5 (nếu có):** ........................................... (Mã SV: ....................)

---

## 1. BÀI TOÁN VÀ PHẠM VI

* **Mục tiêu:** Xây dựng hạ tầng Web Server phục vụ việc công bố bảng tin, thông báo và cho phép người dùng (học sinh, sinh viên, giảng viên) tải tài liệu học tập.
* **Phạm vi triển khai:** 
  * Tập trung vào kiến trúc mạng, cấu hình dịch vụ web (Apache/Nginx/IIS), phân quyền tệp tin, an toàn bảo mật, giám sát tài nguyên và sao lưu phục hồi.
  * **Không yêu cầu** lập trình giao diện hoặc cổng thông tin phức tạp (sử dụng HTML/CSS tĩnh hoặc hệ thống CMS cơ bản).

---

## 2. PHÂN TÍCH VÀ THỰC HIỆN YÊU CẦU BẮT BUỘC

### 2.1. Kiến trúc luồng yêu cầu (Request Flow Architecture)
*Sơ đồ luồng từ Trình duyệt (Client) qua DNS, IP, Cổng dịch vụ đến Web Server và File System:*

```text
[ Trình duyệt (Browser) ]
        │
        ▼ (1) Phân giải tên miền (DNS Query)
[ DNS Server / Local hosts ] ──► Trả về IP tương ứng (vd: 192.168.1.100)
        │
        ▼ (2) Thiết lập kết nối TCP (SYN -> SYN/ACK -> ACK) tới IP:Port (80/443)
[ Mạng / Firewall / NAT ]
        │
        ▼ (3) Tiếp nhận Socket connection
[ HTTP Server (Nginx / Apache / IIS) ]
        │ 
        ├─ Phân tích HTTP Request (Method, Path, Headers)
        ├─ Kiểm tra VirtualHost / Server Block khớp Domain/Port
        ▼ (4) Ánh xạ URL sang Document Root trên đĩa cứng
[ Quyền tiến trình (Process Permissions / DAC / SELinux) ]
        │
        ▼ (5) Truy xuất tập tin vật lý
[ Hệ thống tập tin (File System / HDD / SSD) ]
        │
        ▼ (6) Đóng gói HTTP Response (Status 200/404, Headers, Data)
[ Trình duyệt hiển thị trang / Tải tài liệu ]
```

### 2.2. Triển khai Web Server và cấu hình môi trường
* **Hệ điều hành lựa chọn:** Linux (Ubuntu Server / Rocky Linux) hoặc Windows Server.
* **Web Server lựa chọn:** Nginx / Apache HTTPD.
* **Cấu hình Tên miền nội bộ:**
  * Cấu hình DNS Server (BIND9/dnsmasq) hoặc file cấu hình cục bộ (`/etc/hosts` trên Linux hoặc `C:\Windows\System32\drivers\etc\hosts` trên Windows).
  * Ví dụ: `192.168.1.50   school.internal.edu.vn`
* **Cấu hình Document Root & Virtual Host:**
  * Thư mục chứa web: `/var/www/school_web/`
  * Quyền sở hữu và phân quyền (Linux):
    ```bash
    chown -R www-data:www-data /var/www/school_web
    chmod -R 755 /var/www/school_web
    ```
* **Quyền hạn tiến trình:** Web server chỉ chạy dưới quyền tài khoản không đặc quyền (`www-data` hoặc `nobody`), tuyệt đối không cho phép tiến trình chạy dưới quyền `root`/`Administrator`.

### 2.3. Giám sát tài nguyên, HTTP Status Code và Log hệ thống
* **Mã trạng thái HTTP cần kiểm soát:**
  * `200 OK`: Truy cập trang hoặc tải tài liệu thành công.
  * `301/302`: Điều hướng (Redirect).
  * `403 Forbidden`: Truy cập thư mục quản trị hoặc tệp bị cấm.
  * `404 Not Found`: Đường dẫn tài liệu không tồn tại.
  * `500 Internal Server Error`: Lỗi cấu hình máy chủ hoặc mã nguồn.
* **Giám sát Process & Resource:**
  * Xem PID và lượng tiêu thụ CPU/RAM: `top`, `htop`, `ps aux | grep nginx`.
  * Đọc log truy cập và lỗi theo thời gian thực:
    ```bash
    tail -f /var/log/nginx/access.log
    tail -f /var/log/nginx/error.log
    ```
* **Vai trò của Hệ điều hành:**
  * Quản lý tiến trình (Process Scheduling) và đa luồng/đa tiến trình (Worker processes).
  * Quản lý bộ nhớ (Virtual Memory, Buffer/Cache) khi phục vụ file tĩnh dung lượng lớn.
  * Quản lý Networking Stack (TCP Window, Socket buffers, Epoll/Select) khi có tải cao.

### 2.4. Phân quyền quản trị, Sao lưu phục hồi và Phương án HTTPS
* **Giới hạn truy cập trang quản trị (`/admin`):**
  * Giới hạn dải IP nội bộ hoặc sử dụng Basic Authentication (`.htpasswd` / `auth_basic`).
* **Kế hoạch sao lưu (Backup Policy):**
  * Script tự động nén thư mục web (`/var/www/school_web`) và cấu hình (`/etc/nginx/`).
  * Thực hiện sao lưu định kỳ qua `cronjob`.
* **Phương án triển khai HTTPS thực tế:**
  * **Môi trường nội bộ (Lab/Staging):** Tạo Self-signed Certificate bằng `OpenSSL`.
  * **Môi trường sản xuất (Production):** 
    * Sử dụng chứng chỉ miễn phí tự động gia hạn: **Let's Encrypt** thông qua công cụ **Certbot**.
    * Hoặc sử dụng chứng chỉ thương mại (Organization Validation - OV / Extended Validation - EV) cho cổng thông tin chính thức của nhà trường.
    * Bật TLS 1.3, cấu hình HTTP Strict Transport Security (HSTS) và tự động chuyển hướng HTTP (port 80) sang HTTPS (port 443).

---

## 3. KỊCH BẢN KIỂM THỬ NGHIỆM THU TỐI THIỂU (ACCEPTANCE TESTS)

| Mã test | Kịch bản thực hiện | Kết quả mong đợi | Bằng chứng cần nộp |
| :--- | :--- | :--- | :--- |
| **KT1** | **Truy cập & Tải tài liệu:**<br>Từ máy Client trong mạng LAN, mở trình duyệt gõ `http://school.internal.edu.vn`, xem trang chủ thông báo và nhấn tải một file PDF bài giảng. | Trang hiển thị đúng nội dung; file tải về thành công nguyên vẹn; log máy chủ ghi mã `200 OK`. | - Ảnh chụp màn hình Client tải tệp.<br>- Trích xuất dòng log tương ứng trong file `access.log`. |
| **KT2** | **Phân biệt lỗi DNS vs Dừng dịch vụ:**<br>1. Sửa sai DNS trên Client (đổi IP domain thành IP không tồn tại) rồi truy cập web.<br>2. Sửa lại DNS đúng, sau đó trên Server tắt dịch vụ web (`systemctl stop nginx`) rồi Client truy cập web. | - TH 1: Trình duyệt báo `DNS_PROBE_FINISHED_NXDOMAIN` hoặc `Server IP address could not be found`. Lệnh `ping domain` không tìm thấy host.<br>- TH 2: Trình duyệt báo `ERR_CONNECTION_REFUSED`. Lệnh `ping domain` vẫn thông, nhưng port 80/443 không phản hồi. | - Bảng so sánh 2 trường hợp.<br>- Ảnh chụp thông báo trình duyệt và kết quả lệnh `curl`/`ping`/`telnet` của cả 2 trường hợp. |
| **KT3** | **Khôi phục từ bản sao lưu:**<br>1. Xóa nội dung website hoặc sửa tệp `index.html` làm trang web lỗi.<br>2. Chạy lệnh/script restore từ file backup đã tạo trước đó. | Trang web hiển thị lại đúng trạng thái ban đầu, tài liệu tải lại bình thường. | - Ảnh chụp trang web bị lỗi/trắng.<br>- Lệnh giải nén bản sao lưu.<br>- Ảnh trang web đã hoạt động lại. |

---

## 4. BẢNG PHÂN CÔNG VÀ MINH CHỨNG CÁ NHÂN

| Vai trò | Phân công nhiệm vụ | Sản phẩm / Minh chứng cần nộp |
| :--- | :--- | :--- |
| **SV1** | **Thiết kế luồng truy cập và DNS** | File sơ đồ kiến trúc luồng; file cấu hình DNS/hosts; video/ảnh chụp kết quả phân giải domain từ client (`nslookup`, `dig`). |
| **SV2** | **Cài đặt Web Server & Cấu hình HĐH** | Toàn bộ file cấu hình Web Server (`nginx.conf` hoặc `httpd.conf`); thông số cấu hình mạng và tường lửa trên server. |
| **SV3** | **Thiết lập quyền, Log và Sao lưu** | Script phân quyền; script sao lưu tự động (bash/powershell); cấu hình xác thực khu vực quản trị; kịch bản test KT3. |
| **SV4** | **Đo tài nguyên và Chẩn đoán lỗi** | Bảng số liệu giám sát tải (sử dụng `ab` hoặc `wrk` để bắn tải); báo cáo phân biệt lỗi KT2; bằng chứng log trạng thái HTTP. |
| **SV5** *(nếu có)* | **Tối ưu hóa & Nghiệm thu tổng hợp** | Tổng hợp hồ sơ nộp, thực hiện phần mở rộng tự chọn (Reverse Proxy, Caching hoặc HTTPS TLS Certificate). |

---

## 5. CÂU HỎI VẤN ĐÁP BẢO VỆ & HƯỚNG DẪN TRẢ LỜI

### Câu hỏi: *"Ping thành công nhưng Website không mở được thì kiểm tra gì tiếp?"*

**Trình tự chẩn đoán theo mô hình OSI / TCP-IP:**

1. **Ý nghĩa:** Lệnh `ping` sử dụng giao thức **ICMP** (tầng Network/Internet). Ping thành công chứng tỏ kết nối vật lý, cấu hình IP, định tuyến (Routing) và Gateway từ Client đến Server hoàn toàn bình thường. Website không mở được (HTTP/HTTPS - tầng Application, chạy trên nền TCP) nghĩa là vấn đề nằm ở tầng Transport trở lên.
2. **Các bước kiểm tra tiếp theo:**
   * **Bước 1: Kiểm tra trạng thái tiến trình dịch vụ trên Server:**
     * Lệnh: `systemctl status nginx` (hoặc `apache2`).
     * Kiểm tra xem tiến trình có đang chạy (`active (running)`) hay đã bị crash/dừng.
   * **Bước 2: Kiểm tra cổng dịch vụ (Port Listening):**
     * Lệnh trên Server: `ss -tuln | grep -E ':80|:443'` hoặc `netstat -an`.
     * Xem Web server có đang thực sự lắng nghe trên cổng 80/443 của card mạng đó hay chỉ lắng nghe trên `127.0.0.1` (localhost).
   * **Bước 3: Kiểm tra Tường lửa (Firewall):**
     * Kiểm tra tường lửa trên Server: `ufw status` (Ubuntu), `firewall-cmd --list-all` (RHEL/CentOS) hoặc Windows Firewall xem port 80/443 có bị chặn (DROP/REJECT) không.
   * **Bước 4: Kiểm tra kết nối TCP từ phía Client:**
     * Từ máy Client dùng lệnh: `telnet <IP_Server> 80` hoặc `Test-NetConnection -ComputerName <IP_Server> -Port 80` (Powershell) hoặc `curl -Iv http://<IP_Server>`.
     * Nếu không kết nối được: vấn đề do Port bị chặn hoặc dịch vụ chưa mở.
     * Nếu kết nối được cổng nhưng trình duyệt vẫn lỗi: xem lại cấu hình VirtualHost (Server Name có đúng domain client gửi trong header `Host:` không) hoặc mã lỗi HTTP trả về (403, 500,...).

---

## 6. PHẦN MỞ RỘNG TỰ CHỌN (GỢI Ý THỰC HIỆN)

* **Đề xuất:** Triển khai **Nginx làm Reverse Proxy & Caching** đứng trước **Apache Web Server** (hoặc so sánh hiệu năng giữa **Nginx** và **Apache** khi phục vụ tệp tĩnh).
* **Tiêu chí so sánh:**
  * Thông lượng xử lý (Requests per second - RPS) dùng công cụ `ApacheBench (ab)`.
  * Thời gian phản hồi trung bình (Latency).
  * Lượng chiếm dụng RAM và CPU của hệ thống khi có 1,000 yêu cầu đồng thời.