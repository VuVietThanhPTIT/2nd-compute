<!-- ORIGINAL 0 BEGIN -->
## Mục lục

- [Kiến trúc phần cứng](#kiến-trúc-phần-cứng)
  - [1. Bản đồ Topo Phần cứng (Physical Hardware Hierarchy)](#1-bản-đồ-topo-phần-cứng-physical-hardware-hierarchy)
  - [2. Vi mô Uncore & Bus Fabric: Nơi I/O giao thoa với Bộ nhớ](#2-vi-mô-uncore--bus-fabric-nơi-io-giao-thoa-với-bộ-nhớ)
    - [A. Duy trì Nhất quán Bộ nhớ (Cache Coherency) trong DMA](#a-duy-trì-nhất-quán-bộ-nhớ-cache-coherency-trong-dma)
    - [B. Động cơ IOMMU (Intel VT-d / AMD-Vi) ở cấp phần cứng](#b-động-cơ-iommu-intel-vt-d--amd-vi-ở-cấp-phần-cứng)
    - [C. Integrated Memory Controller (IMC) & DRAM Bus](#c-integrated-memory-controller-imc--dram-bus)
  - [3. Giao thức PCIe ở Mức Vi phân (Từ TLP đến Tín hiệu Điện)](#3-giao-thức-pcie-ở-mức-vi-phân-từ-tlp-đến-tín-hiệu-điện)
  - [4. Vi kiến trúc của Bộ điều khiển SSD (NVMe Controller ASIC)](#4-vi-kiến-trúc-của-bộ-điều-khiển-ssd-nvme-controller-asic)
  - [5. Tầng Vật lý Vi mô: Tế bào nhớ NAND Flash (Điện tử & Cơ học Lượng tử)](#5-tầng-vật-lý-vi-mô-tế-bào-nhớ-nand-flash-điện-tử--cơ-học-lượng-tử)
    - [A. Hiện tượng Hầm phiếu Lượng tử (Fowler-Nordheim Tunneling)](#a-hiện-tượng-hầm-phiếu-lượng-tử-fowler-nordheim-tunneling)
    - [B. Dịch chuyển Điện áp Ngưỡng (Vth - Threshold Voltage)](#b-dịch-chuyển-điện-áp-ngưỡng-vth---threshold-voltage)
    - [C. Quy trình Ghi xung Từng bước (ISPP - Incremental Step Pulse Programming)](#c-quy-trình-ghi-xung-từng-bước-ispp---incremental-step-pulse-programming)
  - [6. Thước đo Độ trễ Toàn cảnh: Kim tự tháp Thời gian (Latency Scale)](#6-thước-đo-độ-trễ-toàn-cảnh-kim-tự-tháp-thời-gian-latency-scale)
  - [Chi tiết logic và vai trò từng giai đoạn](#chi-tiết-logic-và-vai-trò-từng-giai-đoạn)
    - [1. Khởi tạo từ Userspace và Chuyển đổi Đặc quyền](#1-khởi-tạo-từ-userspace-và-chuyển-đổi-đặc-quyền)
    - [2. Tầng VFS và Điểm đệm Page Cache (Lần sao chép thứ nhất)](#2-tầng-vfs-và-điểm-đệm-page-cache-lần-sao-chép-thứ-nhất)
    - [3. Bộ phận Ghi lùi Bất đồng bộ (Writeback Subsystem)](#3-bộ-phận-ghi-lùi-bất-đồng-bộ-writeback-subsystem)
    - [4. Phân giải Hệ thống tệp (Filesystem - ext4 / XFS)](#4-phân-giải-hệ-thống-tệp-filesystem---ext4--xfs)
    - [5. Tầng Khối Đa hàng đợi (Generic Block Layer - blk-mq)](#5-tầng-khối-đa-hàng-đợi-generic-block-layer---blk-mq)
    - [6. Trình điều khiển Thiết bị (Device Driver: nvme / virtio-blk)](#6-trình-điều-khiển-thiết-bị-device-driver-nvme--virtio-blk)
    - [7. Động cơ DMA và Tuyến truyền thông Liên kết (Bus Interconnect)](#7-động-cơ-dma-và-tuyến-truyền-thông-liên-kết-bus-interconnect)
    - [8. Thiết bị Lưu trữ Vật lý (SSD NVMe Controller & NAND Flash)](#8-thiết-bị-lưu-trữ-vật-lý-ssd-nvme-controller--nand-flash)
    - [9. Đường hồi tiếp Hoàn tất qua Ngắt Phần cứng (Interrupt Loop)](#9-đường-hồi-tiếp-hoàn-tất-qua-ngắt-phần-cứng-interrupt-loop)
  - [Bảng đối chiếu thực thể: Phần mềm Logic vs. Phần cứng Vật lý](#bảng-đối-chiếu-thực-thể-phần-mềm-logic-vs-phần-cứng-vật-lý)

- [Các hàm hệ thống](#các-hàm-hệ-thống)
  - [GIAI ĐOẠN 1: GUEST OS — TỪ SYSTEM CALL ĐẾN VIRTQUEUE](#giai-đoạn-1-guest-os--từ-system-call-đến-virtqueue)
    - [1. sys_write() / ksys_write()](#1-sys_write--ksys_write)
    - [2. vfs_write() → blkdev_write_iter()](#2-vfs_write--blkdev_write_iter)
    - [3. blk_mq_submit_bio()](#3-blk_mq_submit_bio)
    - [4. virtio_queue_rq()](#4-virtio_queue_rq)
    - [5. virtqueue_add_outbuf() / virtqueue_add()](#5-virtqueue_add_outbuf--virtqueue_add)
    - [6. virtqueue_notify() / iowrite16()](#6-virtqueue_notify--iowrite16)
  - [GIAI ĐOẠN 2: RANH GIỚI ẢO HÓA — BẪY PHẦN CỨNG VM-EXIT & KVM](#giai-đoạn-2-ranh-giới-ảo-hóa--bẫy-phần-cứng-vm-exit--kvm)
    - [7. Phần cứng CPU: Kích hoạt VM-Exit](#7-phần-cứng-cpu-kích-hoạt-vm-exit)
    - [8. vmx_handle_exit() → handle_ept_misconfig()](#8-vmx_handle_exit--handle_ept_misconfig)
    - [9. kvm_io_bus_write() → ioeventfd_write()](#9-kvm_io_bus_write--ioeventfd_write)
    - [10. eventfd_signal() & Hoàn tất Fast Path](#10-eventfd_signal--hoàn-tất-fast-path)
  - [GIAI ĐOẠN 3: HOST USERSPACE — QEMU IOTHREAD & DỊCH ĐỊA CHỈ](#giai-đoạn-3-host-userspace--qemu-iothread--dịch-địa-chỉ)
    - [11. epoll_wait() thức dậy](#11-epoll_wait-thức-dậy)
    - [12. virtio_blk_data_plane_handle_output()](#12-virtio_blk_data_plane_handle_output)
    - [13. virtqueue_pop() → address_space_map()](#13-virtqueue_pop--address_space_map)
    - [14. laio_do_submit() → io_submit()](#14-laio_do_submit--io_submit)
  - [GIAI ĐOẠN 4: HOST KERNEL STORAGE STACK (blk-mq, dm-multipath, SCSI)](#giai-đoạn-4-host-kernel-storage-stack-blk-mq-dm-multipath-scsi)
    - [15. __x64_sys_io_submit() → io_submit_one()](#15-__x64_sys_io_submit--io_submit_one)
    - [16. blkdev_write_iter() → iov_iter_get_pages2()](#16-blkdev_write_iter--iov_iter_get_pages2)
    - [17. dm_submit_bio() → multipath_clone_and_map()](#17-dm_submit_bio--multipath_clone_and_map)
    - [18. scsi_queue_rq() → sd_init_command()](#18-scsi_queue_rq--sd_init_command)
  - [GIAI ĐOẠN 5: GIAO THỨC iSCSI & CARD MẠNG NIC DMA ĐẨY RA MẠNG](#giai-đoạn-5-giao-thức-iscsi--card-mạng-nic-dma-đẩy-ra-mạng)
    - [19. iscsi_queuecommand() → iscsi_prep_scsi_cmd_pdu()](#19-iscsi_queuecommand--iscsi_prep_scsi_cmd_pdu)
    - [20. kernel_sendpage() / sock_sendmsg() → tcp_write_xmit()](#20-kernel_sendpage--sock_sendmsg--tcp_write_xmit)
    - [21. dma_map_page() (IOMMU Translation)](#21-dma_map_page-iommu-translation)
    - [22. nic_start_xmit() → writel() (Kích hoạt NIC Hardware)](#22-nic_start_xmit--writel-kích-hoạt-nic-hardware)
    - [23. Phần cứng: NIC Scatter-Gather DMA Fetch & Serialization](#23-phần-cứng-nic-scatter-gather-dma-fetch--serialization)
  - [GIAI ĐOẠN 6: TARGET LƯU TRỮ VÀ HOÀN TẤT TRUYỀN DẪN VỀ HOST](#giai-đoạn-6-target-lưu-trữ-và-hoàn-tất-truyền-dẫn-về-host)
    - [24. Xử lý tại Storage Target (NetApp SDS / Flash Array)](#24-xử-lý-tại-storage-target-netapp-sds--flash-array)
    - [25. Phần cứng: Tín hiệu ngắt MSI-X Interrupt](#25-phần-cứng-tín-hiệu-ngắt-msi-x-interrupt)
    - [26. iscsi_tcp_data_recv() → scsi_finish_command()](#26-iscsi_tcp_data_recv--scsi_finish_command)
    - [27. aio_complete() → eventfd_signal()](#27-aio_complete--eventfd_signal)
  - [GIAI ĐOẠN 7: QEMU CẬP NHẬT VIRTQUEUE VÀ TIÊM NGẮT ẢO VÀO VM](#giai-đoạn-7-qemu-cập-nhật-virtqueue-và-tiêm-ngắt-ảo-vào-vm)
    - [28. io_getevents() → virtio_blk_req_complete()](#28-io_getevents--virtio_blk_req_complete)
    - [29. Báo hiệu qua irqfd: eventfd_write()](#29-báo-hiệu-qua-irqfd-eventfd_write)
    - [30. kvm_set_irq() & Tiêm ngắt ảo (Virtual IRQ Injection)](#30-kvm_set_irq--tiêm-ngắt-ảo-virtual-irq-injection)
    - [31. Guest Kernel: virtio_blk_done() → Đánh thức ứng dụng](#31-guest-kernel-virtio_blk_done--đánh-thức-ứng-dụng)
  - [BẢNG ĐỐI CHIẾU CHUỖI BIẾN ĐỔI BỘ NHỚ VÀ ĐỊA CHỈ TRUY CẬP](#bảng-đối-chiếu-chuỗi-biến-đổi-bộ-nhớ-và-địa-chỉ-truy-cập)

- [CÁC THIẾT BỊ ẢO & GIAO DIỆN LOGIC XUẤT HIỆN TRONG DATAPATH](#các-thiết-bị-ảo--giao-diện-logic-xuất-hiện-trong-datapath)
  - [TẦNG 1: BÊN TRONG GUEST VM (THIẾT BỊ ẢO MỨC HỆ ĐIỀU HÀNH KHÁCH)](#tầng-1-bên-trong-guest-vm-thiết-bị-ảo-mức-hệ-điều-hành-khách)
    - [1. Thiết bị khối ảo: /dev/vdb (Guest Virtual Block Device)](#1-thiết-bị-khối-ảo-devvdb-guest-virtual-block-device)
    - [2. Thiết bị PCI ảo: virtio-blk-pci (Emulated PCI Endpoint)](#2-thiết-bị-pci-ảo-virtio-blk-pci-emulated-pci-endpoint)
    - [3. Bộ điều khiển ngắt ảo: vAPIC (Virtual Local APIC)](#3-bộ-điều-khiển-ngắt-ảo-vapic-virtual-local-apic)
  - [TẦNG 2: RANH GIỚI HYPERVISOR & HOST USERSPACE (GIAO DIỆN ĐIỀU PHỐI)](#tầng-2-ranh-giới-hypervisor--host-userspace-giao-diện-điều-phối)
    - [4. Giao diện sự kiện nhanh: ioeventfd](#4-giao-diện-sự-kiện-nhanh-ioeventfd)
    - [5. Khối điều khiển giả lập: QEMU virtio-blk Emulated Controller](#5-khối-điều-khiển-giả-lập-qemu-virtio-blk-emulated-controller)
    - [6. Giao diện tiêm ngắt: irqfd](#6-giao-diện-tiêm-ngắt-irqfd)
  - [TẦNG 3: HOST KERNEL STORAGE STACK (THIẾT BỊ LƯU TRỮ LOGIC TRÊN HOST)](#tầng-3-host-kernel-storage-stack-thiết-bị-lưu-trữ-logic-trên-host)
    - [7. Thiết bị Multipath ảo: /dev/mapper/mpath0 (hay dm-0)](#7-thiết-bị-multipath-ảo-devmappermpath0-hay-dm-0)
    - [8. Bộ điều khiển HBA ảo: scsi_hostX](#8-bộ-điều-khiển-hba-ảo-scsi_hostx)
    - [9. Thiết bị đĩa SCSI logic: /dev/sda và /dev/sdb](#9-thiết-bị-đĩa-scsi-logic-devsda-và-devsdb)
    - [10. Giao diện mạng Socket ảo: Kernel struct socket](#10-giao-diện-mạng-socket-ảo-kernel-struct-socket)
  - [TẦNG 4: STORAGE FABRIC & TARGET (THỰC THỂ LOGIC TẠI TỦ ĐĨA)](#tầng-4-storage-fabric--target-thực-thể-logic-tại-tủ-đĩa)
    - [11. Cụm định danh logic Target: IQN, TPG và LUN 0](#11-cụm-định-danh-logic-target-iqn-tpg-và-lun-0)
  - [BẢNG ĐỐI CHIẾU: THIẾT BỊ ẢO VS PHẦN CỨNG THẬT TƯƠNG ỨNG](#bảng-đối-chiếu-thiết-bị-ảo-vs-phần-cứng-thật-tương-ứng)
# Kiến trúc phần cứng

<!-- ORIGINAL 0 END -->

<!-- ORIGINAL 1 BEGIN -->

Kiến trúc máy tính ở luồng I/O được phân tách từ mức cổng logic, các bus kết nối trên đế bán dẫn (silicon), cho tới sự dịch chuyển của điện tích trong tế bào nhớ flash theo cấu trúc từ lõi CPU xuống đến mức vật lý vi mô.

<!-- ORIGINAL 1 END -->

<!-- ORIGINAL 3 BEGIN -->

## 1\. Bản đồ Topo Phần cứng (Physical Hardware Hierarchy)

<!-- ORIGINAL 3 END -->

<!-- ORIGINAL 5-50 BEGIN -->

```text
+══════════════════════════════════════════════════════════════════════════════════════════════════════════════+
║                                  CPU DIE (SOCKET / SYSTEM ON CHIP)                                           ║
║                                                                                                              ║
║   ┌────────────────────────┐  ┌────────────────────────┐                                                     ║
║   │  CPU CORE 0            │  │  CPU CORE 1            │                                                     ║
║   │  - ALU / FPU           │  │  - ALU / FPU           │                                                     ║
║   │  - L1 Data (32-48KB)   │  │  - L1 Data             │                                                     ║
║   │  - L2 Cache (1-2MB)    │  │  - L2 Cache            │                                                     ║
║   └───────────┬────────────┘  └───────────┬────────────┘                                                     ║
║               │                           │                                                                  ║
║   ════════════╧═══════════════════════════╧══════════════════════════════════════ (Coherent On-Chip Interconnect)║
║                  MESH / RING BUS / CROSSBAR (Chạy giao thức Snooping / Directory)                            ║
║   ════════════╤═══════════════════════════════════════════════╤══════════════════════════════════════════════║
║               │                                               │                                              ║
║   ┌───────────┴────────────┐                     ┌────────────┴──────────────────────────────────────────┐   ║
║   │ LAST LEVEL CACHE (LLC) │                     │ UNCORE / SYSTEM AGENT                                 │   ║
║   │ Shared L3 Cache        │                     │                                                       │   ║
║   └────────────────────────┘                     │  ┌─────────────────────────┐ ┌──────────────────────┐ │   ║
║                                                  │  │ Integrated Memory       │ │ PCIe ROOT COMPLEX    │ │   ║
║                                                  │  │ Controller (IMC)        │ │                      │ │   ║
║                                                  │  │ - Command/Address Queues│ │ - Root Port Registers│ │   ║
║                                                  │  │ - Arbiter & PHY         │ │ - IOMMU Engine       │ │   ║
║                                                  │  └────────────┬────────────┘ └──────────┬───────────┘ │   ║
║                                                  └───────────────┼─────────────────────────┼─────────────┘   ║
+══════════════════════════════════════════════════════════════════┼═════════════════════════┼═════════════════+
                                                                   │ DDR5 Bus (128-bit)      │ PCIe Lanes (x4)
                                                                   ▼                         ▼
                                                  +─────────────────────────+   +─────────────────────────────+
                                                  │ HOST DRAM CHIPS         │   │ SSD CONTROLLER (ASIC)       │
                                                  │ (DRAM Cells: 1T-1C)     │   │                             │
                                                  │ - Row Buffer / Sense Amp│   │ - PCIe PHY / SerDes         │
                                                  │ - Chứa Page Cache Frames│   │ - DMA Controller Engine     │
                                                  +─────────────────────────+   │ - Multi-Core ARM/RISC-V     │
                                                                                │ - Controller SRAM / DRAM    │
                                                                                │ - Flash Memory Controller   │
                                                                                │ - Hardware LDPC Engine      │
                                                                                +──────────────┬──────────────+
                                                                                               │ Open NAND Flash
                                                                                               │ Interface (ONFi)
                                                                                               ▼
                                                                                +─────────────────────────────+
                                                                                │ NAND FLASH CHIP (DIE)       │
                                                                                │ - Wordlines (WL) / Bitlines │
                                                                                │ - Charge Trap Nitride Layer │
                                                                                │ - High Voltage Charge Pumps │
                                                                                +─────────────────────────────+
```

<!-- ORIGINAL 5-50 END -->

<!-- ORIGINAL 52 BEGIN -->

## 2\. Vi mô Uncore & Bus Fabric: Nơi I/O giao thoa với Bộ nhớ

<!-- ORIGINAL 52 END -->

<!-- ORIGINAL 53 BEGIN -->

Khi thiết bị I/O đọc/ghi dữ liệu từ máy chủ mà không qua CPU (DMA), nó phải đi xuyên qua cấu trúc liên kết vi mô của CPU:

<!-- ORIGINAL 53 END -->

<!-- ORIGINAL 55 BEGIN -->

### A. Duy trì Nhất quán Bộ nhớ (Cache Coherency) trong DMA

<!-- ORIGINAL 55 END -->

<!-- ORIGINAL 56 BEGIN -->

- Khi SSD DMA đọc một khung trang trên RAM máy chủ, dữ liệu mới nhất có thể chưa được ghi xuống DRAM mà đang nằm trên L1/L2/L3 Cache của Core 0 (ở trạng thái Modified theo giao thức MESI/MOESI).

<!-- ORIGINAL 56 END -->

<!-- ORIGINAL 58 BEGIN -->

- **Phần cứng xử lý:** Bộ điều khiển PCIe Root Complex gửi yêu cầu thăm dò (Snoop Request) lên mạng lưới liên kết on-chip (Ring/Mesh Bus).

<!-- ORIGINAL 58 END -->

<!-- ORIGINAL 60 BEGIN -->

- Bộ lọc Snoop Filter (nằm cạnh LLC) kiểm tra xem địa chỉ này có nằm trong cache của lõi nào không.

<!-- ORIGINAL 60 END -->

<!-- ORIGINAL 62 BEGIN -->

- Nếu Core 0 giữ trang bẩn, bộ điều khiển cache của Core 0 bị ép xả dòng cache đó ra bus để phục vụ giao dịch DMA, hoặc can thiệp trực tiếp cung cấp dữ liệu cho PCIe Controller mà không cần qua DRAM.

<!-- ORIGINAL 62 END -->

<!-- ORIGINAL 64 BEGIN -->

### B. Động cơ IOMMU (Intel VT-d / AMD-Vi) ở cấp phần cứng

<!-- ORIGINAL 64 END -->

<!-- ORIGINAL 65 BEGIN -->

IOMMU là một MMU dành riêng cho thiết bị ngoại vi, thực thi dịch địa chỉ hoàn toàn bằng phần cứng:

<!-- ORIGINAL 65 END -->

<!-- ORIGINAL 67 BEGIN -->

- Thiết bị SSD gửi một gói tin có địa chỉ bộ nhớ I/O ảo (IOVA - I/O Virtual Address).

<!-- ORIGINAL 67 END -->

<!-- ORIGINAL 69 BEGIN -->

- **IOTLB (I/O Translation Lookaside Buffer):** Một mảng bộ nhớ cache SRAM cực nhanh nằm trên PCIe Root Port kiểm tra xem cặp ánh xạ IOVA → HPA (Host Physical Address) đã có sẵn chưa.

<!-- ORIGINAL 69 END -->

<!-- ORIGINAL 71 BEGIN -->

- **Hardware Page Walk:** Nếu IOTLB Miss, phần cứng IOMMU tự động phát sinh các chu kỳ đọc bộ nhớ tới cây bảng phân trang của IOMMU nằm trên RAM (Context Tables → PASID Tables → 4-Level Page Table) để tìm địa chỉ vật lý thực tế, bảo vệ RAM máy chủ khỏi thiết bị ghi đè trái phép.

<!-- ORIGINAL 71 END -->

<!-- ORIGINAL 73 BEGIN -->

### C. Integrated Memory Controller (IMC) & DRAM Bus

<!-- ORIGINAL 73 END -->

<!-- ORIGINAL 74 BEGIN -->

Khi địa chỉ vật lý đến được IMC:

<!-- ORIGINAL 74 END -->

<!-- ORIGINAL 76 BEGIN -->

- **IMC chia địa chỉ thành:** Channel -\> DIMM -\> Rank -\> Bank Group -\> Bank -\> Row -\> Column.

<!-- ORIGINAL 76 END -->

<!-- ORIGINAL 78 BEGIN -->

- **Kích hoạt hàng (ACTIVATE - tRCD):** Một điện thế được nạp vào đường chọn hàng (Wordline) của DRAM chip. Hàng triệu tụ điện nhỏ (tụ 1-Transistor 1-Capacitor: 1T-1C) xả điện tích yếu ớt ra các đường cột (Bitlines).

<!-- ORIGINAL 78 END -->

<!-- ORIGINAL 80 BEGIN -->

- **Bộ khuếch đại cảm ứng (Sense Amplifiers):** Đọc chênh lệch điện áp vài chục milivolt, khuếch đại thành mức logic 0/1 và nạp vào Row Buffer (SRAM đệm trên chip DRAM).

<!-- ORIGINAL 80 END -->

<!-- ORIGINAL 82 BEGIN -->

- **Lệnh đọc/ghi cột (READ/WRITE - tCL / tCAS):** IMC truyền dữ liệu qua các chân tín hiệu DQ theo xung nhịp đôi (DDR), chuyển khối dữ liệu 64-byte (Cacheline) vào bộ đệm của PCIe Root Complex.

<!-- ORIGINAL 82 END -->

<!-- ORIGINAL 84 BEGIN -->

## 3\. Giao thức PCIe ở Mức Vi phân (Từ TLP đến Tín hiệu Điện)

<!-- ORIGINAL 84 END -->

<!-- ORIGINAL 85 BEGIN -->

Dữ liệu không chạy dưới dạng "byte" đơn thuần trên bo mạch chủ; nó được băm nhỏ thành gói tin và biến đổi thành dạng sóng vi sai tần số cực cao.

<!-- ORIGINAL 85 END -->

<!-- ORIGINAL 88-114 BEGIN -->

```text
┌────────────────────────────────────────────────────────────────────────┐
│ 1. TRANSACTION LAYER (TLP)                                             │
│    [ Header (12-16B) ] [ Data Payload (4KB max) ] [ TLP Digest/ECRC ]   │
│    (Định nghĩa: Ghi bộ nhớ Memory Write, địa chỉ đích 64-bit)          │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
┌───────────────────────────────────▼────────────────────────────────────┐
│ 2. DATA LINK LAYER (DLLP & Phối hợp độ tin cậy)                        │
│    [ Seq Num (12-bit) ] [ TLP ] [ 32-bit LCRC ]                        │
│    - Cơ chế Credit-Based Flow Control (Cấp hạn ngạch phát/nhận)        │
│    - Replay Buffer: Lưu bản sao; nếu LCRC lỗi -> Phát NAK và truyền lại│
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
┌───────────────────────────────────▼────────────────────────────────────┐
│ 3. LOGICAL PHYSICAL LAYER (Mã hóa và Đóng khung)                       │
│    - Mã hóa 128b/130b (PCIe Gen 3/4/5): 2-bit Sync Header + 128-bit    │
│    - Scrambler LFSR: Trộn bit chống cộng hưởng tần số điện từ (EMI)    │
│    - Striping: Chia các byte đều qua các Lane (Lane 0, 1, 2, 3)        │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
┌───────────────────────────────────▼────────────────────────────────────┐
│ 4. ELECTRICAL PHYSICAL LAYER (PHY / Tín hiệu tương tự)                 │
│    - SerDes (Serializer/Deserializer): Biến 32/64-bit song song -> Nối tiếp│
│    - Tín hiệu vi sai (Differential Signaling): D+ và D- (Điện áp ~800mV)│
│    - Clock and Data Recovery (CDR): Trích xuất xung nhịp từ dữ liệu    │
│    - De-emphasis & CTLE/DFE Equalization (Khử méo dạng sóng)           │
└────────────────────────────────────────────────────────────────────────┘
```

<!-- ORIGINAL 88-114 END -->

<!-- ORIGINAL 116 BEGIN -->

- Tại sao lại là tín hiệu vi sai (Differential Signaling)? Thay vì truyền 1 đường dây và so với Mass (dễ bị nhiễu do xung điện cao tần), PCIe dùng 2 dây dẫn xoắn/chạy song song: Vtx+-Vtx-. Khi có nhiễu sóng điện từ môi trường ngoài, cả 2 dây cùng tăng/giảm một mức áp như nhau, nên hiệu số điện áp giữa hai dây vẫn giữ nguyên dạng bit 0 và 1.

<!-- ORIGINAL 116 END -->

<!-- ORIGINAL 118 BEGIN -->

- **Credit-based Flow Control (Kiểm soát lưu lượng dựa trên Tín dụng):** Trước khi gửi 4KB dữ liệu, bộ phát phải kiểm tra xem bộ đệm của SSD Controller còn đủ "Credit" (đơn vị 16 byte) không. Cơ chế này loại bỏ hoàn toàn hiện tượng tràn bộ đệm (Buffer Overflow) ở mức phần cứng, không bao giờ phải hủy gói (Drop packet) như giao thức TCP/IP trên mạng.

<!-- ORIGINAL 118 END -->

<!-- ORIGINAL 120 BEGIN -->

## 4\. Vi kiến trúc của Bộ điều khiển SSD (NVMe Controller ASIC)

<!-- ORIGINAL 120 END -->

<!-- ORIGINAL 121 BEGIN -->

Bên trong con chip SSD Controller là một máy tính thu nhỏ chuyên dụng:

<!-- ORIGINAL 121 END -->

<!-- ORIGINAL 124-154 BEGIN -->

```text
              +═════════════════════════════════════════════════════════+
               ║                   NVMe CONTROLLER ASIC                  ║
               ║                                                         ║
PCIe Lanes ────╫─► [ PCIe PHY / SerDes Core ]                            ║
               ║               │                                         ║
               ║   [ PCIe Controller & Direct Register Logic ]           ║
               ║   - Chứa thanh ghi Doorbell thực tế                     ║
               ║   - Hardware SQ/CQ Parser (Tự quét lệnh từ Host DRAM)   ║
               ║               │                                         ║
               ║   [ AXI / NoC Bus Matrix (Băng thông hàng chục GB/s) ]  ║
               ║      │                    │                    │        ║
               ║      ▼                    ▼                    ▼        ║
               ║  [DMA ENGINE]     [CPU SUBSYSTEM]      [LOCAL MEMORY]   ║
               ║  Scatter-Gather   3-8x ARM Cortex-R    - TCM SRAM       ║
               ║  Engines          hoặc RISC-V Cores    - DDR4/DDR5 Ctrl ║
               ║  Tự động hút      Chạy FTL Firmware      chứa bảng L2P  ║
               ║  dữ liệu Host     (Wear Leveling,        (Logical to    ║
               ║  vào Controller   Garbage Collection)    Physical Map)  ║
               ║      │                                                  ║
               ║      ▼                                                  ║
               ║  [HARDWARE LDPC ENCODER/DECODER ENGINE]                 ║
               ║  Tạo mã sửa lỗi Parity bảo vệ dữ liệu                   ║
               ║      │                                                  ║
               ║      ▼                                                  ║
               ║  [FLASH MEMORY CONTROLLER (FMC)]                        ║
               ║  8 đến 16 Kênh Độc Lập (Channels)                       ║
               ║  Mỗi kênh kết nối từ 4 đến 8 Die (Interleaving)         ║
               +══════╤══════════════════════════════════════════════════+
                      │ ONFi 4.x / 5.x Bus (1200 - 2400 MT/s)
                      ▼
               [ NAND FLASH DIES ]
```

<!-- ORIGINAL 124-154 END -->

<!-- ORIGINAL 156 BEGIN -->

- **Bộ phân tích hàng đợi phần cứng (Hardware SQ Engine):** Khi CPU máy chủ ghi vào thanh ghi MMIO Doorbell, mạch logic trên controller lập tức ghi nhận chỉ mục mới mà không làm phiền vi xử lý của SSD. Bộ điều khiển tự động tạo yêu cầu DMA Memory Read để đọc gói 64-byte NVMe Command từ RAM máy chủ về bộ nhớ nội bộ của nó.

<!-- ORIGINAL 156 END -->

<!-- ORIGINAL 158 BEGIN -->

- **Scatter-Gather DMA Engine:** Phân tích các con trỏ vật lý PRP/SGL trong lệnh, chia nhỏ thành các gói DMA Burst tối ưu và kéo dữ liệu 4KB từ RAM máy chủ vào Controller DRAM Buffer.

<!-- ORIGINAL 158 END -->

<!-- ORIGINAL 160 BEGIN -->

- **Mã sửa lỗi phần cứng (LDPC Engine - Low-Density Parity-Check):** Dữ liệu được đẩy qua các khối logic toán học thực thi ma trận nhị phân để sinh ra hàng trăm byte dữ liệu kiểm tra chéo (Parity). Đây là yêu cầu bắt buộc vì chip nhớ NAND có tỷ lệ lỗi bit tự nhiên rất cao.

<!-- ORIGINAL 160 END -->

<!-- ORIGINAL 162 BEGIN -->

## 5\. Tầng Vật lý Vi mô: Tế bào nhớ NAND Flash (Điện tử & Cơ học Lượng tử)

<!-- ORIGINAL 162 END -->

<!-- ORIGINAL 163 BEGIN -->

Điểm dừng chân cuối cùng của dữ liệu là các cổng bẫy điện tích (Charge Trap Flash) nằm sâu trong các phiến silicon của chip nhớ NAND 3D.

<!-- ORIGINAL 163 END -->

<!-- ORIGINAL 166-177 BEGIN -->

```text
                 Cấu trúc Vi mô Tế bào nhớ Charge Trap (CTF)

           Wordline (Cổng điều khiển - Control Gate) [ Kim loại ]
     ═════════════════════════════════════════════════════════════════════
           Blocking Oxide (Lớp oxit chặn - SiO2 / Al2O3)
     ---------------------------------------------------------------------
     ● ● ● ● ● ●  CHARGE TRAP NITRIDE LAYER (Lớp SiN giữ electron) ● ● ● ●
     ---------------------------------------------------------------------
           Tunnel Oxide (Lớp oxit hầm phiếu mỏng vài nanomet)
     ═════════════════════════════════════════════════════════════════════
                     Kênh dẫn Silicon (P-type Silicon Substrate)
                     Source ◄────────────────────────► Drain
```

<!-- ORIGINAL 166-177 END -->

<!-- ORIGINAL 179 BEGIN -->

### A. Hiện tượng Hầm phiếu Lượng tử (Fowler-Nordheim Tunneling)

<!-- ORIGINAL 179 END -->

<!-- ORIGINAL 180 BEGIN -->

Để ghi dữ liệu (Program):

<!-- ORIGINAL 180 END -->

<!-- ORIGINAL 182 BEGIN -->

- Bộ điều khiển kích hoạt các mạch bơm điện áp nội bộ (Charge Pumps) trên die NAND, nâng điện thế lên mức cực cao: +18V đến +20V đặt vào Wordline.

<!-- ORIGINAL 182 END -->

<!-- ORIGINAL 184 BEGIN -->

- Đường kênh dẫn (Channel/Substrate) được nối đất (0V).

<!-- ORIGINAL 184 END -->

<!-- ORIGINAL 186 BEGIN -->

- **Hiệu ứng lượng tử:** Điện trường cực mạnh tạo ra độ dốc thế năng khiến các hạt electron tự do trong kênh silicon vượt qua rào cản thế năng của lớp cách điện mỏng (Tunnel Oxide) thông qua hiện tượng Hầm phiếu Lượng tử (Quantum Tunneling) và chui vào nằm kẹt lại bên trong lớp bẫy điện tích (Charge Trap Nitride).

<!-- ORIGINAL 186 END -->

<!-- ORIGINAL 188 BEGIN -->

### B. Dịch chuyển Điện áp Ngưỡng (Vth - Threshold Voltage)

<!-- ORIGINAL 188 END -->

<!-- ORIGINAL 189 BEGIN -->

- Khi electron bị giam giữ trong lớp Charge Trap, chúng tạo ra một điện trường ngược làm chắn bớt điện trường từ Wordline.

<!-- ORIGINAL 189 END -->

<!-- ORIGINAL 191 BEGIN -->

- Để làm cho kênh dẫn đóng/ngắt, CPU của SSD lúc này phải đặt một điện áp lớn hơn mức bình thường vào Wordline. Điện áp tối thiểu này gọi là Điện áp ngưỡng (Vth).

<!-- ORIGINAL 191 END -->

<!-- ORIGINAL 193 BEGIN -->

- **Tế bào đã bị xóa (Trống):** Ít electron → Vth thấp (Quy ước mức logic 1).

<!-- ORIGINAL 193 END -->

<!-- ORIGINAL 195 BEGIN -->

- **Tế bào đã được ghi:** Nhiều electron → Vth cao (Quy ước mức logic 0).

<!-- ORIGINAL 195 END -->

<!-- ORIGINAL 198-208 BEGIN -->

```text
Mật độ tế bào
      ▲
      │       Tế bào Trống (E)             Tế bào Đã Ghi (P)
      │          (Bit = 1)                     (Bit = 0)
      │        ┌───────────┐                 ┌───────────┐
      │       /             \               /             \
      │      /               \             /               \
      │     /                 \           /                 \
──────┼────┴───────────────────┴─────────┴───────────────────┴────► Điện áp (V)
      │                       │           │
      │                       └─ V_read ──┘
```

<!-- ORIGINAL 198-208 END -->

<!-- ORIGINAL 210 BEGIN -->

- Với chip TLC (3 bits/cell), một ô nhớ duy nhất phải kiểm soát số lượng electron chính xác đến mức vi mô để tạo ra 23=8 trạng thái điện áp Vth khác nhau.

<!-- ORIGINAL 210 END -->

<!-- ORIGINAL 212 BEGIN -->

- Với chip QLC (4 bits/cell), nó phải phân chia điện áp thành 16 trạng thái tách biệt. Chỉ cần vài chục hạt electron rò rỉ qua lớp màng cách điện do lão hóa vật liệu là giá trị bit bị lật, đòi hỏi khối LDPC ở bước 4 phải can thiệp giải mã toán học.

<!-- ORIGINAL 212 END -->

<!-- ORIGINAL 214 BEGIN -->

### C. Quy trình Ghi xung Từng bước (ISPP - Incremental Step Pulse Programming)

<!-- ORIGINAL 214 END -->

<!-- ORIGINAL 215 BEGIN -->

NAND Controller không nạp 20V liên tục một lần vì sẽ làm thủng lớp oxit hoặc nạp thừa electron:

<!-- ORIGINAL 215 END -->

<!-- ORIGINAL 217 BEGIN -->

- Nó phóng một xung điện áp ngắn (ví dụ 14V).

<!-- ORIGINAL 217 END -->

<!-- ORIGINAL 219 BEGIN -->

- Tạm dừng để phát xung đọc (Program Verify) kiểm tra xem Vth đã đạt đúng mức mong muốn chưa.

<!-- ORIGINAL 219 END -->

<!-- ORIGINAL 221 BEGIN -->

- Nếu chưa đạt, tăng điện áp thêm một bước nhỏ ΔV (ví dụ +0.2V) và tiếp tục phóng xung tiếp theo.

<!-- ORIGINAL 221 END -->

<!-- ORIGINAL 223 BEGIN -->

- Quá trình lặp đi lặp lại hàng chục chu kỳ cho đến khi lượng điện tích tích tụ đạt ngưỡng.

<!-- ORIGINAL 223 END -->

<!-- ORIGINAL 225 BEGIN -->

## 6\. Thước đo Độ trễ Toàn cảnh: Kim tự tháp Thời gian (Latency Scale)

<!-- ORIGINAL 225 END -->

<!-- ORIGINAL 226 BEGIN -->

Sự chênh lệch tốc độ giữa các thành phần phần cứng trong toàn bộ chuỗi I/O:

<!-- ORIGINAL 226 END -->

<!-- ORIGINAL 228 BEGIN -->

| Tầng phần cứng | Thao tác vi mô thực thi | Thời gian điển hình | Quy đổi tương đối (Nếu 1 chu kỳ CPU = 1 Giây) |
| --- | --- | --- | --- |
| CPU L1 Cache | Đọc dữ liệu biến con trỏ | ~1 ns (4 chu kỳ) | 1 giây |
| CPU LLC (L3) | Tra cứu dữ liệu chia sẻ | ~10-15 ns | 12 giây |
| Host DRAM | Kích hoạt Row Buffer, xả tụ điện | ~60-80 ns | 1 phút |
| PCIe Transmit | Mã hóa SerDes, chuyển gói TLP | ~100-250 ns | 3 phút |
| IOMMU Translation | Tra cứu IOTLB / Đi bộ Page Table | ~100-500 ns | 5 phút |
| Controller SRAM | Firmware FTL phân giải bảng L2P | ~1-5 µs | 1 giờ |
| NAND Read (TLC) | Sense Amp phân biệt 8 mức điện áp | ~30-50 µs | 半 ngày (12 giờ) |
| NAND Program (Ghi) | Phóng xung ISPP hầm phiếu electron | ~500 - 1500 µs | 12 ngày |
| NAND Erase (Xóa) | Phóng điện xóa sạch cả một Block | ~5 - 10 ms | 3 đến 4 tháng |

<!-- ORIGINAL 228 END -->

<!-- ORIGINAL 229 BEGIN -->

Toàn bộ các tầng kiến trúc phức tạp — từ Page Cache của Hệ điều hành, cơ chế Ghi lùi bất đồng bộ (Writeback), Đa hàng đợi (blk-mq) cho đến các bộ đệm DRAM trên SSD Controller — sinh ra chỉ để phục vụ một mục đích duy nhất: Giấu đi khoảng trễ thời gian khổng lồ giữa tốc độ tính toán cấp độ nano-giây của bán dẫn CPU và tốc độ nạp điện tích lượng tử cấp độ mili-giây của tế bào nhớ NAND.

<!-- ORIGINAL 229 END -->

<!-- ORIGINAL 244 BEGIN -->

**Plaintext**

<!-- ORIGINAL 244 END -->

<!-- ORIGINAL 245-324 BEGIN -->

```text
+══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════+
║                                           KIẾN TRÚC TỔNG QUÁT LUỒNG WRITE I/O                                                   ║
+══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════+

   TẦNG PHẦN MỀM / LOGIC (OS & KERNEL)                                        TẦNG PHẦN CỨNG VẬT LÝ (HARDWARE)
┌──────────────────────────────────────────────┐                           ┌──────────────────────────────────────────────┐
│ 1. USERSPACE APPLICATION                     │                           │ [CPU - Ring 3 (User Mode)]                   │
│    - buffer[4096] trên Stack/Heap            │                           │ - User Virtual Address (UVA)                 │
│    - glibc: nạp RAX=1, RDI=fd, RSI=&buf      │                           │ - GPRs: RAX, RDI, RSI, RDX                   │
└──────────────────────┬───────────────────────┘                           └──────────────────────┬───────────────────────┘
                       │ SYSCALL (Privilege Switch)                                               │ MSR IA32_LSTAR, TSS sp0
                       ▼                                                                          ▼
┌──────────────────────────────────────────────┐                           ┌──────────────────────────────────────────────┐
│ 2. VFS & PAGE CACHE                          │                           │ [CPU - Ring 0 (Kernel Mode) + RAM (DRAM)]    │
│    - sys_write() -> vfs_write()              │                           │ - Kernel Stack riêng biệt                    │
│    - Tra cứu XArray trong struct address_space│                           │ - MMU: Kiểm tra SMAP (CR4.SMAP, lệnh STAC)   │
│    - Cấp phát struct page / folio            │                           │ - CPU Memcpy: Đọc DRAM (UVA) -> ghi DRAM     │
│    - copy_from_user() payload                │ ── Lần 1: CPU Copy ─────► │   (Kernel Page Frame 4KB)                    │
│    - Đánh dấu folio_mark_dirty()             │                           │ - CPU L1/L2/L3 Cache được nạp dữ liệu        │
│    - sysretq: BÁO THÀNH CÔNG VỀ USER APP     │                           │                                              │
└──────────────────────┬───────────────────────┘                           └──────────────────────────────────────────────┘
                       │ (Bất đồng bộ - Asynchronous Boundary: App đã chạy tiếp, dữ liệu vẫn nằm trên RAM)
                       ▼
┌──────────────────────────────────────────────┐                           ┌──────────────────────────────────────────────┐
│ 3. WRITEBACK SUBSYSTEM                       │                           │ [CPU Background Cores]                       │
│    - Luồng kworker/flush-x thức giấc         │                           │ - Timer Interrupts định kỳ                   │
│    - Quét cây XArray tìm trang Dirty         │                           │ - Đọc cấu trúc metadata từ DRAM              │
│    - Khóa trang: PG_locked, xóa PG_dirty     │                           │                                              │
└──────────────────────┬───────────────────────┘                           └──────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐                           ┌──────────────────────────────────────────────┐
│ 4. FILESYSTEM (ext4 / XFS)                   │                           │ [DRAM - Metadata Structures]                 │
│    - Ánh xạ File Offset -> Logical Block     │                           │ - Superblock, Inode Table, Extent Tree       │
│    - Ghi Journal (JBD2) đảm bảo tính toàn vẹn│                           │ - Chuyển đổi LBN sang Physical Block (PBN)   │
│    - Đóng gói dữ liệu thành struct bio       │                           │                                              │
└──────────────────────┬───────────────────────┘                           └──────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐                           ┌──────────────────────────────────────────────┐
│ 5. GENERIC BLOCK LAYER (blk-mq)              │                           │ [CPU Per-Core Queues & DRAM]                 │
│    - Nhận struct bio -> Chia/Gộp (Merge/Split)│                           │ - Software Staging Queue (blk_mq_ctx)        │
│    - Tạo struct request                      │                           │ - Hardware Dispatch Queue (blk_mq_hw_ctx)    │
│    - I/O Scheduler (none / mq-deadline / bfq)│                           │                                              │
└──────────────────────┬───────────────────────┘                           └──────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐                           ┌──────────────────────────────────────────────┐
│ 6. DEVICE DRIVER (nvme / virtio-blk)         │                           │ [PCIe Root Complex & MMIO Registers]         │
│    - Dịch request thành lệnh phần cứng       │                           │ - Driver ghi địa chỉ vật lý (HPA) của Page   │
│    - Tạo danh sách PRP List / SGL descriptors│                           │   vào Submission Queue (SQ) trên Host DRAM   │
│    - Ghi chuông thông báo (Ring Doorbell)   │ ── Ghi MMIO ────────────► │ - Ghi thanh ghi BAR của thiết bị qua PCIe Bus│
└──────────────────────────────────────────────┘                           └──────────────────────┬───────────────────────┘
                                                                                                  │ PCIe TLP Packet
                                                                                                  ▼
┌──────────────────────────────────────────────┐                           ┌──────────────────────────────────────────────┐
│ 7. DMA ENGINE & INTERCONNECT BUS             │                           │ [Bus Controller & IOMMU]                     │
│    - Bỏ qua CPU hoàn toàn (Zero-CPU Copy)    │                           │ - IOMMU (Intel VT-d/AMD-Vi) dịch IOVA -> HPA │
│    - Thiết bị tự làm Bus Master              │ ── Lần 2: DMA Transfer ──►│ - PCIe Lanes (Gen4/Gen5 x4): Direct DMA Read │
│    - Kéo dữ liệu từ DRAM Host về Storage     │                           │   từ Host DRAM vào SSD RAM Buffer            │
└──────────────────────────────────────────────┘                           └──────────────────────┬───────────────────────┘
                                                                                                  │
                                                                                                  ▼
┌──────────────────────────────────────────────┐                           ┌──────────────────────────────────────────────┐
│ 8. STORAGE CONTROLLER & MEDIA                │                           │ [Physical SSD / NVMe Device]                 │
│    - Tiếp nhận Command từ NVMe SQ            │                           │ - ASIC / ARM multi-core Controller           │
│    - Flash Translation Layer (FTL) ánh xạ    │                           │ - On-board DRAM Controller Buffer            │
│    - Wear Leveling & ECC Generation          │                           │ - Power-Loss Protection (PLP Capacitors)     │
│    - Ghi điện tích vào ô nhớ vật lý          │ ── Flash Programming ───► │ - NAND Flash Chips (Die -> Plane -> Block)   │
└──────────────────────┬───────────────────────┘                           └──────────────────────────────────────────────┘
                       │
                       │ 9. COMPLETION & INTERRUPT (MSI-X)
                       ▼
┌──────────────────────────────────────────────┐                           ┌──────────────────────────────────────────────┐
│ 10. DRIVER ISR & BLOCK COMPLETION            │                           │ [Interrupt Routing & CPU Local APIC]         │
│    - Driver đọc NVMe Completion Queue (CQ)   │                           │ - Thiết bị bắn PCIe MSI-X Interrupt          │
│    - blk_mq_complete_request()               │                           │ - CPU kích hoạt Hard-IRQ -> Soft-IRQ         │
│    - bio_endio(): Mở khóa trang (unlock_page)│                           │ - Giải phóng tài nguyên trên Host DRAM       │
│    - Đánh dấu trang Clean, kết thúc vòng đời │                           │                                              │
└──────────────────────────────────────────────┘                           └──────────────────────────────────────────────┘
```

<!-- ORIGINAL 245-324 END -->

<!-- ORIGINAL 326 BEGIN -->

## Chi tiết logic và vai trò từng giai đoạn

<!-- ORIGINAL 326 END -->

<!-- ORIGINAL 327 BEGIN -->

### 1\. Khởi tạo từ Userspace và Chuyển đổi Đặc quyền

<!-- ORIGINAL 327 END -->

<!-- ORIGINAL 328 BEGIN -->

- Logic phần mềm:

<!-- ORIGINAL 328 END -->

<!-- ORIGINAL 330 BEGIN -->

- Tiến trình nắm giữ vùng đệm dữ liệu buffer tại Địa chỉ Ảo Người dùng (User Virtual Address - UVA) nằm trên Stack hoặc Heap.

<!-- ORIGINAL 330 END -->

<!-- ORIGINAL 332 BEGIN -->

- Thư viện glibc đóng gói các tham số vào thanh ghi phần cứng theo chuẩn gọi hàm System V AMD64 ABI: RAX = 1 (mã syscall write), RDI = fd, RSI = &buffer, RDX = 4096.

<!-- ORIGINAL 332 END -->

<!-- ORIGINAL 334 BEGIN -->

- Thiết bị phần cứng tham gia:

<!-- ORIGINAL 334 END -->

<!-- ORIGINAL 336 BEGIN -->

- **CPU Core:** Thực thi ở cấp đặc quyền Ring 3.

<!-- ORIGINAL 336 END -->

<!-- ORIGINAL 338 BEGIN -->

- **Chỉ lệnh SYSCALL:** Phần cứng CPU tự động lưu con trỏ lệnh kế tiếp vào RCX, lưu cờ trạng thái vào R11, đổi cờ CPL sang Ring 0, nạp con trỏ Stack hạt nhân từ TSS (tss.sp0), và nhảy thẳng tới địa chỉ ghi trong thanh ghi đặc quyền MSR IA32\_LSTAR (hàm entry\_SYSCALL\_64).

<!-- ORIGINAL 338 END -->

<!-- ORIGINAL 340 BEGIN -->

- **Bản chất:** Đây là bước chuyển mức đặc quyền phần cứng (Privilege Switch), luồng thực thi vẫn thuộc ngữ cảnh của tiến trình gọi (Process Context).

<!-- ORIGINAL 340 END -->

<!-- ORIGINAL 342 BEGIN -->

### 2\. Tầng VFS và Điểm đệm Page Cache (Lần sao chép thứ nhất)

<!-- ORIGINAL 342 END -->

<!-- ORIGINAL 343 BEGIN -->

- Logic phần mềm:

<!-- ORIGINAL 343 END -->

<!-- ORIGINAL 345 BEGIN -->

- **sys\_write() → vfs\_write():** Tra cứu số nguyên fd trong bảng File Descriptor (fdt) của tiến trình để lấy con trỏ struct file.

<!-- ORIGINAL 345 END -->

<!-- ORIGINAL 347 BEGIN -->

- **Gọi hàm thao tác tương ứng của hệ thống tệp:** file-\>f\_op-\>write\_iter().

<!-- ORIGINAL 347 END -->

<!-- ORIGINAL 349 BEGIN -->

- **Hệ điều hành chia file offset thành các chỉ mục trang:** index=offset≫12 (cho trang 4KB). Kernel tra cứu cấu trúc cây XArray (trong struct address\_space) của Inode.

<!-- ORIGINAL 349 END -->

<!-- ORIGINAL 351 BEGIN -->

- Nếu trang chưa có (Cache Miss), kernel gọi bộ cấp phát phân trang hạt nhân lấy một struct page/folio vật lý mới và chèn vào cây.

<!-- ORIGINAL 351 END -->

<!-- ORIGINAL 353 BEGIN -->

- **Thực thi sao chép dữ liệu thông qua copy\_from\_user():** Đây là lần sao chép payload đầu tiên (CPU Memcpy).

<!-- ORIGINAL 353 END -->

<!-- ORIGINAL 355 BEGIN -->

- Đánh dấu trang bằng cờ PG\_dirty.

<!-- ORIGINAL 355 END -->

<!-- ORIGINAL 357 BEGIN -->

- Thiết bị phần cứng tham gia:

<!-- ORIGINAL 357 END -->

<!-- ORIGINAL 359 BEGIN -->

- **CPU MMU & Bit SMAP:** Khi ở Ring 0, CPU dùng tính năng phần cứng SMAP (Supervisor Mode Access Prevention - bit 21 thanh ghi CR4) để chặn kernel truy cập trái phép bộ nhớ User Space. Kernel phải kích hoạt lệnh assembly stac để tạm tắt bảo vệ trước khi memcpy, và chạy lệnh clac ngay sau đó để bật lại.

<!-- ORIGINAL 359 END -->

<!-- ORIGINAL 361 BEGIN -->

- **CPU L1/L2/L3 Cache & Bus DRAM:** Dữ liệu payload được CPU đọc từ dải DRAM của Userspace và ghi vào dải DRAM mới được cấp phát cho Page Cache của Kernel.

<!-- ORIGINAL 361 END -->

<!-- ORIGINAL 363 BEGIN -->

- **Hoàn tất lệnh write():** CPU thực thi sysretq, trả về số byte đã ghi cho User Application. Ứng dụng tiếp tục chạy; thiết bị lưu trữ vật lý lúc này hoàn toàn chưa nhận được bất kỳ tín hiệu nào.

<!-- ORIGINAL 363 END -->

<!-- ORIGINAL 365 BEGIN -->

### 3\. Bộ phận Ghi lùi Bất đồng bộ (Writeback Subsystem)

<!-- ORIGINAL 365 END -->

<!-- ORIGINAL 366 BEGIN -->

- Logic phần mềm:

<!-- ORIGINAL 366 END -->

<!-- ORIGINAL 368 BEGIN -->

- Dữ liệu nằm chờ trong RAM dưới dạng các trang bẩn (Dirty Pages).

<!-- ORIGINAL 368 END -->

<!-- ORIGINAL 370 BEGIN -->

- Hạt nhân đánh thức các tiến trình nền kworker/flush-x dựa trên 3 điều kiện kích hoạt:

<!-- ORIGINAL 370 END -->

<!-- ORIGINAL 372 BEGIN -->

- **Định kỳ theo thời gian:** Cấu hình dirty\_writeback\_centisecs (mặc định mỗi 5 giây).

<!-- ORIGINAL 372 END -->

<!-- ORIGINAL 374 BEGIN -->

- **Ngưỡng dung lượng:** Tỷ lệ trang bẩn vượt quá sysctl dirty\_background\_ratio (hoặc dirty\_ratio).

<!-- ORIGINAL 374 END -->

<!-- ORIGINAL 376 BEGIN -->

- **Yêu cầu cưỡng bức:** Ứng dụng chủ động gọi các hàm đồng bộ dữ liệu fsync(), fdatasync(), hoặc sync().

<!-- ORIGINAL 376 END -->

<!-- ORIGINAL 378 BEGIN -->

- Tiến trình flush duyệt cây XArray, khóa trang bằng cờ PG\_locked để ngăn race condition, xóa cờ PG\_dirty, và chuyển trang vào danh sách chờ ghi xuống tầng lưu trữ.

<!-- ORIGINAL 378 END -->

<!-- ORIGINAL 380 BEGIN -->

- Thiết bị phần cứng tham gia:

<!-- ORIGINAL 380 END -->

<!-- ORIGINAL 382 BEGIN -->

- **Programmable Interval Timer (PIT) / APIC Timer:** Phát ngắt định kỳ đánh thức các core CPU đang rảnh rỗi để lên lịch chạy luồng kworker.

<!-- ORIGINAL 382 END -->

<!-- ORIGINAL 384 BEGIN -->

### 4\. Phân giải Hệ thống tệp (Filesystem - ext4 / XFS)

<!-- ORIGINAL 384 END -->

<!-- ORIGINAL 385 BEGIN -->

- Logic phần mềm:

<!-- ORIGINAL 385 END -->

<!-- ORIGINAL 387 BEGIN -->

- **File hệ thống ánh xạ:** Một file là chuỗi logic liên tục, nhưng trên đĩa vật lý các khối có thể bị phân mảnh.

<!-- ORIGINAL 387 END -->

<!-- ORIGINAL 389 BEGIN -->

- Driver hệ thống tệp (ví dụ ext4\_writepages) tra cứu cấu trúc cây Extent Tree của Inode để chuyển đổi từ Số Khối Logic (Logical Block Number - LBN) sang Số Khối Vật lý (Physical Block Number - PBN) của ổ đĩa.

<!-- ORIGINAL 389 END -->

<!-- ORIGINAL 391 BEGIN -->

- **Journaling (Ghi nhật ký - JBD2):** Metadata (kích thước file, mtime, con trỏ extent mới) được ghi vào vùng Journal trước để đảm bảo tính nhất quán cấu trúc khi mất điện đột ngột.

<!-- ORIGINAL 391 END -->

<!-- ORIGINAL 393 BEGIN -->

- **Đóng gói yêu cầu I/O thành cấu trúc chuẩn của Linux Block Layer:** struct bio. Một bio chứa danh sách các đoạn bộ nhớ phân tán (bio\_vec) gồm con trỏ trỏ tới các trang struct page trong Page Cache và vị trí sector đĩa đích.

<!-- ORIGINAL 393 END -->

<!-- ORIGINAL 395 BEGIN -->

### 5\. Tầng Khối Đa hàng đợi (Generic Block Layer - blk-mq)

<!-- ORIGINAL 395 END -->

<!-- ORIGINAL 396 BEGIN -->

- Logic phần mềm:

<!-- ORIGINAL 396 END -->

<!-- ORIGINAL 398 BEGIN -->

- blk-mq (Multi-Queue Block Layer) tiếp nhận các struct bio.

<!-- ORIGINAL 398 END -->

<!-- ORIGINAL 400 BEGIN -->

- **Tối ưu hóa:** Kernel tiến hành Bio Merge (nếu các bio trỏ tới các sector đĩa nằm liền kề nhau, chúng được gộp thành một request lớn duy nhất) hoặc Bio Split (nếu request vượt quá giới hạn phần cứng max\_sectors\_kb).

<!-- ORIGINAL 400 END -->

<!-- ORIGINAL 402 BEGIN -->

- Các bio được đóng gói vào cấu trúc struct request.

<!-- ORIGINAL 402 END -->

<!-- ORIGINAL 404 BEGIN -->

- **I/O Scheduler:** Các thuật toán như none (thường dùng cho NVMe), mq-deadline (đảm bảo hạn chót tránh đói I/O), hoặc bfq (chia đều băng thông) sắp xếp lại thứ tự ưu tiên của request.

<!-- ORIGINAL 404 END -->

<!-- ORIGINAL 406 BEGIN -->

- Cấu trúc dữ liệu liên kết phần cứng:

<!-- ORIGINAL 406 END -->

<!-- ORIGINAL 408 BEGIN -->

- **Software Staging Queues (blk\_mq\_ctx):** Được phân bổ trên từng CPU core để loại bỏ hiện tượng tranh chấp khóa (Lock Contention) giữa các luồng.

<!-- ORIGINAL 408 END -->

<!-- ORIGINAL 410 BEGIN -->

- **Hardware Dispatch Queues (blk\_mq\_hw\_ctx):** Ánh xạ tương ứng trực tiếp tới số lượng hàng đợi phần cứng mà thiết bị lưu trữ hỗ trợ.

<!-- ORIGINAL 410 END -->

<!-- ORIGINAL 412 BEGIN -->

### 6\. Trình điều khiển Thiết bị (Device Driver: nvme / virtio-blk)

<!-- ORIGINAL 412 END -->

<!-- ORIGINAL 413 BEGIN -->

- Logic phần mềm & Chuẩn bị phần cứng:

<!-- ORIGINAL 413 END -->

<!-- ORIGINAL 415 BEGIN -->

- Driver lấy struct request ra khỏi hàng đợi dispatch và dịch nó thành khuôn mẫu lệnh riêng của thiết bị (ví dụ: NVMe Write Command 64-byte).

<!-- ORIGINAL 415 END -->

<!-- ORIGINAL 417 BEGIN -->

- **Scatter-Gather Lists (SGL) / Physical Region Pages (PRP):** Do các trang Page Cache 4KB trong DRAM có thể nằm phân tán ở các địa chỉ vật lý ngẫu nhiên, driver tạo ra một danh sách các con trỏ chứa Địa chỉ Vật lý Máy chủ (Host Physical Address - HPA) của các trang này để đưa vào lệnh NVMe.

<!-- ORIGINAL 417 END -->

<!-- ORIGINAL 419 BEGIN -->

- Ghi gói lệnh NVMe (64 bytes) vào vùng bộ nhớ Submission Queue (SQ) — vùng nhớ này nằm trên chính thanh RAM của máy chủ.

<!-- ORIGINAL 419 END -->

<!-- ORIGINAL 421 BEGIN -->

- Kích hoạt phần cứng (Ring Doorbell):

<!-- ORIGINAL 421 END -->

<!-- ORIGINAL 423 BEGIN -->

- Driver ghi số hiệu chỉ mục (index) mới của hàng đợi vào thanh ghi Doorbell Register của controller NVMe.

<!-- ORIGINAL 423 END -->

<!-- ORIGINAL 425 BEGIN -->

- Thiết bị phần cứng tham gia:

<!-- ORIGINAL 425 END -->

<!-- ORIGINAL 427 BEGIN -->

- **MMIO (Memory-Mapped I/O):** Thanh ghi Doorbell được ánh xạ vào không gian địa chỉ vật lý thông qua Base Address Register (BAR) của thiết bị trên bus PCI.

<!-- ORIGINAL 427 END -->

<!-- ORIGINAL 429 BEGIN -->

- **PCIe Controller:** Thao tác ghi Doorbell từ CPU được chuyển đổi thành một gói tin giao dịch trên bus PCIe (PCIe TLP - Transaction Layer Packet: Memory Write), bắn tín hiệu tới vi mạch điều khiển của ổ cứng.

<!-- ORIGINAL 429 END -->

<!-- ORIGINAL 431 BEGIN -->

### 7\. Động cơ DMA và Tuyến truyền thông Liên kết (Bus Interconnect)

<!-- ORIGINAL 431 END -->

<!-- ORIGINAL 432 BEGIN -->

- **Bản chất:** Hoàn toàn không có sự tham gia của CPU trong việc vận chuyển dữ liệu payload ở giai đoạn này.

<!-- ORIGINAL 432 END -->

<!-- ORIGINAL 434 BEGIN -->

- Cơ chế hoạt động:

<!-- ORIGINAL 434 END -->

<!-- ORIGINAL 436 BEGIN -->

- Controller trên thiết bị nhận được thông báo chuông (Doorbell) qua bus PCIe.

<!-- ORIGINAL 436 END -->

<!-- ORIGINAL 438 BEGIN -->

- Thiết bị đóng vai trò là Bus Master, chủ động kích hoạt động cơ DMA Engine tích hợp sẵn trên bo mạch điều khiển của nó.

<!-- ORIGINAL 438 END -->

<!-- ORIGINAL 440 BEGIN -->

- Động cơ DMA đọc danh sách địa chỉ PRP/SGL từ Submission Queue trên Host DRAM, sau đó phát các yêu cầu đọc trực tiếp (PCIe Inbound Memory Read) để hút từng khối dữ liệu 4KB từ các khung trang Page Cache trên Host DRAM về bộ đệm nội bộ của thiết bị.

<!-- ORIGINAL 440 END -->

<!-- ORIGINAL 442 BEGIN -->

- Thiết bị phần cứng tham gia:

<!-- ORIGINAL 442 END -->

<!-- ORIGINAL 444 BEGIN -->

- **IOMMU (Intel VT-d / AMD-Vi):** Nằm giữa bus PCIe và thanh RAM máy chủ. IOMMU kiểm tra quyền truy cập bộ nhớ của thiết bị và thực hiện dịch địa chỉ I/O ảo (IOVA) sang địa chỉ vật lý (HPA), bảo vệ hệ thống khỏi các cuộc tấn công DMA độc hại.

<!-- ORIGINAL 444 END -->

<!-- ORIGINAL 446 BEGIN -->

- **PCIe Root Complex & PCIe Lanes:** Kênh truyền dẫn vật lý tốc độ cao (ví dụ: PCIe 4.0 x4 cung cấp băng thông thực tế ~7.8 GB/s) kết nối trực tiếp socket CPU/RAM với khe cắm thiết bị lưu trữ.

<!-- ORIGINAL 446 END -->

<!-- ORIGINAL 448 BEGIN -->

### 8\. Thiết bị Lưu trữ Vật lý (SSD NVMe Controller & NAND Flash)

<!-- ORIGINAL 448 END -->

<!-- ORIGINAL 449 BEGIN -->

- Logic phần cứng bên trong thiết bị (Firmware & Controller):

<!-- ORIGINAL 449 END -->

<!-- ORIGINAL 451 BEGIN -->

- Dữ liệu được DMA đưa tạm vào bộ đệm DRAM tích hợp của SSD Controller.

<!-- ORIGINAL 451 END -->

<!-- ORIGINAL 453 BEGIN -->

- Bộ vi xử lý nhúng (ASIC Controller / ARM multi-core) chạy thuật toán Flash Translation Layer (FTL):

<!-- ORIGINAL 453 END -->

<!-- ORIGINAL 455 BEGIN -->

- Ánh xạ Địa chỉ Khối Logic (LBA từ OS) sang Địa chỉ Khối Vật lý trên chip nhớ Flash (Physical Page Address - PPA).

<!-- ORIGINAL 455 END -->

<!-- ORIGINAL 457 BEGIN -->

- **Thuật toán Wear Leveling:** Điều phối dữ liệu ghi vào các ô nhớ có chu kỳ xóa thấp nhằm chống chai chip.

<!-- ORIGINAL 457 END -->

<!-- ORIGINAL 459 BEGIN -->

- Tạo mã sửa lỗi ECC / LDPC gắn kèm dữ liệu.

<!-- ORIGINAL 459 END -->

<!-- ORIGINAL 461 BEGIN -->

- Thiết bị phần cứng lưu trữ cuối cùng:

<!-- ORIGINAL 461 END -->

<!-- ORIGINAL 463 BEGIN -->

- **NAND Flash Media:** Dữ liệu được ghi điện áp vào các mảng tế bào nhớ (NAND Cells: SLC, TLC, QLC) trên từng Die/Plane. Quá trình này tiêu tốn thời gian lớn nhất về mặt cơ chế vật lý (vài chục đến hàng trăm micro-giây cho thao tác nạp điện áp Flash Program).

<!-- ORIGINAL 463 END -->

<!-- ORIGINAL 465 BEGIN -->

- **Tụ chống mất điện (Power-Loss Protection - PLP):** Trên các dòng Enterprise SSD, dữ liệu chỉ cần nằm an toàn trong DRAM của SSD là controller đã có thể báo hoàn tất, vì dàn tụ điện PLP đảm bảo đủ năng lượng để xả toàn bộ dữ liệu từ DRAM SSD vào NAND nếu nguồn điện bị ngắt đột ngột.

<!-- ORIGINAL 465 END -->

<!-- ORIGINAL 467 BEGIN -->

### 9\. Đường hồi tiếp Hoàn tất qua Ngắt Phần cứng (Interrupt Loop)

<!-- ORIGINAL 467 END -->

<!-- ORIGINAL 468 BEGIN -->

- Tương tác phần cứng → Phần mềm:

<!-- ORIGINAL 468 END -->

<!-- ORIGINAL 470 BEGIN -->

- Sau khi dữ liệu đã được nạp an toàn, controller NVMe ghi một bản ghi 16-byte vào Completion Queue (CQ) (nằm trên Host DRAM) và kích hoạt một ngắt MSI-X (Message Signaled Interrupt) qua bus PCIe.

<!-- ORIGINAL 470 END -->

<!-- ORIGINAL 472 BEGIN -->

- Gói tin MSI-X gửi tín hiệu ngắt thẳng tới bộ xử lý ngắt Local APIC của một Core CPU xác định.

<!-- ORIGINAL 472 END -->

<!-- ORIGINAL 474 BEGIN -->

- CPU dừng luồng đang chạy, lưu ngữ cảnh và kích hoạt hàm xử lý ngắt cứng (Hard-IRQ / ISR) của driver thiết bị.

<!-- ORIGINAL 474 END -->

<!-- ORIGINAL 476 BEGIN -->

- Driver nâng mức xử lý lên ngắt mềm (Soft-IRQ / Tasklet / ksoftirqd) để gọi blk\_mq\_complete\_request().

<!-- ORIGINAL 476 END -->

<!-- ORIGINAL 478 BEGIN -->

- Tầng Block layer thông báo cho hệ thống tệp và gọi hàm bio\_endio().

<!-- ORIGINAL 478 END -->

<!-- ORIGINAL 480 BEGIN -->

- Kernel mở khóa trang (unlock\_page()), giải phóng cờ chờ I/O, chuyển trạng thái trang trong Page Cache thành trang sạch (Clean Page). Toàn bộ vòng đời của một thao tác I/O kết thúc.

<!-- ORIGINAL 480 END -->

<!-- ORIGINAL 481 BEGIN -->

## Bảng đối chiếu thực thể: Phần mềm Logic vs. Phần cứng Vật lý

<!-- ORIGINAL 481 END -->

<!-- ORIGINAL 482 BEGIN -->

| Tầng kiến trúc | Thành phần Logic (Software / Kernel) | Thành phần Phần cứng (Physical Hardware) |
| --- | --- | --- |
| Userspace | User Virtual Address (UVA), Buffer array, glibc | CPU Core (Ring 3), Thanh ghi chung (RAX, RDI, RSI, RDX) |
| Giao tiếp Cổng | sys\_write(), Entry Assembly entry\_SYSCALL\_64 | MSR IA32\_LSTAR, Control Register CR3, TSS sp0 |
| Bộ đệm Tệp tin | VFS struct file, Inode, XArray, Page Cache, folio | Khung trang DRAM (Host Physical Address), Bộ nhớ đệm L1/L2/L3, SMAP (CR4) |
| Lên lịch Ghi lùi | kworker/flush, Thuật toán tính ngưỡng Dirty Ratio | Ngắt APIC Timer định kỳ của CPU |
| Hệ thống tệp | ext4\_writepages, Extent Tree, bio, Nhật ký JBD2 | Bộ nhớ DRAM lưu trữ siêu dữ liệu (Metadata) |
| Đa hàng đợi Khối | blk-mq, struct request, I/O Scheduler (none, bfq) | Software Context Queues chia theo từng Core CPU |
| Trình điều khiển | Driver nvme.ko, Bảng PRP List / SGL, NVMe SQ/CQ | Vùng nhớ MMIO BAR trên PCIe, Thanh ghi Doorbell |
| Tuyến vận chuyển | Cấu hình kênh DMA, IOVA Mapping | Bộ điều khiển IOMMU, Tuyến PCIe (Lanes x4/x8), PCIe Root Complex |
| Bộ điều khiển Đĩa | Firmware SSD, Flash Translation Layer (FTL), LDPC | Chip ASIC Controller, On-board DRAM Buffer của SSD, Tụ điện PLP |
| Môi trường lưu trữ | LBA-to-PPA Translation Table | Ô nhớ bán dẫn NAND Flash (Floating Gate / Charge Trap) |
| Báo hoàn tất | ISR Driver, Bottom-Half BLOCK\_SOFTIRQ, bio\_endio | Gói tin ngắt PCIe MSI-X, Bộ điều khiển Local APIC trên CPU |

<!-- ORIGINAL 482 END -->

<!-- ORIGINAL 485 BEGIN -->

## Các hàm hệ thống


Dưới đây là vi phẫu toàn bộ chuỗi hàm được kích hoạt theo trình tự thời gian cho luồng ghi từ Guest VM (virtio-blk, O\_DIRECT, cache=none, io=native) qua KVM/QEMU → Host Kernel (blk-mq → dm-multipath → iscsi\_tcp) → Flash Target, kèm chi tiết tham số, tác động phần cứng và biến đổi bộ nhớ.

<!-- ORIGINAL 485 END -->

<!-- ORIGINAL 487 BEGIN -->

### GIAI ĐOẠN 1: GUEST OS — TỪ SYSTEM CALL ĐẾN VIRTQUEUE

<!-- ORIGINAL 487 END -->

<!-- ORIGINAL 488 BEGIN -->

Ở giai đoạn này, luồng thực thi chạy hoàn toàn bên trong máy ảo dưới chế độ CPU VMX Non-Root Operation.

<!-- ORIGINAL 488 END -->

<!-- ORIGINAL 490 BEGIN -->

**Plaintext**

<!-- ORIGINAL 490 END -->

<!-- ORIGINAL 491-493 BEGIN -->

```text
write() ──> ksys_write() ──> vfs_write() ──> blkdev_write_iter()
  └──> blk_mq_submit_bio() ──> virtio_queue_rq()
         └──> virtqueue_add_outbuf() ──> virtqueue_notify() (Ghi MMIO Doorbell)
```

<!-- ORIGINAL 491-493 END -->

<!-- ORIGINAL 495 BEGIN -->

#### 1\. sys\_write() / ksys\_write()

<!-- ORIGINAL 495 END -->

<!-- ORIGINAL 496 BEGIN -->

**Signature:** `ssize_t ksys_write(unsigned int fd, const char __user *buf, size_t count);`

<!-- ORIGINAL 496 END -->

<!-- ORIGINAL 498 BEGIN -->

- **Ngữ cảnh:** Guest User Space (Ring 3) → Guest Kernel (Ring 0) qua chỉ lệnh syscall.

<!-- ORIGINAL 498 END -->

<!-- ORIGINAL 500 BEGIN -->

- Tham số chính:

<!-- ORIGINAL 500 END -->

<!-- ORIGINAL 502 BEGIN -->

- **fd:** File descriptor của thiết bị khối (ví dụ /dev/vdb).

<!-- ORIGINAL 502 END -->

<!-- ORIGINAL 504 BEGIN -->

- **buf:** Con trỏ Guest Virtual Address (GVA) trỏ tới buffer 4096B (ví dụ 0x00401000).

<!-- ORIGINAL 504 END -->

<!-- ORIGINAL 506 BEGIN -->

- **count:** 4096 (kích thước ghi tính bằng bytes).

<!-- ORIGINAL 506 END -->

<!-- ORIGINAL 508 BEGIN -->

- **Vai trò & Bộ nhớ:** Lưu con trỏ lệnh RIP vào RCX, đổi sang Kernel Stack của Guest. MMU Guest giữ quyền kiểm soát bảng phân trang cấp 1 (Guest Page Table - GPT).

<!-- ORIGINAL 508 END -->

<!-- ORIGINAL 510 BEGIN -->

#### 2\. vfs\_write() → blkdev\_write\_iter()

<!-- ORIGINAL 510 END -->

<!-- ORIGINAL 511 BEGIN -->

**Signature:** `ssize_t blkdev_write_iter(struct kiocb *iocb, struct iov_iter *from);`

<!-- ORIGINAL 511 END -->

<!-- ORIGINAL 513 BEGIN -->

- **Ngữ cảnh:** VFS Guest định tuyến tới driver thiết bị khối.

<!-- ORIGINAL 513 END -->

<!-- ORIGINAL 515 BEGIN -->

- Tham số chính:

<!-- ORIGINAL 515 END -->

<!-- ORIGINAL 517 BEGIN -->

- **iocb-\>ki\_flags:** Chứa cờ IOCB\_DIRECT (do mở bằng O\_DIRECT).

<!-- ORIGINAL 517 END -->

<!-- ORIGINAL 519 BEGIN -->

- **from:** Cấu trúc mô tả mảng bộ nhớ người dùng (IOVEC).

<!-- ORIGINAL 519 END -->

<!-- ORIGINAL 521 BEGIN -->

- **Vai trò & Bộ nhớ:** Bỏ qua hoàn toàn Page Cache của Guest. Không cấp phát page cache folio, không gọi copy\_from\_user.

<!-- ORIGINAL 521 END -->

<!-- ORIGINAL 523 BEGIN -->

#### 3\. blk\_mq\_submit\_bio()

<!-- ORIGINAL 523 END -->

<!-- ORIGINAL 524 BEGIN -->

**Signature:** `void blk_mq_submit_bio(struct bio *bio);`

<!-- ORIGINAL 524 END -->

<!-- ORIGINAL 526 BEGIN -->

- **Ngữ cảnh:** Tầng Block đa hàng đợi của Guest Kernel (blk-mq).

<!-- ORIGINAL 526 END -->

<!-- ORIGINAL 528 BEGIN -->

- Tham số chính:

<!-- ORIGINAL 528 END -->

<!-- ORIGINAL 530 BEGIN -->

- **bio-\>bi\_iter.bi\_sector:** Sector đích trên đĩa ảo /dev/vdb.

<!-- ORIGINAL 530 END -->

<!-- ORIGINAL 532 BEGIN -->

- **bio-\>bi\_io\_vec\[0\]:** Chứa trang RAM vật lý của Guest (GPA) sau khi gọi pin\_user\_pages().

<!-- ORIGINAL 532 END -->

<!-- ORIGINAL 534 BEGIN -->

- **Vai trò & Bộ nhớ:** Ghim cứng khung trang RAM tại địa chỉ Guest Physical Address (GPA) (ví dụ 0x00105000). Tạo struct request và đưa vào hàng đợi phần mềm (Software Staging Queue) của vCPU hiện tại.

<!-- ORIGINAL 534 END -->

<!-- ORIGINAL 536 BEGIN -->

#### 4\. virtio\_queue\_rq()

<!-- ORIGINAL 536 END -->

<!-- ORIGINAL 537 BEGIN -->

**Signature:** `blk_status_t virtio_queue_rq(struct blk_mq_hw_ctx *hctx, const struct blk_mq_queue_data *bd);`

<!-- ORIGINAL 537 END -->

<!-- ORIGINAL 539 BEGIN -->

- **Ngữ cảnh:** Driver thiết bị ảo hóa virtio\_blk.ko trong Guest.

<!-- ORIGINAL 539 END -->

<!-- ORIGINAL 541 BEGIN -->

- Tham số chính:

<!-- ORIGINAL 541 END -->

<!-- ORIGINAL 543 BEGIN -->

- **bd-\>rq:** Cấu trúc request cần dispatch.

<!-- ORIGINAL 543 END -->

<!-- ORIGINAL 545 BEGIN -->

- **Vai trò & Bộ nhớ:** Bóc tách request thành 3 phần: Header lệnh (struct virtio\_blk\_outhdr), Data payload (4096B), và Status byte.

<!-- ORIGINAL 545 END -->

<!-- ORIGINAL 547 BEGIN -->

#### 5\. virtqueue\_add\_outbuf() / virtqueue\_add()

<!-- ORIGINAL 547 END -->

<!-- ORIGINAL 548 BEGIN -->

**Signature:** `int virtqueue_add_outbuf(struct virtqueue *vq, struct scatterlist *sg, unsigned int num, void *data, gfp_t gfp);`

<!-- ORIGINAL 548 END -->

<!-- ORIGINAL 550 BEGIN -->

- **Ngữ cảnh:** Driver Virtio thao tác trên cấu trúc bộ nhớ dùng chung (Shared Memory).

<!-- ORIGINAL 550 END -->

<!-- ORIGINAL 552 BEGIN -->

- Tham số chính:

<!-- ORIGINAL 552 END -->

<!-- ORIGINAL 554 BEGIN -->

- **vq:** Con trỏ tới Virtqueue (Split Ring).

<!-- ORIGINAL 554 END -->

<!-- ORIGINAL 556 BEGIN -->

- **sg:** Danh sách phân tán gồm 3 phần tử trỏ vào GPA.

<!-- ORIGINAL 556 END -->

<!-- ORIGINAL 558 BEGIN -->

- **Vai trò & Bộ nhớ:** Điền thông tin vào Descriptor Table:

<!-- ORIGINAL 558 END -->

<!-- ORIGINAL 560 BEGIN -->

- Desc\[0\].addr = GPA\_Header, cờ VRING\_DESC\_F\_NEXT.

<!-- ORIGINAL 560 END -->

<!-- ORIGINAL 562 BEGIN -->

- Desc\[1\].addr = 0x00105000 (GPA của 4096B Payload), Desc\[1\].len = 4096.

<!-- ORIGINAL 562 END -->

<!-- ORIGINAL 564 BEGIN -->

- Desc\[2\].addr = GPA\_Status, cờ VRING\_DESC\_F\_WRITE.

<!-- ORIGINAL 564 END -->

<!-- ORIGINAL 566 BEGIN -->

- Cập nhật avail-\>ring\[avail-\>idx % qsz\] = 0, tăng biến đếm avail-\>idx. Payload không bị copy, chỉ có descriptor được ghi vào RAM.

<!-- ORIGINAL 566 END -->

<!-- ORIGINAL 568 BEGIN -->

#### 6\. virtqueue\_notify() / iowrite16()

<!-- ORIGINAL 568 END -->

<!-- ORIGINAL 569 BEGIN -->

**Signature:** `void virtqueue_notify(struct virtqueue *vq);`

<!-- ORIGINAL 569 END -->

<!-- ORIGINAL 571 BEGIN -->

- **Ngữ cảnh:** Driver Guest phát tín hiệu Doorbell.

<!-- ORIGINAL 571 END -->

<!-- ORIGINAL 573 BEGIN -->

- Tác động phần cứng:

<!-- ORIGINAL 573 END -->

<!-- ORIGINAL 575 BEGIN -->

- **CPU thực thi lệnh ghi assembly:** movw %ax, (%rdi) vào thanh ghi MMIO Queue Notify trên PCI BAR của virtio-blk.

<!-- ORIGINAL 575 END -->

<!-- ORIGINAL 577 BEGIN -->

- Vì địa chỉ MMIO này không có ánh xạ thực trong bảng EPT (Extended Page Table), CPU phần cứng kích hoạt bẫy VM-Exit.

<!-- ORIGINAL 577 END -->

<!-- ORIGINAL 579 BEGIN -->

### GIAI ĐOẠN 2: RANH GIỚI ẢO HÓA — BẪY PHẦN CỨNG VM-EXIT & KVM

<!-- ORIGINAL 579 END -->

<!-- ORIGINAL 580 BEGIN -->

Tại thời điểm này, CPU chuyển quyền từ VMX Non-Root sang VMX Root (Host Kernel).

<!-- ORIGINAL 580 END -->

<!-- ORIGINAL 582 BEGIN -->

**Plaintext**

<!-- ORIGINAL 582 END -->

<!-- ORIGINAL 583-585 BEGIN -->

```text
[PHẦN CỨNG: VM-Exit] ──> vmx_handle_exit() ──> handle_ept_misconfig()
  └──> kvm_io_bus_write() ──> ioeventfd_write() ──> eventfd_signal()
         └──> [PHẦN CỨNG: VM-Entry quay lại Guest]
```

<!-- ORIGINAL 583-585 END -->

<!-- ORIGINAL 587 BEGIN -->

#### 7\. Phần cứng CPU: Kích hoạt VM-Exit

<!-- ORIGINAL 587 END -->

<!-- ORIGINAL 588 BEGIN -->

- **Cơ chế:** CPU phần cứng Intel/AMD tự động:

<!-- ORIGINAL 588 END -->

<!-- ORIGINAL 590 BEGIN -->

- Đóng băng luồng vCPU của Guest.

<!-- ORIGINAL 590 END -->

<!-- ORIGINAL 592 BEGIN -->

- Nạp toàn bộ trạng thái thanh ghi vCPU (RIP, RSP, CR3, segment registers) vào vùng nhớ cấu trúc VMCS (Virtual Machine Control Structure).

<!-- ORIGINAL 592 END -->

<!-- ORIGINAL 594 BEGIN -->

- Đổi trạng thái thực thi sang VMX Root Operation (Host Ring 0).

<!-- ORIGINAL 594 END -->

<!-- ORIGINAL 596 BEGIN -->

- Nạp con trỏ lệnh Host RIP trỏ tới hàm xử lý exit của nhân Linux Host: vmx\_handle\_exit().

<!-- ORIGINAL 596 END -->

<!-- ORIGINAL 598 BEGIN -->

#### 8\. vmx\_handle\_exit() → handle\_ept\_misconfig()

<!-- ORIGINAL 598 END -->

<!-- ORIGINAL 599 BEGIN -->

**Signature:** `int vmx_handle_exit(struct kvm_vcpu *vcpu);`

<!-- ORIGINAL 599 END -->

<!-- ORIGINAL 601 BEGIN -->

- **Ngữ cảnh:** Module nhân kvm-intel.ko trên Host.

<!-- ORIGINAL 601 END -->

<!-- ORIGINAL 603 BEGIN -->

- Tham số chính:

<!-- ORIGINAL 603 END -->

<!-- ORIGINAL 605 BEGIN -->

- **vcpu:** Cấu trúc mô tả vCPU đang bị ngắt.

<!-- ORIGINAL 605 END -->

<!-- ORIGINAL 607 BEGIN -->

- **vmcs\_read32(VM\_EXIT\_REASON):** Trả về mã lỗi do EPT misconfiguration (truy cập MMIO).

<!-- ORIGINAL 607 END -->

<!-- ORIGINAL 609 BEGIN -->

- **Vai trò:** Phân tích địa chỉ GPA bị truy cập, nhận diện đây là thao tác ghi vào chuông Doorbell của Virtio.

<!-- ORIGINAL 609 END -->

<!-- ORIGINAL 611 BEGIN -->

#### 9\. kvm\_io\_bus\_write() → ioeventfd\_write()

<!-- ORIGINAL 611 END -->

<!-- ORIGINAL 612 BEGIN -->

**Signature:** `int ioeventfd_write(struct kvm_vcpu *vcpu, struct kvm_io_device *dev, gpa_t addr, int len, const void *val);`

<!-- ORIGINAL 612 END -->

<!-- ORIGINAL 614 BEGIN -->

- **Ngữ cảnh:** Phân hệ ảo hóa I/O nhanh của KVM trong nhân Host.

<!-- ORIGINAL 614 END -->

<!-- ORIGINAL 616 BEGIN -->

- Tham số chính:

<!-- ORIGINAL 616 END -->

<!-- ORIGINAL 618 BEGIN -->

- **addr:** Địa chỉ GPA của thanh ghi Doorbell.

<!-- ORIGINAL 618 END -->

<!-- ORIGINAL 620 BEGIN -->

- **val:** Chỉ số hàng đợi (queue index).

<!-- ORIGINAL 620 END -->

<!-- ORIGINAL 622 BEGIN -->

- **Vai trò:** KVM tra cứu thiết bị bắt sự kiện bus và gọi trực tiếp eventfd\_signal().

<!-- ORIGINAL 622 END -->

<!-- ORIGINAL 624 BEGIN -->

#### 10\. eventfd\_signal() & Hoàn tất Fast Path

<!-- ORIGINAL 624 END -->

<!-- ORIGINAL 625 BEGIN -->

**Signature:** `__u64 eventfd_signal(struct eventfd_ctx *ctx, __u64 n);`

<!-- ORIGINAL 625 END -->

<!-- ORIGINAL 627 BEGIN -->

- **Ngữ cảnh:** Host Kernel IPC.

<!-- ORIGINAL 627 END -->

<!-- ORIGINAL 629 BEGIN -->

- Vai trò & Tác động:

<!-- ORIGINAL 629 END -->

<!-- ORIGINAL 631 BEGIN -->

- Tăng biến đếm nội bộ 64-bit của ioeventfd lên 1.

<!-- ORIGINAL 631 END -->

<!-- ORIGINAL 633 BEGIN -->

- Đánh thức luồng đang chờ file descriptor này bằng cách đưa nó vào hàng đợi chạy (wake\_up\_locked\_poll).

<!-- ORIGINAL 633 END -->

<!-- ORIGINAL 635 BEGIN -->

- **Tối ưu quan trọng:** KVM không thoát ra ngoài QEMU (không trả về vòng lặp ioctl(KVM\_RUN)). KVM lập tức thực hiện VM-Entry đưa vCPU quay trở lại chạy tiếp mã Guest. Việc xử lý I/O được bàn giao hoàn toàn cho Host IOThread chạy song song.

<!-- ORIGINAL 635 END -->

<!-- ORIGINAL 637 BEGIN -->

### GIAI ĐOẠN 3: HOST USERSPACE — QEMU IOTHREAD & DỊCH ĐỊA CHỈ

<!-- ORIGINAL 637 END -->

<!-- ORIGINAL 638 BEGIN -->

Luồng QEMU IOThread chạy độc lập trên một CPU Core của Host, đón nhận tín hiệu và phát Syscall bất đồng bộ.

<!-- ORIGINAL 638 END -->

<!-- ORIGINAL 640 BEGIN -->

**Plaintext**

<!-- ORIGINAL 640 END -->

<!-- ORIGINAL 641-643 BEGIN -->

```text
epoll_wait() thức dậy ──> virtio_blk_data_plane_handle_output()
  └──> virtqueue_pop() (Dịch GPA -> HVA)
         └──> virtio_blk_handle_request() ──> laio_do_submit() ──> io_submit()
```

<!-- ORIGINAL 641-643 END -->

<!-- ORIGINAL 645 BEGIN -->

#### 11\. epoll\_wait() thức dậy

<!-- ORIGINAL 645 END -->

<!-- ORIGINAL 646 BEGIN -->

**Signature:** `int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);`

<!-- ORIGINAL 646 END -->

<!-- ORIGINAL 648 BEGIN -->

- **Ngữ cảnh:** Vòng lặp sự kiện AioContext của QEMU IOThread (Host User Space - Ring 3).

<!-- ORIGINAL 648 END -->

<!-- ORIGINAL 650 BEGIN -->

- **Tác động:** Tín hiệu từ ioeventfd ở Giai đoạn 2 làm hàm này trả về. Bộ điều phối CPU của Linux thực hiện Context Switch, đưa luồng QEMU IOThread vào thực thi.

<!-- ORIGINAL 650 END -->

<!-- ORIGINAL 652 BEGIN -->

#### 12\. virtio\_blk\_data\_plane\_handle\_output()

<!-- ORIGINAL 652 END -->

<!-- ORIGINAL 653 BEGIN -->

**Signature:** `bool virtio_blk_data_plane_handle_output(VirtIODevice *vdev, VirtQueue *vq);`

<!-- ORIGINAL 653 END -->

<!-- ORIGINAL 655 BEGIN -->

- **Ngữ cảnh:** QEMU Block Layer Datapath (User Space).

<!-- ORIGINAL 655 END -->

<!-- ORIGINAL 657 BEGIN -->

- **Vai trò:** Hàm callback chuyên trách xử lý các Descriptor mới được đẩy vào Virtqueue.

<!-- ORIGINAL 657 END -->

<!-- ORIGINAL 659 BEGIN -->

#### 13\. virtqueue\_pop() → address\_space\_map()

<!-- ORIGINAL 659 END -->

<!-- ORIGINAL 660 BEGIN -->

**Signature:** `void *virtqueue_pop(VirtQueue *vq, size_t sz);`

<!-- ORIGINAL 660 END -->

<!-- ORIGINAL 662 BEGIN -->

- **Ngữ cảnh:** QEMU dịch chuyển không gian bộ nhớ.

<!-- ORIGINAL 662 END -->

<!-- ORIGINAL 664 BEGIN -->

- Tham số chính:

<!-- ORIGINAL 664 END -->

<!-- ORIGINAL 666 BEGIN -->

- Đọc Desc\[1\].addr (GPA 0x00105000).

<!-- ORIGINAL 666 END -->

<!-- ORIGINAL 668 BEGIN -->

- Vai trò & Dịch địa chỉ (Điểm cốt lõi):

<!-- ORIGINAL 668 END -->

<!-- ORIGINAL 670 BEGIN -->

- QEMU gọi hàm ánh xạ bộ nhớ nội bộ qemu\_map\_ram\_ptr():

<!-- ORIGINAL 670 END -->

<!-- ORIGINAL 672 BEGIN -->

- \$\$\\text\{GPA \} (0x00105000) \\longrightarrow \\text\{Host Virtual Address (HVA: \} 0x7f11a000)\$\$

<!-- ORIGINAL 672 END -->

<!-- ORIGINAL 673 BEGIN -->

- Cấu trúc VirtQueueElement được tạo ra, chứa con trỏ iov\_base = 0x7f11a000 trỏ thẳng vào vùng RAM máy ảo trên không gian của QEMU.

<!-- ORIGINAL 673 END -->

<!-- ORIGINAL 675 BEGIN -->

- **Kiểm tra Zero-copy:** Vì các trang nhớ căn chỉnh chuẩn 4 KiB, QEMU không gọi qemu\_iovec\_to\_buf(), không tạo Bounce Buffer, dữ liệu payload 4096B giữ nguyên vị trí vật lý.

<!-- ORIGINAL 675 END -->

<!-- ORIGINAL 677 BEGIN -->

#### 14\. laio\_do\_submit() → io\_submit()

<!-- ORIGINAL 677 END -->

<!-- ORIGINAL 678 BEGIN -->

**Signature:** `int io_submit(aio_context_t ctx_id, long nr, struct iocb **iocbpp);`

<!-- ORIGINAL 678 END -->

<!-- ORIGINAL 680 BEGIN -->

- **Ngữ cảnh:** QEMU phát lời gọi hệ thống xuống Host Kernel (Ring 3 → Ring 0).

<!-- ORIGINAL 680 END -->

<!-- ORIGINAL 682 BEGIN -->

- Tham số chính:

<!-- ORIGINAL 682 END -->

<!-- ORIGINAL 684 BEGIN -->

- **ctx\_id:** Bối cảnh Linux Native AIO của IOThread.

<!-- ORIGINAL 684 END -->

<!-- ORIGINAL 686 BEGIN -->

- **iocb-\>aio\_fildes:** File descriptor đại diện cho /dev/mapper/mpath0.

<!-- ORIGINAL 686 END -->

<!-- ORIGINAL 688 BEGIN -->

- **iocb-\>aio\_lio\_opcode:** IOCB\_CMD\_PWRITEV.

<!-- ORIGINAL 688 END -->

<!-- ORIGINAL 690 BEGIN -->

- **iocb-\>aio\_buf:** Con trỏ HVA 0x7f11a000.

<!-- ORIGINAL 690 END -->

<!-- ORIGINAL 692 BEGIN -->

- **iocb-\>aio\_nbytes:** 4096.

<!-- ORIGINAL 692 END -->

<!-- ORIGINAL 694 BEGIN -->

- **Tác động:** Chỉ lệnh assembly syscall được kích hoạt trên Host.

<!-- ORIGINAL 694 END -->

<!-- ORIGINAL 696 BEGIN -->

### GIAI ĐOẠN 4: HOST KERNEL STORAGE STACK (blk-mq, dm-multipath, SCSI)

<!-- ORIGINAL 696 END -->

<!-- ORIGINAL 697 BEGIN -->

Host Kernel tiếp nhận yêu cầu ghi trực tiếp (O\_DIRECT từ cờ của io\_submit), định tuyến qua Multipath và dịch sang lệnh SCSI.

<!-- ORIGINAL 697 END -->

<!-- ORIGINAL 699 BEGIN -->

**Plaintext**

<!-- ORIGINAL 699 END -->

<!-- ORIGINAL 700-702 BEGIN -->

```text
__x64_sys_io_submit() ──> io_submit_one() ──> blkdev_write_iter()
  └──> blk_mq_submit_bio() ──> dm_submit_bio() (Multipath Selector)
         └──> scsi_queue_rq() (Tạo CDB 0x2A) ──> iscsi_queuecommand()
```

<!-- ORIGINAL 700-702 END -->

<!-- ORIGINAL 704 BEGIN -->

#### 15\. \_\_x64\_sys\_io\_submit() → io\_submit\_one()

<!-- ORIGINAL 704 END -->

<!-- ORIGINAL 705 BEGIN -->

**Signature:** `static int io_submit_one(struct kioctx *ctx, struct iocb __user *user_iocb, bool compat);`

<!-- ORIGINAL 705 END -->

<!-- ORIGINAL 707 BEGIN -->

- **Ngữ cảnh:** Host Kernel Native AIO Subsystem (Ring 0).

<!-- ORIGINAL 707 END -->

<!-- ORIGINAL 709 BEGIN -->

- **Vai trò:** Đọc cấu trúc iocb từ User Space của QEMU, khởi tạo đối tượng struct kiocb.

<!-- ORIGINAL 709 END -->

<!-- ORIGINAL 711 BEGIN -->

#### 16\. blkdev\_write\_iter() → iov\_iter\_get\_pages2()

<!-- ORIGINAL 711 END -->

<!-- ORIGINAL 712 BEGIN -->

**Signature:** `ssize_t iov_iter_get_pages2(struct iov_iter *i, struct page **pages, size_t maxsize, unsigned max_pages, size_t *start);`

<!-- ORIGINAL 712 END -->

<!-- ORIGINAL 714 BEGIN -->

- **Ngữ cảnh:** Host Memory Management Subsystem.

<!-- ORIGINAL 714 END -->

<!-- ORIGINAL 716 BEGIN -->

- Vai trò & Dịch địa chỉ:

<!-- ORIGINAL 716 END -->

<!-- ORIGINAL 718 BEGIN -->

- Tiếp nhận con trỏ HVA 0x7f11a000.

<!-- ORIGINAL 718 END -->

<!-- ORIGINAL 720 BEGIN -->

- MMU Host tra cứu bảng phân trang của Host (init\_mm.pgd), dịch:

<!-- ORIGINAL 720 END -->

<!-- ORIGINAL 722 BEGIN -->

- \$\$\\text\{HVA \} (0x7f11a000) \\longrightarrow \\text\{Host Physical Address (Host PA: \} 0x1a2b3000)\$\$

<!-- ORIGINAL 722 END -->

<!-- ORIGINAL 723 BEGIN -->

- Gọi pin\_user\_pages() ghim cứng trang RAM vật lý 0x1a2b3000 của Host.

<!-- ORIGINAL 723 END -->

<!-- ORIGINAL 725 BEGIN -->

- **Đóng gói vào struct bio:** bio-\>bi\_io\_vec\[0\].bv\_page trỏ vào Host PA 0x1a2b3000.

<!-- ORIGINAL 725 END -->

<!-- ORIGINAL 727 BEGIN -->

#### 17\. dm\_submit\_bio() → multipath\_clone\_and\_map()

<!-- ORIGINAL 727 END -->

<!-- ORIGINAL 728 BEGIN -->

**Signature:** `static int multipath_clone_and_map(struct dm_target *ti, struct request *rq, union map_info *map_context, struct request **clone);`

<!-- ORIGINAL 728 END -->

<!-- ORIGINAL 730 BEGIN -->

- **Ngữ cảnh:** Phân hệ Device Mapper (dm\_multipath.ko).

<!-- ORIGINAL 730 END -->

<!-- ORIGINAL 732 BEGIN -->

- Tham số chính:

<!-- ORIGINAL 732 END -->

<!-- ORIGINAL 734 BEGIN -->

- **ti:** Target instance đại diện cho /dev/mapper/mpath0.

<!-- ORIGINAL 734 END -->

<!-- ORIGINAL 736 BEGIN -->

- Vai trò & Quyết định đường đi:

<!-- ORIGINAL 736 END -->

<!-- ORIGINAL 738 BEGIN -->

- Bộ chọn đường (Path Selector, ví dụ thuật toán service-time) kiểm tra tải các đường dẫn vật lý: chọn /dev/sda (nối qua NIC 1 tới Portal 1).

<!-- ORIGINAL 738 END -->

<!-- ORIGINAL 740 BEGIN -->

- Hàm clone tạo bản sao metadata của request và chuyển hướng con trỏ xuống hàng đợi request\_queue của driver SCSI bên dưới.

<!-- ORIGINAL 740 END -->

<!-- ORIGINAL 742 BEGIN -->

#### 18\. scsi\_queue\_rq() → sd\_init\_command()

<!-- ORIGINAL 742 END -->

<!-- ORIGINAL 743 BEGIN -->

**Signature:** `blk_status_t scsi_queue_rq(struct blk_mq_hw_ctx *hctx, const struct blk_mq_queue_data *bd);`

<!-- ORIGINAL 743 END -->

<!-- ORIGINAL 745 BEGIN -->

- **Ngữ cảnh:** Tầng SCSI Upper/Mid Layer (sd\_mod.ko, scsi\_mod.ko).

<!-- ORIGINAL 745 END -->

<!-- ORIGINAL 747 BEGIN -->

- Vai trò & Dịch lệnh:

<!-- ORIGINAL 747 END -->

<!-- ORIGINAL 749 BEGIN -->

- Khởi tạo cấu trúc struct scsi\_cmnd.

<!-- ORIGINAL 749 END -->

<!-- ORIGINAL 751 BEGIN -->

- **Xây dựng SCSI Command Descriptor Block (CDB):** Lệnh WRITE\_10 (Opcode 0x2A), ghi 8 sectors vào LBA chỉ định.

<!-- ORIGINAL 751 END -->

<!-- ORIGINAL 753 BEGIN -->

- Chuyển mảng bio\_vec thành danh sách phân tán bộ nhớ (Scatter-Gather List - SG List): phần tử sg\[0\] mang địa chỉ Host PA 0x1a2b3000, độ dài 4096B.

<!-- ORIGINAL 753 END -->

<!-- ORIGINAL 755 BEGIN -->

### GIAI ĐOẠN 5: GIAO THỨC iSCSI & CARD MẠNG NIC DMA ĐẨY RA MẠNG

<!-- ORIGINAL 755 END -->

<!-- ORIGINAL 756 BEGIN -->

SCSI Command được đóng gói thành iSCSI PDU, chuyển qua TCP stack và được phần cứng NIC hút từ RAM bằng DMA.

<!-- ORIGINAL 756 END -->

<!-- ORIGINAL 758 BEGIN -->

**Plaintext**

<!-- ORIGINAL 758 END -->

<!-- ORIGINAL 759-762 BEGIN -->

```text
iscsi_queuecommand() ──> iscsi_prep_scsi_cmd_pdu() ──> kernel_sendpage()
  └──> tcp_write_xmit() ──> dev_queue_xmit()
         └──> dma_map_page() (IOMMU: Host PA -> IOVA) ──> writel() (PCIe Doorbell NIC)
                └──> [PHẦN CỨNG NIC DMA BURST READ] ──> Phát xung Laser ra dây
```

<!-- ORIGINAL 759-762 END -->

<!-- ORIGINAL 764 BEGIN -->

#### 19\. iscsi\_queuecommand() → iscsi\_prep\_scsi\_cmd\_pdu()

<!-- ORIGINAL 764 END -->

<!-- ORIGINAL 765 BEGIN -->

**Signature:** `int iscsi_queuecommand(struct Scsi_Host *host, struct scsi_cmnd *sc);`

<!-- ORIGINAL 765 END -->

<!-- ORIGINAL 767 BEGIN -->

- **Ngữ cảnh:** Driver iscsi\_tcp.ko / libiscsi.ko.

<!-- ORIGINAL 767 END -->

<!-- ORIGINAL 769 BEGIN -->

- Vai trò:

<!-- ORIGINAL 769 END -->

<!-- ORIGINAL 771 BEGIN -->

- **Khởi tạo iSCSI Command PDU Header (48 bytes):** Opcode = 0x01 (SCSI Command), cờ ghi W, LUN ID, gán mã định danh nhiệm vụ ITT (Initiator Task Tag).

<!-- ORIGINAL 771 END -->

<!-- ORIGINAL 773 BEGIN -->

- Ghép Header và SG-List (trỏ vào payload) chuẩn bị đẩy vào socket.

<!-- ORIGINAL 773 END -->

<!-- ORIGINAL 775 BEGIN -->

#### 20\. kernel\_sendpage() / sock\_sendmsg() → tcp\_write\_xmit()

<!-- ORIGINAL 775 END -->

<!-- ORIGINAL 776 BEGIN -->

**Signature:** `static bool tcp_write_xmit(struct sock *sk, unsigned int mss_now, int nonagle, int push_one, gfp_t gfp);`

<!-- ORIGINAL 776 END -->

<!-- ORIGINAL 778 BEGIN -->

- **Ngữ cảnh:** Linux Kernel TCP/IP Network Stack.

<!-- ORIGINAL 778 END -->

<!-- ORIGINAL 780 BEGIN -->

- Vai trò & Cơ chế Zero-copy Socket:

<!-- ORIGINAL 780 END -->

<!-- ORIGINAL 782 BEGIN -->

- Cấp phát cấu trúc quản lý mạng struct sk\_buff (skb).

<!-- ORIGINAL 782 END -->

<!-- ORIGINAL 784 BEGIN -->

- TCP stack gắn TCP Header (Port 3260) và IP Header.

<!-- ORIGINAL 784 END -->

<!-- ORIGINAL 786 BEGIN -->

- Khối dữ liệu 4096B được gắn vào mảng phân mảnh skb\_shinfo(skb)-\>frags\[0\] trỏ trực tiếp tới trang Host PA 0x1a2b3000. Không dùng memcpy để chép dữ liệu vào bộ đệm socket.

<!-- ORIGINAL 786 END -->

<!-- ORIGINAL 788 BEGIN -->

#### 21\. dma\_map\_page() (IOMMU Translation)

<!-- ORIGINAL 788 END -->

<!-- ORIGINAL 789 BEGIN -->

**Signature:** `dma_addr_t dma_map_page_attrs(struct device *dev, struct page *page, size_t offset, size_t size, enum dma_data_direction dir, unsigned long attrs);`

<!-- ORIGINAL 789 END -->

<!-- ORIGINAL 791 BEGIN -->

- **Ngữ cảnh:** Linux DMA Mapping API & Phần cứng Intel VT-d / AMD-Vi.

<!-- ORIGINAL 791 END -->

<!-- ORIGINAL 793 BEGIN -->

- Tham số chính:

<!-- ORIGINAL 793 END -->

<!-- ORIGINAL 795 BEGIN -->

- **page:** Khung trang Host PA 0x1a2b3000.

<!-- ORIGINAL 795 END -->

<!-- ORIGINAL 797 BEGIN -->

- **dir:** DMA\_TO\_DEVICE.

<!-- ORIGINAL 797 END -->

<!-- ORIGINAL 799 BEGIN -->

- Tác động phần cứng:

<!-- ORIGINAL 799 END -->

<!-- ORIGINAL 801 BEGIN -->

- Bộ vi xử lý IOMMU trên bo mạch chủ lập trình bảng trang dịch địa chỉ:

<!-- ORIGINAL 801 END -->

<!-- ORIGINAL 803 BEGIN -->

- \$\$\\text\{Host PA \} (0x1a2b3000) \\longrightarrow \\text\{IOVA / DMA Address \} (0x80001000)\$\$

<!-- ORIGINAL 803 END -->

<!-- ORIGINAL 804 BEGIN -->

- Trả về địa chỉ bus IOVA mà Card mạng có thể truy cập được qua bus PCIe.

<!-- ORIGINAL 804 END -->

<!-- ORIGINAL 806 BEGIN -->

#### 22\. nic\_start\_xmit() → writel() (Kích hoạt NIC Hardware)

<!-- ORIGINAL 806 END -->

<!-- ORIGINAL 807 BEGIN -->

**Signature:** `void writel(u32 val, void __iomem *addr);`

<!-- ORIGINAL 807 END -->

<!-- ORIGINAL 809 BEGIN -->

- **Ngữ cảnh:** Driver card mạng Ethernet (ví dụ ixgbe, mlx5\_core).

<!-- ORIGINAL 809 END -->

<!-- ORIGINAL 811 BEGIN -->

- Vai trò & Tác động bus:

<!-- ORIGINAL 811 END -->

<!-- ORIGINAL 813 BEGIN -->

- **Driver ghi Descriptor vào vòng truyền (TX Ring Buffer trên RAM):** Descriptor nạp con trỏ IOVA 0x80001000 và độ dài 4096B.

<!-- ORIGINAL 813 END -->

<!-- ORIGINAL 815 BEGIN -->

- CPU thực thi lệnh ghi MMIO writel() vào thanh ghi chuông Doorbell của card mạng.

<!-- ORIGINAL 815 END -->

<!-- ORIGINAL 817 BEGIN -->

- Phần cứng phát sinh một gói tin PCIe TLP Memory Write (MWr) truyền qua các làn cáp PCIe vào ASIC của NIC.

<!-- ORIGINAL 817 END -->

<!-- ORIGINAL 819 BEGIN -->

#### 23\. Phần cứng: NIC Scatter-Gather DMA Fetch & Serialization

<!-- ORIGINAL 819 END -->

<!-- ORIGINAL 820 BEGIN -->

- Thực thi phần cứng:

<!-- ORIGINAL 820 END -->

<!-- ORIGINAL 822 BEGIN -->

- Chip điều khiển NIC (Bus Master) phát tín hiệu PCIe Memory Read dồn dập (Burst Read) ngược lên RAM máy chủ.

<!-- ORIGINAL 822 END -->

<!-- ORIGINAL 824 BEGIN -->

- Gói tin đi qua IOMMU, IOMMU dịch IOVA 0x80001000 về RAM vật lý 0x1a2b3000.

<!-- ORIGINAL 824 END -->

<!-- ORIGINAL 826 BEGIN -->

- DMA Engine của NIC rút 4096 bytes dữ liệu từ RAM nạp vào bộ đệm SRAM của NIC (CPU tải 0%).

<!-- ORIGINAL 826 END -->

<!-- ORIGINAL 828 BEGIN -->

- Bộ phát quang SerDes biến đổi chuỗi bit nhị phân thành các xung ánh sáng (photons) truyền qua module quang SFP+ dọc theo sợi cáp quang đến Storage Target.

<!-- ORIGINAL 828 END -->

<!-- ORIGINAL 830 BEGIN -->

### GIAI ĐOẠN 6: TARGET LƯU TRỮ VÀ HOÀN TẤT TRUYỀN DẪN VỀ HOST

<!-- ORIGINAL 830 END -->

<!-- ORIGINAL 831 BEGIN -->

**Plaintext**

<!-- ORIGINAL 831 END -->

<!-- ORIGINAL 832-834 BEGIN -->

```text
[Storage Target ghi Flash] ──> Gửi TCP ACK / iSCSI Response
  └──> [NIC Host: Bắn ngắt MSI-X] ──> napi_schedule() ──> iscsi_tcp_data_recv()
         └──> scsi_finish_command() ──> aio_complete() ──> eventfd_signal()
```

<!-- ORIGINAL 832-834 END -->

<!-- ORIGINAL 836 BEGIN -->

#### 24\. Xử lý tại Storage Target (NetApp SDS / Flash Array)

<!-- ORIGINAL 836 END -->

<!-- ORIGINAL 837 BEGIN -->

- Bộ điều khiển Target tiếp nhận xung quang, bóc tách Ethernet → IP → TCP → iSCSI PDU → SCSI CDB.

<!-- ORIGINAL 837 END -->

<!-- ORIGINAL 839 BEGIN -->

- Ghi 4096 bytes vào Bộ đệm an toàn NVRAM/DRAM có tụ điện bảo vệ.

<!-- ORIGINAL 839 END -->

<!-- ORIGINAL 841 BEGIN -->

- Bộ điều khiển mảng đĩa phát lệnh nạp điện tích ghi bền vững vào các ô nhớ NAND Flash Array.

<!-- ORIGINAL 841 END -->

<!-- ORIGINAL 843 BEGIN -->

- Target tạo gói tin iSCSI Response PDU (Status = 0x00 GOOD, mang đúng mã định danh ITT ban đầu) và gửi về Host qua TCP.

<!-- ORIGINAL 843 END -->

<!-- ORIGINAL 845 BEGIN -->

#### 25\. Phần cứng: Tín hiệu ngắt MSI-X Interrupt

<!-- ORIGINAL 845 END -->

<!-- ORIGINAL 846 BEGIN -->

- Cổng quang trên Host tiếp nhận tín hiệu, NIC DMA ghi bản tin phản hồi vào RX Ring.

<!-- ORIGINAL 846 END -->

<!-- ORIGINAL 848 BEGIN -->

- **NIC kích hoạt ngắt MSI-X:** Bắn gói tin PCIe MWr vào thanh ghi 0xFEE00000 của bộ điều khiển ngắt Local APIC (LAPIC) của CPU Host.

<!-- ORIGINAL 848 END -->

<!-- ORIGINAL 850 BEGIN -->

- Một CPU Core của Host bị ngắt dòng lệnh, nhảy vào thực thi trình phục vụ ngắt Hard-IRQ.

<!-- ORIGINAL 850 END -->

<!-- ORIGINAL 852 BEGIN -->

#### 26\. iscsi\_tcp\_data\_recv() → scsi\_finish\_command()

<!-- ORIGINAL 852 END -->

<!-- ORIGINAL 853 BEGIN -->

**Signature:** `void scsi_finish_command(struct scsi_cmnd *cmd);`

<!-- ORIGINAL 853 END -->

<!-- ORIGINAL 855 BEGIN -->

- **Ngữ cảnh:** Tầng ngắt mềm mạng (NET\_RX\_SOFTIRQ / NAPI poll) trên Host.

<!-- ORIGINAL 855 END -->

<!-- ORIGINAL 857 BEGIN -->

- Vai trò:

<!-- ORIGINAL 857 END -->

<!-- ORIGINAL 859 BEGIN -->

- Bóc gói TCP, đọc trường ITT, tra bảng băm để lấy đúng struct scsi\_cmnd.

<!-- ORIGINAL 859 END -->

<!-- ORIGINAL 861 BEGIN -->

- **dma\_unmap\_page():** IOMMU hủy ánh xạ IOVA.

<!-- ORIGINAL 861 END -->

<!-- ORIGINAL 863 BEGIN -->

- **Báo cho tầng Block Layer:** gọi blk\_mq\_complete\_request().

<!-- ORIGINAL 863 END -->

<!-- ORIGINAL 865 BEGIN -->

#### 27\. aio\_complete() → eventfd\_signal()

<!-- ORIGINAL 865 END -->

<!-- ORIGINAL 866 BEGIN -->

**Signature:** `void aio_complete(struct kiocb *iocb, long res, long res2);`

<!-- ORIGINAL 866 END -->

<!-- ORIGINAL 868 BEGIN -->

- **Ngữ cảnh:** Host Kernel Linux Native AIO.

<!-- ORIGINAL 868 END -->

<!-- ORIGINAL 870 BEGIN -->

- Vai trò:

<!-- ORIGINAL 870 END -->

<!-- ORIGINAL 872 BEGIN -->

- Ghi kết quả thành công (res = 4096) vào Completion Ring của AIO.

<!-- ORIGINAL 872 END -->

<!-- ORIGINAL 874 BEGIN -->

- Giải phóng unpin\_user\_pages() cho khung trang Host PA 0x1a2b3000.

<!-- ORIGINAL 874 END -->

<!-- ORIGINAL 876 BEGIN -->

- Kích hoạt eventfd\_signal() trên file descriptor liên kết giữa Kernel AIO và QEMU IOThread.

<!-- ORIGINAL 876 END -->

<!-- ORIGINAL 878 BEGIN -->

### GIAI ĐOẠN 7: QEMU CẬP NHẬT VIRTQUEUE VÀ TIÊM NGẮT ẢO VÀO VM

<!-- ORIGINAL 878 END -->

<!-- ORIGINAL 879 BEGIN -->

QEMU thu hoạch kết quả, ghi nhận vào Virtqueue trên RAM Guest và kích hoạt KVM tiêm ngắt cho vCPU.

<!-- ORIGINAL 879 END -->

<!-- ORIGINAL 881 BEGIN -->

**Plaintext**

<!-- ORIGINAL 881 END -->

<!-- ORIGINAL 882-884 BEGIN -->

```text
io_getevents() ──> virtio_blk_req_complete() (Ghi Used Ring)
  └──> eventfd_write(irqfd) ──> kvm_set_irq() ──> [Virtual IRQ Injection qua vAPIC]
         └──> [Guest OS: virtio_blk_done() ──> fio nhận kết quả]
```

<!-- ORIGINAL 882-884 END -->

<!-- ORIGINAL 886 BEGIN -->

#### 28\. io\_getevents() → virtio\_blk\_req\_complete()

<!-- ORIGINAL 886 END -->

<!-- ORIGINAL 887 BEGIN -->

**Signature:** `void virtio_blk_req_complete(VirtIOBlockReq *req, int8_t status);`

<!-- ORIGINAL 887 END -->

<!-- ORIGINAL 889 BEGIN -->

- **Ngữ cảnh:** Luồng QEMU IOThread thức dậy (Host User Space).

<!-- ORIGINAL 889 END -->

<!-- ORIGINAL 891 BEGIN -->

- Tham số chính:

<!-- ORIGINAL 891 END -->

<!-- ORIGINAL 893 BEGIN -->

- **status:** VIRTIO\_BLK\_S\_OK (0).

<!-- ORIGINAL 893 END -->

<!-- ORIGINAL 895 BEGIN -->

- Vai trò & Cập nhật Virtqueue:

<!-- ORIGINAL 895 END -->

<!-- ORIGINAL 897 BEGIN -->

- QEMU ghi giá trị 0 vào Status byte (Desc\[2\]) trên RAM của Guest thông qua con trỏ HVA.

<!-- ORIGINAL 897 END -->

<!-- ORIGINAL 899 BEGIN -->

- Ghi chỉ số chuỗi Descriptor vào Used Ring:

<!-- ORIGINAL 899 END -->

<!-- ORIGINAL 901 BEGIN -->

**C**

<!-- ORIGINAL 901 END -->

<!-- ORIGINAL 902 BEGIN -->

```c
vring_used_write_idx(vq, req->elem.index);
```

<!-- ORIGINAL 902 END -->

<!-- ORIGINAL 905 BEGIN -->

- Tăng con trỏ used-\>idx.

<!-- ORIGINAL 905 END -->

<!-- ORIGINAL 907 BEGIN -->

#### 29\. Báo hiệu qua irqfd: eventfd\_write()

<!-- ORIGINAL 907 END -->

<!-- ORIGINAL 908 BEGIN -->

**Signature:** `int eventfd_write(int fd, eventfd_t value);`

<!-- ORIGINAL 908 END -->

<!-- ORIGINAL 910 BEGIN -->

- **Ngữ cảnh:** QEMU User Space gọi Syscall báo cho KVM.

<!-- ORIGINAL 910 END -->

<!-- ORIGINAL 912 BEGIN -->

- **Tác động:** Ghi giá trị 1 vào file descriptor irqfd đã được gắn kết với đường ngắt của VM.

<!-- ORIGINAL 912 END -->

<!-- ORIGINAL 914 BEGIN -->

#### 30\. kvm\_set\_irq() & Tiêm ngắt ảo (Virtual IRQ Injection)

<!-- ORIGINAL 914 END -->

<!-- ORIGINAL 915 BEGIN -->

**Signature:** `int kvm_set_irq(struct kvm *kvm, int irq_source_id, u32 irq, int level, bool line_status);`

<!-- ORIGINAL 915 END -->

<!-- ORIGINAL 917 BEGIN -->

- **Ngữ cảnh:** Module KVM trong nhân Host.

<!-- ORIGINAL 917 END -->

<!-- ORIGINAL 919 BEGIN -->

- Tác động phần cứng vi kiến trúc:

<!-- ORIGINAL 919 END -->

<!-- ORIGINAL 921 BEGIN -->

- KVM lập trình trực tiếp vào bộ điều khiển ngắt ảo (vAPIC) của vCPU trong cấu trúc VMCS.

<!-- ORIGINAL 921 END -->

<!-- ORIGINAL 923 BEGIN -->

- **Nếu phần cứng hỗ trợ Intel APIC-v (Posted Interrupts):** CPU phần cứng tự động cập nhật bảng Posted-Interrupt Descriptor và gửi một thông báo IPI đặc biệt để vCPU tự nhận ngắt mà không cần ép vCPU phải VM-Exit.

<!-- ORIGINAL 923 END -->

<!-- ORIGINAL 925 BEGIN -->

- **Nếu không có APIC-v:** vCPU bị bẫy để KVM nạp vector ngắt vào bảng IDT ảo của Guest.

<!-- ORIGINAL 925 END -->

<!-- ORIGINAL 927 BEGIN -->

#### 31\. Guest Kernel: virtio\_blk\_done() → Đánh thức ứng dụng

<!-- ORIGINAL 927 END -->

<!-- ORIGINAL 928 BEGIN -->

- **Ngữ cảnh:** Trình phục vụ ngắt (ISR) của driver virtio-blk trong Guest OS.

<!-- ORIGINAL 928 END -->

<!-- ORIGINAL 930 BEGIN -->

- Hành động cuối cùng:

<!-- ORIGINAL 930 END -->

<!-- ORIGINAL 932 BEGIN -->

- Thu hồi các Descriptor từ Used Ring.

<!-- ORIGINAL 932 END -->

<!-- ORIGINAL 934 BEGIN -->

- **Gọi hàm hoàn tất của Block Layer:** blk\_mq\_end\_request(rq, BLK\_STS\_OK).

<!-- ORIGINAL 934 END -->

<!-- ORIGINAL 936 BEGIN -->

- Bộ điều phối của Guest chuyển trạng thái tiến trình fio từ ngủ sang TASK\_RUNNING.

<!-- ORIGINAL 936 END -->

<!-- ORIGINAL 938 BEGIN -->

- CPU Guest thực thi chỉ lệnh sysretq, trả về giá trị 4096 cho hàm write(). Lệnh ghi từ VM xuống chip Flash chính thức kết thúc toàn bộ hành trình.

<!-- ORIGINAL 938 END -->

<!-- ORIGINAL 939 BEGIN -->

## BẢNG ĐỐI CHIẾU CHUỖI BIẾN ĐỔI BỘ NHỚ VÀ ĐỊA CHỈ TRUY CẬP

<!-- ORIGINAL 939 END -->

<!-- ORIGINAL 940 BEGIN -->

| Giai đoạn | Hàm xử lý chính | Định dạng địa chỉ sử dụng | Vị trí Payload thực tế | Hành vi sao chép bộ nhớ |
| --- | --- | --- | --- | --- |
| Guest App → Driver | sys\_write → virtqueue\_add | GVA → GPA (0x00105000) | RAM vật lý máy chủ | 0 copy payload (Ghim trang GPA) |
| Virtualization Trap | vmx\_handle\_exit → ioeventfd\_write | Địa chỉ MMIO Doorbell | RAM vật lý máy chủ | 0 copy (Phần cứng bẫy VM-Exit) |
| QEMU IOThread | virtqueue\_pop → io\_submit | GPA → HVA (0x7f11a000) | RAM vật lý máy chủ | 0 copy (Ánh xạ bảng trang) |
| Host Kernel → SCSI | blkdev\_write\_iter → scsi\_queue\_rq | HVA → Host PA (0x1a2b3000) | RAM vật lý máy chủ | 0 copy (pin\_user\_pages) |
| Mạng & Hardware DMA | dma\_map\_page → nic\_start\_xmit | Host PA → IOVA (0x80001000) | RAM → NIC SRAM | 0 CPU copy (NIC Hardware DMA) |
| Fabric → Flash Media | Xử lý tại NetApp Target Controller | Cáp quang / NVRAM / NAND | NIC SRAM → NAND Flash | Phần cứng Target tự ghi |

<!-- ORIGINAL 940 END -->

<!-- ORIGINAL 941 BEGIN -->

## 

CÁC THIẾT BỊ ẢO & GIAO DIỆN LOGIC XUẤT HIỆN TRONG DATAPATH 
Plaintext

<!-- ORIGINAL 941 END -->

<!-- ORIGINAL 942-989 BEGIN -->

```text
========================================================================================================================
             SƠ ĐỒ TỔNG THỂ CÁC THIẾT BỊ ẢO & GIAO DIỆN LOGIC XUẤT HIỆN TRONG DATAPATH
========================================================================================================================

 [ GUEST VM ]
   │
   ├─ [1] /dev/vdb (Guest Virtual Block Device)
   │    └── Giao diện khối ảo cho ứng dụng (fio)
   │
   ├─ [2] virtio-blk-pci (Emulated PCI Device - BDF ảo 00:05.0)
   │    └── Thanh ghi ảo MMIO Doorbell (BAR0 / BAR4)
   │
   └─ [3] vAPIC (Virtual Local APIC trong vCPU)
        └── Bộ điều khiển ngắt ảo tiếp nhận Virtual IRQ

 ══════════════════════════════════════════════════════════════════════════════════ [ Ranh giới KVM / QEMU ]
 [ HYPERVISOR & HOST USERSPACE ]
   │
   ├─ [4] ioeventfd (Anonymous Inode / Fast IPC Interface)
   │    └── Cầu nối kernel-to-userspace kích hoạt khi vCPU gõ Doorbell ảo
   │
   ├─ [5] QEMU Emulated virtio-blk Controller (Cấu trúc C trong QEMU Process)
   │    └── Thực thể giả lập phần cứng Virtio, ánh xạ RAM và phát Host Syscall
   │
   └─ [6] irqfd (Virtual Interrupt Signaling Interface)
        └── Cầu nối để QEMU báo cho KVM tiêm ngắt ảo vào vAPIC

 ══════════════════════════════════════════════════════════════════════════════════ [ Ranh giới Host Kernel ]
 [ HOST KERNEL STORAGE STACK ]
   │
   ├─ [7] /dev/mapper/mpath0 (Device Mapper Multipath Virtual Device - dm-0)
   │    └── Ổ đĩa ảo gom nhiều đường truyền, điều phối failover/load-balancing
   │
   ├─ [8] scsi_hostX (Virtual SCSI HBA / Transport Class)
   │    └── Bộ điều khiển Host Bus Adapter ảo do driver iscsi_tcp giả lập
   │
   ├─ [9] /dev/sda & /dev/sdb (Virtual SCSI Block Devices / LUN Representation)
   │    └── Các file block device đại diện cho từng đường iSCSI Session vật lý
   │
   └─ [10] Kernel TCP Socket (Kernel struct socket / File Descriptor ảo)
        └── Giao diện bọc gói iSCSI PDU sang tầng mạng TCP/IP

 ══════════════════════════════════════════════════════════════════════════════════ [ Ranh giới Fabric / Target ]
 [ STORAGE FABRIC & TARGET ]
   │
   └─ [11] Target IQN / TPG / LUN 0 (Storage Target Logical Entities)
        └── Điểm cuối logic (Logical Unit) tại tủ đĩa Flash NetApp
========================================================================================================================
```

<!-- ORIGINAL 942-989 END -->

<!-- ORIGINAL 991 BEGIN -->

### TẦNG 1: BÊN TRONG GUEST VM (THIẾT BỊ ẢO MỨC HỆ ĐIỀU HÀNH KHÁCH)

<!-- ORIGINAL 991 END -->

<!-- ORIGINAL 992 BEGIN -->

#### 1\. Thiết bị khối ảo: /dev/vdb (Guest Virtual Block Device)

<!-- ORIGINAL 992 END -->

<!-- ORIGINAL 993 BEGIN -->

- **Bản chất:** Là một device node thuộc lớp block (cdev/bdev interface) do driver virtio\_blk.ko khởi tạo trong sysfs của Guest.

<!-- ORIGINAL 993 END -->

<!-- ORIGINAL 995 BEGIN -->

- **Đặc tính kỹ thuật:** Không có đĩa từ hay chip nhớ vật lý nào tương ứng bên dưới hệ điều hành Guest; nó chỉ là điểm móc (mount point / target) để ứng dụng mở bằng hàm open("/dev/vdb", O\_DIRECT \| O\_RDWR).

<!-- ORIGINAL 995 END -->

<!-- ORIGINAL 997 BEGIN -->

- **Vai trò:** Cung cấp chuẩn giao tiếp POSIX Block I/O cho ứng dụng, tiếp nhận struct bio từ VFS và chuyển giao xuống hàng đợi virtqueue.

<!-- ORIGINAL 997 END -->

<!-- ORIGINAL 999 BEGIN -->

#### 2\. Thiết bị PCI ảo: virtio-blk-pci (Emulated PCI Endpoint)

<!-- ORIGINAL 999 END -->

<!-- ORIGINAL 1000 BEGIN -->

- **Bản chất:** Một thiết bị ngoại vi ảo được QEMU cắm vào bo mạch chủ ảo (Q35 hoặc i440FX) của VM, thường có địa chỉ PCI ảo dạng 00:05.0 (Vendor ID: 0x1af4, Device ID: 0x1001 hoặc 0x1042).

<!-- ORIGINAL 1000 END -->

<!-- ORIGINAL 1002 BEGIN -->

- Giao diện thanh ghi ảo (Virtual MMIO Registers):

<!-- ORIGINAL 1002 END -->

<!-- ORIGINAL 1004 BEGIN -->

- Thiết bị này phơi bày các thanh ghi cấu hình thông qua PCI BAR0/BAR4.

<!-- ORIGINAL 1004 END -->

<!-- ORIGINAL 1006 BEGIN -->

- Trong đó, thanh ghi quan trọng nhất là Queue Notify Register (MMIO Doorbell). Khi Guest ghi chỉ số hàng đợi vào thanh ghi này, nó không kích hoạt mạch điện thật mà kích hoạt bẫy vi kiến trúc VM-Exit của CPU vật lý.

<!-- ORIGINAL 1006 END -->

<!-- ORIGINAL 1008 BEGIN -->

#### 3\. Bộ điều khiển ngắt ảo: vAPIC (Virtual Local APIC)

<!-- ORIGINAL 1008 END -->

<!-- ORIGINAL 1009 BEGIN -->

- **Bản chất:** Một cấu trúc dữ liệu mô phỏng chip quản lý ngắt cục bộ của CPU, nằm bên trong cấu trúc phần cứng VMCS (Intel VT-x) và được quản lý bởi module nhân kvm.ko.

<!-- ORIGINAL 1009 END -->

<!-- ORIGINAL 1011 BEGIN -->

- **Vai trò:** Đóng vai trò là "chiếc loa ảo" tiếp nhận tín hiệu ngắt. Khi Host hoàn tất I/O, KVM không gửi tín hiệu điện lên chân ngắt của CPU vật lý mà ghi vector ngắt vào bảng vAPIC page để vCPU của Guest thức dậy chạy hàm xử lý ngắt (virtio\_blk\_done).

<!-- ORIGINAL 1011 END -->

<!-- ORIGINAL 1013 BEGIN -->

### TẦNG 2: RANH GIỚI HYPERVISOR & HOST USERSPACE (GIAO DIỆN ĐIỀU PHỐI)

<!-- ORIGINAL 1013 END -->

<!-- ORIGINAL 1014 BEGIN -->

#### 4\. Giao diện sự kiện nhanh: ioeventfd

<!-- ORIGINAL 1014 END -->

<!-- ORIGINAL 1015 BEGIN -->

- **Bản chất:** Là một cơ chế giao tiếp liên tiến trình (IPC) siêu nhẹ được tạo bằng syscall eventfd(), liên kết trực tiếp giữa phân hệ ảo hóa KVM trong Host Kernel và vòng lặp sự kiện của QEMU.

<!-- ORIGINAL 1015 END -->

<!-- ORIGINAL 1017 BEGIN -->

- Cơ chế hoạt động:

<!-- ORIGINAL 1017 END -->

<!-- ORIGINAL 1019 BEGIN -->

- Được gán cố định với địa chỉ thanh ghi MMIO Doorbell của thiết bị virtio-blk-pci qua lệnh ioctl(KVM\_IOEVENTFD).

<!-- ORIGINAL 1019 END -->

<!-- ORIGINAL 1021 BEGIN -->

- Khi VM-Exit xảy ra tại Doorbell, KVM ghi giá trị vào ioeventfd này, biến tín hiệu bẫy phần cứng thành một sự kiện file descriptor sẵn sàng để đọc (POLLIN), đánh thức vòng lặp epoll\_wait của QEMU IOThread mà không cần đánh thức toàn bộ tiến trình QEMU.

<!-- ORIGINAL 1021 END -->

<!-- ORIGINAL 1023 BEGIN -->

#### 5\. Khối điều khiển giả lập: QEMU virtio-blk Emulated Controller

<!-- ORIGINAL 1023 END -->

<!-- ORIGINAL 1024 BEGIN -->

- **Bản chất:** Thực thể phần mềm chạy trong không gian User Space của Host (Host Ring 3), đại diện bởi cấu trúc C VirtIOBlock trong mã nguồn QEMU.

<!-- ORIGINAL 1024 END -->

<!-- ORIGINAL 1026 BEGIN -->

- Vai trò:

<!-- ORIGINAL 1026 END -->

<!-- ORIGINAL 1028 BEGIN -->

- Giữ bảng ánh xạ bộ nhớ máy ảo (AddressSpaceDispatch).

<!-- ORIGINAL 1028 END -->

<!-- ORIGINAL 1030 BEGIN -->

- Đọc các Descriptor ảo (chứa địa chỉ GPA), chuyển đổi thành con trỏ bộ nhớ thật trong tiến trình QEMU (HVA), sau đó đóng vai trò như một Initiator trung gian phát lệnh io\_submit() xuống tầng lưu trữ Host.

<!-- ORIGINAL 1030 END -->

<!-- ORIGINAL 1032 BEGIN -->

#### 6\. Giao diện tiêm ngắt: irqfd

<!-- ORIGINAL 1032 END -->

<!-- ORIGINAL 1033 BEGIN -->

- **Bản chất:** Tương tự ioeventfd, nhưng chạy theo chiều ngược lại (từ QEMU User Space → KVM Kernel).

<!-- ORIGINAL 1033 END -->

<!-- ORIGINAL 1035 BEGIN -->

- **Vai trò:** Được cấu hình qua lệnh ioctl(KVM\_IRQFD). Khi QEMU nhận được kết quả I/O hoàn tất từ Host Kernel, nó ghi một số 64-bit vào irqfd. KVM bắt sự kiện này và ngay lập tức nạp vector ngắt vào vAPIC của Guest mà không yêu cầu QEMU phải can thiệp sâu vào luồng thực thi của CPU.

<!-- ORIGINAL 1035 END -->

<!-- ORIGINAL 1037 BEGIN -->

### TẦNG 3: HOST KERNEL STORAGE STACK (THIẾT BỊ LƯU TRỮ LOGIC TRÊN HOST)

<!-- ORIGINAL 1037 END -->

<!-- ORIGINAL 1038 BEGIN -->

#### 7\. Thiết bị Multipath ảo: /dev/mapper/mpath0 (hay dm-0)

<!-- ORIGINAL 1038 END -->

<!-- ORIGINAL 1039 BEGIN -->

- **Bản chất:** Thiết bị khối ảo do phân hệ Device Mapper của Linux (dm\_multipath.ko) tạo ra trong Host Kernel.

<!-- ORIGINAL 1039 END -->

<!-- ORIGINAL 1041 BEGIN -->

- Vai trò:

<!-- ORIGINAL 1041 END -->

<!-- ORIGINAL 1043 BEGIN -->

- **Trừu tượng hóa topology mạng lưu trữ:** Ứng dụng QEMU chỉ nhìn thấy duy nhất một block device /dev/mapper/mpath0.

<!-- ORIGINAL 1043 END -->

<!-- ORIGINAL 1045 BEGIN -->

- **Bộ chọn đường logic (Path Selector):** Bên dưới mpath0 không trực tiếp chứa dữ liệu; nó nắm giữ một bảng định tuyến gồm nhiều đường dẫn vật lý độc lập (Active/Passive hoặc Active/Active). Nó nhân bản (clone) struct request và chuyển hướng I/O xuống các thiết bị SCSI thành phần.

<!-- ORIGINAL 1045 END -->

<!-- ORIGINAL 1047 BEGIN -->

#### 8\. Bộ điều khiển HBA ảo: scsi\_hostX

<!-- ORIGINAL 1047 END -->

<!-- ORIGINAL 1048 BEGIN -->

- **Bản chất:** Giao diện điều khiển máy chủ SCSI (SCSI Host Bus Adapter) logic được khai báo trong hệ thống sysfs của Host (/sys/class/scsi\_host/hostX).

<!-- ORIGINAL 1048 END -->

<!-- ORIGINAL 1050 BEGIN -->

- **Vai trò:** Trong kiến trúc iSCSI truyền thống, máy chủ không cắm card HBA Fibre Channel vật lý. Driver iscsi\_tcp.ko tự đăng ký nó như một HBA ảo với SCSI Mid-layer. Điều này đánh lừa phân hệ SCSI của Linux Kernel rằng hệ thống đang có một card điều khiển lưu trữ thực thụ gắn trên bus.

<!-- ORIGINAL 1050 END -->

<!-- ORIGINAL 1052 BEGIN -->

#### 9\. Thiết bị đĩa SCSI logic: /dev/sda và /dev/sdb

<!-- ORIGINAL 1052 END -->

<!-- ORIGINAL 1053 BEGIN -->

- **Bản chất:** Các block device SCSI (struct scsi\_device) do driver đĩa sd\_mod tạo ra.

<!-- ORIGINAL 1053 END -->

<!-- ORIGINAL 1055 BEGIN -->

- Đặc tính kỹ thuật:

<!-- ORIGINAL 1055 END -->

<!-- ORIGINAL 1057 BEGIN -->

- Đây không phải là các ổ cứng cắm trên khe cắm SATA/SAS của máy chủ.

<!-- ORIGINAL 1057 END -->

<!-- ORIGINAL 1059 BEGIN -->

- Mỗi thiết bị này thực chất là một iSCSI Session độc lập được ánh xạ qua mạng từ các cổng portal khác nhau của mảng đĩa Target (ví dụ: /dev/sda đi qua NIC 1 tới Target IP 1, /dev/sdb đi qua NIC 2 tới Target IP 2). Chúng đóng vai trò là "nô lệ" (slave paths) nằm bên dưới bộ điều khiển /dev/mapper/mpath0.

<!-- ORIGINAL 1059 END -->

<!-- ORIGINAL 1061 BEGIN -->

#### 10\. Giao diện mạng Socket ảo: Kernel struct socket

<!-- ORIGINAL 1061 END -->

<!-- ORIGINAL 1062 BEGIN -->

- **Bản chất:** Điểm cuối kết nối mạng mức nhân (In-kernel TCP Endpoint) do iscsi\_tcp duy trì để giữ kết nối tới cổng 3260 của Target.

<!-- ORIGINAL 1062 END -->

<!-- ORIGINAL 1064 BEGIN -->

- **Vai trò:** Biến đổi các yêu cầu ghi dữ liệu khối (SCSI Blocks) thành các dòng dữ liệu truyền thông (TCP Streams), gắn cờ quản lý cửa sổ trượt và phân mảnh TCP.

<!-- ORIGINAL 1064 END -->

<!-- ORIGINAL 1066 BEGIN -->

### TẦNG 4: STORAGE FABRIC & TARGET (THỰC THỂ LOGIC TẠI TỦ ĐĨA)

<!-- ORIGINAL 1066 END -->

<!-- ORIGINAL 1067 BEGIN -->

#### 11\. Cụm định danh logic Target: IQN, TPG và LUN 0

<!-- ORIGINAL 1067 END -->

<!-- ORIGINAL 1068 BEGIN -->

- **iSCSI Qualified Name (IQN):** Chuỗi định danh duy nhất của tủ lưu trữ (ví dụ iqn.1992-08.com.netapp:sn.123456).

<!-- ORIGINAL 1068 END -->

<!-- ORIGINAL 1070 BEGIN -->

- **Target Portal Group (TPG):** Tập hợp logic các địa chỉ IP và cổng mạng của tủ đĩa cho phép kết nối đa đường (Multipathing).

<!-- ORIGINAL 1070 END -->

<!-- ORIGINAL 1072 BEGIN -->

- **Logical Unit Number (LUN 0):** Phân vùng không gian lưu trữ ảo được cắt từ khối ổ đĩa Flash RAID/Pool trên tủ NetApp. Đây chính là đích đến cuối cùng mà các lệnh SCSI CDB (0x2A) trỏ tới trước khi controller phần cứng của Target ghi dữ liệu vào các chip nhớ Flash NAND vật lý.

<!-- ORIGINAL 1072 END -->

<!-- ORIGINAL 1073 BEGIN -->

## BẢNG ĐỐI CHIẾU: THIẾT BỊ ẢO VS PHẦN CỨNG THẬT TƯƠNG ỨNG

<!-- ORIGINAL 1073 END -->

<!-- ORIGINAL 1074 BEGIN -->

| Thiết bị ảo / Giao diện logic | Tầng xuất hiện | Thực thể phần cứng thật tương ứng (nếu không ảo hóa) | Chi phí hiệu năng (Overhead) phát sinh |
| --- | --- | --- | --- |
| /dev/vdb | Guest OS | Ổ cứng SSD cắm trực tiếp trên máy chủ | Tạo thêm một lớp hàng đợi blk-mq riêng trong Guest |
| virtio-blk-pci | KVM / QEMU | Card điều khiển Host Controller vật lý trên bus PCIe | Gây ra VM-Exit khi ghi thanh ghi MMIO Doorbell |
| vAPIC | vCPU (VMCS) | Chip LAPIC phần cứng trên CPU socket | Độ trễ do tiêm ngắt ảo (Interrupt Injection latency) |
| ioeventfd & irqfd | Hypervisor IPC | Đường dây dẫn tín hiệu ngắt/chuông vật lý | Tốn chu kỳ CPU cho Context Switch giữa các luồng Host |
| /dev/mapper/mpath0 | Host Kernel | Chip vi xử lý Multipath phần cứng trên tủ SAN | CPU Host phải chạy giải thuật chọn đường và clone request |
| scsi\_hostX | Host Kernel | Card HBA cắm khe PCIe (Broadcom/QLogic FC HBA) | Phải bọc/mở nhiều tầng driver (sd → scsi → iscsi) |
| /dev/sda / /dev/sdb | Host Kernel | Ổ đĩa SAS/SATA cắm cục bộ | Duy trì hàng đợi, timer kiểm tra timeout mạng iSCSI |

<!-- ORIGINAL 1074 END -->
