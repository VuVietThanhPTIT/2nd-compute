# TÀI LIỆU KỸ THUẬT: TOÀN TẬP KIẾN TRÚC VÀ BẢN CHẤT I/O CỦA VIRTIO

---

# I. BỐI CẢNH LỊCH SỬ & LÝ DO VIRTIO RA ĐỜI

Trước năm 2008, hệ sinh thái ảo hóa trên Linux rơi vào tình trạng **phân mảnh mã nguồn trầm trọng**:

* Mỗi giải pháp ảo hóa tự duy trì một tập driver I/O độc quyền: Xen dùng `xen-blkfront`/`xen-netfront`, KVM phụ thuộc vào QEMU để mô phỏng các con chip vật lý cổ điển (như card mạng Intel e1000, Realtek RTL8139, hay bộ điều khiển đĩa IDE), trong khi các dự án nghiên cứu như `lguest` hay `User-Mode Linux (UML)` lại tự viết cơ chế chia sẻ bộ nhớ riêng.
* **Hậu quả:** Mỗi khi cộng đồng muốn tối ưu hóa một thuật toán truyền gói tin hay bộ đệm I/O, họ phải viết lại và bảo trì mã nguồn trên hàng chục codebase khác nhau. Việc bảo trì nhân Linux trở thành một gánh nặng khổng lồ.

**Bước ngoặt mang tên Rusty Russell (Linux Kernel 2.6.24 - 2008):**

* Kỹ sư hạt nhân Rusty Russell đề xuất **VirtIO** với triết lý: *Xây dựng một lớp trừu tượng hóa chuẩn (Common Abstraction Layer) cho giao tiếp I/O Paravirtualization trên toàn bộ hệ điều hành Linux.*
* VirtIO tách rời hoàn toàn:
1. **Tầng giao tiếp thiết bị (Device Abstraction):** OS chỉ thấy các hàm trừu tượng như "gửi khối đĩa", "gửi gói tin".
2. **Tầng vận chuyển (Transport):** Cách thức các byte dữ liệu được chuyển qua lại giữa Guest và Hypervisor.


* Năm 2013, tổ chức chuẩn hóa quốc tế **OASIS** chính thức tiếp quản VirtIO, biến nó thành tiêu chuẩn mở của toàn ngành công nghiệp ảo hóa (VirtIO Standard v1.0, v1.1, v1.2), hỗ trợ từ Linux, Windows (thông qua bộ driver VirtIO của Red Hat), đến FreeBSD.

---

# II. FULL VIRTUALIZATION VS. PARAVIRTUALIZATION (BẢN CHẤT I/O)

Sự khác biệt cốt lõi giữa hai mô hình không nằm ở việc "chạy máy ảo như thế nào", mà nằm ở **cơ chế bẫy ngắt phần cứng (Hardware Trapping) và sự tương tác giữa Driver với CPU**.

```text
+──────────────────────────────────────────────────────────────────────────────────────────────────+
|                    SO SÁNH BẢN CHẤT LUỒNG I/O GIỮA FULL VÀ PARAVIRTUALIZATION                    |
+──────────────────────────────────────────────────────────────────────────────────────────────────+

  A. FULL VIRTUALIZATION (Giả lập phần cứng cổ điển: e1000, IDE)
     Guest OS (Unmodified)             CPU Hardware (VT-x)              Hypervisor (QEMU/KVM)
  ┌─────────────────────────┐         ┌──────────────────┐            ┌────────────────────────┐
  │ Driver e1000 ghi MMIO   │ ──────► │ Trap: EPT Fault  │ ─────────► │ Bắt ngắt, phân tích    │
  │ (Thanh ghi TX Control)  │         │ (VM-Exit)        │            │ thanh ghi, mô phỏng    │
  │                         │ ◄────── │ VM-Entry         │ ◄───────── │ mạch logic chip e1000  │
  │ Driver ghi MMIO         │ ──────► │ Trap: EPT Fault  │ ─────────► │ Bắt ngắt, đọc buffer   │
  │ (Thanh ghi TX Buffer)   │         │ (VM-Exit)        │            │ từ RAM Guest           │
  │                         │ ◄────── │ VM-Entry         │ ◄───────── │                        │
  │ Driver ghi MMIO         │ ──────► │ Trap: EPT Fault  │ ─────────► │ Kích hoạt gửi packet   │
  │ (Thanh ghi TX Length)   │         │ (VM-Exit)        │            │ ra card mạng thật      │
  └─────────────────────────┘         └──────────────────┘            └────────────────────────┘
  * TỐN KÉM: 1 gói tin I/O = 5 đến 20 lần VM-Exit! Chu kỳ CPU bị thiêu rụi vì đổi ngữ cảnh.

────────────────────────────────────────────────────────────────────────────────────────────────────

  B. PARAVIRTUALIZATION (VirtIO: Hợp đồng Bộ nhớ Chia sẻ)
     Guest OS (VirtIO Driver)          CPU Hardware (VT-x)              Host Backend (KVM/vhost)
  ┌─────────────────────────┐         ┌──────────────────┐            ┌────────────────────────┐
  │ Nạp 64 gói tin vào RAM  │         │ (Không trap)     │            │                        │
  │ (Virtqueue Shared Ring) │         │                  │            │                        │
  │                         │         │                  │            │                        │
  │ Ghi 1 lệnh Kick Doorbell│ ──────► │ Trap: 1 VM-Exit  │ ─────────► │ Backend đọc 1 lượt     │
  │ (Báo có việc mới)       │         │ (Hoặc 0 nếu poll)│            │ toàn bộ 64 gói từ RAM  │
  └─────────────────────────┘         └──────────────────┘            └────────────────────────┘
  * HIỆU QUẢ: 64 gói tin I/O = 1 lần VM-Exit duy nhất (Cơ chế Batching / Gom cụm).

```

