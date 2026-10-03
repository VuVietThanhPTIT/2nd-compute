
## Lịch sử tiến hóa

**Giai đoạn 1: Emulator thuần (2003-2007)**

- Fabrice Bellard viết QEMU năm 2003, dùng _dyngen_ để dịch binary động (full emulation, chưa có KVM).
- Mô hình **single thread + `select()/poll()` loop**: CPU emulation và device emulation chạy chung một thread.

**Giai đoạn 2: KVM và fork qemu-kvm (2007-2012)**

- KVM vào Linux 2.6.20 (2007). Avi Kivity và team Qumranet fork QEMU thành **qemu-kvm** để thêm `KVM_RUN` loop.
- 2008: QEMU 0.10 thay dyngen bằng **TCG** (Tiny Code Generator).
- 2007-2008: **virtio** ra đời (Rusty Russell), mở đường cho paravirtual I/O.
- Kiến trúc lúc này: mỗi vCPU một thread, nhưng mọi thứ dùng chung **Big QEMU Lock (BQL, trước gọi là global mutex)**.
- **qemu-kvm được merge ngược vào upstream ở QEMU 1.3 (2012)**.

**Giai đoạn 3: Refactor nền tảng (2011-2013)**

- **Memory API** (Avi Kivity): mô tả guest physical address space dạng cây `MemoryRegion`. Đây là gốc của việc dispatch MMIO/PIO hiện nay.
- **QOM** (QEMU Object Model): chuẩn hóa device, bus, machine.
- **QMP** thay cho human monitor làm giao diện cho libvirt.
- **ioeventfd/irqfd**: guest ghi vào doorbell thì KVM kernel signal một eventfd, không cần exit lên QEMU main loop.

**Giai đoạn 4: Tách I/O khỏi BQL (2013-2018)**

- **virtio-blk dataplane** (QEMU 1.4+), sau thành **IOThread** (2.1) với `AioContext` riêng. Mỗi IOThread chạy event loop riêng để xử lý virtqueue, không cần BQL.
- **vhost-net** (kernel) và **vhost-user** (QEMU 2.1, userspace daemon như DPDK/SPDK): data path rời hẳn khỏi QEMU process, QEMU chỉ làm control plane (qua unix socket) và share guest memory qua `memfd`/hugepage.
- **Block layer coroutines**: code I/O viết kiểu đồng bộ nhưng chạy async.
- **MTTCG** (Multi-threaded TCG, ~2.9 đến 3.x): TCG cũng chạy mỗi vCPU một thread. Trước đó TCG chỉ dùng một thread cho mọi vCPU.

**Giai đoạn 5: Hiệu năng và đa dạng hóa (2019-2023)**

- **Migration multifd** (4.0): nhiều kênh song song cho live migration, hỗ trợ RDMA/zero-copy.
- **microvm machine type** (4.2): cho workload nhẹ, boot nhanh (cạnh tranh với Firecracker).
- **io_uring** backend cho block (5.0), **virtio-fs** (5.0), **vDPA** (vhost-vdpa, ~5.x).
- **KVM dirty ring** (Linux 5.11, QEMU ~6.x): thay dirty bitmap để migration hiệu quả hơn.
- **vfio-user / multi-process QEMU**: tách device emulation thành process riêng qua socket.
- BQL được đổi tên API (`qemu_mutex_lock_iothread` thành `bql_lock()` ở khoảng 8.x) và các vùng MMIO dần được chuyển sang xử lý không cần BQL.
- **Confidential computing**: AMD SEV/SEV-ES/SNP, Intel TDX, dùng `guest_memfd` (QEMU 8.x-9.x) để bộ nhớ guest không map vào userspace host.

**Giai đoạn 6: Hiện tại (2024-nay, 9.x-10.x)**

- **`iothread-vq-mapping`** (9.0): virtio-blk multi-queue, mỗi virtqueue gắn một IOThread riêng, scale tốt trên NVMe nhanh.
- **Rust trong QEMU**: hạ tầng build Rust vào tree (9.x), đã có vài device viết bằng Rust (pl011, HPET...). Đây là hướng dài hạn nhằm giảm bug memory safety.
- Tiếp tục gỡ BQL khỏi các subsystem, hoàn thiện multifd, cải thiện hỗ trợ ARM và RISC-V cho cloud.

## Tóm lại: so sánh mô hình

|                  | Cũ (~2010)          | Hiện tại                                           |
| ---------------- | ------------------- | -------------------------------------------------- |
| vCPU             | pthread + `KVM_RUN` | **giống hệt**                                      |
| Lock             | BQL cho mọi thứ     | BQL chỉ cho phần còn lại, I/O nằm ở IOThread       |
| Virtio data path | QEMU main loop      | IOThread, vhost kernel, hoặc **vhost-user (SPDK)** |
| Doorbell (kick)  | exit lên QEMU       | ioeventfd, không lên QEMU                          |
| Interrupt        | QEMU inject         | irqfd / in-kernel irqchip                          |
| Migration        | 1 luồng             | multifd, dirty ring                                |

## Liên hệ với PoC SPDK vhost-user-blk của bạn

Đây chính là hướng "đưa data path ra khỏi QEMU": guest kick virtqueue, ioeventfd báo thẳng cho SPDK vhost target (poll mode), SPDK xử lý I/O qua NVMe-oF/RDMA rồi irqfd báo hoàn thành. QEMU vCPU thread không dính vào đường I/O. Vòng `KVM_RUN` trong đoạn bạn paste chỉ còn gánh các exit "chậm" hoặc control-path.

Nếu muốn, mình đi sâu vào một phần: luồng exit MMIO với ioeventfd (code `kvm_cpu_exec`), BQL cụ thể giữ ở đâu, hay so sánh IOThread với vhost-user cho workload NVMe.