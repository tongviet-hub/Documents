## 7. PHẦN MỞ RỘNG: TRIỂN KHAI WEBSITE TRƯỜNG HỌC TRÊN KUBERNETES

### 7.1. Mục tiêu và lý do thực hiện

Mục 6 của đề gợi ý phần mở rộng theo hướng Reverse Proxy, Caching hoặc HTTPS. Ngoài phần triển khai Nginx truyền thống trên Ubuntu (`nginx-school`, tên miền `truong.test`) đã trình bày ở các mục trước, nhóm thực hiện thêm một phương án hiện đại hơn: **đưa website của trường lên nền tảng điều phối container Kubernetes** và công khai ra Internet bằng tên miền thật có HTTPS.

* **Hệ thống được triển khai:** cổng thông tin trường **Lumina** gồm website (React, phục vụ bằng Nginx), 3 dịch vụ API (Portal, Academic, Finance) và cơ sở dữ liệu PostgreSQL.
* **Địa chỉ truy cập:** `https://lumina.synnie.dev`
* **Nền tảng:** K3s (bản Kubernetes gọn nhẹ) chạy trên máy chủ Ubuntu của nhóm bằng k3d, phiên bản `v1.31.14+k3s1`.
* **Mục đích so sánh:** cùng một bài toán (công bố website cho trường, an toàn, có log, có giám sát, có sao lưu), đặt cách làm thủ công trên một máy chủ cạnh cách làm theo kiểu hạ tầng điện toán đám mây, để thấy mỗi yêu cầu của đề được giải quyết ra sao ở hai mô hình.

> Phần này **bổ sung**, không thay thế phần Nginx truyền thống. Các yêu cầu bắt buộc của đề vẫn được đáp ứng đầy đủ ở mục 2 và 3; mục 7 cho thấy cùng các khái niệm đó (DNS, cổng, tiến trình, quyền, log, sao lưu, HTTPS) được áp dụng lại ở quy mô lớn hơn.

---

### 7.2. Kiến trúc và luồng yêu cầu trên Kubernetes

#### 7.2.1. Sơ đồ luồng yêu cầu

Sơ đồ dưới đây giữ nguyên 6 bước của sơ đồ mục 2.1 để dễ đối chiếu:

```text
[ Trình duyệt (Browser) ]
        │
        ▼ (1) Phân giải tên miền: lumina.synnie.dev
[ DNS công khai (Cloudflare) ] ──► Trả về IP của Cloudflare (không lộ IP máy chủ trường)
        │
        ▼ (2) Kết nối TCP + bắt tay TLS tới cổng 443 của Cloudflare (HTTPS)
[ Cloudflare Edge: giữ chứng chỉ HTTPS, chống tấn công ]
        │   Đường hầm mã hoá (Cloudflare Tunnel) do máy chủ CHỦ ĐỘNG kết nối ra ngoài
        │   → router/tường lửa của trường KHÔNG cần mở cổng 80/443 nào
        ▼
┌──────────────────────── Cụm Kubernetes "synnie" (máy chủ Ubuntu) ────────────────────────┐
│ [ Pod cloudflared ] (đầu cuối đường hầm)                                                 │
│        │                                                                                 │
│        ▼ (3) Tiếp nhận kết nối, phân tích HTTP Request (Host, Path)                      │
│ [ Traefik – Ingress Controller ]  ≈ "VirtualHost" của Nginx                              │
│        ├── Host = lumina.synnie.dev, Path /api/v1/...  ──► Service "portal"               │
│        └── Host = lumina.synnie.dev, Path /  (còn lại) ──► Service "frontend"             │
│        │                                                                                 │
│        ▼ (4) Service chia tải tới các Pod (DNS nội bộ: tên Service → IP ảo)              │
│ [ Pod frontend x2 ]  Nginx không root, trả file HTML/CSS/JS tĩnh                         │
│ [ Pod portal  x2 ]  ──► [ academic-api ] ──► [ postgres (StatefulSet) ]                  │
│                     └─► [ finance-api ]                                                  │
│        │                                                                                 │
│        ▼ (5) Truy xuất dữ liệu                                                           │
│ [ File tĩnh nằm sẵn trong image (chỉ đọc) ]   [ Ổ đĩa bền vững PVC 2Gi cho PostgreSQL ]   │
└──────────────────────────────────────────────────────────────────────────────────────────┘
        │
        ▼ (6) HTTP Response (200/404/429/503...) đi ngược lại đường hầm → Cloudflare → Trình duyệt
```

