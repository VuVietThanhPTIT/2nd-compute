
> **Mạch tư duy nối tiếp:**
> - **Bài 03:** Máy ảo Guest đẩy yêu cầu I/O qua `virtqueue` trên bộ nhớ chia sẻ (Shared Memory).
> - **Bài 04:** Compute Host giao tiếp với mảng đĩa NetApp SDS qua mạng bằng giao thức `NVMe-oF` (TCP hoặc RoCEv2).
> - **Bài 05:** Đặt **SPDK** vào đúng vị trí trung tâm trên Compute Host để làm nhạc trưởng: **SPDK nhận yêu cầu từ VM bằng gì, biểu diễn ổ đĩa bên trong nó bằng gì, và bắn gói tin ra card mạng bằng gì?**

## Mục lục
1. Bảng thuật ngữ
2. Nội dung kiến thức và kết quả cần đạt
3. Vị trí thực tế của SPDK trong chuỗi truyền dẫn I/O
4. Trái tim trừu tượng của SPDK: Lớp bdev (Block Device)
5. Phân biệt rạch ròi hai đầu: Vhost Target vs. NVMe-oF Initiator
6. Vi kiến trúc Threading Model: Reactor, Poller và cái giá 100% CPU
7. Kênh điều khiển JSON-RPC: Tách biệt hoàn toàn với Data Plane
8. Căn chỉnh phần cứng: Shared Memory, Hugepages và bài toán NUMA Alignment
9. Chiến lược Lab 2 giai đoạn: Tách rời phụ thuộc để đẩy nhanh Job 1
10. Chuỗi định danh sống còn cho Job 1 (Identity Mapping Chain)
11. Tổng kết 

---
## Bảng thuật ngữ

| Thuật ngữ | Định nghĩa bản chất kỹ thuật |
| :--- | :--- |
| **SPDK** | *Storage Performance Development Kit*: Bộ framework mã nguồn mở chạy hoàn toàn ở User Space, tối ưu hóa I/O bằng cơ chế Polling, Lockless và Kernel Bypass. |
| **bdev (Block Device)** | Tầng trừu tượng hóa thiết bị khối thống nhất bên trong SPDK (tương đương với `struct block_device` trong Linux Kernel). |
| **bdev module** | Driver backend cụ thể hiện thực hóa `bdev` (ví dụ: `bdev_malloc`, `bdev_nvme`, `bdev_aio`). |
| **vhost target** | Module bên trong SPDK đóng vai trò backend phục vụ đĩa ảo cho QEMU/VM thông qua giao thức `vhost-user`. |
| **NVMe-oF initiator**| Driver mạng trong SPDK chủ động khởi tạo kết nối và gửi lệnh NVMe qua mạng (TCP/RDMA) tới storage từ xa. |
| **Reactor** | Vòng lặp sự kiện (Event Loop) chạy cố định trên một CPU core được gán cứng (CPU Pinning) cho SPDK. |
| **Poller** | Hàm xử lý được đăng ký vào Reactor để chạy lặp liên tục, chuyên kiểm tra trạng thái hàng đợi và đón nhận completion. |
| **SPDK Thread** | Đơn vị thực thi logic nhẹ (lightweight thread) của SPDK, được gắn vào một Reactor vật lý, vận hành theo cơ chế bất đồng bộ không khóa. |
| **JSON-RPC** | Giao diện điều khiển (Control Plane) của SPDK qua Unix Domain Socket, dùng để cấu hình động (tạo bdev, gán vhost controller). |
| **NUMA Node** | Cụm gồm một hoặc nhiều CPU socket gắn trực tiếp với một kênh RAM vật lý cục bộ. |

---

## Nội dung kiến thức và kết quả cần đạt