### 1. Phân tích sâu: Full Virtualization (Mô phỏng đầy đủ)

* **Cơ chế hoạt động:** Hypervisor tạo ra một bản sao kỹ thuật số 100% của một con chip vật lý có thật (ví dụ: Intel 82540EM Gigabit Ethernet). Guest OS chạy một hệ điều hành nguyên bản và nạp đúng driver do Intel viết cho con chip đó.
* **Cái giá phải trả của bẫy ngắt (The Hardware Trap Penalty):**
* Driver của Intel nghĩ rằng nó đang nói chuyện với phần cứng vật lý qua các cổng I/O (`in`/`out` instructions) hoặc các dải địa chỉ bộ nhớ Memory-Mapped I/O (MMIO).
* Trong thực tế, các địa chỉ MMIO đó là vùng nhớ không tồn tại thật trên bảng phân trang mở rộng **EPT (Extended Page Tables)** của CPU.
* Khi Guest cố ghi một giá trị vào thanh ghi ảo: Phần cứng CPU (Intel VT-x) phát hiện vi phạm phân trang và lập tức dừng toàn bộ máy ảo, kích hoạt một cú **VM-Exit**.
* CPU chuyển quyền điều khiển từ Guest về KVM (Host Kernel Ring 0), sau đó KVM thường phải chuyển tiếp (Context Switch) quyền lên QEMU (Host Userspace Ring 3) để chạy đoạn mã C mô phỏng lại mạch logic của chip Intel.
* **Hậu quả:** Để xử lý chỉ **một gói tin mạng duy nhất**, driver có thể phải cấu hình 5–10 thanh ghi liên tiếp $\to$ gây ra 5–10 lần VM-Exit. Mỗi cú VM-Exit tốn từ vài trăm đến hàng nghìn chu kỳ xung nhịp CPU và quét sạch bộ nhớ đệm L1/L2/L3 Cache (Cache Pollution).



### 2. Phân tích sâu: Paravirtualization (Bán ảo hóa - VirtIO)

* **Cơ chế hoạt động:** Loại bỏ hoàn toàn ảo tưởng về việc "phải giống phần cứng thật". Hệ điều hành Guest được cài một driver chuyên dụng (**VirtIO Frontend Driver**) và nhận thức rõ: *"Ta đang chạy trong máy ảo, đối tác của ta là Hypervisor"*.
* **Hợp đồng bộ nhớ chung (Memory-Contract):** Thay vì giao tiếp qua thanh ghi vật lý, Guest và Host ký một bản hợp đồng thông qua một cấu trúc dữ liệu nằm trong RAM dùng chung gọi là **Virtqueue (Ring Buffer)**:
1. Guest ghi thẳng thông tin hàng chục request vào RAM.
2. Guest chỉ gõ chuông thông báo (Doorbell Kick) **một lần duy nhất** cho cả cụm request.
3. Host đọc trực tiếp từ RAM, không cần phân tích trạng thái thanh ghi vi mạch giả lập.



### Bảng So sánh Chi tiết: Full Virtualization vs. Paravirtualization

| Tiêu chí kỹ thuật | Full Virtualization (e1000, IDE) | Paravirtualization (VirtIO) |
| --- | --- | --- |
| **Sự nhận thức của Guest OS** | Hoàn toàn "mù tịt", nghĩ mình là máy vật lý | Nhận thức rõ ràng mình đang trong môi trường ảo hóa |
| **Yêu cầu chỉnh sửa / Driver** | Không cần sửa Guest OS, dùng driver gốc | Bắt buộc cài VirtIO driver (tích hợp sẵn từ Linux 2.6.25+) |
| **Giao diện phần cứng** | Thanh ghi phần cứng giả lập (MMIO / Port I/O) | Bảng chỉ mục và Ring Buffer trên RAM chia sẻ |
| **Tần suất VM-Exit (Trap)** | Cực cao (1 thao tác I/O = nhiều lần VM-Exit) | Cực thấp (Gom cụm hàng trăm I/O = 1 VM-Exit; Polling = 0) |
| **Chi phí sao chép Payload** | Thường qua nhiều tầng đệm mô phỏng | Zero-Copy: Host đọc trực tiếp khung trang RAM của Guest |
| **Thông lượng & Độ trễ** | Thấp, độ trễ không ổn định, trồi sụt | Cao, tiệm cận hiệu năng phần cứng vật lý |
| **Tính tương thích** | Hoàn hảo với mọi OS cổ xưa (DOS, Windows 98) | Cần có driver cho từng hệ điều hành tương ứng |

