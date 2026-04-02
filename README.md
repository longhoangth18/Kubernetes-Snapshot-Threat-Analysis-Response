# K8s DFIR Lab – Phase 1: Logging & DFIR Preparation

## Objective

Thiết lập nền tảng phục vụ DFIR trên Kubernetes, bao gồm:

* Audit logging (control-plane visibility)
* Log hệ thống và container runtime
* Bộ công cụ phục vụ snapshot, phân tích tĩnh và động

Mục tiêu của phase này là đảm bảo:

* Có đủ dữ liệu để điều tra
* Có đủ công cụ để xử lý incident

---

## Phase 1 – Checklist

### Logging

* [ ] Enable Kubernetes audit logging
* [ ] Xác nhận log tại `/var/log/kubernetes/audit.log`
* [ ] Kiểm tra kubelet log
* [ ] Kiểm tra container runtime log

### Tooling

* [ ] Cài đặt tool snapshot (CRIU)
* [ ] Cài đặt tool static analysis
* [ ] Cài đặt tool dynamic analysis
* [ ] Cài đặt tool runtime/debug

### Validation

* [ ] Log ghi nhận API activity
* [ ] Tool hoạt động bình thường
* [ ] Có thể đọc log và inspect container

---

## 1. Audit Logging (Control Plane Visibility)

<details>
<summary>Cấu hình audit logging</summary>

### Mục đích

Ghi lại toàn bộ hành vi API:

* ai thực hiện
* thao tác gì
* resource nào bị tác động

### Log path

```bash
/var/log/kubernetes/audit.log
```

### Kiểm tra

```bash
sudo tail -f /var/log/kubernetes/audit.log
```

### Test

```bash
kubectl create ns dfir-test
kubectl delete ns dfir-test
```

### Ý nghĩa DFIR

Audit log cung cấp:

* timeline cấp control-plane
* truy vết hành vi attacker qua API
* phát hiện truy cập secrets, RBAC abuse

</details>

---

## 2. Node & Runtime Logging

<details>
<summary>Log hệ thống và container runtime</summary>

### Mục đích

Thu thập log ở cấp node để bổ sung cho audit log.

### Các nguồn log quan trọng

```bash
# kubelet
journalctl -u kubelet

# container runtime
crictl ps
crictl logs <container-id>

# pod log
kubectl logs <pod>
```

### Ý nghĩa DFIR

* kubelet log: hành vi scheduling, container lifecycle
* runtime log: stdout/stderr của container
* kết hợp với audit log để dựng timeline đầy đủ

</details>

---

## 3. Snapshot & Restore Tooling

<details>
<summary>Chuẩn bị công cụ checkpoint</summary>

### Mục đích

* snapshot container đang chạy
* lưu state: memory, process, filesystem
* phục vụ forensic

### Tools

| Tool          | Mục đích                   |
| ------------- | -------------------------- |
| criu          | checkpoint/restore process |
| checkpointctl | đọc checkpoint             |
| buildah       | build image từ checkpoint  |
| crictl        | debug container            |

### Cài đặt

```bash
sudo apt update
sudo apt install -y criu jq curl tar
```

### Ý nghĩa DFIR

Checkpoint là artifact quan trọng nhất:

* giữ lại RAM (tránh mất khi pod bị xóa)
* lưu process tree
* lưu network state

</details>

---

## 4. Static Analysis Tooling

<details>
<summary>Phân tích artifact</summary>

### Mục đích

Phân tích dữ liệu từ checkpoint:

* file bị sửa
* process
* memory
* credential

### Tools

| Tool          | Mục đích               |
| ------------- | ---------------------- |
| checkpointctl | inspect process, mount |
| crit          | parse CRIU image       |
| strings       | extract data từ memory |
| grep          | tìm IOC                |
| jq            | parse JSON             |

### Ví dụ

```bash
strings checkpoint/* | grep http
```

### Ý nghĩa DFIR

Có thể tìm:

* URL C2
* token
* command attacker

</details>

---

## 5. Dynamic Analysis Tooling

<details>
<summary>Phân tích hành vi runtime</summary>