#### 7.2.2. Đối chiếu 6 bước với mô hình truyền thống

| Bước | Mô hình Nginx truyền thống (mục 2.1) | Mô hình Kubernetes |
| :--- | :--- | :--- |
| (1) DNS | `/etc/hosts` hoặc DNS nội bộ trỏ `truong.test` → `192.168.102.114` | DNS công khai trỏ `lumina.synnie.dev` → Cloudflare; bên trong cụm có thêm DNS nội bộ (CoreDNS) để các dịch vụ gọi nhau bằng **tên** (`academic-api`, `postgres`) |
| (2) TCP/TLS | Client kết nối thẳng IP:80 của máy chủ | Client kết nối 443 tới Cloudflare; máy chủ tự mở đường hầm ra ngoài, không cần IP công khai hay mở cổng |
| (3) Web server | Nginx đọc `Host`, chọn `server_name` | Traefik đọc `Host` + đường dẫn, chọn Service theo luật **Ingress** |
| (4) Phân quyền tiến trình | Nginx chạy dưới `svc_nginx`, AppArmor, systemd sandbox | Mỗi container chạy user không root, bỏ toàn bộ capability, seccomp, hệ thống tệp chỉ đọc (mục 7.3) |
| (5) File system | `/var/www/school/current` trên đĩa máy chủ | File tĩnh **đóng gói trong image**, dữ liệu động trên **PersistentVolume** |
| (6) Response | Nginx trả về, ghi `access.log` | Pod trả về, log được gom tập trung vào Loki (mục 7.3) |

#### 7.2.3. Các thành phần đã triển khai

| Thành phần | Loại đối tượng Kubernetes | Số bản chạy | Vai trò |
| :--- | :--- | :---: | :--- |
| `frontend` | Deployment + Service | 2 | Nginx (`nginx-unprivileged`) phục vụ website tĩnh |
| `portal` | Deployment + Service | 2 | Cổng API duy nhất nhận yêu cầu từ trình duyệt |
| `academic-api` | Deployment + Service | 1 | Đăng nhập, phân quyền, người dùng |
| `finance-api` | Deployment + Service | 1 | Dịch vụ học phí |
| `postgres` | StatefulSet + PVC 2Gi | 1 | Cơ sở dữ liệu, dữ liệu nằm trên ổ đĩa bền vững |
| `postgres-backup` | CronJob + PVC 2Gi | theo lịch | Sao lưu cơ sở dữ liệu mỗi đêm |
| `portal` | Ingress (Traefik) | – | Định tuyến theo tên miền và đường dẫn |
| Cloudflare Tunnel | Deployment `cloudflared` | 1 | Đưa website ra Internet, HTTPS |
| ArgoCD, Prometheus, Grafana, Loki, Alloy | Ứng dụng hệ thống | – | Triển khai tự động, giám sát, log tập trung |

Toàn bộ cấu hình được viết dưới dạng tệp YAML và lưu trong kho Git (`Infra/apps/portal/`). Không có thao tác cấu hình thủ công nào trên máy chủ cho website này: muốn thay đổi thì sửa tệp, đẩy lên Git, hệ thống tự áp dụng (mục 7.3, dòng "Triển khai").

---

### 7.3. Đối chiếu từng yêu cầu bắt buộc: Nginx truyền thống và Kubernetes