---

# III. KIẾN TRÚC TỔNG QUÁT CỦA VIRTIO 
Thay vì giả lập phần cứng phức tạp, hệ thống sử dụng một tập hợp thiết bị ảo chuẩn hóa và tối giản: Guest cài driver VirtIO (Frontend), Host cài engine xử lý VirtIO (Backend). Hai bên bắt tay trực tiếp qua bộ nhớ chia sẻ chung (Shared Memory), triệt tiêu các thao tác mô phỏng dư thừa

```text
========================================= GUEST OS (VIRTUAL MACHINE) =========================================
 [ Userspace App ]          [ Userspace App ]          [ Userspace App ]          [ Userspace App ]
   (curl / nginx)             (fio / dd / db)             (qemu-ga)                  (app-socket)
         │                          │                          │                          │
─────────┼──────────────────────────┼──────────────────────────┼──────────────────────────┼───────────────────
 [ Kernel Space ]                   │                          │                          │
   Socket Layer               VFS / Block Layer                │                          │
         │                          │                          │                          │
         ▼                          ▼                          ▼                          ▼
 ┌───────────────┐          ┌───────────────┐          ┌───────────────┐          ┌───────────────┐
 │  virtio-net   │          │  virtio-blk   │          │ virtio-serial │          │  virtio-vsock │
 │ (Network Dev) │          │ (Block Disk)  │          │(Console/Agent)│          │ (Fast Socket) │
 └───────┬───────┘          └───────┬───────┘          └───────┬───────┘          └───────┬───────┘
         │ drivers/net/             │ drivers/block/           │ drivers/char/            │ net/vmw_vsock/
         │ virtio_net.c             │ virtio_blk.c             │ virtio_console.c         │ virtio_transport.c
         └──────────────────────────┴─────────────┬────────────┴──────────────────────────┘
                                                  │
                                                  ▼
                        ┌───────────────────────────────────────────────────┐
                        │              CORE VIRTIO FRAMEWORK                │
                        │             drivers/virtio/virtio.c               │
                        │ - Quản lý vòng đời thiết bị (device_status)       │
                        │ - Xử lý đàm phán tính năng (Feature Negotiation)  │
                        └─────────────────────────┬─────────────────────────┘
                                                  │
                                                  ▼
                        ┌───────────────────────────────────────────────────┐
                        │              VIRTIO TRANSPORT LAYER               │
                        │  drivers/virtio/virtio_pci_modern.c (hoặc mmio)   │
                        │ - Ánh xạ thanh ghi cấu hình (PCI Common Config)   │
                        │ - Quản lý vector ngắt MSI-X (config & queue)      │
                        └─────────────────────────┬─────────────────────────┘
                                                  │
                                                  ▼
                        ┌───────────────────────────────────────────────────┐
                        │             VIRTQUEUE DATA FABRIC                 │
                        │            drivers/virtio/virtio_ring.c           │
                        │ - Quản lý 3 bảng: Descriptor, Available, Used Ring│
                        │ - Thực thi rào cản bộ nhớ (Memory Barriers: wmb)  │
                        └─────────────────────────┬─────────────────────────┘
==================================================│===========================================================
                                RANH GIỚI BỘ NHỚ VÀ PHẦN CỨNG ẢO HÓA
            Shared Memory (Guest RAM ◄────────────────────────► Host Address Space)
            Doorbell Kick (Guest MMIO Write ──► EPT Trap ──► KVM ioeventfd)
            Interrupt (Host ──► KVM irqfd ──► Virtual MSI-X ──► Guest LAPIC)
==================================================│===========================================================
                                       HOST SYSTEM / HYPERVISOR
                                                  │
                        ┌─────────────────────────┴─────────────────────────┐
                        ▼                                                   ▼
       ┌─────────────────────────────────┐                 ┌─────────────────────────────────┐
       │     HOST USERSPACE (Ring 3)     │                 │      HOST KERNEL (Ring 0)       │
       │                                 │                 │                                 │
       │ [QEMU Process]                  │                 │ [KVM Subsystem]                 │
       │  - PCI Configuration Emulation  │                 │  - ioeventfd / irqfd routing    │
       │  - Feature Negotiation Backend  │                 │                                 │
       │                                 │                 │ [vhost-kernel: vhost_net/scsi]  │
       │ [QEMU IOThread (Mô hình 1)]     │                 │  - Data Plane kthreads          │
       │  - virtqueue parsing            │                 │  - Giao tiếp trực tiếp với      │
       │  - Gọi syscall: io_uring/aio    │                 │    Linux Block/Net Layer        │
       │                                 │                 └────────────────┬────────────────┘
       │ [vhost-user: SPDK/DPDK (Mô hình3]│                                  │
       │  - Bypass Kernel hoàn toàn      │                                  │
       │  - Polling Mode Driver (PMD)    │                                  │
       └────────────────┬────────────────┘                                  │
                        │                                                   │
                        └─────────────────────────┬─────────────────────────┘
                                                  │
                                                  ▼
                        ┌───────────────────────────────────────────────────┐
                        │          PHẦN CỨNG THỰC TẾ (PHYSICAL HW)          │
                        │     NVMe SSD / Physical NIC / Hardware Switch     │
                        └───────────────────────────────────────────────────┘

```