### Mục đích

Phân tích container sau khi restore:

* syscall
* network
* execution flow

### Tools

| Tool         | Mục đích            |
| ------------ | ------------------- |
| tcpdump      | capture network     |
| strace       | trace syscall       |
| ss / netstat | kiểm tra connection |
| lsof         | file descriptor     |

### Cài đặt

```bash
sudo apt install -y tcpdump strace net-tools lsof
```

### Ý nghĩa DFIR

* quan sát hành vi attacker
* phát hiện C2
* theo dõi command execution

</details>

---

## 6. Runtime & Debug Tooling

<details>
<summary>Quan sát hệ thống</summary>

### Tools

| Tool       | Mục đích        |
| ---------- | --------------- |
| kubectl    | quản lý cluster |
| crictl     | debug container |
| journalctl | log hệ thống    |
| tail       | theo dõi log    |

### Kiểm tra

```bash
kubectl get nodes
crictl ps
```

</details>

---

## 7. DFIR Workflow Mapping

<details>
<summary>Mapping tổng thể</summary>

| Phase            | Thành phần          |
| ---------------- | ------------------- |
| Detection        | audit log           |
| Snapshot         | criu                |
| Storage          | filesystem          |
| Static analysis  | checkpointctl, crit |
| Dynamic analysis | tcpdump, strace     |
| Investigation    | grep, strings       |

### Flow

1. Detect từ audit log
2. Snapshot container
3. Lưu checkpoint
4. Phân tích:

   * static
   * dynamic

</details>

---

## Phase 1 Completion Criteria

Phase 1 được coi là hoàn thành khi:

* Audit log ghi nhận được API activity
* Có thể xem log kubelet và container runtime
* Tool CRIU hoạt động
* Có thể inspect container bằng crictl
* Có thể đọc và phân tích log

---

## Next Phase

Phase 2: Runtime Detection và trigger checkpoint




K8s Lab
I. Tổng thể hệ thống Lab
Môi trường lab này được triển khai dưới dạng một cụm Kubernetes nội bộ hoạt động trong mạng private, cung cấp dịch vụ Jupyter trên nền tảng cloud. Trong cụm hiện diện ít nhất hai workload chính:
Pod 1: jupyter-lab
Đây là điểm truy cập ban đầu của người dùng. Thông qua giao diện Jupyter, người dùng có thể đăng nhập và thực thi mã Python bên trong container.
Pod 2: internal-processor
Đây là dịch vụ nội bộ mà Pod 1 có thể nhận diện thông qua cơ chế service discovery mặc định của Kubernetes. Dịch vụ này để lộ một endpoint fetch dùng cho xử lý nội bộ.
Ngoài hai workload chính, hệ thống còn bao gồm:
Kubernetes API Server, chịu trách nhiệm điều phối và quản trị toàn bộ cụm.
Node hạ tầng vận hành các pod, sử dụng địa chỉ IP private thuộc dải 192.168.x.x.
II. Phase 1 – Mục tiêu của Red Team
Trong giai đoạn đầu, mục tiêu của Red Team là bắt đầu từ Pod 1 (jupyter-lab) như một điểm foothold ban đầu, từ đó thực hiện lateral movement sang Pod 2 (internal-processor), sau đó tiếp tục mở rộng phạm vi kiểm soát nhằm thoát khỏi ranh giới container và tiến tới mức ảnh hưởng trên toàn bộ Kubernetes host.
Workflow attack 
​
Initial Access & Reconnaissance via Jupyter Lab
1. Tổng quan
Truy cập vào Jupyter, lợi dụng khả năng thực thi mã Python trong môi trường notebook để gọi lệnh hệ thống thông qua os.system, từ đó phục vụ cho quá trình khai thác ban đầu.

2. Kỹ thuật khai thác
Attacker sử dụng module subprocess của Python để thực thi lệnh hệ thống, nhằm xác định context bảo mật và thông tin môi trường của container:
​
2.2. Kết quả thu thập được