| Yêu cầu của đề | Đã làm với Nginx truyền thống | Cách làm trên Kubernetes | Nhận xét |
| :--- | :--- | :--- | :--- |
| **Tên miền** (2.2) | `truong.test` trong `/etc/hosts`, chỉ dùng trong LAN | Tên miền thật `lumina.synnie.dev` trên DNS công khai; trong cụm dùng DNS nội bộ theo tên Service | Truy cập được từ bất kỳ đâu; các dịch vụ không phụ thuộc IP cố định |
| **Web server / VirtualHost** (2.2) | Nginx `server_name truong.test`, truy cập bằng IP trần bị cắt (444) | Traefik Ingress: chỉ nhận `Host: lumina.synnie.dev`; `/api/v1` → `portal`, còn lại → `frontend` | Cùng nguyên lý khớp `Host`; luật viết dạng khai báo, áp dụng không cần reload tay |
| **Document Root, `chown`/`chmod`** (2.2) | `/var/www/school/current` (symlink vào `releases/`), phân quyền `www-data`/`svc_nginx` | Nội dung web được **đóng gói vào image** lúc build; container chạy với `readOnlyRootFilesystem: true`, chỉ `/tmp` (emptyDir) ghi được | Không ai sửa lén được trang web đang chạy, kể cả khi chiếm được tiến trình |
| **Không chạy root** (2.2) | Toàn bộ Nginx (kể cả master) chạy `svc_nginx`, chỉ giữ 1 capability `CAP_NET_BIND_SERVICE` | `runAsNonRoot: true` (uid 101 cho Nginx, 1000 cho Node.js), `capabilities: drop: [ALL]`, `allowPrivilegeEscalation: false`; Nginx nghe cổng 8080 nên **không cần capability nào** | Chặt hơn một bậc: 0 capability thay vì 1 |
| **Sandbox / kiểm soát truy cập bắt buộc** | systemd sandbox (điểm exposure giảm), AppArmor profile | `seccompProfile: RuntimeDefault` (chặn syscall nguy hiểm), không gắn token truy cập Kubernetes (`automountServiceAccountToken: false`) | Cùng mục tiêu "giới hạn những gì tiến trình làm được", nhưng áp dụng đồng loạt cho mọi container bằng vài dòng cấu hình |
| **Tường lửa** | `ufw` chỉ mở cổng 80 cho dải LAN và Tailscale | **NetworkPolicy chặn mặc định cả chiều vào lẫn chiều ra**, rồi mở đúng từng đường: Traefik → frontend/portal; portal → academic/finance; academic → postgres; mọi Pod → DNS. Không Pod nào được gọi ra Internet | Tường lửa đặt tới **từng dịch vụ**, không chỉ ở biên máy chủ. Một Pod bị chiếm cũng không dò sang Pod khác hay tải mã độc về được |
| **Trần tài nguyên** | cgroup qua systemd: `MemoryMax=300M`, `CPUQuota=80%`, `TasksMax=200` | `resources.requests/limits` cho từng container (vd. portal: yêu cầu 50m CPU / 96Mi, tối đa 256Mi RAM); `ResourceQuota` cho cả namespace (tối đa 4Gi RAM, 20 Pod); `LimitRange` gán mặc định | Bên dưới Kubernetes cũng dùng **cgroup** của Linux, tức là cùng cơ chế HĐH ở mục 2.3, chỉ khác cách khai báo |
| **Mã trạng thái HTTP** (2.3) | Đếm mã trong `access.log` bằng `awk` | Mỗi dòng log của API ghi mã trạng thái, thời gian xử lý và `request_id`; Loki cho phép lọc trực tiếp, vd. `{namespace="portal", status=~"5.."}` | Ngoài 200/404/500, hệ thống còn dùng **429 Too Many Requests** (chống dò mật khẩu, quá tải) và **503** (không còn Pod nào sẵn sàng) |
| **Log** (2.3) | `tail -f /var/log/nginx-school/access.log` | `kubectl logs` cho từng Pod; **Loki + Alloy** gom log của mọi Pod và cả sự kiện Kubernetes, giữ 7 ngày, xem trong Grafana | Log vẫn còn khi Pod đã bị xoá; truy vết được **một yêu cầu đi qua nhiều dịch vụ** bằng `request_id` |
| **PID, CPU, RAM** (2.3) | `top`, `ps aux`, `systemctl show -p MemoryCurrent` | `kubectl top pods`; Prometheus thu số liệu liên tục, Grafana vẽ biểu đồ; **9 luật cảnh báo** gửi về Slack (Pod khởi động lại liên tục, dịch vụ chết, DB chết, hết RAM, sao lưu lỗi...) | Chuyển từ "người vào xem" sang "hệ thống tự báo khi có sự cố" |
| **Giới hạn trang quản trị** (2.4) | Basic Auth (`auth_basic`) hoặc giới hạn dải IP cho `/admin` | Đăng nhập bằng tài khoản thật (JWT, phân quyền theo vai trò sinh viên/giảng viên/quản trị); giới hạn **10 lần đăng nhập/phút** và 60 yêu cầu/phút mỗi IP; các API nội bộ **không có đường nào từ ngoài vào** | Kiểm soát ở tầng ứng dụng thay vì tầng web server. Để giới hạn theo IP chính xác, Traefik được cấu hình giữ **IP thật** của người dùng (`forwardedHeaders.trustedIPs`) |
| **Sao lưu** (2.4) | Script nén `/var/www/school` + `/etc/nginx` chạy bằng `cron`; deploy/rollback bằng symlink | **CronJob** `postgres-backup` chạy 2h sáng mỗi ngày: `pg_dump` → nén `gzip` → kiểm tra file nén bằng `gzip -t` → giữ 7 bản gần nhất. Mã nguồn và cấu hình nằm trong Git nên không cần sao lưu máy chủ | Thứ cần sao lưu thay đổi: trang web và cấu hình đã nằm trong Git/image, **chỉ còn dữ liệu** là phải sao lưu |
| **HTTPS** (2.4) | Đề xuất self-signed (lab) hoặc Let's Encrypt/Certbot (thật) | Chứng chỉ do Cloudflare cấp và **tự gia hạn**; HTTPS bật ở biên, đường hầm từ Cloudflare vào máy chủ cũng được mã hoá | Không phải tự quản lý khoá riêng hay gia hạn chứng chỉ |
| **Triển khai / cập nhật** | Script `school-deploy.sh` (deploy, verify, rollback), `reload` không ngắt kết nối | **CI/CD + GitOps**: đẩy mã nguồn → GitHub Actions kiểm thử, build image, quét lỗ hổng (Trivy, chặn mức CRITICAL), ký số image (cosign) → ghi phiên bản mới vào Git → **ArgoCD** tự đồng bộ vào cụm. Quay lui = `git revert` | Mỗi phiên bản gắn với đúng một commit (`sha-<mã commit>`), biết chính xác máy chủ đang chạy bản nào |
| **Không gián đoạn khi cập nhật** | `reload` giữ PID master; `restart` làm rớt 3 yêu cầu | **Rolling update** `maxUnavailable: 0`: bản mới sẵn sàng mới tắt bản cũ; frontend và portal chạy 2 bản; **PodDisruptionBudget** luôn giữ ≥ 1 bản khi bảo trì | Có kiểm tra sức khoẻ (`startupProbe`, `readinessProbe`, `livenessProbe`) trước khi đưa bản mới vào phục vụ |
| **Tự phục hồi** | Dịch vụ chết thì quản trị viên phải `systemctl start` | Deployment luôn giữ đủ số bản: Pod chết tự được tạo lại; ai sửa tay trên cụm thì ArgoCD (`selfHeal`) tự đưa về đúng như Git | Đây là khác biệt lớn nhất so với mô hình một máy chủ |