Sau bài học này, bạn cần:
1. **Chỉ rõ vị trí của SPDK**: Nhìn thấy cấu trúc "2 mặt" của SPDK (một mặt hướng về Guest VM, một mặt hướng về Storage Target).
2. **Hiểu tường tận tầng `bdev`**: Tại sao mọi backend (RAM disk, SSD local hay NVMe-oF remote) đều được đối xử như nhau trong SPDK.
3. **Phân biệt hai khái niệm dễ nhầm**: **SPDK Vhost Target** (nằm trên Compute Host) và **Storage Target** (nằm trên NetApp SDS).
4. **Hiểu rõ vi kiến trúc Reactor/Poller**: Lý do vì sao một CPU core chạy SPDK luôn ăn 100% công suất, và sự khác biệt giữa Polling và Interrupt trong SPDK.
5. **Nắm vững nguyên lý NUMA Alignment**: Biết cách ghép nối CPU, RAM và Card mạng cùng một NUMA node để triệt tiêu độ trễ liên kết UPI.
6. **Xây dựng tư duy tách rời cho Job 1**: Triển khai thành công lab thử nghiệm với `malloc bdev` trước khi phụ thuộc vào phần cứng SAN/RDMA thật.

---

## 1. Vị trí thực tế của SPDK trong chuỗi truyền dẫn I/O

SPDK trên máy chủ Compute Host là một **tiến trình User Space độc lập** (thường là daemon `spdk_tgt`). Nó đóng vai trò làm cây cầu bắc ngang giữa máy ảo và mạng lưu trữ:

```text
================================ GUEST VM ================================
  [ Ứng dụng / Database ] ──> [ ext4 / xfs ] ──> [ virtio-blk driver ]
                                                       │
                                                       ▼
                                            [ Hàng đợi Virtqueue ]
============================= COMPUTE HOST ==============================
                                                       │
                           (Shared Hugepages Memory)   ▼
                    +──────────────────────────────────────────────────+
                    |             SPDK PROCESS (spdk_tgt)              |
                    |                                                  |
                    |  [ 1. Vhost-user-blk Controller ]                |
                    |       - Lắng nghe Virtqueue của VM               |
                    |       - Đọc/ghi buffer trên RAM chia sẻ          |
                    |                         │                        |
                    |                         ▼                        |
                    |  [ 2. Tầng trừu tượng bdev (Block Device) ]      |
                    |       - Định tuyến I/O bất đồng bộ               |
                    |                         │                        |
                    |                         ▼                        |
                    |  [ 3. NVMe-oF Initiator Driver ]                 |
                    |       - Đóng gói lệnh NVMe (TCP hoặc RDMA)       |
                    +─────────────────────────│────────────────────────+
                                              │
                                              ▼ (Bypass Host Kernel qua VFIO/UIO)
============================= HARDWARE NETWORK ==========================
                     [ SmartNIC Mellanox ConnectX-6 ]
                                      │
                                      ▼ (Cáp quang RoCEv2 / Switch Lossless)
============================= STORAGE TARGET ============================
                     [ NetApp SDS Target (Subsystem) ]

```

### Dòng chảy dữ liệu (Data Path) qua SPDK:

1. **Tiếp nhận:** SPDK đọc trực tiếp chỉ số Descriptor từ `Virtqueue` trên RAM chung mà không làm phiền đến Host Kernel.
2. **Quy đổi:** `vhost-user-blk` chuyển đổi yêu cầu Virtio thành lệnh khối nội bộ của SPDK (`spdk_bdev_io`).
3. **Định tuyến:** Lớp `bdev` xác định ổ đĩa này gắn với backend nào (ở đây là một NVMe Namespace từ xa).
4. **Truyền dẫn:** `NVMe-oF Initiator` của SPDK trực tiếp ra lệnh cho card mạng đẩy dữ liệu qua mạng tới tủ NetApp Target bằng RDMA.
5. **Nghiệm thu:** Khi NetApp phản hồi, card mạng báo về hàng đợi hoàn tất của SPDK Initiator. SPDK cập nhật kết quả vào `Used Ring` của Virtqueue và gõ chuông ảo (`irqfd`) đánh thức Guest VM.

---

## 2. Trái tim trừu tượng của SPDK: Lớp bdev (Block Device)

Tương tự như tầng Virtual File System (VFS) hay Block Layer của Linux Kernel, SPDK sở hữu một tầng trừu tượng hóa cực kỳ mạnh mẽ mang tên **`bdev`**.

