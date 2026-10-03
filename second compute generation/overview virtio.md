

# I  , Bối cảnh & Lý do VirtIO ra đời
- Vấn đề phân mảnh: linux lúc đó có nhiều giải pháp ảo hóa mạng khác nhau  (KVM, lguest, User-Mode Linux)  , gây phân mảnh và tốn công bảo trì 
- -> **Rusty Russell:** Tác giả sáng tạo ra VirtIO nhằm cung cấp một **lớp trừu tượng hóa chuẩn (common abstraction layer)** chung cho toàn bộ Linux, giúp chuẩn hóa giao tiếp I/O Paravirtualization và tái sử dụng mã nguồn trên mọi nền tảng ảo hóa.
# II , Full Virtualization vs. Paravirtualization (Bản chất I/O)

- **Full Virtualization:** Hypervisor phải mô phỏng phần cứng vật lý ở tầng thấp nhất (ví dụ giả lập card mạng thật). Việc này giúp chạy được hệ điều hành nguyên bản không cần sửa, nhưng cực kỳ cồng kềnh, phức tạp và chậm chạp do phát sinh quá nhiều bẫy ngắt (trap).
    
- **Paravirtualization (VirtIO):** Hệ điều hành Guest biết mình đang ở trong máy ảo và chủ động phối hợp với Hypervisor. Driver máy ảo (Front-end) và Hypervisor (Back-end) thống nhất giao tiếp qua một giao thức chung để tối ưu hiệu năng. Điểm đánh đổi duy nhất là Guest OS cần có driver chuyên dụng.

# III , architecture : 
- **Front-end Drivers (trong Guest OS):** Các driver như `virtio_net` (mạng), `virtio_blk` (đĩa), `virtio_balloon` (co giãn RAM), console, v.v.
- **Back-end Drivers (trong Hypervisor / QEMU):** Nhận lệnh từ Front-end và trực tiếp xử lý với phần cứng thật.
- **Cầu nối trung gian (Virtqueue / Ring):** Hàng đợi ảo kết nối hai đầu, được triển khai dưới dạng các vòng buffer (ring buffer) trong bộ nhớ chia sẻ.
```
================================ GUEST OS ================================
 [virtio-blk]       [virtio-net]       [virtio-pci]     [virtio-balloon]    [virtio-console]
(Block Storage)    (Network Card)      (PCI Transport)   (Dynamic Memory)   (Serial Console)
       │                  │                  │                  │                  │
 ./drivers/block/   ./drivers/net/     ./drivers/virtio/  ./drivers/virtio/  ./drivers/virtio/
   virtio-blk.c       virtio-net.c       virtio-pci.c     virtio-balloon.c   virtio-console.c
       │                  │                  │                  │                  │
       └──────────────────┴──────────┬───────┴──────────────────┴──────────────────┘
                                     │
                                     ▼
                       ┌───────────────────────────┐
                       │          virtio           │  <-- Lớp trừu tượng hóa chung
                       │                           │      (Core Framework)
                       │ ./drivers/virtio/virtio.c │
                       └─────────────┬─────────────┘
                                     │
                                     ▼
                       ┌───────────────────────────┐
                       │         Transport         │  <-- Cơ chế hàng đợi vòng
                       │   (Virtqueue / vring)     │      (Shared Memory Ring Buffer)
                       │ ./drivers/virtio/         │
                       │        virtio_ring.c      │
                       └─────────────┬─────────────┘
=====================================│====================================
                        Shared Memory (RAM) / Doorbell (MMIO)
=====================================│====================================
                                     ▼
                       ┌───────────────────────────┐
                       │    virtio back-end        │  <-- Xử lý I/O thật
                       │        drivers            │      (QEMU Userspace,
                       │  (QEMU / vhost / HW)      │       vhost Kernel, vDPA)
                       └───────────────────────────┘
================================ HOST SYSTEM =============================
```


## IV Flow 
1 . Quá trình khởi tạo thiết bị (Initialization Phase)
**Device Discovery (Nhận diện thiết bị):** Thiết bị được phơi ra (expose) cho Guest qua bus **PCI/PCIe** Khi Guest boot, cơ chế dò tìm PCI sẽ đọc Vendor ID / Device ID để tự động nạp đúng driver VirtIO trong Guest Kernel
2 .**Feature Negotiation (Đàm phán tính năng):** QEMU gửi yêu cầu lấy danh sách tính năng hỗ trợ từ Backend, sau đó so khớp với khả năng của Driver để quyết định những tính năng nào sẽ được bật (ví dụ: kích thước buffer, các cờ offload)

