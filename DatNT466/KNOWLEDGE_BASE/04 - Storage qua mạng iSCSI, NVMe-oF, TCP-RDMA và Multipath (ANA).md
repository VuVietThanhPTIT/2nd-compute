> **Mạch nối tiếp:**
> - **Bài 01 & 02:** Khái niệm I/O cơ bản và luồng xử lý nội bộ của nhân Linux.
> - **Bài 03:** Vượt ranh giới ảo hóa từ Guest VM sang Compute Host bằng `vhost-user-blk`.
> - **Bài 04:** Rời khỏi Compute Host. Dữ liệu làm thế nào để đi xuyên qua mạng vật lý (Fabric) để tới tủ đĩa **NetApp SDS Target**, và các giao thức mạng giải quyết bài toán băng thông, độ trễ và tính sẵn sàng cao (High Availability) ra sao.

---

## Mục lục
1. Bảng thuật ngữ
2. Nội dung kiến thức và kết quả cần đạt
3. Mô hình tổng thể: Phân định Initiator, Target và Fabric
4. Giao thức truyền thống: SCSI và iSCSI (Baseline hiện tại)
5. Cuộc cách mạng NVMe over Fabrics (NVMe-oF)
6. So sánh tầng vận chuyển: NVMe/TCP vs. NVMe/RDMA (RoCEv2)
7. Cơ chế đa đường (Multipathing): DM-Multipath vs. NVMe ANA
8. Ma trận 5 phương án Benchmark & Cạm bẫy đo lường
9. Tổng kết 

## Bảng thuật ngữ

| Thuật ngữ | Định nghĩa bản chất |
| :--- | :--- |
| **Initiator** | Phía phát sinh và gửi yêu cầu I/O (ở đây là tiến trình trên Compute Host: Kernel iSCSI hoặc SPDK). |
| **Target** | Phía tiếp nhận, bóc tách lệnh và điều khiển các ổ đĩa vật lý để thực thi I/O (ở đây là hệ thống NetApp SDS). |
| **SCSI / CDB** | *Small Computer System Interface*: Bộ tập lệnh lưu trữ truyền thống; sử dụng cấu trúc khối mô tả lệnh *Command Descriptor Block*. |
| **iSCSI / IQN** | Giao thức bọc lệnh SCSI vào các gói tin TCP/IP; định danh thiết bị bằng chuỗi *iSCSI Qualified Name*. |
| **LUN (Logical Unit Number)** | Đơn vị lưu trữ khối logic được chia sẻ từ Target trong thế giới SCSI/iSCSI. |
| **NVMe-oF** | *NVMe over Fabrics*: Chuẩn mở rộng cho phép truyền các lệnh NVMe nguyên bản qua mạng (Ethernet, Fibre Channel, InfiniBand). |
| **Subsystem / Namespace** | Các khái niệm của NVMe tương đương với Target / LUN của SCSI. Subsystem chứa các Namespace (vùng LBA liên tục). |
| **NQN** | *NVMe Qualified Name*: Chuỗi định danh duy nhất cho Host Initiator và Subsystem Target. |
| **RDMA** | *Remote Direct Memory Access*: Kỹ thuật card mạng bốc dữ liệu trực tiếp từ RAM máy này đổ sang RAM máy khác qua mạng, bypass hoàn toàn CPU/OS cả hai đầu. |
| **RoCEv2** | *RDMA over Converged Ethernet v2*: Chạy RDMA trên mạng Ethernet/IP tiêu chuẩn bằng cách bọc frame RDMA vào gói tin UDP. |
| **ANA (Asymmetric Namespace Access)**| Cơ chế của NVMe cho phép Target thông báo cho Initiator biết đường dẫn (path) nào đang tối ưu (Optimized) hoặc không tối ưu (Non-Optimized). |

---

## Nội dung kiến thức và kết quả cần đạt