```text
               +───────────────────────────────────────────────+
               |              TẦNG TRÊN (Consumers)            |
               |  vhost-user-blk  |  iSCSI Target  |  NVMe-oF  |
               +───────────────────────┬───────────────────────+
                                       │
                                       ▼ (spdk_bdev API)
               +───────────────────────────────────────────────+
               |             SPDK BDEV CORE LAYER              |
               |      (Quản lý Block size, dung lượng, UUID)   |
               +───────────────────────┬───────────────────────+
                                       │
                   ┌───────────────────┼───────────────────┐
                   ▼                   ▼                   ▼
          [ bdev_malloc ]       [ bdev_nvme (PCIe) ] [ bdev_nvme (oF) ]
            (RAM Disk)           (SSD cắm máy chủ)    (NetApp từ xa)

```

### 1. Bản chất của `bdev`

* `bdev` là một cấu trúc phần mềm (`struct spdk_bdev`) định nghĩa các thuộc tính cơ bản của một ổ đĩa: **kích thước sector** (512B hoặc 4KiB), **tổng số block**, **UUID**, và **con trỏ hàm xử lý đọc/ghi**.
* Đối với tầng trên (`vhost-user-blk`), nó chỉ quan tâm: *"Tôi có một cái bdev tên là `bdev_vol123`, tôi muốn đọc 4 KiB tại LBA 100"*. Nó hoàn toàn không cần biết dữ liệu đó nằm trên RAM hay nằm ở tủ đĩa cách xa 100 mét.

### 2. Các loại `bdev module` quan trọng trong lộ trình:

| Loại bdev | Tên RPC khởi tạo | Dữ liệu lưu ở đâu? | Ý nghĩa trong dự án của bạn |
| --- | --- | --- | --- |
| **`bdev_malloc`** | `bdev_malloc_create` | Trên RAM cấp phát của SPDK | **Cứu tinh cho Lab:** Dùng để viết code tích hợp OpenStack (Job 1) mà không cần chờ phần cứng NetApp/Mellanox. |
| **`bdev_nvme` (Local)** | `bdev_nvme_attach_controller` (PCIe) | SSD NVMe cắm trực tiếp trên mainboard Compute | Dùng để đo kiểm hiệu năng chuẩn của NVMe local trước khi so sánh với mạng. |
| **`bdev_nvme` (Fabrics)** | `bdev_nvme_attach_controller` (RDMA/TCP) | Namespace trên tủ đĩa NetApp SDS | **Mục tiêu cuối cùng của dự án:** Biến ổ đĩa NetApp từ xa thành bdev trong SPDK. |

### 3. Cơ chế I/O Bất đồng bộ (Asynchronous Callback)

* Mọi thao tác đọc/ghi xuống `bdev` đều là **Non-blocking (không chặn)**.
* Hàm gửi lệnh `spdk_bdev_read()` nhận vào một con trỏ hàm phản hồi (Callback Function). SPDK gửi lệnh xong sẽ lập tức quay sang làm việc khác. Khi phần cứng hoàn thành, worker thread của SPDK sẽ tự động gọi hàm callback để trả kết quả cho caller.

---

## 3. Phân biệt rạch ròi hai đầu: Vhost Target vs. NVMe-oF Initiator

Đây là điểm gây nhầm lẫn lớn nhất cho các kỹ sư mới tiếp cận SPDK: **"Trong hệ thống này, đâu là Target, đâu là Initiator?"**

```text
[ Guest VM ]  <==== (vhost-user) ====>  [ SPDK ENGINE ]  <==== (NVMe-oF) ====>  [ NetApp SDS ]
                                        ┌─────────────┐
                                        │ vhost target│ ──> Phục vụ VM
                                        ├─────────────┤
                                        │  Initiator  │ ──> Gọi NetApp
                                        └─────────────┘

```

### Bảng đối chiếu bản chất:

