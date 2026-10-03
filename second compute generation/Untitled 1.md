

## Guest

**virtio-blk**

- Một queue (hoặc multiqueue) gắn với blk-mq trong guest; `virtio_blk.c` biến request thành descriptor chain.
- Cấu trúc một request: header (type/sector), data buffers, status byte.
- Feature bits (`VIRTIO_BLK_F_*`): flush, discard, write-zeroes, multiqueue, seg_max, size_max.
- Chỉ có block device, không có lệnh SCSI.

**virtio-scsi**

- Mô hình HBA ảo: controlq, eventq, request queues. Mỗi LUN là một `sd` device (sda, sdb…).
- Hỗ trợ nhiều LUN trên một controller, SCSI passthrough, persistent reservation.
- Đường đi trong guest dài hơn (qua SCSI mid-layer), đổi lại tương thích rộng. Đây là điểm nên so sánh latency với virtio-blk.

**Chung cho cả hai**

- Vring, kick (ghi vào notify register gây VM exit hoặc ioeventfd), interrupt injection (MSI-X, irqfd).
- Event index / notification suppression để giảm số lần kick và interrupt.
- Guest I/O scheduler, queue depth, `max_sectors_kb`.

## Userspace

**QEMU**

- Block layer: `BlockDriverState`, `BlockBackend`, protocol/format driver (`file`, `host_device`, `raw`, `qcow2`).
- **AioContext**: event loop (`aio_poll`), fd handler, bottom half, timer. Mỗi iothread sở hữu một AioContext.
- **IOThread**: tách xử lý I/O khỏi main loop. Cần hiểu vì sao main loop làm chậm I/O và cách pin iothread vào CPU.
- **Multi IOThread / multiqueue**: mỗi virtqueue gắn một iothread (`iothread-vq-mapping` ở bản QEMU mới). Quan hệ giữa `num-queues`, số vCPU và số iothread.
- Backend AIO: `linux-aio` (io=native), `threads` (thread pool + `pwritev`), `io_uring`. Hiểu rõ điều kiện của từng loại.
- Chuyển đổi địa chỉ: guest physical → host virtual qua `MemoryRegion`, `address_space_map`.

**iscsid / open-iscsi**

- iscsid là daemon điều khiển, không nằm trên data path. Nó lo login, session, reconnect, timeout, error recovery.
- `iscsiadm`: node/session/discovery DB, các tham số (`node.session.queue_depth`, `cmds_max`, `nr_sessions`, replacement_timeout).
- Giao tiếp với kernel qua netlink (`scsi_transport_iscsi`). Cần phân biệt control plane (iscsid) và data plane (kernel).

**multipathd** (nên thêm vào cặp với iscsid)

- Path checker, `multipath.conf`, wwid, blacklist (bạn đã gặp ở lab), uevent từ udev.

## Kernel

**Ba bộ module iSCSI** (mình đoán bạn muốn nói thế này, nếu khác thì bảo mình)

1. **Initiator core + transport**: `scsi_transport_iscsi`, `libiscsi`, `libiscsi_tcp`, `iscsi_tcp`.
2. **Transport thay thế / offload**: `ib_iser` (iSCSI qua RDMA), `bnx2i`, `cxgb4i`… (HBA offload).
3. **Target (LIO)**: `target_core_mod`, `iscsi_target_mod`, backstore (`target_core_file`, `iblock`, `rd_mcp`), configfs. Khớp với phía SDS trong lab của bạn.

**vhost I/O**

- `vhost.c` (core: worker thread, vring, memory table), `vhost/scsi.c`, `vhost/net.c`, `vhost/vdpa.c`.
- Vì sao vhost-kernel dùng worker kthread, ioeventfd/irqfd nối với KVM.
- Phân biệt vhost-kernel với vhost-user (backend chạy ngoài kernel).

**dm-multipath**

- Request-based dm: `dm-mpath.c`, `dm-rq.c`, path group, selector, `queue_if_no_path`, failover/failback.
- Khác biệt giữa bio-based dm và request-based dm, và tác động tới việc có copy payload hay không.

**SCSI**