1. **Phân biệt rạch ròi 3 tầng:** Tầng Tập lệnh lưu trữ (SCSI vs. NVMe), Tầng Giao thức mạng (iSCSI vs. NVMe-oF) và Tầng Vận chuyển (TCP vs. RDMA).
2. **Giải thích vì sao iSCSI trở thành nút thắt cổ chai** khi kết hợp với hạ tầng All-Flash NVMe hiện đại.
3. **Hiểu bản chất của RDMA/RoCEv2**: Không chỉ có ưu điểm "độ trễ thấp" mà còn đi kèm các ràng buộc khắt khe về hạ tầng mạng (Lossless Ethernet, PFC, ECN).
4. **So sánh cơ chế xử lý đa đường**: Sự khác nhau giữa Linux DM-Multipath truyền thống và NVMe ANA Native Multipath.
5. **Nắm vững ma trận kiểm thử**: Biết cách cô lập từng biến số khi benchmark so sánh hiệu năng trong dự án.

---

## 1. Mô hình tổng thể: Phân định Initiator, Target và Fabric

Toàn bộ hệ thống lưu trữ phân tán đều hoạt động theo mô hình Client - Server chuyên biệt:

```text
+──────────────────────────────── COMPUTE HOST ───────────────────────────────+
| [ Guest VM: /dev/vdb ]                                                       |
|       │                                                                      |
|       ▼ (Virtqueue / Shared Memory)                                          |
| [ STORAGE INITIATOR ] ──> Kẻ khởi xướng: Chủ động tạo kết nối và phát lệnh   |
|  - Phương án cũ: Host Kernel iSCSI Driver                                    |
|  - Phương án mới: SPDK NVMe-oF Initiator (User Space)                        |
+───────│──────────────────────────────────────────────────────────────────────+
        │
        ▼ (Gói tin lưu trữ chạy trên dây cáp mạng)
+──────────────────────────────── STORAGE FABRIC ──────────────────────────────+
| [ Cáp quang / Switch Leaf-Spine ]                                            |
|  - Chạy giao thức TCP/IP truyền thống HOẶC RoCEv2 (Lossless Network)         |
+───────│──────────────────────────────────────────────────────────────────────+
        │
        ▼
+─────────────────────────── STORAGE TARGET (NetApp SDS) ──────────────────────+
| [ STORAGE TARGET ]    ──> Kẻ cung cấp: Lắng nghe, nhận lệnh, đọc/ghi chip    |
|                           Flash và gửi kết quả hoàn tất về cho Initiator.    |
+──────────────────────────────────────────────────────────────────────────────+

```

* **Điểm mù cần tránh:**
Guest VM bên trong hoàn toàn **không biết và không cần biết** phía sau Compute đang dùng cáp quang gì, giao thức gì. Guest chỉ thấy duy nhất một ổ đĩa `/dev/vdb`. Mọi việc đóng gói dữ liệu thành gói tin iSCSI hay NVMe-oF là trách nhiệm độc quyền của **Initiator trên Compute Host**.

---

## 2. Giao thức truyền thống: SCSI và iSCSI (Baseline hiện tại)

Hệ thống hiện tại của team bạn đang sử dụng **iSCSI** nối tới NetApp. Để hiểu vì sao cần thay thế, phải hiểu iSCSI hoạt động thế nào.

```text
[ Lệnh đọc/ghi từ App ]
         │
         ▼
[ SCSI Command (CDB) ]  --> Lệnh chuẩn hóa: READ(10), WRITE(10)...
         │
         ▼
[ iSCSI Layer (RFC 7143) ] --> Bọc SCSI CDB vào các đơn vị PDU (Protocol Data Unit)
         │
         ▼
[ TCP/IP Stack (Kernel) ]  --> Chia nhỏ PDU thành các TCP Segment, thêm IP Header
         │
         ▼
[ Cáp mạng Ethernet ]   --> Gửi gói tin qua mạng LAN/SAN

```

### 1. Bản chất kỹ thuật của iSCSI

