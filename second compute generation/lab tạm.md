Mình tách thành 4 kịch bản, vì số lần copy phụ thuộc vào cấu hình. Quy ước:

- **Copy CPU**: CPU chạy `memcpy`/`copy_from_user` (đây là thứ cần đếm).
- **DMA**: thiết bị tự đọc/ghi RAM, không tính là memcpy nhưng tốn băng thông bus.
- **Remap/pin**: chỉ đổi con trỏ, descriptor hoặc ghim page, không copy payload.

Con số dưới đây là **suy ra từ kiến trúc**, nên phải kiểm chứng bằng đo (cuối bài).

## A. Máy thường, write buffered (`echo ... > file`)

|Bước|Hàm / cấu trúc|Copy payload|
|---|---|---|
|1|`write()` → `entry_SYSCALL_64` → `ksys_write` → `vfs_write`|không|
|2|Filesystem ghi vào page cache (`generic_perform_write`)|**Copy #1**: `copy_from_user` từ user buffer vào page cache|
|3|Page dirty, chờ writeback (`write()` trả về ở đây)|không|
|4|Writeback tạo `bio` trỏ vào page cache|remap|
|5|blk-mq → driver (SCSI/NVMe) dựng SG list, `dma_map_sg`|remap|
|6|Controller DMA đọc từ page cache|DMA|

**Tổng: 1 copy CPU + 1 DMA.**

## B. Máy thường, `O_DIRECT`

Bỏ bước page cache. Kernel ghim page của user buffer (`pin_user_pages`), dựng bio từ đúng các page đó, controller DMA thẳng từ user buffer. **Tổng: 0 copy CPU + 1 DMA.** Điều kiện là buffer và offset phải đúng alignment, nếu không sẽ lỗi `EINVAL` hoặc phát sinh bounce.

## C. VM qua iSCSI + multipath (chính là lab của bạn)

`driver=virtio_blk, cache=none, io=native`, ổ `mpatha` gắn vào guest, storage host dùng LIO + fileio.

|Tầng|Điều gì xảy ra|Copy payload|
|---|---|---|
|Guest userspace|App gọi `write()`||
|Guest kernel|Buffered: copy vào guest page cache|**#1** (không có nếu app dùng O_DIRECT)|
|Guest `virtio_blk`|Dựng descriptor chain: header 16B + các buffer trỏ tới page của guest + status 1B|remap|
|KVM|Guest ghi doorbell, VM exit, ioeventfd đánh thức QEMU|không|
|QEMU|Đọc descriptor, map guest memory (GPA → HVA), dựng iovec|remap (header/status chỉ vài byte)|
|QEMU `io=native`|`io_submit` lên fd `/dev/mapper/mpatha` mở bằng `O_DIRECT` (do `cache=none`)|không|
|Host kernel|Ghim page của QEMU (chính là RAM guest), dựng bio|remap/pin|
|dm-multipath|Chọn path, remap bio/request sang `sdX`|không|
|SCSI + `iscsi_tcp`|Đóng gói CDB + PDU header (vài chục byte, copy bằng sendmsg), payload đưa vào skb bằng page reference|không đáng kể ở data|
|TCP/NIC|NIC DMA đọc payload từ RAM|DMA|
|Storage host, LIO rx|`iscsi_target` nhận TCP, copy từ skb vào buffer của target|**#2**|
|LIO `fileio`|`vfs_iter_write` ghi vào page cache của file backing (dù `write-thru` vẫn đi qua page cache rồi mới sync)|**#3**|
|Storage host|Writeback → block layer → DMA xuống disk|DMA|

**Tổng: 3 copy CPU nếu guest ghi buffered, 2 nếu guest dùng O_DIRECT.** Hai copy sau nằm ở phía target, không phải ở host KVM. Nếu target dùng backstore `iblock` (block device thay cho file), copy #3 biến mất.

Đây cũng là lý do ở host KVM không thấy memcpy lớn ở vhost: với `cache=none` + O_DIRECT, đường host gần như zero-copy cho payload.

**Những điều kiện làm phát sinh thêm copy:**

- Buffer không aligned theo yêu cầu của O_DIRECT: QEMU phải dùng **bounce buffer** (copy thêm).
- NIC (hoặc NIC ảo) không hỗ trợ scatter-gather/checksum offload: kernel có thể phải gộp skb (copy).
- Phiên bản kernel khác nhau: `sendpage` đã được thay bằng `MSG_SPLICE_PAGES` ở kernel mới, hành vi gửi payload có thể khác, cần đối chiếu source đúng version.
- Header/data digest (CRC32C) chỉ đọc dữ liệu, tốn CPU nhưng không tính là copy.

## D. vhost-user-blk + SPDK + NVMe-oF/RDMA (hướng dự án)