---

# IV. VI PHẪU VIRTQUEUE (VRING) — BẢN THIẾT KẾ DỮ LIỆU CỐT LÕI

Virtqueue là cấu trúc dữ liệu tuần hoàn nằm trên vùng nhớ RAM liên tục (Physically Contiguous Memory do Guest Kernel cấp phát bằng hàm `dma_alloc_coherent`).

Trong chuẩn **VirtIO Split Ring (v1.0)**, một Virtqueue gồm 3 mảng bộ nhớ tách rời:

```text
+─────────────────────────────────────────────────────────────────────────────────────────────────+
|                                CẤU TRÚC CHI TIẾT SPLIT VIRTQUEUE                                |
+─────────────────────────────────────────────────────────────────────────────────────────────────+

  1. BẢNG MÔ TẢ (DESCRIPTOR TABLE) — Kích thước: Queue Size x 16 Bytes
  Index    [ addr (64-bit GPA) ]    [ len (32-bit) ]    [ flags (16-bit) ]    [ next (16-bit) ]
  ┌─────┐ ┌──────────────────────┐ ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
  │  0  │ │  0x1a2b3000 (Header) │ │        16        │ │ NEXT (0x1)       │ │        1         │
  ├─────┤ ├──────────────────────┤ ├──────────────────┤ ├──────────────────┤ ├──────────────────┤
  │  1  │ │  0x1a2b4000 (Payload)│ │       4096       │ │ NEXT (0x1)       │ │        2         │
  ├─────┤ ├──────────────────────┤ ├──────────────────┤ ├──────────────────┤ ├──────────────────┤
  │  2  │ │  0x1a2b5000 (Status) │ │        1         │ │ WRITE (0x2)      │ │        0         │
  └─────┘ └──────────────────────┘ └──────────────────┘ └──────────────────┘ └──────────────────┘
  * flags: NEXT = Còn phần tử kế tiếp; WRITE = Host ghi vào (Read-only đối với Host nếu bit này = 0).

  2. VÒNG SẴN SÀNG (AVAILABLE RING) — Guest ghi, Host đọc (Báo việc mới)
  ┌──────────────────┬──────────────────┬────────────────────────────────────────────┬────────────┐
  │  flags (16-bit)  │   idx (16-bit)   │          ring[Queue Size] (16-bit)         │ used_event │
  │ (NO_INTERRUPT:0) │ (Tăng đơn điệu)  │ [ Index 0 ] [ Index 3 ] [   ...   ] [      ]│  (16-bit)  │
  └──────────────────┴──────────────────┴──────▲─────────────────────────────────────┴────────────┘
                                               │
                                               └─► Chứa Head Index (Phần tử đầu chuỗi Descriptor)

  3. VÒNG ĐÃ DÙNG (USED RING) — Host ghi, Guest đọc (Báo hoàn tất)
  ┌──────────────────┬──────────────────┬────────────────────────────────────────────┬────────────┐
  │  flags (16-bit)  │   idx (16-bit)   │       ring[Queue Size] (struct vring_used_elem)         │
  │   (NO_NOTIFY:0)  │ (Tăng đơn điệu)  │ [ id: 0, len: 4097 ] [ id: 3, len: 64 ]... │avail_event │
  └──────────────────┴──────────────────┴────────────────────────────────────────────┴────────────┘

```

* **Quy ước quyền truy cập:**
* **Descriptor Table:** Chứa danh sách các khối bộ nhớ phân tán. Chuỗi descriptor (Chaining) cho phép gom một tác vụ I/O gồm Header, Payload và Status byte lại với nhau mà không cần sao chép dữ liệu.
* **Available Ring:** Guest là bên sản xuất (Producer), Host là bên tiêu thụ (Consumer). Biến `idx` tăng đơn điệu không giới hạn; vị trí thực tế trong mảng được tính bằng phép toán modulo: $\text{slot} = \text{idx} \pmod{\text{Queue Size}}$.
* **Used Ring:** Host là bên sản xuất (Producer), Guest là bên tiêu thụ (Consumer). Host trả về đúng `id` của Head Descriptor và ghi số byte thực tế đã tương tác vào trường `len`.



---

# V. TOÀN BỘ VÒNG ĐỜI VÀ LUỒNG THỰC THI (LIFECYCLE & RUNTIME FLOW)

---

## Giai đoạn 1: Khởi tạo và Bắt tay Thiết bị (Initialization Phase)

