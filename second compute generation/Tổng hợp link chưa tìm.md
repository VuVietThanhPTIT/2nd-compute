Mình chưa tìm được animation hay mô phỏng tương tác nào cho đúng ý "phần mềm giả lập ổ đĩa". Những tài liệu dưới đây là gần nhất, xếp từ dễ vào đến sâu.

## 1. Hiểu cơ chế guest và host nói chuyện với nhau (virtqueue)

- **Red Hat blog: "Virtqueues and virtio ring: How the data travels"**: https://www.redhat.com/en/blog/virtqueues-and-virtio-ring-how-data-travels
    - Đây là bài có hình minh họa rõ nhất mình tìm được. Bài giải thích các vùng bộ nhớ dùng chung: descriptor ring, avail ring để driver đưa buffer cho device, và used ring để device báo đã xong, kèm cả chained và indirect descriptor.
    - Nó khớp với phần "vring, kick, interrupt" trong bảng của bạn.
- **Tài liệu kernel "Virtio on Linux"**: https://docs.kernel.org/driver-api/virtio/virtio.html
    - Giải thích guest driver và device ở hypervisor giao tiếp qua shared memory bằng virtqueue, tức ring buffer chứa các descriptor.
    - Bài này cũng trỏ tới virtio spec v1.2 (OASIS), chương 2.5 "Virtqueues" là phần chính.

## 2. Phía QEMU: thiết bị ổ đĩa ảo thực sự chạy thế nào

- **Video và slide KVM Forum 2024 của Stefan Hajnoczi, "IOThread Virtqueue Mapping"**: https://blog.vmsplice.net/2024/10/video-and-slides-available-for-iothread.html
    - Đây là tài liệu đúng nhất cho câu hỏi của bạn. Slide có mục "How IOThreads process virtio-blk I/O".
    - Talk nói về việc trước đây QEMU xử lý mọi queue của virtio-blk trong một thread, và tính năng mới từ QEMU 9.0 cho phép chia queue cho nhiều thread.
    - Slide PDF: https://kvm-forum.qemu.org/2024/IOThread_Virtqueue_Mapping_EGsYiZC.pdf
- **Bài của Red Hat Developers về cùng tính năng**: https://developers.redhat.com/articles/2024/07/09/scaling-virtio-blk-disk-io-iothread-virtqueue-mapping
    - Có hình minh họa một thiết bị virtio-blk 4 queue được gán cho 2 IOThread.
    - Tính năng này được thiết kế cho io="native".
- **Video KVM Forum 2017, "Applying Polling Techniques to QEMU: Reducing virtio-blk I/O Latency"**: https://blog.vmsplice.net/2017/11/video-and-slides-available-for-applying.html
    - Talk nói về tối ưu AioContext polling, giúp giảm latency cho virtio-blk và virtio-scsi khi dùng iothread.
    - Phần "AioContext" trong bảng của bạn có thể học từ đây.
- **Slide "A Practical Look at QEMU's Block Layer Primitives"**: https://events.static.linuxfound.org/sites/events/files/slides/A-Practical-Look-at-QEMU-Block-Layer-Primitives.pdf
    - Slide trình bày các khái niệm block device trong QEMU như backend (NBD, qcow2, raw…), cách cấu hình và các thao tác live.
    - Phù hợp để hiểu tầng `BlockBackend` và format driver.

## 3. Đọc code thật (rất ngắn, nhìn là thấy luồng)

- **`hw/block/dataplane/virtio-blk.c`** của QEMU: https://github.com/qemu/qemu/blob/729962f6db43bf262a446f5e13d900ffb3c54a88/hw/block/dataplane/virtio-blk.c
    - Đây là bản cũ nên đơn giản hơn nhiều so với code hiện tại, dễ đọc để nắm khung.
    - Trong bản này có thể thấy vòng lặp lấy request từ vring, gom lại để submit xuống block layer (`blk_io_plug`, `virtio_submit_multiwrite`), tắt và bật lại notification khi vring rỗng.
    - Nó cũng đặt context cho block backend vào AioContext của thread riêng (`blk_set_aio_context`).