* **SCSI (Small Computer System Interface):** Ra đời từ những năm 1980 cho ổ đĩa cơ (HDD). SCSI được thiết kế quanh mô hình **hàng đợi đơn hoặc rất ít hàng đợi (Single Queue)** có khóa bảo vệ (Mutex Lock).
* **iSCSI:** Lấy nguyên vẹn tập lệnh SCSI cổ điển đóng gói vào gói tin TCP (cổng mặc định `3260`). Thiết bị được chia thành các **LUN (Logical Unit Number)** và định danh bằng chuỗi **IQN** (ví dụ: `iqn.1992-08.com.netapp:sn.12345`).

### 2. Vì sao iSCSI tạo ra "nút thắt cổ chai" với SSD hiện đại?

1. **Lệch pha thế hệ:** Ổ cứng All-Flash NVMe có thể xử lý song song hàng triệu IOPS, nhưng bộ tập lệnh SCSI và cấu trúc đóng gói PDU của iSCSI lại cồng kềnh, sinh ra chi phí CPU parsing rất lớn.
2. **Chi phí TCP Stack trong Kernel:** Mỗi gói tin iSCSI đi qua nhân Linux phải chịu sự điều phối của hệ điều hành: cấp phát `sk_buff`, tính toán TCP Checksum, xử lý ngắt mạng (Network IRQ) và copy dữ liệu nhiều lần giữa các tầng bộ đệm.
3. **Tranh chấp khóa (Lock Contention):** Mô hình hàng đợi của iSCSI không thể co giãn tuyến tính theo số lượng CPU Core lớn của các dòng vi xử lý máy chủ Compute đời mới.

---

## 3. Cuộc cách mạng NVMe over Fabrics (NVMe-oF)

NVMe ra đời để thay thế hoàn toàn SCSI cho các thiết bị lưu trữ bán dẫn thể rắn (Flash NAND). **NVMe over Fabrics (NVMe-oF)** là tiêu chuẩn mở rộng, mang nguyên vẹn các ưu điểm của NVMe cục bộ (PCIe) truyền qua hạ tầng mạng.

### 1. Bảng đối chiếu cấu trúc: iSCSI vs. NVMe-oF

| Đặc tính kiến trúc | iSCSI (SCSI over TCP) | NVMe-oF (NVMe over Fabrics) |
| --- | --- | --- |
| **Bộ tập lệnh** | Phức tạp, cồng kềnh, hàng trăm lệnh kế thừa thời HDD. | Tinh giản, chỉ gồm vài lệnh cơ sở: *Read, Write, Flush...* |
| **Kích thước lệnh** | Biến thiên tùy loại CDB (6, 10, 12, 16 bytes). | Cố định chuẩn xác: **64 bytes** cho lệnh gửi, **16 bytes** cho phản hồi. |
| **Cơ chế hàng đợi** | Thường là 1 hàng đợi chung, tối đa 256/1024 lệnh, cần Lock. | **Đa hàng đợi cực lớn:** Hỗ trợ tới 64.000 Queue, mỗi Queue sâu 64.000 lệnh. |
| **Tối ưu đa nhân CPU** | Kém; các CPU Core dễ tranh chấp một hàng đợi iSCSI. | Hoàn hảo; **mỗi CPU Core sở hữu riêng một cặp Queue (Submission/Completion)**, chạy song song không cần khóa (Lockless). |
| **Đơn vị lưu trữ logic** | **LUN** | **Namespace** (nằm trong một Subsystem) |
| **Chuỗi định danh** | **IQN** (`iqn.xyz...`) | **NQN** (`nqn.2014-08.org.nvmexpress:...`) |

### 2. Các thực thể cốt lõi trong NVMe-oF