3 . **Virtqueue Configuration (Cấu hình hàng đợi):**

- QEMU cấp phát bộ nhớ RAM dùng chung dưới dạng **Shared Hugepages** khi boot
- QEMU chuyển file descriptor (fd) của vùng RAM này sang Backend qua Unix Socket để Backend gọi `mmap()` vào không gian nhớ của nó
- VirtIO Driver trong Guest dành riêng một phần vùng RAM này để khởi tạo các hàng đợi logic (`controlq`, `eventq`, `tx`, `rx`, ...)
4 . 
### 1. Driver VirtIO là gì và nó giúp Guest "trực tiếp" thế nào?

#### Driver là gì trong ngữ cảnh này?

- Trong máy thật: Driver là phần mềm trung gian giúp Kernel hệ điều hành nói chuyện với một con chip vật lý cụ thể (ví dụ driver card mạng Realtek, driver đồ họa NVIDIA). Driver biết thanh ghi nào của con chip nằm ở đâu để điều khiển.
- Trong VirtIO: **VirtIO Driver** (hay còn gọi là **Frontend Driver**) là các module trong Kernel của Guest (như `virtio_net.ko`, `virtio_blk.ko`). Điểm khác biệt là nó **không nói chuyện với con chip vật lý nào**, mà nó tuân theo chuẩn VirtIO để ghi/đọc dữ liệu vào một cấu trúc bộ nhớ chia sẻ chung giữa Guest và Host gọi là **Virtqueue (Shared Memory)**.
#### VirtIO có bỏ hoàn toàn Interrupt và Trap để "trực tiếp" ra phần cứng thật không?

> **Câu trả lời chính xác: KHÔNG.** Guest **không trực tiếp chạm vào phần cứng vật lý bên ngoài**, và nó **vẫn phải dùng Trap & Interrupt**, nhưng cách thức thực hiện đã được tối ưu hóa triệt để.
> 
>   

- **Ai mới trực tiếp nói chuyện với phần cứng ngoài?**
    
    Là **Host (QEMU / KVM / vhost)**. Guest chỉ đưa yêu cầu vào RAM chia sẻ, Host đứng ra đọc RAM đó rồi thay mặt Guest ghi xuống ổ cứng thật hoặc gửi qua card mạng thật.  
    
- **Tại sao không bỏ được Trap (VM-Exit)?**
    Khi Guest nhét 100 gói tin vào RAM chia sẻ, Host không thể cứ chạy vòng lặp vô tận `while(true)` để soi RAM (vì sẽ ăn 100% CPU Host). Guest bắt buộc phải **gõ chuông (Kick / Doorbell)** báo cho Host biết. Thao tác gõ chuông này thực chất là ghi 1 byte vào thanh ghi ảo $\rightarrow$ **Vẫn kích hoạt VM-Exit (Trap)**.  
    
- **Vậy VirtIO nhanh hơn Full Emulation ở điểm nào?**  
    1. **Batching (Gom nhóm):** Full Emulation gửi 1 gói tin có thể tốn hàng chục lần VM-Exit (mô phỏng từng thanh ghi phần cứng). VirtIO gom 64 hoặc 128 gói tin vào hàng đợi rồi chỉ cần **1 lần VM-Exit duy nhất** để kick Host.
        
    2. **Zero-Copy / Shared Memory:** Host đọc trực tiếp từ RAM của Guest, không cần sao chép dữ liệu qua lại nhiều tầng trung gian.
         
    3. **Polling Mode (vhost-user/DPDK):** Ở các hệ thống cực đoan về tốc độ, Host có thể bật polling để quét RAM liên tục, lúc này mới thực sự loại bỏ được VM-Exit của chuông kick.
_(Lưu ý: Nếu muốn Guest trực tiếp điều khiển 100% phần cứng vật lý mà bỏ qua cả QEMU/VirtIO, người ta dùng công nghệ **PCI Passthrough / SR-IOV** kết hợp VFIO, không phải VirtIO)._

### 2. Tổng hợp tất cả các loại VirtIO phổ biến trong thực tế