| Thành phần | Nó nhìn về phía nào? | Vai trò kỹ thuật | Thuật ngữ chính xác |
| --- | --- | --- | --- |
| **SPDK vhost-blk** | Hướng về **Guest VM (Bên trong Compute)** | Đóng vai trò là một **Target ảo**. Nó cung cấp một đĩa ảo thông qua Unix Socket để tiến trình QEMU kết nối vào. | **Vhost Target** |
| **SPDK bdev** | Nằm ở **Trung tâm của SPDK** | Làm vật trung gian. Chuyển đổi yêu cầu từ Vhost Target sang cho NVMe-oF Initiator. | **Block Device Abstraction** |
| **SPDK nvme-of** | Hướng về **NetApp SDS (Bên ngoài mạng)** | Đóng vai trò là một **Client (Khởi tạo kết nối)**. Nó chủ động bắt tay mạng với NetApp để lấy Namespace về. | **NVMe-oF Initiator** |
| **NetApp SDS** | Nằm ở **Tủ đĩa từ xa (Storage SAN)** | Đóng vai trò là mảng lưu trữ vật lý thực sự. Tiếp nhận lệnh từ Compute Host và ghi chip Flash. | **Storage Target (Subsystem)** |

> [!CAUTION] Cảnh báo lỗi tư duy khi làm Job 1:
> * Tạo xong kết nối `NVMe-oF Initiator` tới NetApp **mới chỉ sinh ra được một `bdev**`. Lúc này Guest VM **hoàn toàn chưa thấy gì**.
> * Bạn bắt buộc phải thực hiện tiếp bước thứ hai: Lấy `bdev` đó tạo thành một `vhost-blk controller` gắn với một **Unix Socket**, sau đó cấu hình Libvirt/QEMU cắm vào socket này thì Guest VM mới xuất hiện `/dev/vdb`.
> 
> 

---

## 4. Vi kiến trúc Threading Model: Reactor, Poller và cái giá 100% CPU

Tại sao SPDK có thể đạt được hàng triệu IOPS với độ trễ P99 cực phẳng? Câu trả lời nằm ở kiến trúc luồng **Share-Nothing (Không chia sẻ gì cả)**.

```text
[ CPU CORE 1 (Pinning) ]                          [ CPU CORE 2 (Pinning) ]
+─────────────────────────────────────────+      +─────────────────────────────────────────+
| REACTOR (Event Loop vô tận)             |      | REACTOR (Event Loop vô tận)             |
|                                         |      |                                         |
|  [ SPDK Thread A ]                      |      |  [ SPDK Thread B ]                      |
|    ├── Poller 1: Quét Virtqueue VM 1   |      |    ├── Poller 1: Quét Virtqueue VM 2   |
|    └── Poller 2: Quét Completion NIC 1  |      |    └── Poller 2: Quét Completion NIC 2  |
|                                         |      |                                         |
|  * Lockless (Không Mutex/Spinlock)      |      |  * Độc lập tài nguyên hoàn toàn         |
|  * Chạy liên tục (Busy-polling 100%)    |      |  * Trao đổi qua Message Passing         |
+─────────────────────────────────────────+      +─────────────────────────────────────────+

```

### 1. Khối xây dựng vi kiến trúc:

* **Reactor:** Là một vòng lặp sự kiện cơ sở (`while(1)`). SPDK tạo ra mỗi Reactor chạy cố định trên một Core CPU vật lý (thông qua tham số `-m` CPU Core Mask).
* **SPDK Thread:** Không phải là `pthread` của Linux! Đây là một cấu trúc trừu tượng chạy trên một Reactor. Một Reactor có thể điều phối nhiều SPDK Thread.
* **Poller:** Là một con trỏ hàm được đăng ký vào Reactor. Cứ mỗi vòng lặp, Reactor gọi lại hàm Poller này để kiểm tra:
* *Có I/O nào mới từ VM đẩy vào Virtqueue không?*
* *Có gói tin nào mới từ card mạng Mellanox báo về không?*



### 2. Nguyên lý "Không chia sẻ" (Share-Nothing Architecture)