```text
[ STORAGE TARGET: NetApp SDS ]
  └── NVMe Subsystem (NQN: nqn.2026-09.com.netapp:subsystem-storage-01)
       ├── Controller ảo 1 (Cung cấp các Queue I/O cho Host A)
       ├── Controller ảo 2 (Cung cấp các Queue I/O cho Host B)
       └── Danh sách Namespaces:
            ├── Namespace ID 1 (NSID 1 - 500GB)  ──> Volume gắn cho VM 1
            └── Namespace ID 2 (NSID 2 - 1000GB) ──> Volume gắn cho VM 2

```

* **Subsystem:** Là một thực thể Target độc lập, có một địa chỉ định danh duy nhất gọi là **NQN (NVMe Qualified Name)**.
* **Namespace:** Tương đương với LUN. Là một vùng nhớ logic liên tục chứa các khối dữ liệu (LBA). Một Subsystem có thể chứa nhiều Namespace được đánh số ID (`NSID 1`, `NSID 2`).
* **Multi-Queue Mapping:** Khi Initiator (SPDK) kết nối tới Subsystem, nó tạo ra một kênh điều khiển (**Admin Queue**) và **nhiều Queue I/O độc lập**. Mỗi thread worker của SPDK có riêng một Queue, triệt tiêu hoàn toàn sự cạnh tranh giữa các core CPU.

---

## 4. So sánh tầng vận chuyển: NVMe/TCP vs. NVMe/RDMA (RoCEv2)

NVMe-oF là **tầng giao thức lưu trữ (Storage Protocol)**. Nó cần một **tầng vận chuyển (Transport Layer)** bên dưới để đẩy dữ liệu đi. Hiện nay có hai phương án vận chuyển phổ biến nhất trên hạ tầng Ethernet:

```text
           +─────────────────────────────────────────────+
           |           NVMe-oF (Tầng lưu trữ)            |
           +─────────────────────────────────────────────+
                         │                 │
            ┌────────────┘                 └────────────┐
            ▼                                           ▼
+──────────────────────────+               +──────────────────────────+
|      NVMe over TCP       |               |      NVMe over RDMA      |
|  (Dùng socket TCP/IP)    |               |    (Dùng RoCEv2 / IB)    |
+──────────────────────────+               +──────────────────────────+
| - Chạy trên switch thường|               | - Cần SmartNIC / RoCEv2  |
| - Dễ triển khai          |               | - Bypass CPU & Kernel    |
| - Tốn CPU xử lý TCP      |               | - Cần cấu hình PFC / ECN |
+──────────────────────────+               +──────────────────────────+

```

### 1. NVMe over TCP: Lựa chọn thực tế, dễ tiếp cận

* **Nguyên lý:** Lệnh NVMe (64 bytes) và payload dữ liệu được bọc trực tiếp vào các TCP Segment tiêu chuẩn.
* **Ưu điểm:** Chạy được trên **100% switch mạng Ethernet tiêu chuẩn hiện có**, không yêu cầu cấu hình mạng đặc biệt, vượt qua được các router mạng IP (Routable).
* **Nhược điểm:** Vẫn chịu chi phí của giao thức TCP (TCP windowing, ACK, checksum), độ trễ cao hơn RDMA từ $10 - 20\ \mu s$, và tiêu tốn CPU Host hơn.

### 2. NVMe over RDMA (RoCEv2): Vũ khí hủy diệt độ trễ

* **Nguyên lý:** Sử dụng công nghệ **RDMA** bọc trong gói tin UDP (gọi là RoCEv2 - RDMA over Converged Ethernet v2). Card mạng (Mellanox ConnectX) tự đọc dữ liệu từ RAM của Compute Host và bắn thẳng sang RAM của NetApp Target.
* **Ưu điểm:**
* **Zero-Copy tuyệt đối:** CPU hoàn toàn không tham gia sao chép dữ liệu.
* **Kernel Bypass:** SPDK điều khiển trực tiếp phần cứng NIC từ User Space.
* **Độ trễ chạm đáy:** Độ trễ truyền dẫn qua mạng giảm xuống chỉ còn khoảng $2 - 5\ \mu s$.



### 3. Cái giá phải trả khi dùng RoCEv2 (Rủi ro kỹ thuật)