Chuẩn VirtIO (chuẩn hóa bởi tổ chức OASIS) định nghĩa rất nhiều loại thiết bị chuyên biệt:
#### Nhóm Lưu trữ & Tệp tin (Storage & Filesystem)

- **virtio-blk (Block Device):** Thiết bị lưu trữ dạng khối. Dùng cho ổ cứng chính của máy ảo (`/dev/vda`), tối ưu cho đọc/ghi raw block tốc độ cao.
 
- **virtio-scSI:** Thiết bị lưu trữ chuẩn SCSI ảo. Khác với `virtio-blk` (chỉ gắn từng đĩa đơn giản), `virtio-scsi` cho phép gắn hàng trăm ổ đĩa (LUNs) sau một controller duy nhất, hỗ trợ đầy đủ tập lệnh SCSI phức tạp (như lệnh passthrough SCSI, TRIM/UNMAP, quản trị mảng đĩa SAN).
    
- **virtio-fs (Virtio Filesystem):** Chia sẻ trực tiếp một thư mục từ Host vào Guest với hiệu năng gần như local disk, khắc phục điểm yếu chậm chạp của NFS hay 9P protocol.
  
#### Nhóm Mạng (Networking)

- **virtio-net:** Card mạng ảo paravirtualized. Hỗ trợ đa hàng đợi (multiqueue), checksum offloading, TSO/LSO (Large Send Offload), kết hợp với `vhost-net` hoặc Open vSwitch cho thông lượng hàng chục Gbps.
    
#### Nhóm Bộ nhớ & Quản trị tài nguyên (Memory Management)
- **virtio-balloon:** Thiết bị "bóng bay" co giãn RAM. Khi Host thiếu RAM, nó bảo driver balloon trong Guest "phồng lên" (chiếm dụng bộ nhớ của Guest và trả quyền dùng RAM vật lý đó về cho Host). Khi Guest cần RAM, quả bóng "xẹp xuống".
- **virtio-mem:** Cơ chế cắm/rút RAM nóng (Hotplug/Hotunplug memory) linh hoạt theo từng khối nhỏ (chunk size vài chục MB), khắc phục hạn chế cứng nhắc của balloon truyền thống.

#### Nhóm Đồ họa & Hiển thị (Graphics & Display)

- **virtio-gpu:** Card đồ họa ảo 2D/3D. Có thể chuyển tiếp (forward) các lệnh dựng hình OpenGL/Vulkan từ Guest xuống GPU thật của Host để render (Virglrenderer).
#### Nhóm Giao tiếp liên tiến trình & Hệ thống (IPC & System)

- **virtio-serial (virtio-console):** Cổng giao tiếp dạng character/stream giữa Host và Guest. Đây chính là kênh truyền tin cậy được dùng bởi **QEMU Guest Agent (`qemu-ga`)** để Host ra lệnh cho Guest (như đóng băng filesystem lúc snapshot, đổi mật khẩu root).
    
 
- **virtio-vsock (Virtual Sockets):** Cung cấp cơ chế giao tiếp Socket mạng (chuẩn `AF_VSOCK`) giữa ứng dụng trên Host và ứng dụng trong Guest mà **không cần cấu hình IP hay Card mạng**. Tốc độ truyền tin cực nhanh qua bộ nhớ chia sẻ.
- **virtio-rng:** Bộ sinh số ngẫu nhiên phần cứng ảo (Random Number Generator). Chuyển nguồn entropy ngẫu nhiên chất lượng cao từ `/dev/urandom` của Host vào Guest để tránh việc Guest bị treo do thiếu entropy khi khởi động hoặc mã hóa.

- **virtio-iommu:** Bộ quản lý địa chỉ I/O ảo, phục vụ quản lý phân vùng bộ nhớ an toàn (DMA protection) khi chạy lồng ảo hóa (Nested Virtualization).
    

### Tóm tắt trực quan luồng đi

```
[Ứng dụng trong Guest]
        │
[VirtIO Frontend Driver]  <-- Nằm trong Guest Kernel (Ghi dữ liệu vào RAM)
        │
   (Vòng lặp RAM - Virtqueue)
        │
  [Kick qua Doorbell]     <-- Kích hoạt Trap/VM-Exit nhẹ để báo hiệu
        │
[VirtIO Backend / QEMU]   <-- Nằm ở Host (Đọc RAM, gọi Driver vật lý thực thi)
        │
[Phần cứng vật lý thật]
```