- Mid-layer (`scsi_lib.c`), upper driver (`sd`), lower driver (HBA driver).
- `scsi_cmnd`, command queueing, `queue_depth`, error handler (abort → reset → offline), timeout.

## Hardware

**DMA**

- DMA engine, scatter-gather list, descriptor ring, cache coherency, bounce buffer (swiotlb).
- DMA API của kernel (`dma_map_sg`, `dma_alloc_coherent`).
- Đây là chỗ giải thích vì sao O_DIRECT và pinned memory có thể zero-copy.

**IOMMU**

- Dịch IOVA → PA cho thiết bị, bảo vệ DMA, passthrough mode vs translated mode.
- Liên quan tới vIOMMU khi guest dùng thiết bị passthrough, và tới vhost-user nếu backend map memory.

**MMU**

- Page table, TLB, EPT/NPT (2 tầng dịch địa chỉ cho VM), hugepage giảm TLB miss.
- Giúp trả lời câu "VA / PA" trên bảng trắng.

**Storage/host controller (HBA cắm PCIe)**

- PCIe: BAR, doorbell, MSI-X, queue của controller, firmware.
- HBA offload: ASIC xử lý TCP/iSCSI, bypass network stack. So sánh với NIC thường (TSO, checksum offload) + software iSCSI.

**Disk controller (trong ổ)**

- Firmware, FTL (với SSD), NAND/HDD mechanics, write cache trong ổ, NCQ/command queue, FUA và flush.
- NVMe: submission/completion queue, doorbell, namespace.

**RDMA** (chỗ mình sửa lại)

- RDMA không phải "header định danh tiến trình source/dest". Bản chất là NIC đọc/ghi thẳng vào bộ nhớ đã đăng ký của máy bên kia, không qua CPU/kernel của bên đó trên data path.
- Khái niệm cốt lõi: **Memory Region (MR)** và `lkey/rkey`, **Protection Domain**, **Queue Pair** (SQ/RQ) và **Completion Queue**, work request (SEND, RDMA WRITE, RDMA READ), verbs API.
- Transport: InfiniBand, RoCE v2 (chạy trên UDP/IP), iWARP (chạy trên TCP). Header đáng biết là BTH (có Destination QP number và PSN), nên có thể ý bạn nhắc đến phần định danh QP.
- Với NVMe-oF/RDMA, controller NVMe-oF cần đăng ký bộ nhớ guest thì mới DMA thẳng được. Đây là điểm nối trực tiếp với vhost-user-blk + SPDK.

## Luồng hoạt động của ổ và đường lên (completion path)

Phần này bảng của bạn ghi "tìm hiểu ổ cứng, phần luồng copy thật", nên nên vẽ cả chiều lên:

- **Ổ đĩa**: nhận command → cache/FTL hoặc seek → ghi → báo hoàn thành (hoặc flush).
- **Controller → CPU**: interrupt (MSI-X) hoặc polling → ISR → softirq/threaded IRQ → `blk_mq_complete_request` → `bio_endio`.
- **Với iSCSI**: Data-In/Response PDU → TCP receive → `iscsi_tcp` xử lý PDU → `scsi_done` → complete lên block layer. Có thêm tầng network ngược lại.
- **Kernel → userspace**: wakeup thread chờ (hoặc `io_getevents`, eventfd, io_uring CQE).
- **Với VM**: QEMU nhận completion → ghi vào used ring → irqfd inject interrupt vào guest → guest `virtblk_done` → complete lên guest block layer → báo cho app trong guest. Đây là toàn bộ chiều ngược lại cần vẽ.
- Cần đo thêm: latency của từng tầng (blktrace D2C trong host, rồi so với latency fio trong guest) để biết overhead nằm ở đâu.

## Cách ghép các tầng lại

Gợi ý bạn làm một bảng 3 cột để bản thân tự lấp đầy dần: **tầng**, **struct/hàm/tracepoint đại diện**, **có copy payload không**. Khi bảng này đầy đủ cho cả chiều xuống (write) và chiều lên (completion) thì bạn đã trả lời được câu hỏi gốc trên bảng trắng.

Nếu muốn, mình dựng sẵn bảng này (để bạn điền tiếp) hoặc vẽ sơ đồ luồng write + completion qua cả 4 tầng.