* Trong Linux Kernel, khi nhiều core cùng xử lý một hàng đợi, chúng phải dùng các biến khóa (Spinlock/Mutex). Khi tải cao, các CPU core liên tục tranh chấp khóa khiến hiệu năng bị suy giảm nghiêm trọng.
* SPDK phân chia rạch ròi: Mỗi Reactor sở hữu riêng bộ nhớ đệm, riêng Queue mạng và riêng hàng đợi Virtqueue. Nếu Thread A muốn gửi dữ liệu sang Thread B, nó dùng cơ chế **Message Passing** (ghi tin nhắn vào ring buffer nội bộ không khóa).

### 3. Bản chất của việc "CPU luôn ăn 100%"

* Khi gõ lệnh `htop` trên Host, bạn sẽ thấy các CPU Core dành cho SPDK **luôn luôn đỏ rực ở mức 100%**, ngay cả khi VM đang tắt hoặc không hề có I/O nào phát sinh.
* **Lý do:** Reactor liên tục chạy vòng lặp Poller ở tốc độ hàng triệu lần/giây để sẵn sàng đón nhận I/O ở mức nano-giây.
* **Cơ chế Interrupt Mode của SPDK:** SPDK có hỗ trợ chuyển sang cơ chế ngắt khi rảnh rỗi (`spdk_interrupt_mode`). Tuy nhiên, tính năng này làm tăng độ trễ và không phải transport nào cũng hỗ trợ hoàn hảo. Trong môi trường Cloud hiệu năng cao, các kỹ sư thường chấp nhận **cách ly riêng một vài core CPU chuyên dụng (Isolcpus)** chỉ để phục vụ SPDK Polling.

---

## 5. Kênh điều khiển JSON-RPC: Tách biệt hoàn toàn với Data Plane

Để điều khiển tiến trình `spdk_tgt`, bạn không sửa file cấu hình tĩnh rồi restart dịch vụ. SPDK cung cấp giao diện **JSON-RPC qua Unix Domain Socket** (mặc định tại `/var/run/spdk.sock`).

```text
[ SCRIPT AUTOMATION (Job 1) ]
  (Python os-brick / rpc.py)
            │
            ▼ (JSON-RPC qua Unix Domain Socket: /var/run/spdk.sock)
+──────────────────────────────────────────────────────────────────────────────+
| [ CONTROL PLANE ]                                                            |
| SPDK RPC Server: Lắng nghe, phân tích cú pháp JSON, gọi hàm nội bộ           |
|                  để cấu hình bdev / vhost controller.                        |
+──────────────────────────────────────┬───────────────────────────────────────+
                                       │ (Cấu hình xong)
                                       ▼
+──────────────────────────────────────────────────────────────────────────────+
| [ DATA PLANE ]                                                               |
| Guest VM  <════════════ (Shared Hugepages Memory) ════════════>  NetApp Target|
|                                                                              |
| * Dữ liệu đọc/ghi chạy độc lập, TUYỆT ĐỐI KHÔNG GỌI RPC TRONG KHI ĐANG I/O!  |
+──────────────────────────────────────────────────────────────────────────────+

```

### 3 lệnh JSON-RPC kinh điển tạo nên Data Path:

#### Bước 1: Kết nối tới Storage Target (Tạo bdev)

Initiator của SPDK bắt tay với tủ NetApp để tạo một bdev mang tên `Nvme0n1`:

```bash
python3 scripts/rpc.py bdev_nvme_attach_controller \
    -b Nvme0 \
    -t rdma \
    -a 192.168.100.10 \
    -s 4420 \
    -f ipv4 \
    -n nqn.2026-09.com.netapp:subsystem-01

```

#### Bước 2: Tạo Vhost-blk Controller (Mở socket cho QEMU)

Gắn `bdev` vừa tạo vào một Unix socket mới để chuẩn bị trao cho VM:

```bash
python3 scripts/rpc.py vhost_create_blk_controller \
    --ctrlr vhost.vol123 \
    --dev-name Nvme0n1 \
    -u /var/run/spdk/vhost.vol123.sock

```

#### Bước 3: Kiểm tra trạng thái