- **Commit "dataplane: add virtqueue vring code"**: https://qemu.googlesource.com/qemu/+/88807f89d945acad54c8365ff7b6ef0f0d0ddd56%5E%21/trace-events
    - Commit message giải thích: thread dataplane chạy ngoài global mutex nên không dùng được các hàm truy cập bộ nhớ thông thường của QEMU. Vì vậy guest memory được map sẵn từ trước để truy cập nhanh.
    - Phần này liên quan trực tiếp tới câu "guest physical → host virtual" trong bảng của bạn.

## 4. Cách tự "quan sát" mô phỏng đang diễn ra

QEMU có sẵn trace event cho virtio-blk. Trong commit ở trên có `virtio_blk_handle_read` và `virtio_blk_handle_write`, in ra request, sector và số sector. Bạn có thể bật các trace này (`-trace 'virtio_blk_*'`) rồi chạy `dd` hay `fio` trong guest. Cách này cho thấy từng request của guest đi vào QEMU ra sao, và rất hợp để điền cột "hàm/tracepoint đại diện" trong bảng.

## Phần chưa có trong kết quả tìm được

Mình không tìm được bài nào mô tả riêng cách QEMU giả lập thiết bị kiểu cũ (IDE/AHCI/NVMe emulated, bắt từng lần guest ghi vào thanh ghi rồi QEMU xử lý). Đó là phía "không paravirtual", và so với virtio thì nó chậm hơn vì mỗi thao tác thanh ghi gây một VM exit. Nếu cần, mình tìm riêng phần này. Mình cũng có thể tìm thêm các hypervisor nhỏ như kvmtool hay Firecracker, vì code của chúng ngắn hơn QEMU nhiều và dễ đọc hơn để thấy một thiết bị virtio-blk được dựng từ đầu thế nào.

Mình xếp theo đúng các nhóm bạn liệt kê, mỗi mục ghi bài đó giúp hiểu điểm nào. Một điểm mình sửa so với lần trước: "3 bộ modules của iscsi" theo README của open-iscsi là `scsi_transport_iscsi`, `libiscsi` và `iscsi_tcp`, nên mình đoán sai ở bản trước (xem mục Kernel).

## Guest: virtio-blk, virtio-scsi

- **QEMU blog "Configuring virtio-blk and virtio-scsi Devices"**: https://www.qemu.org/2021/01/19/virtio-blk-scsi-configuration/
    - Bài khuyên dùng virtio-blk khi cần hiệu năng, dùng virtio-scsi khi gắn trên 28 disk hoặc cần đủ tính năng SCSI. Persistent Reservation chỉ virtio-scsi hỗ trợ, và SCSI passthrough đã bị bỏ khỏi driver virtio-blk từ Linux 5.6.
    - Đây là bài nên đọc đầu tiên cho phần so sánh hai loại.
- **Thread LKML "virtio-scsi: first version"**: https://lkml.iu.edu/hypermail/linux/kernel/1112.1/00189.html
    - Đây là cuộc tranh luận về lý do tạo virtio-scsi bên cạnh virtio-blk. Virtio-scsi chỉ là một transport SCSI, còn virtio-blk thì không có task management function nên xử lý lỗi trong guest kém hơn.
    - Bài này cho thấy lý do thiết kế rõ hơn tài liệu hướng dẫn.
- **Benchmark trên mailing list OpenStack (2017)**: https://lists.openstack.org/pipermail/openstack/2017-March/019031.html
    - Người dùng đo thấy virtio-scsi chậm hơn khoảng 11,68% khi ghi và 4,83% khi đọc, trên kernel và QEMU rất cũ.
    - Chỉ nên xem như một số liệu tham khảo, không phải kết luận chung.

## Userspace

**QEMU AioContext, IOThread**

- **"QEMU Internals: Event loops" (Stefan Hajnoczi)**: https://blog.vmsplice.net/2020/08/qemu-internals-event-loops.html
    - Bài giải thích AioContext là event loop gốc của QEMU, chạy bằng `aio_poll()`. Main loop và IOThread chạy event loop, và IOThread cho hiệu năng tốt nhất.
    - Đây là bài chính cho phần "AioContext".