---

### 7.4. Kịch bản kiểm thử KT1–KT3 trên Kubernetes

#### KT1 – Truy cập trang web

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://lumina.synnie.dev/      # 200
curl -s https://lumina.synnie.dev/api/v1/health                          # {"status":"ok","service":"portal"}
kubectl logs -n portal deploy/frontend --tail=3                          # dòng log tương ứng
```

Dòng log thực tế ghi nhận được từ Pod `frontend` (định dạng giống `access.log` của Nginx):

```text
10.42.0.157 - - [18/Nov/2026:04:59:59 +0000] "GET / HTTP/1.1" 200 1812 "-" "ApacheBench/2.3" "10.42.0.1"
10.42.0.1 - - [18/Nov/2026:05:00:04 +0000] "GET / HTTP/1.1" 200 1812 "-" "kube-probe/1.31" "-"
```

Dòng thứ hai là **kubelet** tự gọi kiểm tra sức khoẻ (probe) định kỳ, một loại yêu cầu không tồn tại trong mô hình truyền thống.

#### KT2 – Phân biệt lỗi DNS và dừng dịch vụ

| Trường hợp | Mô hình Nginx truyền thống | Mô hình Kubernetes |
| :--- | :--- | :--- |
| **1. Sai DNS** (gõ sai tên miền / trỏ sai IP) | `DNS_PROBE_FINISHED_NXDOMAIN`, `ping` không tìm thấy host | Giống hệt: `DNS_PROBE_FINISHED_NXDOMAIN`. Lỗi DNS xảy ra **trước khi** yêu cầu tới hạ tầng nên không phụ thuộc mô hình |
| **2. Dừng dịch vụ web** | `systemctl stop nginx-school` → `ERR_CONNECTION_REFUSED`, cổng 80 không mở | Cổng 443 **vẫn mở** (Cloudflare và Traefik vẫn chạy) nên **không có** `ERR_CONNECTION_REFUSED`. Khi không còn Pod nào sẵn sàng, Traefik trả **HTTP 503 Service Unavailable** |
| **3. Một tiến trình bị kill** | Dịch vụ dừng cho tới khi quản trị viên bật lại | Pod bị xoá được **tạo lại tự động** trong vài giây; nhờ chạy 2 bản, người dùng không thấy lỗi |

Lệnh thực hiện trường hợp 2 và 3 (làm gián đoạn website trong thời gian ngắn, chỉ chạy khi demo):

```bash
# Trường hợp 3: xoá một Pod, quan sát Pod mới được tạo thay thế
kubectl delete pod -n portal -l app.kubernetes.io/name=frontend --wait=false
kubectl get pods -n portal -w