RDMA được thiết kế ban đầu cho môi trường không bao giờ rớt gói tin (InfiniBand). Khi đem chạy trên mạng Ethernet vốn có bản chất "cho phép rớt gói" (Best-effort), hệ thống bắt buộc phải xây dựng mạng **Lossless Ethernet**:

* **PFC (Priority-based Flow Control):** Cơ chế tạm dừng luồng dữ liệu ở tầng switch để tránh tràn buffer và mất gói tin. Nếu cấu hình sai, PFC có thể gây ra hiện tượng **PFC Deadlock** (toàn bộ switch bị treo cứng vòng lặp) làm sập toàn bộ mạng DC.
* **ECN (Explicit Congestion Notification):** Cơ chế đánh dấu cờ báo nghẽn mạng để hai đầu Host tự động hạ tốc độ truyền trước khi switch bị tràn bộ đệm.
* **Ràng buộc:** Cần sự phối hợp cực kỳ chặt chẽ giữa team System và team Network. **Không thể chỉ cắm card Mellanox vào là tự động có mạng RoCEv2 ổn định.**

---

## 5. Cơ chế đa đường (Multipathing): DM-Multipath vs. NVMe ANA

Trong môi trường sản xuất (Enterprise/Cloud), Compute Host luôn nối tới Target qua ít nhất **2 đường mạng vật lý độc lập** (2 cổng NIC, 2 dây quang, 2 switch tách biệt) để đảm bảo:

1. **High Availability (HA):** Khi 1 switch hoặc 1 cổng cáp bị đứt, I/O tự động chuyển sang đường còn lại mà VM không bị treo đơ (Failover).
2. **Load Balancing:** Phân bổ tải đọc/ghi trên nhiều cổng mạng để tăng băng thông tổng (Throughput).

```text
               +─────────────────────────────────────────────+
               |           COMPUTE HOST: /dev/vdb            |
               +─────────────────────────────────────────────+
                                │             │
                    ┌───────────┘             └───────────┐
                    ▼ (Path A: NIC 1)                     ▼ (Path B: NIC 2)
              [ Switch A ]                          [ Switch B ]
                    │                                     │
                    └───────────┐             ┌───────────┘
                                ▼             ▼
               +─────────────────────────────────────────────+
               |        NETAPP STORAGE CONTROLLER            |
               +─────────────────────────────────────────────+

```

### 1. Phương án truyền thống: Linux DM-Multipath (`multipathd`)

* Hoạt động ở tầng Kernel Block Layer của Linux thông qua module `dm-multipath`.
* Trực tiếp gom các block device đơn lẻ (`/dev/sdb`, `/dev/sdc`) thành một thiết bị ảo thống nhất (`/dev/mapper/mpatha`).
* Hoạt động theo cơ chế kiểm tra định kỳ (Path Checker ping SCSI định kỳ). Khi có sự cố, thời gian phát hiện và chuyển mạch (Failover time) thường mất từ **vài giây đến vài chục giây**, có thể làm nghẽn I/O tạm thời trong VM.

### 2. Phương án hiện đại: NVMe ANA (Asymmetric Namespace Access)

Trong các tủ đĩa All-Flash hiện đại, các Controller thường chạy theo mô hình cụm (Cluster). Một Namespace có thể được truy cập qua nhiều Controller, nhưng Controller đang trực tiếp sở hữu chip nhớ đó sẽ xử lý nhanh nhất.

Chuẩn NVMe định nghĩa cơ chế **ANA (Asymmetric Namespace Access)**, cho phép Target chủ động gắn nhãn trạng thái cho từng đường dẫn:

```text
Path A (Nối Controller 1) ──> Trạng thái: [ ANA OPTIMIZED ]      ──> Ưu tiên gửi I/O (Độ trễ thấp nhất)
Path B (Nối Controller 2) ──> Trạng thái: [ ANA NON-OPTIMIZED ]  ──> Đường dự phòng (Đi vòng qua liên kết cụm)
Path C (Controller chết)  ──> Trạng thái: [ ANA INACCESSIBLE ]    ──> Cấm gửi I/O, tự động cô lập ngay

```