Thông tin
Giá trị
Ý nghĩa bảo mật
User Context
uid=1000(jovyan) gid=100(users)
Container chạy với non-root user, hạn chế một số vector escalation
Hostname
jupyter-lab-6694c6bcfd-j44tb
Xác nhận đang trong pod Jupyter, namespace corp-internal
INTERNAL_PROCESSOR_SVC_SERVICE_HOST
10.101.66.187
🎯 ClusterIP của service mục tiêu - entry point cho lateral movement
INTERNAL_PROCESSOR_SVC_PORT_8080_TCP_ADDR
10.101.66.187:8080
Port và địa chỉ để kết nối tới Flask app internal
JUPYTER_LAB_SVC_PORT_8888_TCP_ADDR
10.103.208.208:8888
Thông tin service của chính pod hiện tại
🔹 3. Phân tích kỹ thuật
3.1. Service Discovery qua Environment Variables
Kubernetes tự động inject environment variables vào container cho các Service trong cùng namespace theo định dạng:

​
→ Từ đó cũng như ta xác định được service đang chạy trên môi trường k8s
Leo quyền bằng phương pháp chroot
Sau khi xác lập foothold trong container jupyter-lab với user context jovyan (uid=1000), attacker tiến hành đánh giá khả năng leo quyền (privilege escalation). Mặc dù container được cấu hình chạy với non-root user, việc kết hợp giữa thiếu hardened security context và khả năng truy cập filesystem đã mở ra vector tấn công leo quyền thông qua kỹ thuật filesystem manipulation và namespace escape.
⚠️ Technical Clarification: chroot đơn thuần không trực tiếp leo quyền từ uid=1000 lên root. Tuy nhiên, trong môi trường Kubernetes với cấu hình bảo mật chưa tối ưu, attacker có thể kết hợp nhiều kỹ thuật (writable hostPath mount, SUID binary exploitation, hoặc kernel vulnerability) để đạt được mục tiêu tương tự.
Sau khi chroot, có thểt thoả sức cài tool không bị giới hạn về quyền

Reconaise BRAC
Sau khi xác lập foothold trong container jupyter-lab, attacker tiến hành khám phá cơ chế xác thực nội bộ của Kubernetes. Việc phát hiện ServiceAccount token được tự động mount vào filesystem container đã mở ra khả năng tương tác với Kubernetes API Server — bước then chốt để đánh giá quyền hạn (RBAC) và xác định vector leo quyền tiếp theo.
2.1. ServiceAccount Token Discovery
Kubernetes tự động mount credentials của ServiceAccount vào đường dẫn chuẩn trong container