# Trường hợp 2: phải TẮT tự đồng bộ của ArgoCD trước, nếu không ArgoCD bật lại ngay
kubectl patch application portal -n argocd --type merge -p '{"spec":{"syncPolicy":{"automated":null}}}'
kubectl scale deploy/frontend -n portal --replicas=0
curl -s -o /dev/null -w '%{http_code}\n' https://lumina.synnie.dev/      # mong đợi 503
# Khôi phục: bật lại tự đồng bộ, ArgoCD tự trả về 2 bản
kubectl patch application portal -n argocd --type merge -p '{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'
```

**Kết luận KT2:** ở mô hình Kubernetes, câu hỏi chẩn đoán ở mục 5 ("ping được nhưng web không mở") vẫn đúng về nguyên lý, nhưng phải xét thêm nhiều tầng: Cloudflare → đường hầm → Traefik → Service → Pod. Mã lỗi HTTP (502/503/504) thay cho lỗi kết nối TCP cho biết lỗi nằm ở tầng nào.

#### KT3 – Khôi phục từ bản sao lưu

Bản sao lưu được tạo tự động (thực tế ghi nhận ngày 18/11/2026):

```text
Đã tạo /backup/portal-20261118-020915.sql.gz (1456 bytes)
-rw-r--r--    1 postgres postgres    1.4K Nov 18 02:09 portal-20261118-020915.sql.gz
```

Quy trình khôi phục và kiểm chứng:

1. Chạy sao lưu ngay: `kubectl create job -n portal --from=cronjob/postgres-backup backup-tay-$(date +%s)`.
2. Tạo một Pod tạm có gắn ổ sao lưu, nạp file `.sql.gz` mới nhất vào một cơ sở dữ liệu tạm `restore_test`.
3. So sánh số bản ghi bảng `users` giữa cơ sở dữ liệu thật và bản khôi phục: **hai con số phải bằng nhau**.
4. Xoá cơ sở dữ liệu tạm và Pod tạm.

Khôi phục **nội dung website** không cần bản sao lưu: trang web nằm trong image gắn với commit Git, nên chỉ cần `git revert` commit triển khai lỗi trong kho `Infra`, ArgoCD tự đưa phiên bản cũ trở lại. Cách này tương đương lệnh `school-deploy.sh rollback` ở mô hình truyền thống nhưng có lịch sử đầy đủ trên Git.

---

### 7.5. Số liệu đo và so sánh

#### 7.5.1. Thông lượng

Đo bằng cùng công cụ và tham số với phần Nginx truyền thống (`ab -n 1000 -c 20`), gọi trực tiếp vào Traefik trên máy chủ (không qua Internet), ngày 18/11/2026:

```bash
ab -n 1000 -c 20 -H "Host: lumina.synnie.dev" http://localhost:8080/
```

| Chỉ số | Nginx truyền thống (SV2, 06/11) | Kubernetes – trang tĩnh (`/`) |
| :--- | :---: | :---: |
| Số yêu cầu thành công | 1000/1000 | 1000/1000 |
| Yêu cầu lỗi | 0 | 0 |
| Thông lượng | ~35 500 yêu cầu/giây | **~9 000 yêu cầu/giây** |
| Thời gian phản hồi trung bình | < 1 ms | 2,2 ms |
| 99% yêu cầu xong trong | ≤ 2 ms | ≤ 12 ms |

**Nhận xét:** Kubernetes chậm hơn khoảng 4 lần ở phép đo này vì mỗi yêu cầu phải đi thêm nhiều chặng: Docker (k3d) → Traefik → mạng ảo của cụm → Pod Nginx. Đây là **cái giá của lớp trừu tượng**. Tuy vậy 9 000 yêu cầu/giây vẫn dư rất nhiều so với nhu cầu của một website trường học, và độ trễ 2 ms là không đáng kể so với độ trễ Internet (hàng chục ms). Lưu ý: số của SV2 đo trên bản Nginx cơ bản chưa bật giới hạn tài nguyên, hai phép đo không cùng ngày nên chỉ mang tính tham khảo.

#### 7.5.2. Chống quá tải ở tầng API

Chạy cùng lệnh vào API (`/api/v1/health`):

| Chỉ số | Kết quả |
| :--- | :---: |
| Yêu cầu được phục vụ (200) | 84 |
| Yêu cầu bị từ chối | 916 |
| Mã trả về khi bị từ chối | `429 Too Many Requests`, kèm header `Retry-After: 55` |

Đây là **kết quả mong muốn**: Portal giới hạn mỗi IP 60 yêu cầu/phút (mỗi bản đếm riêng, chạy 2 bản). Một công cụ bắn tải hoặc kẻ dò mật khẩu bị chặn ngay ở tầng ứng dụng, các dịch vụ phía sau và cơ sở dữ liệu không bị ảnh hưởng.

#### 7.5.3. Tài nguyên sử dụng

Đo bằng `kubectl top pods -n portal` (trạng thái bình thường):

| Pod | CPU | RAM |
| :--- | :---: | :---: |
| frontend (mỗi bản) | 1m | 9 Mi |
| portal (mỗi bản) | 1m | 43–46 Mi |
| academic-api | 1m | 60 Mi |
| finance-api | 1m | 38 Mi |
| postgres | 8–11m | 40 Mi |

Nginx trong container chỉ dùng **9 Mi RAM**, tương đương con số ~6,3 MB/worker đo ở mô hình truyền thống. Toàn bộ website (7 Pod) dùng chưa tới 250 Mi RAM. Phần tốn tài nguyên hơn là các thành phần nền (ArgoCD, Prometheus, Grafana, Loki), vốn dùng chung cho mọi ứng dụng trên cụm.

---

### 7.6. Kết quả kiểm chứng thực tế

Các kiểm tra sau được chạy trên hệ thống đang hoạt động ngày 18/11/2026:

| # | Kiểm tra | Lệnh | Kết quả | Ý nghĩa |
| :---: | :--- | :--- | :--- | :--- |
| 1 | Website truy cập được từ Internet qua HTTPS | `curl -I https://lumina.synnie.dev/` | `200` | Đường DNS → Cloudflare → đường hầm → Traefik → Pod hoạt động |
| 2 | API hoạt động | `curl https://lumina.synnie.dev/api/v1/health` | `{"status":"ok","service":"portal"}` | Định tuyến `/api/v1` tới Portal đúng |
| 3 | Pod khác **không** gọi được API nội bộ | `kubectl exec deploy/frontend -- wget http://academic-api:8081/...` | `Connection refused` | NetworkPolicy chặn đúng |
| 4 | Portal gọi được API nội bộ | `kubectl exec deploy/portal -- wget http://academic-api:8081/...` | `{"status":"ok"}` | Đường được phép vẫn thông |
| 5 | Pod không ra được Internet | `kubectl exec deploy/portal -- wget https://example.com` | Bị chặn | Chặn chiều ra hoạt động |
| 6 | Không ghi được vào mã nguồn | `kubectl exec deploy/portal -- touch /app/x` | `Read-only file system` | Hệ thống tệp chỉ đọc |
| 7 | Toàn bộ ứng dụng đồng bộ với Git | `kubectl get application -n argocd` | 7 ứng dụng `Synced` / `Healthy` | Trạng thái cụm khớp với cấu hình trong Git |
| 8 | Số Pod chạy | `kubectl get pods -n portal` | 7 Pod `Running`, 0 lần khởi động lại | Hệ thống ổn định |

