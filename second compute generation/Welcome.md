# Logic tổng thể của tài liệu

Cả tài liệu là một chuỗi lập luận: **hiện tại chậm ở đâu → loại bỏ từng lớp gây chậm → làm thế nào để tích hợp vào OpenStack → đo để chứng minh.**

---

## 1. Vấn đề: vì sao iSCSI + multipath "hết đà"

Chuỗi hiện tại (chính là lab bạn đang làm) như sau:

```
App trong VM → FS guest → virtio-blk → QEMU (giả lập đĩa) → Block layer kernel host
→ dm-multipath → iSCSI initiator → TCP/IP stack kernel → NIC → SAN/SDS
```

Mỗi lớp đều tốn: **copy dữ liệu, syscall, context switch, ngắt (interrupt), lock**. Từng cái nhỏ, cộng lại thành độ trễ lớn.

- **QEMU emulated block device**: QEMU giả lập một đĩa cho guest, mỗi I/O phải đi qua tiến trình QEMU.
- **iothread**: thread riêng của QEMU để xử lý I/O, tránh tranh với main loop. Tăng iothread bằng số vCPU là tối ưu tận cùng của hướng này, nhưng vẫn không xóa được các lớp kernel phía dưới.
- **Tail latency (P99, P99.99)**: P99 là mốc mà 99% request nhanh hơn nó, P99.99 là 1 trong 10.000 request chậm nhất. Ứng dụng như database bị ảnh hưởng bởi request chậm nhất chứ không phải trung bình. Đó là chỗ "thua metal vài chục %".

**Kết luận của tài liệu:** tối ưu tiếp trong khuôn khổ cũ không đủ, phải đổi **protocol** (iSCSI → NVMe-oF) và **datapath** (kernel → userspace, TCP → RDMA).

---

## 2. Các khối kiến thức nền

### 2.1 Protocol: SCSI → NVMe → NVMe-oF

- **SCSI** thiết kế cho đĩa quay: chỉ một hàng đợi lệnh, nhiều lớp dịch lệnh.
- **NVMe** thiết kế cho SSD: tới 64K hàng đợi, mỗi hàng sâu 64K lệnh, mỗi CPU core có queue riêng nên gần như không tranh lock, lệnh rất gọn.
- **NVMe-oF (over Fabrics)**: mang NVMe qua mạng thay vì PCIe. Thuật ngữ tương ứng:
    - **Subsystem (NQN)** ≈ iSCSI target
    - **Namespace** ≈ LUN
    - **Transport**: RDMA, TCP hoặc FC, tức "đường ống" chở lệnh NVMe.

### 2.2 Transport: TCP vs RDMA

- **NVMe/TCP**: chạy trên TCP/IP stack của kernel. Dễ triển khai (mạng Ethernet thường) nhưng tốn CPU để xử lý TCP.
- **RDMA (Remote Direct Memory Access)**: NIC đọc/ghi thẳng vào bộ nhớ máy bên kia, **bypass kernel** và **zero-copy**, CPU gần như không tham gia.
- **RoCEv2**: RDMA chạy trên Ethernet/UDP/IP (khác InfiniBand cần hạ tầng riêng).
- **PFC + ECN**: RDMA rất nhạy với mất gói, nên mạng phải **lossless**. PFC (Priority Flow Control) yêu cầu bên gửi tạm dừng theo từng lớp ưu tiên khi switch sắp đầy. ECN (Explicit Congestion Notification) báo tắc nghẽn sớm để bên gửi giảm tốc trước khi phải drop gói. Vì thế Job 2 phải nhờ team mạng (CSO) audit cấu hình switch.
- **libibverbs / rdma-core**: thư viện userspace để nói chuyện trực tiếp với NIC RDMA. **WQE (Work Queue Entry)** là một "phiếu yêu cầu" gửi vào hàng đợi của NIC.
- **Mellanox ConnectX-6 Lx**: card mạng có ASIC hỗ trợ RDMA và offload, thay cho HBA/NIC thường.

### 2.3 Kernel path vs Userspace path

||Kernel (nvme-rdma / nvme-tcp)|Userspace (SPDK)|
|---|---|---|
|Cách nhận việc|**Interrupt**: có việc thì ngắt CPU|**Polling**: một core quay vòng hỏi liên tục|
|Chi phí|Context switch, syscall|Không có, nhưng đốt 100% một core|
|Quản lý tài nguyên|Linh hoạt, dùng chung|Phải dành riêng core|
|Ưu điểm|Đơn giản, tiết kiệm khi rảnh|Độ trễ thấp, ổn định, IOPS/core cao|