* **Cơ chế cập nhật tức thời:** Target chủ động thông báo trạng thái mạng cho Initiator thông qua lệnh bất đồng bộ (Asynchronous Event Request - AER). Khi Controller 1 gặp sự cố, Target lập tức báo Host chuyển Path B thành `Optimized`. Quá trình failover diễn ra ở mức **mili-giây**, VM hoàn toàn không cảm nhận được sự gián đoạn.

### 3. Lưu ý sống còn về Multipath trong SPDK

* **SPDK KHÔNG dùng `multipathd` của nhân Linux!**
Vì SPDK chạy hoàn toàn ở User Space và bypass kernel, nó không thể sử dụng `/dev/mapper/mpathX`.
* **Cơ chế riêng của SPDK:** SPDK tự tích hợp thư viện **NVMe Multipath** nội bộ bên trong `bdev_nvme`. Khi bạn viết script JSON-RPC cho **Job 1**, bạn phải cấu hình tham số `bdev_nvme_set_options -n` để kích hoạt tính năng multipath nội bộ của SPDK, cho phép SPDK tự theo dõi các trạng thái ANA và tự động failover ngay trong User Space.

---

## 6. Ma trận 5 phương án Benchmark & Cạm bẫy đo lường

Để chứng minh giải pháp SPDK mang lại giá trị thật sự cho hội đồng kỹ thuật, bạn cần nắm vững ma trận 5 trạng thái kiến trúc:

```text
[BASELINE]                                                                     [MỤC TIÊU CUỐI]
Kernel iSCSI  ───>  Kernel NVMe/TCP  ───>  Kernel NVMe/RDMA  ───>  SPDK NVMe/TCP  ───>  SPDK NVMe/RDMA
   (P1)                    (P2)                   (P3)                 (P4)                  (P5)

```

| Cấu hình | Backend trên Compute | Giao thức mạng | Transport Layer | Mục đích kiểm thử (Cô lập biến số) |
| --- | --- | --- | --- | --- |
| **Phương án 1 (P1)** | QEMU / Kernel | iSCSI | TCP/IP | **Baseline hiện tại:** Đo hiệu năng hệ thống cũ để lấy mốc so sánh. |
| **Phương án 2 (P2)** | QEMU / Kernel | NVMe-oF | TCP/IP | **Đo tác động của Giao thức:** Giữ nguyên TCP, chuyển SCSI $\to$ NVMe xem nhanh hơn bao nhiêu. |
| **Phương án 3 (P3)** | QEMU / Kernel | NVMe-oF | RDMA (RoCEv2) | **Đo tác động của Phần cứng mạng:** Vẫn dùng Kernel nhưng đổi sang RDMA xem giảm trễ thế nào. |
| **Phương án 4 (P4)** | **SPDK vhost-user** | NVMe-oF | TCP/IP | **Đo tác động của SPDK thuần túy:** Bỏ qua Kernel trên Host nhưng vẫn chạy trên mạng TCP thường. |
| **Phương án 5 (P5)** | **SPDK vhost-user** | NVMe-oF | RDMA (RoCEv2) | **Sức mạnh tối đa:** Kết hợp User Space Polling + Mạng RDMA Zero-Copy. |

> [!CAUTION] Cạm bẫy khoa học khi làm báo cáo Benchmark!
> Nếu bạn chỉ đo **Phương án 1** rồi nhảy cóc thẳng sang **Phương án 5** và kết luận: *"SPDK giúp tăng IOPS gấp 4 lần và giảm 70% Tail Latency"*, kết luận đó **hoàn toàn thiếu cơ sở khoa học**.
> Ban giám khảo hoặc Tech Lead sẽ chất vấn ngay: *"Hiệu năng tăng là nhờ SPDK hay nhờ card mạng xịn Mellanox RoCEv2? Hay do đổi từ iSCSI sang NVMe?"*
> **Cách làm chuẩn mực:** Thực hiện các bước đo trung gian (P2, P3 hoặc P4). Điều này giúp chứng minh chính xác: đổi sang NVMe đóng góp bao nhiêu %, mạng RDMA đóng góp bao nhiêu %, và việc Kernel Bypass của SPDK đóng góp bao nhiêu %.