**Minh chứng cần nộp kèm (ảnh chụp):**

* Trình duyệt mở `https://lumina.synnie.dev` có biểu tượng ổ khoá HTTPS.
* `kubectl get pods,svc,ingress,networkpolicy -n portal`.
* Giao diện ArgoCD: ứng dụng `portal` ở trạng thái Synced/Healthy, sơ đồ các tài nguyên.
* Grafana → Explore → Loki: truy vấn `{namespace="portal"}` hiển thị log.
* Tin cảnh báo gửi về Slack.
* GitHub Actions: một lần chạy CD thành công (build → quét → ký → cập nhật Infra → smoke test).
* Kết quả 8 kiểm tra trong bảng trên và kết quả `ab`.

---

### 7.7. Đánh giá so sánh hai mô hình

| Tiêu chí | Nginx truyền thống trên một máy chủ | Kubernetes |
| :--- | :--- | :--- |
| Độ phức tạp để học và dựng | Thấp, phù hợp nắm vững nền tảng HĐH và mạng | Cao, nhiều khái niệm mới (Pod, Service, Ingress, NetworkPolicy, GitOps) |
| Hiệu năng thô (1 máy) | Cao hơn (~35 500 yêu cầu/giây) | Thấp hơn (~9 000 yêu cầu/giây) do thêm nhiều chặng |
| Tài nguyên nền | Rất ít | Tốn thêm RAM cho ArgoCD, giám sát, log |
| Tự phục hồi khi lỗi | Không, cần người can thiệp | Có: tự tạo lại Pod, tự đưa cấu hình về như Git |
| Cập nhật không gián đoạn | Có (`reload`), nhưng `restart` vẫn rớt yêu cầu | Có, mặc định qua rolling update + kiểm tra sức khoẻ |
| Tái lập hệ thống trên máy mới | Phải làm lại từng bước theo tài liệu | Áp dụng lại các tệp YAML trong Git |
| Bảo mật mạng | Tường lửa ở biên máy chủ | Tường lửa tới từng dịch vụ, mặc định chặn |
| Log và giám sát | Đọc file log, chạy `top` thủ công | Tập trung, có biểu đồ và cảnh báo tự động |
| Chẩn đoán lỗi | Đơn giản, ít tầng | Khó hơn, phải hiểu nhiều tầng |
| HTTPS | Tự quản lý chứng chỉ | Cloudflare quản lý, tự gia hạn |