- **`docs/multiple-iothreads.txt` trong source QEMU**: https://github.com/qemu/qemu/blob/ef558696b5c688a8a3bef4ab8f6b27937cc24c89/docs/multiple-iothreads.txt
    - Tài liệu nói về main loop, IOThread, global mutex và các dịch vụ AioContext (event notifier, fd handler…), cùng quy tắc acquire/release context.
    - Đây là tài liệu thiết kế gốc, hơi cũ nhưng khái niệm vẫn đúng.
- **`iothread.c`**: https://lxr.missinglinkelectronics.com/qemu+v2.9.0/iothread.c
    - Code ngắn, cho thấy vòng lặp của IOThread và các tham số polling (`poll_max_ns`, `poll_grow`, `poll_shrink`).

**Multi IOThread**

- **Talk KVM Forum 2024, "IOThread Virtqueue Mapping"**: https://blog.vmsplice.net/2024/10/video-and-slides-available-for-iothread.html
    - Đây là talk có video và slide về việc gán nhiều IOThread cho các virtqueue của virtio-blk, để tận dụng multi-queue của block layer trên host.
    - Slide: https://kvm-forum.qemu.org/2024/IOThread_Virtqueue_Mapping_EGsYiZC.pdf
    - Slide trỏ tới talk "Multiqueue in the block layer" của Kevin Wolf và Emanuele Giuseppe Esposito: https://www.youtube.com/watch?v=Ubped0PgvZI
- **Bài của Red Hat Developers**: https://developers.redhat.com/articles/2024/07/09/scaling-virtio-blk-disk-io-iothread-virtqueue-mapping
    - Bài có hình minh họa một thiết bị virtio-blk 4 queue được gán cho 2 IOThread, kèm cấu hình libvirt. Tính năng này được thiết kế cho `io="native"`.

**iscsid / open-iscsi**

- **README của open-iscsi**: https://github.com/openSUSE/open-iscsi
    - README nêu rõ phần kernel xử lý data path (iSCSI read/write). Toàn bộ control plane (discovery, login/logout, xử lý lỗi cấp connection, Nop-In/Nop-Out) nằm ở userspace, gồm daemon `iscsid` và tool `iscsiadm`.
    - Bài này trả lời trực tiếp câu hỏi "iscsid nằm ở đâu trên đường đi".
- **Arch Wiki "Open-iSCSI"**: https://wiki.archlinux.org/title/Open-iSCSI
    - Bài có sơ đồ ASCII iscsiadm ↔ iscsid ↔ kernel modules, cùng lệnh login, tham số CHAP và xem session bằng `iscsiadm -m session -P 3`.

## Kernel

**Ba bộ module iSCSI (initiator)**

- Theo README ở trên, phần kernel của open-iSCSI gồm đúng ba module: `scsi_transport_iscsi.ko`, `libiscsi.ko` và `iscsi_tcp.ko`.
- Phần hướng dẫn cài đặt trong README cũng liệt kê thêm `libiscsi_tcp.ko`.
- Nếu ý bạn là phía target, đọc mục LIO ngay dưới.

**LIO (iSCSI target trong kernel)**

- **Arch Wiki "iSCSI/LIO"**: https://wiki.archlinux.org/title/ISCSI/LIO
    - Bài nêu LIO là iSCSI target trong kernel, hai module quan trọng là `target_core_mod` và `iscsi_target_mod`.
    - Bài kèm cách cấu hình bằng `targetcli`.
- **Commit "target: Add LIO target core"**: https://android-review.linaro.org/plugins/gitiles/kernel/hikey-linaro/+/c66ac9db8d4ad9994a02b3e933ea2ccc643e1fe5%5E%21/drivers/target/Kconfig
    - Commit liệt kê tính năng của LIO (Persistent Reservations, ALUA, error recovery, UNMAP…) và các backstore IBLOCK, FILEIO, pSCSI.

**vhost I/O**

- **"QEMU Internals: vhost architecture"**: https://blog.vmsplice.net/2011/09/qemu-internals-vhost-architecture.html
    - Bài giải thích vhost đưa phần giả lập virtio vào kernel, lấy QEMU userspace ra khỏi đường đi. Có vhost worker thread, ioeventfd dùng để kick virtqueue, irqfd dùng để ngắt guest. Bài cũng liệt kê các file nguồn liên quan như `drivers/vhost/vhost.c` và `virt/kvm/eventfd.c`.
    - Bài viết năm 2011 nhưng kiến trúc cơ bản vẫn đúng.