Mọi thiết bị VirtIO đều phải trải qua một quy trình trạng thái hữu hạn (**State Machine**) nghiêm ngặt thông qua thanh ghi chuẩn `device_status`:

```text
[ RESET ] (Status = 0)
    │
    ▼
[ ACKNOWLEDGE ] (Status |= 1) ──► Guest OS nhận diện thấy thiết bị PCI (Vendor: 0x1AF4)
    │
    ▼
[ DRIVER ] (Status |= 2)      ──► Guest Kernel tìm thấy và nạp driver phù hợp (virtio_blk)
    │
    ▼
[ FEATURES_OK ] (Status |= 8) ──► Đàm phán tính năng hoàn tất; cả hai bên chốt danh sách cờ
    │
    ▼
[ DRIVER_OK ] (Status |= 4)   ──► Cấu hình Virtqueue xong; Thiết bị chính thức hoạt động

```

1. **Device Discovery (Nhận diện thiết bị):**
* Quá trình PCI Bus Scan trong Guest đọc không gian cấu hình PCI.
* VirtIO quy ước **Vendor ID luôn là `0x1AF4**`. Dải Device ID từ `0x1040` đến `0x107F` biểu thị thiết bị VirtIO 1.0 Modern (`0x1041` = net, `0x1042` = blk).


2. **Feature Negotiation (Đàm phán tính năng):**
* Thiết bị phơi bày mảng bit tính năng 64-bit (`device_feature`).
* Driver đọc danh sách này, đối chiếu với khả năng của mình, ghi danh sách các bit chấp thuận vào `driver_feature`.
* *Ví dụ các tính năng đàm phán:* `VIRTIO_F_VERSION_1` (Bắt buộc dùng chuẩn 1.0+), `VIRTIO_RING_F_EVENT_IDX` (Bật cơ chế chặn bão ngắt nâng cao), `VIRTIO_BLK_F_RO` (Đĩa chỉ đọc).


3. **Virtqueue Configuration (Cấu hình hàng đợi):**
* Driver chọn từng hàng đợi bằng cách ghi số hiệu vào thanh ghi `queue_select`.
* Đọc `queue_size` để biết kích thước (ví dụ 128, 256 phần tử).
* Guest cấp phát dải RAM liên tục cho 3 bảng của Virtqueue.
* Ghi **Địa chỉ vật lý máy ảo (Guest Physical Address - GPA)** của 3 bảng vào các thanh ghi:
* `queue_desc` $\leftarrow \text{GPA của Descriptor Table}$
* `queue_driver` $\leftarrow \text{GPA của Available Ring}$
* `queue_device` $\leftarrow \text{GPA của Used Ring}$


* Ghi `queue_enable = 1`.


4. **Bật trạng thái DRIVER_OK:**
* Guest ghi `device_status |= 4`. Kể từ thời điểm này, Host Backend bắt đầu kích hoạt luồng xử lý và giám sát hàng đợi.



---

## Giai đoạn 2: Luồng Thực thi Runtime của một Thao tác I/O (Write Path)

Dưới đây là chi tiết chuỗi sự kiện khi ứng dụng trong Guest ghi 4096 bytes dữ liệu xuống ổ đĩa `virtio-blk`:

```text
  GUEST OS (User & Kernel)                  RANH GIỚI PHẦN CỨNG                     HOST (KVM / QEMU Backend)
┌─────────────────────────────┐           ┌──────────────────────┐                ┌───────────────────────────┐
│ 1. Ứng dụng gọi write()     │           │                      │                │                           │
│    VFS/Block Layer điều phối│           │                      │                │                           │
│    vào virtio_blk.c         │           │                      │                │                           │
│                             │           │                      │                │                           │
│ 2. virtqueue_add()          │           │                      │                │                           │
│    - Ghép 3 descriptor      │           │                      │                │                           │
│    - Ghi Available Ring     │           │                      │                │                           │
│    - Rào cản smp_wmb()      │           │                      │                │                           │
│                             │           │                      │                │                           │
│ 3. virtqueue_kick()         │           │                      │                │                           │
│    - Ghi MMIO Doorbell      │ ────────► │ 4. CPU VM-Exit       │ ─────────────► │ 5. KVM kích hoạt ioeventfd│
│                             │           │    (EPT Misconfig)   │                │    Đánh thức Backend      │
│                             │ ◄──────── │    CPU VM-Entry      │                │    (IOThread/vhost)       │
│    (vCPU chạy việc khác)    │           │    (vCPU tiếp tục)   │                │                           │
│                             │           │                      │                │ 6. Backend đọc GPA        │
│                             │           │                      │                │    Dịch sang HVA          │
│                             │           │                      │                │    DMA ghi xuống đĩa thật │
│                             │           │                      │                │                           │
│                             │           │                      │                │ 7. Ghi kết quả Used Ring  │
│                             │           │                      │                │    Bắn tín hiệu irqfd     │
│                             │           │                      │                │                           │
│ 9. ISR: vring_interrupt()   │ ◄──────── │ 8. KVM tiêm ngắt     │ ◄──────────────┘                           │
│    - Đọc Used Ring          │           │    Virtual MSI-X     │                                            │
│    - Thu hồi Descriptor     │           │    vào Guest LAPIC   │                                            │
│    - bio_endio() hoàn tất   │           │                      │                                            │
└─────────────────────────────┘           └──────────────────────┘                                            

```