|Tầng|Điều gì xảy ra|Copy payload|
|---|---|---|
|Guest|Giống trên (buffered: copy vào guest page cache)|**#1** hoặc 0|
|`virtio_blk`|Descriptor trỏ vào RAM guest|remap|
|vhost-user|RAM guest được chia sẻ qua fd (hugepage); SPDK mmap chung vùng này, **poll** vring, không cần VM exit|remap (dịch GPA → VA)|
|SPDK bdev|Con trỏ payload chính là địa chỉ trong RAM guest|không|
|NVMe-oF/RDMA initiator|Bộ nhớ guest đã được đăng ký (memory region), NIC DMA thẳng|DMA|
|Target|NIC đặt dữ liệu vào buffer đã đăng ký, SPDK NVMe driver submit thẳng xuống SSD|DMA|

**Tổng: 0 copy CPU (nếu guest O_DIRECT) hoặc 1 (guest buffered).** Nếu thay RDMA bằng NVMe-oF/TCP thì thêm 1 copy ở mỗi đầu (`sendmsg`/`recv` qua kernel TCP), trừ khi bật zero-copy.

## Bảng tóm tắt

|Kịch bản|Copy CPU (guest buffered)|Copy CPU (O_DIRECT)|Ghi chú|
|---|---|---|---|
|A. Máy thường, buffered|1|n/a||
|B. Máy thường, O_DIRECT|n/a|0||
|C. VM + iSCSI + fileio|3|2|2 copy ở target|
|C'. như C nhưng `iblock`|2|1||
|D. vhost-user + SPDK + RDMA|1|0|cần memory registration|
|D'. vhost-user + SPDK + NVMe/TCP|3|2|thêm copy qua kernel TCP|

Ở mọi kịch bản còn có thêm các cái chi phí không phải memcpy nhưng đáng đo: syscall, context switch, interrupt, VM exit, đổi quyền truy cập cache line.

## Cách đo để kiểm chứng

Đếm byte đi qua các hàm copy, chạy cùng lúc với `fio`:

```bash
# danh sách probe khả dụng (tên hàm khác nhau theo kernel)
bpftrace -l 'kprobe:*copy_from_iter*'
bpftrace -l 'kprobe:*copy_to_iter*'

# tổng byte copy từ user/iov vào kernel, theo tiến trình
bpftrace -e 'kprobe:_copy_from_iter { @from[comm] = sum(arg1); }
             kprobe:_copy_to_iter   { @to[comm]   = sum(arg1); }
             interval:s:10 { exit(); }'

# copy qua socket (TCP rx) và tx
bpftrace -e 'kprobe:skb_copy_datagram_iter { @rx[comm] = sum(arg2); }
             kprobe:tcp_sendmsg { @tx[comm] = count(); }'
```

Cách đọc: chạy `fio` với một khối lượng đã biết (ví dụ ghi 1 GiB). Nếu byte copy ≈ 1 GiB thì có 1 copy payload, ≈ 2 GiB thì 2 copy, v.v. Làm lần lượt:

1. `fio` trong guest với `direct=0` rồi `direct=1`, đo trên **host KVM** (QEMU, kernel).
2. Đo song song trên **storage host** để thấy copy rx và fileio.
3. `perf record -a -g` kèm flamegraph để xem tỷ trọng `memcpy`/`copy_user_*` so với tổng CPU.
4. Đo thêm trong QEMU bằng `perf trace` hoặc uprobe vào `memcpy` để bắt bounce buffer.

So sánh các trường hợp: ghi thẳng vào `/dev/mapper/mpatha` trên host (kịch bản B + iSCSI) và ghi từ guest. Hiệu số cho biết phần ảo hoá thêm vào là bao nhiêu.

Nếu muốn, mình viết sẵn một script bpftrace đầy đủ cho cả host và storage host, kèm bảng ghi kết quả để bạn điền số đo thực tế.


Mình đọc từ 4 ảnh thì bài toán chính là: **một lệnh write từ VM đi xuống ổ như thế nào, khác gì máy thường, có bao nhiêu lần memcpy và đo bằng cách nào.** Bối cảnh là so sánh iSCSI + multipath (local) với vhost. Có vài chỗ chữ tay khó đọc (phần HBA/ASIC/L1/L2 góc trên trái) nên mình đọc theo ngữ cảnh, bạn kiểm tra lại giúp.

## 1. Đường write trên máy thường (baseline)

- `write()` → VFS → page cache hay O_DIRECT → filesystem → block layer → dm (multipath) → SCSI → driver → thiết bị.
- Cấu trúc `struct bio`, `bio_vec`, `request`, `scsi_cmnd`, blk-mq (software/hardware queue), scatter-gather list.
- **Page cache vs O_DIRECT**: `echo "..." > file` đi qua page cache, còn O_DIRECT bypass. Cần nắm yêu cầu alignment (512/4K), pin user page (`get_user_pages`), DMA trực tiếp từ buffer user.
- Flush/FUA/fsync, write-back cache của thiết bị (RAID controller, BBU).
- Phần cứng: RAID controller, HBA, device controller và ai làm gì (bảng vẽ có "HW: RAID, device ctl").

## 2. Cache mode và io mode của QEMU