​
​
Sau khi thu thập được ServiceAccount token từ đường dẫn mount mặc định /var/run/secrets/kubernetes.io/serviceaccount/token, attacker tiến hành phân tích cấu trúc JWT (JSON Web Token) để xác định danh tính, phạm vi ủy quyền, và các thông tin metadata quan trọng phục vụ cho quá trình khai thác tiếp theo.
2.2. Decoded Payload Analysis
Sử dụng công cụ giải mã JWT (jwt.io hoặc jq + base64), attacker trích xuất được các claims quan trọng:
Authentication & Authorization Claims
Claim
Giá trị
Ý nghĩa bảo mật
sub (Subject)
system:serviceaccount:corp-internal:default
🎯 Định danh duy nhất của token — dùng để map tới RBAC policies
iss (Issuer)
https://kubernetes.default.svc.cluster.local
Xác nhận token được ký bởi Kubernetes API Server của cluster hiện tại
aud (Audience)
["https://kubernetes.default.svc.cluster.local"]
Giới hạn token chỉ hợp lệ khi gọi tới API Server nội bộ — ngăn replay attack ra bên ngoài
jti (JWT ID)
1cd9c775-...
ID duy nhất của token — hỗ trợ audit logging và token revocation
3.2. Temporal Claims (Thời hạn sử dụng)
Claim
Giá trị (Unix timestamp)
Giá trị (Human-readable)
Ý nghĩa
iat (Issued At)
1774842811
~ Mar 30, 2026 02:13:31 UTC
Thời điểm token được cấp
nbf (Not Before)
1774842811
~ Mar 30, 2026 02:13:31 UTC
Token chỉ hợp lệ từ thời điểm này
exp (Expiration)
1806378811
~ Mar 30, 2027 02:13:31 UTC
⚠️ Token có hạn ~1 năm — window of opportunity dài cho attacker
warnafter
1774846418
~ Mar 30, 2026 03:13:38 UTC
3.3. Kubernetes-Specific Metadata (kubernetes.io claim)
Field
Giá trị
Ứng dụng khai thác
namespace
corp-internal
Xác định scope RBAC — các permission chỉ áp dụng trong namespace này (trừ cluster-scoped resources)
node.name
longth-cluster
🎯 Xác định tên node vật lý — hữu ích cho node-targeted attacks hoặc lateral movement
node.uid
6f8be7a6-...
Định danh duy nhất của node — dùng để correlate logs hoặc target specific node APIs
pod.name
jupyter-lab-6694c6bcfd-j44tb
Xác định chính xác pod hiện tại — hữu ích cho pod-specific enumeration hoặc cleanup evasion
pod.uid
0ff07e11-...
Định danh duy nhất của pod — ổn định hơn pod name (có thể thay đổi khi restart)
serviceaccount.name
default
🎯 Tên SA để query RBAC: kubectl get rolebinding -n corp-internal -o yaml | grep "default"
2.2. Tooling Challenge & Workaround
Vấn đề: Container không có sẵn kubectl — công cụ CLI tiêu chuẩn để tương tác với Kubernetes API.
​
Giải pháp: Download binary kubectl tương thích trực tiếp từ nguồn chính thức:

2.3. RBAC Enumeration Results
Thực hiện kiểm tra quyền hạn của ServiceAccount default trong namespace corp-internal:


​
→ Chỉ có quyền get trên /api/* và /apis/*  , tuy nhiên với hai quyền này chưa thể lợi dụng được lateralmovement
Ko có quyền nhưng khai phá env
​
​
→ env có khai báo thông tới container 2
​

Phát hiện pod2 có lỗ hổng SSRF
​

Steal SA Token của Pod2
​
Lưu token vào biến cho tiện dùng

Check có quyền cluster-admin không
Check quyền cụ thể: pods/exec
​
TẠO PRIVILEGED POD → ESCAPE RA HOST 

Pod Security Admission (PSA) của namespace corp-internal đang enforce chính sách baseline.
Chính sách này chặn các pod có:
❌ securityContext.privileged: true
❌ hostPID: true, hostNetwork: true, hostIPC: true
❌ hostPath volume mount
⚠️ Quan trọng: Ngay cả khi bạn có cluster-admin, Pod Security Admission vẫn có thể block pod creation nếu vi phạm policy!

Tạo Pod "Baseline-Compliant" 
Tạo pod đơn giản, không privileged, nhưng vẫn có thể exec để lateral movement:
​

→ Nhưng để chiếm host từ pod "baseline-compliant" (không privileged, không hostPath), bạn cần dùng chiến thuật "lách luật":

DÙNG NODE PROXY API QUA API SERVER 
Giải thích (ngắn):
Dùng token cluster-admin để gọi kubelet proxy endpoint qua API Server, trả về full pod specs của tất cả pods trên node — bao gồm hardcoded secrets trong env vars mà bình thường không thấy qua kubectl get pods.

​
​
Pod longth2-test trong namespace default
​
Pod này còn có:
​
Extract credentials từ output 
​
Exec vào pod longth2-test để access host filesystem
Pod này đã có sẵn hostPath: / mount tại /host + privileged: true.
Forensic
dump container ra .tar để DFIR” chính là dùng container checkpoint.
Cụ thể:
bạn không tar thủ công rootfs đang live
mà gọi checkpoint của Kubernetes/CRI
kết quả là kubelet/runtime tạo ra một file checkpoint .tar
file này chứa trạng thái cần cho forensic như memory/process/filesystem diff tùy runtime và nội dung checkpoint
sau đó bạn copy file .tar đó sang /Auto_DFIR
cần cài criu / checkpoint 