### Bước 1: Ghép chuỗi Descriptor & Đóng gói Request (`virtqueue_add`)

Driver `virtio-blk` chia khối ghi 4 KiB thành chuỗi 3 phần tử (Descriptor Chain):

1. **Desc 0 (Header):** Chứa cấu trúc `struct virtio_blk_outhdr` (gồm kiểu lệnh `VIRTIO_BLK_T_OUT`, sector bắt đầu). Cờ: `VRING_DESC_F_NEXT`, trỏ tới Desc 1.
2. **Desc 1 (Payload):** Chứa địa chỉ GPA của buffer 4096 bytes dữ liệu. Cờ: `VRING_DESC_F_NEXT`, trỏ tới Desc 2.
3. **Desc 2 (Status):** Chứa địa chỉ 1 byte trạng thái để Host trả về kết quả. Cờ: `VRING_DESC_F_WRITE` (báo cho Host biết đây là vùng Host sẽ ghi vào).

### Bước 2: Cập nhật Available Ring & Rào cản Bộ nhớ

* Driver lấy chỉ số Head Index (là `0` - trỏ tới Desc 0) nạp vào:

$$\text{avail->ring}[\text{avail->idx} \pmod{\text{Queue Size}}] = 0$$


* **Rào cản bộ nhớ bắt buộc (`smp_wmb()`):** Phần cứng CPU hiện đại có thể tự ý đảo thứ tự thực thi lệnh ghi (Out-of-order store). Lệnh rào cản bộ nhớ ép buộc CPU phải xả toàn bộ dữ liệu của Descriptor Table xuống DRAM trước khi con trỏ `avail->idx` được tăng lên.
* Tăng chỉ số: `avail->idx++`.

### Bước 3: Gõ chuông (Doorbell Kick) & Bẫy ngắt VM-Exit

* Guest kiểm tra cờ `flags` trong Used Ring. Nếu cờ `VRING_USED_F_NO_NOTIFY` **không bật**, Guest bắt đầu thông báo:
* Driver thực thi một chỉ lệnh ghi bộ nhớ vào thanh ghi Doorbell của Virtqueue:

$$\text{writew}(\text{queue\_index}, \text{notify\_mmio\_address})$$


* **Hiện tượng phần cứng:** Địa chỉ này là thanh ghi ảo thuộc BAR 0 của card PCI ảo, không có trang RAM thực nào bảo trợ. Bộ quản lý bộ nhớ MMU kích hoạt **EPT Violation $\to$ VM-Exit**.
* vCPU Guest bị dừng tức thì trong vài trăm nano-giây.

### Bước 4 & 5: Phân giải Tín hiệu tại KVM (`ioeventfd`)

* Nhân KVM đón đầu cú VM-Exit ở Ring 0.
* Nhờ cơ chế **`ioeventfd`**, KVM nhận diện địa chỉ ghi tương ứng với một sự kiện đã đăng ký trước.
* KVM ghi một tín hiệu nguyên tử vào cấu trúc `eventfd` liên kết với backend và **lập tức phát lệnh `VM-Entry` cho vCPU Guest chạy tiếp**.
* **Ý nghĩa:** Tránh hoàn toàn việc chuyển ngữ cảnh nặng nề lên tiến trình QEMU (Heavyweight Exit). Guest OS chỉ cảm nhận một khoảng khựng cực ngắn rồi tiếp tục thực thi các luồng khác.

### Bước 6: Xử lý Backend và Dịch địa chỉ GPA $\to$ HVA

* Backend (QEMU IOThread, `vhost-kernel` worker, hoặc tiến trình SPDK) nhận tín hiệu thức dậy.
* Backend đọc `avail->idx`, xác định Desc 0 là đầu chuỗi.
* Trích xuất các trường `addr` (chứa **GPA - Guest Physical Address**).
* **Dịch địa chỉ bộ nhớ:** Do bộ nhớ của Guest thực chất là một vùng cấp phát trên Host RAM, Backend dùng bảng ánh xạ phân đoạn (Memory Slot) để quy đổi:

$$\text{HVA (Host Virtual Address)} = \text{GPA} - \text{MemorySlot.GuestStart} + \text{MemorySlot.HostStart}$$


* Backend có trong tay con trỏ HVA, tiến hành đọc 4096 bytes trực tiếp từ RAM máy chủ và gửi yêu cầu I/O xuống trình điều khiển lưu trữ vật lý (SSD/NVMe).

### Bước 7 & 8: Hoàn tất, Bắn ngắt và Tiêm vAPIC

