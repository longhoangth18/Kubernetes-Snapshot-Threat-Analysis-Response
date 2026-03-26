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