- **Patch "vhost-scsi: worker per virtqueue"**: https://git.cs.tu-dortmund.de/SYS-OSS/FRET-qemu/src/commit/51396556f0927c3d202f9903db5aa5aa8d4cbd2c/hw/scsi/vhost-scsi.c
    - Patch nói vhost-net có worker thread cho mỗi cặp tx/rx, còn vhost-scsi trước đây dùng một worker chung cho mọi virtqueue.
    - Patch này cho thấy phía vhost cũng đã có hướng tương tự multi-IOThread.

**dm-multipath**

- **Red Hat "Configuring device mapper multipath" (RHEL 8)**: https://access.redhat.com/documentation/vi-vn/red_hat_enterprise_linux/8/html/configuring_device_mapper_multipath/modifying-the-dm-multipath-configuration-file_configuring-device-mapper-multipath
    - Bài mô tả cấu hình `/etc/multipath.conf` gồm các section blacklist, defaults, multipaths, devices, và thứ tự ưu tiên giữa các section.
    - Đây là phần gần với blacklist bạn từng gặp ở lab. Trang mở ra có thể đang ở ngôn ngữ khác, bạn đổi ngôn ngữ trên trang.
- **Patch gốc "device-mapper: multipath"**: https://lkml.indiana.edu/0502.1/0762.html
    - Patch giải thích mô hình path group: mỗi priority group có một path selector chọn đường, và có tùy chọn `queue_if_no_path` làm phương án cuối.
    - Đọc cùng với source hiện tại: https://sre.ring0.de/linux/plain/drivers/md/dm-mpath.c?h=v5.5-rc4 (bản này đã chuyển sang request-based qua `dm-rq.h`).

**SCSI**