Polling dễ hiểu qua ví dụ: interrupt như chuông cửa (đợi người bấm rồi mới ra), polling như đứng canh sẵn ở cửa. Canh sẵn thì phản ứng nhanh nhất nhưng người đó không làm được việc khác.

### 2.4 SPDK

Bộ thư viện userspace của Intel để làm storage tốc độ cao:

- **PMD (Polling Mode Driver)**: driver chạy ở userspace, tự poll thiết bị.
- **Lockless queue**: hàng đợi không cần lock giữa các thread.
- **spdk_tgt**: tiến trình chính (target application).
- **bdev (block device)**: lớp trừu tượng của SPDK. Mọi thứ (NVMe-oF remote, local NVMe, file...) đều thành bdev, và bdev có thể gắn tiếp vào nơi khác.
- **bdev_nvme**: module tạo bdev từ NVMe/NVMe-oF, ở đây là nối tới NetApp qua RDMA.
- **RPC (`rpc.py`)**: cách điều khiển SPDK lúc chạy (tạo bdev, tạo controller...). Đây là "API" mà os-brick sẽ gọi.

### 2.5 Hugepages, isolcpus, NUMA

Ba thứ này phục vụ chung một mục tiêu: **I/O đi nhanh và không bị ai làm phiền**.

- **Hugepages (2MB/1GB)**: trang nhớ lớn, được pin cố định trong RAM (không swap, không đổi địa chỉ vật lý). Điều này cần thiết để NIC và SPDK có thể DMA trực tiếp, đồng thời giảm miss của TLB.
- **isolcpus**: tham số boot để kernel không xếp process thường lên các core này, dành riêng cho SPDK polling thread. Verify bằng `cat /sys/devices/system/cpu/isolated`.
- **NUMA**: máy nhiều socket, mỗi socket có RAM và PCIe riêng. Truy cập "chéo socket" chậm hơn. Vì thế vCPU của VM, hugepages, SPDK thread và NIC RDMA phải **cùng một NUMA node**.

### 2.6 vhost-user-blk (mắt xích quan trọng nhất)

Đây là giao thức cho phép **QEMU giao phần xử lý I/O của đĩa cho một tiến trình khác (SPDK)**:

- **Control path**: qua **Unix socket** (ví dụ `/var/tmp/vhost.0`), dùng để thương lượng, chỉ chạy lúc thiết lập.
- **Data path**: qua **shared memory**. RAM của guest được map chung sang SPDK (bắt buộc dùng hugepages với `share=on`), nên SPDK đọc thẳng virtqueue và dữ liệu trong RAM guest, **zero-copy**.
- **QEMU không còn nằm trên đường đi của I/O**, nó chỉ setup rồi đứng ngoài.

Trong guest vẫn là `virtio-blk` bình thường, guest không biết gì khác. Bên host thì đổi từ `virtio-blk-pci` (QEMU xử lý) sang `vhost-user-blk-pci` (SPDK xử lý).

Trong libvirt XML, cần: `<memoryBacking>` với hugepages và `<access mode='shared'/>`, NUMA cell có `memAccess='shared'`, và disk kiểu `vhostuser` trỏ tới socket (hoặc `<qemu:commandline>` khi libvirt chưa hỗ trợ đủ).

### 2.7 Multipath / HA

- **Kernel**: native NVMe multipath (`nvme_core.multipath=Y`) thay cho dm-multipath, dựa vào **ANA (Asymmetric Namespace Access)**. Mỗi đường được báo trạng thái _Optimized_ hoặc _Non-optimized_, và host ưu tiên đường tối ưu (tương tự ALUA của SCSI, bạn từng thấy `ALUA state: Active/optimized` trong targetcli).
- **SPDK**: `bdev_nvme` tự có multipath/failover. Vì đã bypass kernel nên native multipath của kernel không có tác dụng, phải cấu hình lại ở tầng SPDK.

---

## 3. Data plane: một I/O đi từ VM tới NetApp

1. App ghi file, qua FS guest, driver **virtio-blk** đẩy request vào **virtqueue** (ring buffer).
2. **SPDK polling thread** trên host thấy request ngay trong shared memory (không có exit, không có syscall).
3. Request chuyển qua **bdev layer**, từ "vhost block request" thành "bdev I/O".
4. **NVMe-oF RDMA initiator (userspace)** đóng gói thành lệnh NVMe rồi thành **WQE**, đẩy thẳng xuống NIC qua libibverbs, không qua network stack kernel.
5. NIC gửi qua **RoCEv2**, NetApp target nhận, ghi xuống SSD rồi trả ACK về theo đường ngược lại.