```bash
python3 scripts/rpc.py bdev_get_bdevs
python3 scripts/rpc.py vhost_get_controllers

```

> [!TIP] Lưu ý cho Job 1 (Control Plane Automation):
> Việc viết code trong `os-brick` thực chất là viết một **Python JSON-RPC Client** kết nối tới socket `/var/run/spdk.sock`. Code của bạn phải đảm bảo tính **Idempotency (Bất biến)**: kiểm tra xem bdev đã tồn tại chưa trước khi tạo, và khi detach volume phải dọn dẹp sạch sẽ vhost controller trước khi ngắt kết nối bdev.

---

## 6. Căn chỉnh phần cứng: Shared Memory, Hugepages và bài toán NUMA Alignment

Để đạt được hiệu năng tối đa, phần mềm phải khớp hoàn toàn với kiến trúc vật lý của máy chủ.

### 1. Phân biệt bộ ba khái niệm bộ nhớ:

* **Shared Memory:** Ràng buộc logic để QEMU và SPDK cùng ánh xạ chung một vùng RAM.
* **Hugepages (2MB / 1GB):** Giảm kích thước bảng trang bộ nhớ, triệt tiêu hiện tượng TLB Miss của CPU khi truy xuất hàng chục Gigabyte bộ nhớ.
* **DMA-capable Memory:** Vùng nhớ vật lý liền mạch được khóa lại (pinned) để card mạng hoặc SSD Controller có thể tự do đọc/ghi an toàn.

### 2. Nút thắt cổ chai NUMA (Non-Uniform Memory Access)

Trong máy chủ 2 CPU Socket, mỗi CPU quản lý một nửa số RAM và một số khe cắm PCIe cục bộ. Nếu dữ liệu phải chạy xuyên qua liên kết giữa 2 CPU (Intel UPI / AMD Infinity Fabric), độ trễ sẽ bị tăng gấp đôi!

```text
[ TÌNH THÁNG NGHẼN MẠCH NUMA (TỒI TỆ) ]
  vCPU của VM: Nằm ở CPU Socket 0
  Hugepages RAM: Nằm ở CPU Socket 1  ──> Dữ liệu phải bò qua đường cáp UPI hẹp!
  SPDK Core:    Nằm ở CPU Socket 0  ──> Tăng độ trễ, spike P99 vọt xà.
  Card NIC:     Cắm khe PCIe Socket 1

[ NGUYÊN TẮC NUMA ALIGNMENT CHUẨN MỰC (HOÀN HẢO) ]
  Tất cả cùng nằm trọn vẹn trên NUMA NODE 0:
  [ vCPU VM ] + [ Hugepages RAM ] + [ SPDK Poller Core ] + [ SmartNIC PCIe Slot ]
  ===> Toàn bộ I/O chỉ chạy trong bus nội bộ của Socket 0, độ trễ chạm đáy!

```

---

## 7. Chiến lược Lab 2 giai đoạn: Tách rời phụ thuộc để đẩy nhanh Job 1

Đừng đợi đến khi team Network cấu hình xong switch RoCEv2 hay team Storage bàn giao tủ NetApp thì bạn mới bắt đầu viết code OpenStack. Điều đó sẽ làm bạn bị trễ tiến độ hoàn toàn.

Hãy chia quá trình thực nghiệm thành hai pha:

```text
[ GIAI ĐOẠN 1: LAB LOCAL (Máy ảo cá nhân / Server thường) ]
  VM (virtio-blk) ──> vhost-user socket ──> SPDK Target ──> [ bdev_malloc (RAM) ]
  * Mục tiêu: Viết xong 100% code Cinder/Nova/os-brick, sinh chuẩn XML Libvirt, 
              test thành công Attach/Detach nóng ổ đĩa.

[ GIAI ĐOẠN 2: LAB REMOTE (Phần cứng Enterprise thật) ]
  VM (virtio-blk) ──> vhost-user socket ──> SPDK Target ──> [ bdev_nvme (RDMA) ] ──> NetApp
  * Mục tiêu: Đổi cấu hình RPC sang IP thật của NetApp, đo kiểm hiệu năng IOPS/P99,
              thử nghiệm ngắt dây mạng test Failover ANA.

```