* Khi thiết bị lưu trữ vật lý báo thành công:
* Backend ghi số `0` (`VIRTIO_BLK_S_OK`) vào byte trạng thái tại Desc 2.
* Backend lấy vị trí trống trong Used Ring, gán `used->ring[used_idx].id = 0` và `len = 1` (số byte trạng thái đã ghi).
* Gọi rào cản bộ nhớ `smp_wmb()`, sau đó tăng `used->idx++`.


* **Kích hoạt ngắt qua `irqfd`:** Backend ghi vào file descriptor `irqfd`.
* KVM tiếp nhận tín hiệu, dùng cơ chế ảo hóa ngắt phần cứng (Intel APICv / Virtual Interrupt Injection) để tiêm một ngắt **Virtual MSI-X** trực tiếp vào bảng ngắt của vCPU Guest.

### Bước 9: Phục hồi và Kết thúc tại Guest

* vCPU Guest bị ngắt, kích hoạt hàm xử lý ngắt `vring_interrupt()`.
* Driver kiểm tra Used Ring, nhận diện chuỗi descriptor `0` đã xử lý xong.
* Thu hồi các descriptor `0, 1, 2` đưa về danh sách trống (Free List).
* Báo lên tầng Block Layer (`bio_endio()`), trả kết quả cho ứng dụng `write()`.

---

# VI. CÁC CƠ CHẾ TỐI ƯU HÓA ĐẶC BIỆT CỦA VIRTIO

### 1. Cơ chế Chống bão ngắt: Event Index (`VIRTIO_RING_F_EVENT_IDX`)

Cờ tắt ngắt nhị phân truyền thống (`VRING_AVAIL_F_NO_INTERRUPT`) dễ rơi vào trạng thái tranh chấp (Race Condition). Để khắc phục, VirtIO bổ sung hai trường số học:

* **`used_event`** (nằm ở cuối Available Ring, do Guest ghi): Guest đặt một con số, ví dụ $100$. Host khi xử lý Used Ring chỉ được bắn ngắt khi và chỉ khi `used->idx` chạm mốc $100$.
* **`avail_event`** (nằm ở cuối Used Ring, do Host ghi): Host đặt một ngưỡng, ví dụ $50$. Guest nạp các request $41, 42, 43...$ đều **không được gõ Doorbell**, cho đến khi chạm đúng mốc $50$.
* **Tác dụng:** Giảm hơn 80% số lần VM-Exit và ngắt ảo khi hệ thống chịu tải I/O cao nhờ cơ chế gom cụm thích ứng (Dynamic Batching).

### 2. Chuỗi Mô tả Gián tiếp (Indirect Descriptors)

* **Vấn đề:** Virtqueue có kích thước giới hạn (thường là 128 hoặc 256 phần tử). Nếu mỗi I/O request cần một chuỗi phân tán (Scatter-Gather List) gồm 16 chunks bộ nhớ, chỉ cần 8 request là hàng đợi bị lấp đầy hoàn toàn.
* **Giải pháp:** Cờ `VRING_DESC_F_INDIRECT` cho phép một phần tử Descriptor trong bảng chính trỏ tới **một bảng Descriptor con độc lập** nằm ở vị trí khác trong RAM:
* Toàn bộ 16 chunks của request chỉ chiếm **đúng 1 phần tử** trên Virtqueue chính.
* Tăng sức chứa của hàng đợi lên gấp hàng chục lần, giải phóng tắc nghẽn I/O.



### 3. Packed Virtqueue (Chuẩn VirtIO 1.1+)

* Thay vì phân tán qua 3 mảng (Descriptor Table, Available Ring, Used Ring) gây trượt bộ nhớ đệm CPU L1 Cache liên tục:
* Packed Ring gộp tất cả thành **1 mảng 1 chiều phẳng duy nhất** gồm các phần tử 16 bytes.
* Guest và Host đồng bộ quyền sở hữu thông qua việc lật cờ nhị phân (Wrap Counter bit). CPU hai bên chỉ cần duyệt bộ nhớ tuần tự một chiều, tận dụng tối đa cơ chế nạp trước của phần cứng (Hardware Prefetcher).

---

# VII. PHÂN LOẠI CÁC THIẾT BỊ VIRTIO CHUẨN OASIS