- **`Documentation/scsi/scsi_eh.rst` và `scsi_mid_low_api.rst`** trong kernel tree (bản đọc online: https://gitweb.dragonflybsd.org/linux.git/blob/7a8016d95651fecce5708ed93a24a03a9ad91c80:/Documentation/scsi/scsi_eh.rst)
    - Tài liệu mô tả mỗi lệnh SCSI là một `struct scsi_cmnd`, cách midlayer gọi `queuecommand()`, và cách error handler leo thang qua các callback abort, device reset, bus reset, host reset.
    - Phần này khớp với chuỗi "abort → reset → offline" trong bảng của bạn.

## Hardware

**DMA**

- **Kernel DMA API**: https://www.kernel.org/doc/html/v6.18/core-api/dma-api.html
    - Tài liệu nói về buffer coherent lớn (`dma_alloc_coherent`), DMA pool cho buffer nhỏ, giới hạn địa chỉ DMA và streaming mapping.
    - Đọc kèm "Dynamic DMA mapping Guide" cùng thư mục tài liệu, vì tài liệu này có hướng dẫn nhẹ nhàng hơn.

**IOMMU**

- **Kernel doc "Linux IOMMU Support" (Intel VT-d)**: https://docs.kernel.org/5.17/x86/intel-iommu.html
    - Tài liệu giải thích các thuật ngữ DMAR, DRHD, RMRR, IOVA, cách IOVA được sinh ra khi driver gọi các hàm map, và cách DMA engine báo lỗi bằng interrupt.
- **openEuler "VFIO Device Passthrough Principles (2)"**: https://www.openeuler.org/en/blog/wxggg/2020-11-29-vfio-passthrough-2
    - Bài mô tả cách VFIO và QEMU làm DMA remapping: thiết bị dùng IOVA, và IOMMU ánh xạ IOVA sang địa chỉ vật lý.
    - Bài này nối phần IOMMU với trường hợp passthrough trong VM.

**MMU (EPT/NPT)**

- **arXiv 2006.00380**: https://arxiv.org/pdf/2006.00380
    - Bài mô tả EPT dùng hai tầng page table: tầng guest do OS của guest quản lý, tầng thứ hai nằm ở hypervisor. Khi TLB miss, bộ walker duyệt hai chiều và có thể cần tới 24 lần truy cập bộ nhớ.
- **arXiv 1701.07517**: https://arxiv.org/pdf/1701.07517
    - Bài có hình minh họa page walk hai chiều, và giải thích MMU cache cùng nested TLB làm gì để tăng tốc walk.
- **VMware "Performance Evaluation of Intel EPT"**: https://www.cs.miami.edu/~burt/learning/Csc521.121/docs/Perf_ESX_Intel-EPT-eval.pdf
    - Bài so sánh shadow paging và EPT, và phân tích chi phí page walk khi dùng EPT.
- Ba bài trên đều là bài học thuật, nên đọc phần giải thích hình vẽ trước.

**Storage/host controller và disk controller**

- Mình không tìm được bài riêng về HBA cắm PCIe, hay về HBA offload như bnx2i và cxgb4i. Các bài NVMe dưới đây là gần nhất để hiểu controller phía PCIe (doorbell, queue, MSI-X):
    - **OSDev wiki "NVMe"**: https://wiki.osdev.org/NVMe
        - Bài liệt kê các thanh ghi queue, doorbell của từng submission/completion queue, kích thước entry 64 và 16 byte, và lưu ý về MSI-X vector.
    - **SPDK "Submitting I/O to an NVMe Device"**: https://src.rcs.uwaterloo.ca/xref/spdk/doc/nvme_spec.md
        - Bài giải thích quy trình đặt lệnh vào submission queue, ghi tail doorbell, rồi đọc completion queue và ghi head doorbell.
    - **Kernel doc "NVMe PCI Endpoint Function Target"**: https://docs.kernel.org/nvme/nvme-pci-endpoint-target.html
        - Tài liệu mô tả phía controller: lấy lệnh từ submission queue (bằng DMA hoặc MMIO), poll doorbell, ghi completion entry rồi phát interrupt cho host.
        - Đây là cách nhìn từ phía controller, hiếm khi có.
- **Disk controller (FTL, cache)**:
    - **"Coding for SSDs, Part 3"**: https://codecapsule.com/?p=1865
        - Bài giải thích FTL nằm trong controller SSD, có hai mục đích chính là ánh xạ logical block và garbage collection, kèm write amplification.
    - **"Errors in Flash-Memory-Based SSDs" (survey)**: https://arxiv.org/pdf/1711.11427
        - Phần về FTL và garbage collection giải thích ghi out-of-place, đánh dấu page cũ là invalid, và GC chọn block có nhiều page invalid nhất.
    - Bạn nhắc "flush, FUA, write cache trong ổ" trong bảng, nhưng mình chưa tìm được bài riêng về phần đó.

**RDMA**

- **NVIDIA "RDMA Aware Networks Programming User Manual"**: https://docs.nvidia.com/networking/display/rdmaawareprogrammingv17/typical+application
    - Tài liệu mô tả luồng chuẩn: đăng ký memory region (nhận lkey/rkey), tạo completion queue, tạo queue pair, rồi post work request và poll completion.
- **jcxue "RDMA-Tutorial" wiki**: https://github.com/jcxue/RDMA-Tutorial/wiki
    - Tutorial ví queue pair như địa chỉ của hai đầu giao tiếp (giống socket). Hai bên cần trao đổi `rkey` và `raddr`, tức khóa và địa chỉ vùng nhớ của bên nhận, để bên gửi ghi thẳng vào.
    - Điều này khớp với phần mình sửa trước đó: định danh nằm ở QP, còn "tiến trình" thì không.
- **Slide "RDMA Tutorial" (netdev 0x16)**: https://netdevconf.info/0x16/slides/40/RDMA%20Tutorial.pdf
    - Slide nói đường dữ liệu chạy trên hàng đợi submission/completion bất đồng bộ, thao tác một phía truy cập bộ nhớ của đối tác mà không cần CPU của đối tác, và bộ nhớ phải được đăng ký và pin trước.

## Chưa tìm được

- Luồng nhận PDU trong `iscsi_tcp` (TCP receive → `scsi_done`).
- iSER và NVMe-oF/RDMA.
- vhost-user-blk với SPDK.
- HBA offload.
- Flush/FUA ở mức ổ đĩa.

Nếu bạn muốn, mình tìm tiếp một trong các mục này, theo thứ tự bạn thấy cần nhất cho phần bảng trắng.