---

## 7. Giải đáp câu hỏi kiểm tra tư duy kiến trúc

> **Câu hỏi:** *Nếu Compute Host sử dụng SPDK NVMe/TCP để kết nối tới Storage Target, thì hệ thống đã sử dụng NVMe-oF chưa? Và hệ thống đã sử dụng RDMA chưa?*

### Phân tích và kết luận:

1. **Hệ thống ĐÃ sử dụng NVMe-oF:**
* **Chính xác 100%.** Bởi vì **NVMe-oF (NVMe over Fabrics)** là một chuẩn kiến trúc chung bao gồm nhiều kiểu transport: *NVMe over RDMA, NVMe over TCP, và NVMe over Fibre Channel (FC)*.
* Khi bạn chạy NVMe qua giao thức TCP, nó là phân hệ **NVMe/TCP**, hoàn toàn tuân theo đặc tả kỹ thuật chính thức của chuẩn NVMe-oF do tổ chức *NVM Express Organization* ban hành.


2. **Hệ thống CHƯA sử dụng RDMA:**
* **Chính xác 100%.** **TCP** và **RDMA** là hai tầng vận chuyển (Transport Layer) hoàn toàn khác nhau và loại trừ lẫn nhau trên một kết nối logic:
* Kết nối TCP đẩy dữ liệu qua các luồng byte stream tiêu chuẩn bằng gói tin TCP/IP, có chi phí tính toán checksum, windowing và quản lý gói tin bằng CPU.
* Kết nối RDMA (RoCEv2) sử dụng cơ chế phần cứng của SmartNIC để ghi thẳng vào bộ nhớ từ xa, không hề liên quan đến cơ chế socket của TCP.


* Dùng SPDK NVMe/TCP nghĩa là bạn đang tận dụng **tầng User-Space Polling của SPDK**, nhưng luồng dữ liệu mạng vật lý vẫn đang đi qua giao thức TCP thông thường.



---

## Tổng kết cốt lõi 80/20

1. **Phân định rõ vai trò:** Compute Host đóng vai trò **Initiator** (bên phát lệnh), còn NetApp SDS đóng vai trò **Target** (bên chứa dữ liệu). Guest VM hoàn toàn không biết giao thức mạng phía sau.
2. **iSCSI lỗi thời trước All-Flash:** Hàng đợi đơn, đóng gói PDU phức tạp và overhead TCP Kernel khiến iSCSI trở thành nút thắt cổ chai về độ trễ.
3. **NVMe-oF giải phóng băng thông:** Mang nguyên lý đa hàng đợi (Multi-Queue, Lockless) và tập lệnh 64-byte tinh gọn của NVMe PCIe truyền qua mạng vật lý.
4. **NVMe/TCP vs. NVMe/RDMA:**
* NVMe/TCP chạy trên switch thường, dễ triển khai, độ trễ vừa phải.
* NVMe/RDMA (RoCEv2) cho độ trễ cực thấp (micro-giây), zero-copy nhưng đòi hỏi switch mạng phải cấu hình Lossless Ethernet (PFC/ECN).


5. **Multipathing thế hệ mới:** NVMe ANA cho phép Target trực tiếp thông báo trạng thái tối ưu của đường truyền (Optimized / Non-Optimized), giúp failover ở cấp độ mili-giây.
6. **SPDK Multipath hoạt động độc lập:** SPDK tự quản lý đa đường ở User Space, không sử dụng `multipathd` hay Device Mapper của Linux Kernel.

```

```