| Mã ID | Tên Thiết bị | Mã nguồn Linux | Bản chất & Ứng dụng thực tế |
| --- | --- | --- | --- |
| **`1`** | **`virtio-net`** | `drivers/net/virtio_net.c` | Card mạng ảo. Hỗ trợ Đa hàng đợi (Multi-Queue gắn theo từng vCPU), Checksum Offload, TSO/LSO, kết hợp `vhost-net` cho thông lượng hàng chục Gbps. |
| **`2`** | **`virtio-blk`** | `drivers/block/virtio_blk.c` | Ổ đĩa thô (Raw Block Device). Hiển thị dưới dạng `/dev/vda`. Tối ưu tuyệt đối cho hiệu năng đọc/ghi ngẫu nhiên (IOPS), hỗ trợ lệnh TRIM/Discard. |
| **`3`** | **`virtio-console`** | `drivers/char/virtio_console.c` | Kênh truyền tuần tự và cổng ký tự. Đây là kênh dữ liệu ngầm cho daemon `qemu-guest-agent` giao tiếp với Hypervisor để đóng băng đĩa, đổi mật khẩu. |
| **`4`** | **`virtio-rng`** | `drivers/char/hw_random/virtio-rng.c` | Thiết bị sinh số ngẫu nhiên phần cứng ảo. Kéo entropy thực tế từ `/dev/urandom` của Host bơm vào máy ảo, tránh treo Guest do cạn kiệt entropy khi boot. |
| **`5`** | **`virtio-balloon`** | `drivers/virtio/virtio_balloon.c` | Thiết bị co giãn RAM. Khi Host thiếu RAM, nó ra lệnh cho balloon trong Guest "bơm phồng" (chiếm giữ RAM của Guest) để Host thu hồi lại khung trang RAM vật lý. |
| **`8`** | **`virtio-scsi`** | `drivers/scsi/virtio_scsi.c` | Bộ điều khiển SCSI ảo cao cấp. Khác với `virtio-blk`, nó có thể quản lý hàng nghìn LUN sau 1 controller, hỗ trợ passthrough lệnh SCSI thô (CDB) chuyên dùng cho SAN. |
| **`16`** | **`virtio-gpu`** | `drivers/gpu/drm/virtio/` | Card màn hình ảo DRM/KMS. Chuyển tiếp các lệnh gọi dựng hình OpenGL/Vulkan từ Guest xuống GPU vật lý của Host thông qua thư viện `virglrenderer`. |
| **`19`** | **`virtio-vsock`** | `net/vmw_vsock/virtio_transport.c` | Giao thức mạng Socket không cần IP/Card mạng (`AF_VSOCK`). Giao tiếp cực nhanh giữa ứng dụng Host và Guest qua ID ngữ cảnh (Context ID - CID). |
| **`24`** | **`virtio-mem`** | `drivers/virtio/virtio_mem.c` | Cơ chế cắm/rút nóng RAM linh hoạt theo từng khối nhỏ (Block size 2MB-128MB), thay thế cho cơ chế balloon cứng nhắc truyền thống. |
| **`26`** | **`virtio-fs`** | `fs/fuse/virtio_fs.c` | Chia sẻ thư mục máy chủ vào máy ảo dựa trên giao thức FUSE kết hợp chia sẻ trực tiếp bộ nhớ (DAX Window), khắc phục triệt để độ trễ lớn của NFS và 9P. |
| **`31`** | **`virtio-iommu`** | `drivers/iommu/virtio-iommu.c` | Bộ dịch địa chỉ I/O ảo. Quản lý bản đồ DMA an toàn khi triển khai máy ảo lồng máy ảo (Nested Virtualization: chạy VM/Container bên trong VM). |

---

# VIII. BẢNG TỔNG KẾT ĐỐI CHIẾU KIẾN TRÚC I/O TOÀN CẢNH

| Đặc tính Kỹ thuật | Full Emulation (e1000, IDE) | VirtIO (Chuẩn QEMU/vhost) | VirtIO (vhost-user + Polling SPDK) | PCI Passthrough / SR-IOV |
| --- | --- | --- | --- | --- |
| **Mô hình ảo hóa** | Mô phỏng phần cứng cổ điển | Bán ảo hóa chuẩn hóa | Bán ảo hóa Userspace siêu tốc | Ánh xạ trực tiếp phần cứng thật |
| **Driver trong Guest** | Driver gốc của thiết bị thật | Bộ driver chuẩn VirtIO | Bộ driver chuẩn VirtIO | Driver từ hãng phần cứng thật (Intel/Mellanox) |
| **Cơ chế truyền dữ liệu** | Mô phỏng thanh ghi MMIO/PIO | Vòng nhớ chia sẻ (Virtqueue) | Vòng nhớ chia sẻ (Shared Hugepages) | DMA trực tiếp của phần cứng |
| **Tần suất VM-Exit** | Rất cao (Mỗi I/O tốn nhiều Exit) | Thấp (1 Exit / Batching) | **Triệt tiêu hoàn toàn (0 VM-Exit)** | **0 VM-Exit** |
| **Mức chiếm dụng CPU Host** | Cao (Tốn vào việc trap và switch) | Vừa phải (Tối ưu hóa ngắt) | Rất cao (Chiếm 100% Core chạy poll) | Cực thấp (Phần cứng tự xử lý) |
| **Live Migration (Di trú)** | Hỗ trợ dễ dàng | **Hỗ trợ chuẩn mực, dễ dàng** | Hỗ trợ được (Cần cấu hình phức tạp) | Cực kỳ khó khăn hoặc không thể |
| **Thông lượng / Độ trễ** | Rất thấp, độ trễ cao | Rất cao, độ trễ thấp | Tiệm cận tối đa năng lực phần cứng | Đạt 100% năng lực bare-metal |