So với chuỗi iSCSI ở mục 1, các lớp QEMU emulation, kernel block layer, dm-multipath, iSCSI, TCP/IP stack đều biến mất.

---

## 4. Control plane: OpenStack gắn đĩa như thế nào

Đây là phần **phải viết code** vì OpenStack mặc định giả định đường kernel:

- **Luồng bình thường**: Cinder gọi API NetApp tạo volume, export NVMe subsystem/namespace. Nova gọi **os-brick** (thư viện "connector" chuyên kết nối storage vào host) chạy `nvme connect` ở kernel, rồi đĩa `/dev/nvmeXnY` được đưa vào VM.
- **Luồng custom**: os-brick (hoặc Cinder driver) **không** `nvme connect`, mà gọi **SPDK RPC** để (a) connect NVMe-oF tới NetApp trong tiến trình SPDK, (b) tạo bdev, (c) tạo vhost-user-blk controller và socket. Đường dẫn socket được trả cho Nova/libvirt để đưa vào XML của VM.
- **Cinder SPDK driver** cộng đồng đã có nhưng chưa hoàn thiện với vhost-user-blk trên OpenStack mới, nên cần đánh giá và có thể tự viết connector.

Cinder/Nova/os-brick cũng phải xử lý các thao tác sống còn khác (detach, resize, snapshot, live migration), và đó là phần khó nhất của Job 1.

---

## 5. Benchmark (Mảng 2, Job 2)

Ba mô hình so sánh: **Kernel TCP**, **Kernel RDMA**, **SPDK RDMA**. Nhờ vậy tách được lợi ích của từng thay đổi: đổi transport (TCP→RDMA) giúp bao nhiêu, và đổi datapath (kernel→SPDK) giúp thêm bao nhiêu.

- **fio engine**: `libaio`/`io_uring` cho đường kernel, `spdk` cho userspace.
- **Metric**: IOPS, throughput (MB/s), latency (đặc biệt P99/P99.99), và **IOPS/core** (hiệu quả CPU). Chỉ số cuối quan trọng vì SPDK đốt 100% core polling, nên phải xem có "đáng" không.

---

## 6. Phân việc theo logic dựa trên phụ thuộc

- **Job p**: chuẩn bị hạ tầng, gồm host cô lập để PoC và card Mellanox CX6-Lx.
- **Job 0**: dựng **thủ công từng bước** để chứng minh đường đi hoạt động: hugepages/isolcpus → spdk_tgt kết nối NetApp → tạo vhost controller/socket → chạy QEMU tay → fio baseline. Mỗi bước có lệnh verify riêng, đây là PoC để biết chạy được, chưa cần tích hợp OpenStack.
- **Job 1**: **tự động hóa** những gì Job 0 làm thủ công, tức tích hợp Cinder/os-brick/libvirt.
- **Job 2**: **hạ tầng mạng và đo đạc** (RoCEv2 lossless, 3 mô hình, báo cáo).

Thứ tự tự nhiên là p → 0 → (1 và 2 song song).

---

## 7. Vài lưu ý để bạn nhìn cho chính xác

- Sơ đồ có vẻ gộp "Kernel Path" cùng nhánh với vhost-user-blk, nhưng trên thực tế vhost-user-blk cần một tiến trình userspace phục vụ (SPDK). Đường kernel thường đi theo kiểu khác: `nvme connect` rồi đưa `/dev/nvmeXnY` cho QEMU dùng virtio-blk thường (chính là baseline để so sánh).
- "Posix/sockeye" trong tài liệu chắc là gõ nhầm "posix sockets".
- Cái giá phải trả: mỗi core SPDK bị chiếm 100%, hugepages bị pin cố định, mất một số tính năng tiện lợi của kernel (snapshot, live migration với vhost-user cần thêm công sức), và RoCEv2 đòi hỏi mạng cấu hình đúng. Vì vậy báo cáo cuối cần xét cả **lợi ích latency lẫn chi phí vận hành**.

Trong lab của bạn, chuỗi iSCSI + LIO + virtio-blk chính là **baseline "trước"** của bài toán này. Nếu muốn, mình đi sâu tiếp phần nào (ví dụ libvirt XML cho vhost-user-blk, hoặc lệnh RPC của SPDK theo từng bước Job 0) đều được.