> [!WARNING] Cảnh báo quan trọng về `malloc bdev`:
> `malloc bdev` là một ổ đĩa giả lập trên thanh RAM. Khi tiến trình SPDK tắt hoặc máy tính khởi động lại, **toàn bộ dữ liệu trên đó sẽ bốc hơi 100%**.
> * Bạn chỉ dùng `malloc bdev` để **kiểm tra luồng điều khiển (Control Plane)** và kiểm tra giao thức `vhost-user`.
> * **Tuyệt đối không dùng kết quả benchmark trên `malloc bdev` để đưa vào báo cáo hiệu năng lưu trữ**, vì nó không phản ánh độ trễ của mạng và chip Flash thật.
> 
> 

---

## 8. Chuỗi định danh sống còn cho Job 1 (Identity Mapping Chain)

Nhiệm vụ tối quan trọng trong phần việc **Job 1** của bạn là duy trì tính toàn vẹn của chuỗi định danh sau. Một sai lệch nhỏ ở bất kỳ mắt xích nào cũng sẽ làm máy ảo bị treo hoặc gắn nhầm dữ liệu của khách hàng khác:

```text
[ 1. OpenStack Cinder Volume ID ]
     Ví dụ: `volume-88a9-4b21`
               │
               ▼
[ 2. NetApp Target Namespace & NQN ]
     IP:Port + Target NQN + NSID (ví dụ: NSID 1)
               │
               ▼
[ 3. SPDK bdev Name ]
     Tên định danh bên trong SPDK: `NetApp_vol_88a9`
               │
               ▼
[ 4. SPDK Vhost Controller Name & Socket Path ]
     Controller: `vhost.vol_88a9`
     Socket File: `/var/run/spdk/vhost.vol_88a9.sock`
               │
               ▼
[ 5. Libvirt Domain XML Device ]
     <disk type='vhostuser'>
       <source type='unix' path='/var/run/spdk/vhost.vol_88a9.sock'/>
       <target dev='vdb' bus='virtio'/>
     </disk>

```

Khi thực hiện lệnh **Attach Volume**, code của bạn tạo chuỗi từ trên xuống dưới $[1] \rightarrow [5]$. Khi thực hiện **Detach Volume**, code của bạn phải tháo gỡ an toàn theo chiều ngược lại $[5] \rightarrow [1]$ để đảm bảo không làm rò rỉ tài nguyên bộ nhớ trên Host Compute.

---

## Tổng kết  80/20

1. **SPDK đứng ở vị trí kép trên Compute Host:** Làm **Vhost Target** phục vụ máy ảo Guest qua Shared Memory, và làm **NVMe-oF Initiator** kết nối tới tủ NetApp qua mạng.
2. **`bdev` là lớp trừu tượng hóa chuẩn:** Cho phép gắn kết linh hoạt bất kỳ backend nào (từ RAM disk `malloc` đến kết nối mạng `nvme-of`) mà không làm thay đổi tầng trên.
3. **Threading Model không khóa (Lockless):** Reactor chạy Poller liên tục trên các CPU Core gán cứng, triệt tiêu độ trễ Context Switch nhưng đánh đổi bằng việc tiêu thụ 100% CPU.
4. **JSON-RPC thuộc về Control Plane:** Dùng để cấu hình khởi tạo tài nguyên; các thao tác đọc/ghi I/O thông thường của VM hoàn toàn không đi qua RPC.
5. **NUMA Alignment là chìa khóa của độ trễ:** Gom vCPU, Hugepages, SPDK Core và SmartNIC vào chung một NUMA Node vật lý để dập tắt hiện tượng trễ biến động P99.
6. **Chiến lược phân tầng cho Job 1:** Sử dụng `bdev_malloc` để làm chủ 100% logic tích hợp OpenStack và Libvirt XML trước khi đưa hệ thống lên mạng RDMA và tủ đĩa NetApp thật.