**Hạn chế của phần triển khai hiện tại:**

* Cụm chỉ có **1 node** chạy trên một máy chủ của nhóm: máy tắt hoặc hỏng thì website ngừng. Chạy 2 bản frontend/portal chỉ bảo vệ khỏi lỗi tiến trình, chưa bảo vệ khỏi lỗi phần cứng.
* Bản sao lưu nằm **cùng máy** với cơ sở dữ liệu; hỏng ổ đĩa có thể mất cả hai. Hướng cải thiện: đẩy thêm một bản ra kho lưu trữ ngoài (S3, Cloudflare R2).
* `academic-api` mới chạy 1 bản vì bộ đếm chống dò mật khẩu nằm trong RAM; muốn chạy nhiều bản phải chuyển bộ đếm sang Redis.
* Chưa có kiểm tra từ bên ngoài (đo website có mở được từ Internet hay không theo chu kỳ).

**Khi nào chọn mô hình nào:** với một website tĩnh nhỏ, một máy chủ Nginx được cấu hình cẩn thận như mục 2 là đủ, nhanh và dễ quản trị. Khi hệ thống có nhiều dịch vụ, cần cập nhật thường xuyên, cần tự phục hồi và giám sát tập trung, Kubernetes đem lại lợi ích rõ rệt dù phải trả giá bằng độ phức tạp và tài nguyên.

---

### 7.8. Kết luận phần mở rộng

* Mọi yêu cầu bắt buộc của Đề tài 05 (luồng yêu cầu, tên miền, web server, chạy không root, phân quyền, mã HTTP, log, giám sát tài nguyên, giới hạn khu vực quản trị, sao lưu, HTTPS và các kịch bản KT1–KT3) đều có **phương án tương ứng** trên Kubernetes và đã được kiểm chứng trên hệ thống thật tại `https://lumina.synnie.dev`.
* Các khái niệm nền tảng của môn học vẫn là gốc: Kubernetes không thay thế DNS, TCP, tiến trình, quyền hay cgroup của Linux, mà **tự động hoá và áp dụng chúng đồng loạt** cho nhiều dịch vụ.
* Đổi lại, mô hình này chậm hơn ở phép đo thô và phức tạp hơn khi vận hành. Lựa chọn mô hình phải dựa trên quy mô và yêu cầu thực tế của nhà trường.