- `cache=none` = O_DIRECT phía host, nhưng guest vẫn có page cache riêng. So sánh `writeback`, `writethrough`, `directsync`, `unsafe`.
- `io=native` (Linux AIO/libaio, cần O_DIRECT) vs `threads` vs `io_uring`. Vì sao `native` đi kèm `cache=none`.
- Ý nghĩa của "echo: userspace" trên bảng: write trong guest chỉ là bước đầu.

## 3. Ảo hoá block device (phần "mô phỏng phần cứng bằng phần mềm")

- **virtio-blk vs virtio-scsi** (bảng có `sda`, `sdx`): khác biệt về driver, số queue, passthrough SCSI command.
- **virtqueue/vring**: descriptor table, avail ring, used ring. Cơ chế kick (ioeventfd) và interrupt (irqfd), chi phí VM exit.
- QEMU block layer: AioContext, iothread, dataplane, multiqueue (`num-queues`).
- Driver `virtio_blk` trong guest (blk-mq).
- Bộ nhớ guest: GPA → HVA → HPA, EPT, hugepage. Khớp với phần "VA / PA" trên bảng.

## 4. vhost và vhost-user (trọng tâm)

- vhost-kernel vs **vhost-user-blk**: giao thức qua unix socket, memory table gửi bằng fd (SCM_RIGHTS), process bên ngoài (SPDK) mmap trực tiếp RAM của guest.
- Vì sao vhost giảm copy: descriptor chỉ trỏ tới địa chỉ guest, backend đọc thẳng payload.
- Hình bên phải bảng: `memcpy(src, dst)` với offset/length, cùng vùng nhớ nhìn từ VA và PA. Cần hiểu việc dịch địa chỉ guest sang địa chỉ backend, và khi nào **buộc phải copy** (unaligned, bounce buffer, không map được).
- Yêu cầu hugepage/pinned memory. Nối với RDMA memory registration ở mục 6.

## 5. Đếm memcpy và overhead (câu hỏi lớn nhất trên bảng)

Cần vẽ ra từng điểm copy tiềm năng, rồi **kiểm chứng bằng đo**, không suy đoán:

- guest user → guest page cache (mất nếu O_DIRECT);
- QEMU `pwritev` → host kernel (O_DIRECT có copy không?);
- iSCSI: `tcp_sendmsg` copy vào skb hay dùng sendpage/zero-copy; header/data digest (CRC32C);
- chiều đọc: `skb` → page (`tcp_recvmsg`);
- multipath/dm chỉ remap bio, không copy payload. Cần xác nhận.
- Ngoài memcpy còn có syscall, context switch, interrupt, VM exit, tất cả đều tính vào overhead.

## 6. Phía storage: iSCSI vs NVMe-oF

- **iSCSI**: PDU, login, CmdSN/StatSN, R2T, ImmediateData, FirstBurstLength/MaxBurstLength (đặc biệt quan trọng với write), `iscsi_tcp`, libiscsi, open-iscsi (session, queue_depth, cmds_max).
- **dm-multipath**: path selector, path grouping, `queue_if_no_path`, failover.
- **HBA offload**: iSCSI software initiator vs offload HBA (ASIC xử lý TCP/iSCSI, bypass network stack của kernel) và hệ quả với số lần copy/CPU.
- **NVMe-oF/RDMA**: verbs, QP/CQ, memory region, RDMA WRITE/READ/SEND, NVMe SQ/CQ, doorbell, PRP/SGL.
- **SPDK**: poll mode, reactor, bdev layer, `bdev_nvme`, DPDK, hugepage, lockless ring.

## 7. Công cụ đo

- **fio**: `direct`, `ioengine`, `iodepth`, `numjobs`.
- **blktrace/blkparse/btt** (bảng ghi blk-trace): các event Q/G/I/D/C, tách latency Q2D và D2C.
- **ftrace/trace-cmd** (function_graph, event `block:*`, `scsi:*`, `kvm:*`, `net:*`).
- **perf** (`record -g`, flamegraph) để xem tỷ trọng memcpy. Kprobe/eBPF vào `copy_user_*`, `_copy_from_iter`, `tcp_sendmsg`, `skb_copy_datagram_iter`.
- **bcc/bpftrace**: `biolatency`, `biosnoop`.
- **kvm_stat / perf kvm stat**: đếm VM exit.

## Thứ tự nên học

1. Nắm chắc luồng máy thường (mục 1 + 2), tự vẽ lại và chỉ ra từng struct.
2. Hiểu virtio và vring (mục 3), rồi vhost-user (mục 4).
3. Đo thật, đối chiếu 3 trường hợp: host trực tiếp, VM qua QEMU virtio-blk, VM qua vhost-user. Lab iSCSI + virtio-blk sẵn có của bạn chính là mốc so sánh. Có thể chạy `blktrace` trên `dm-0`/`sdb` ở host và `vdb` ở guest để thấy độ trễ mỗi tầng, và dùng perf/ftrace đếm copy trên đường write.
4. Sau cùng mới sang RDMA/NVMe-oF/SPDK (mục 6), lúc đó bạn đã biết phải so sánh cái gì.

Nếu muốn, mình đi sâu từng mục hoặc soạn kế hoạch thí nghiệm đo memcpy cụ thể trên lab của bạn.