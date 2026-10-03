                
                  CPU / RAM
                    │ PCIe
      ┌──────────────┴──────────
      │                                             
 47:00.0  card RAID HPE MR216i-a Gen10+        
 (Broadcom SAS39xx, driver megaraid_sas)    
 PCIe Gen4 x8 (16GT/s, đang chạy đủ mức)             
      │ SATA 6 Gb/s                                 
      ├── SSD #0 ─┐                         
      └── SSD #1 ─┴── gộp thành ổ ảo               
                       │                           
                  Linux thấy:              
                  /dev/sda (446.6G)           
                  SCSI địa chỉ 0:3:111:0      
                  5 phân vùng sda1..sda5      

### 1. Hệ Thống & Môi trờng Tracing

- **Kernel:** `5.4.0-167-generic` (Ubuntu 20.04 LTS kernel).
- **Hạ tầng Tracing:** Cả `debugfs` và `tracefs` đều đã được mount sẵn sàng tại `/sys/kernel/debug` và `/sys/kernel/tracing`. Bạn có đầy đủ công cụ để sử dụng `ftrace`, `trace-cmd`, hoặc bắt các tracepoint (`block:*`, `scsi:*`, `net:*`).
### 2. Trạng thái 2 ổ đĩa trên máy

| **Tên**        | **Loại (ROTA / TRAN)**                           | **Phần cứng / Controller**                     | **Dung lượng** | **Hiện trạng sử dụng**                                                    |
| -------------- | ------------------------------------------------ | ---------------------------------------------- | -------------- | ------------------------------------------------------------------------- |
| **`/dev/sda`** | `ROTA=0` (SSD)<br><br>  <br>  <br><br>Local Disk | HPE Smart Array / MegaRAID **MR216i-a_Gen10+** | **446.6 GB**   | **Chứa toàn bộ hệ điều hành** (`sda1` đến `sda5`). Root `/` nằm ở `sda5`. |
|                |                                                  |                                                |                |                                                                           |

- **Thông số Block Layer của `/dev/sda`:**
    - **I/O Scheduler:** `[mq-deadline]` (Đang kích hoạt thuật toán mq-deadline, có tùy chọn `none`).
    - **Queue Depth:** `32` (giới hạn phần cứng của controller).
    - **`nr_requests`:** `256` (số lượng request tối đa nằm chờ trong queue).
    - **`write_cache`:** `write through` (ghi trực tiếp, an toàn dữ liệu nhưng không tận dụng write-back cache).
    - **`max_sectors_kb`:** `64` (kích thước I/O tối đa gửi xuống controller trong 1 request là 64 KB).
        

## Công cụ theo từng tầng của slide

|Tầng|Công cụ|Thấy gì|
|---|---|---|
|User → kernel|`strace -T -tt` hoặc tracepoint `syscalls:sys_enter/exit_pwrite64`|Thời điểm vào/ra syscall, thời gian cả lệnh|
|VFS|`ftrace function_graph` (`ksys_pwrite64`, `vfs_write`)|Chuỗi hàm, thời gian từng hàm|
|Filesystem (ext4/xfs)|`ftrace function_graph`, `ext4:*`, `iomap:*`|Ánh xạ file offset → block (extent), chọn đường DIO hay buffered|
|Page cache|`writeback:*`, `filemap:*`, `/proc/meminfo` (Dirty)|Chỉ có ở buffered. Với O_DIRECT bạn sẽ thấy **không có**|
|Block device / bio|`block:block_bio_queue`, `block_bio_remap`, `block_split`|Bio xuất hiện, bị remap hoặc cắt|
|IO scheduler / blk-mq|`block:block_getrq`, `block_rq_insert`, `block_plug/unplug`, merge events|Cấp request, vào hàng đợi, gộp|
|Driver|`block:block_rq_issue`, `scsi:scsi_dispatch_cmd_start`, `nvme:nvme_setup_cmd`, `libata:*`|Lệnh thật gửi cho phần cứng (CDB, hoặc NVMe opcode/slba/nlb)|
|Ngắt, hoàn thành|`irq:irq_handler_entry`, `irq:softirq_entry`, `block:block_rq_complete`, `scsi:scsi_dispatch_cmd_done`|Thiết bị báo xong, ai xử lý|
|Disk|Không trace được từ phần mềm|Chỉ đo gián tiếp (xem cuối)|



## 1. Dòng thời gian một lệnh write (từ `ev.txt`)

Gốc thời gian là lúc `sys_enter_pwrite64` (CPU 15, tiến trình `dw`).

|Mốc|Sự kiện|Tầng trên slide|
|---|---|---|
|+0 µs|`sys_enter_pwrite64`, count=0x1000, pos=0|User → Kernel|
|+4|`writeback_mark_inode_dirty`, `I_DIRTY_SYNC\|I_DIRTY_TIME`|VFS/ext4|
|+20|`block_bio_remap`: `8,0 232118272 + 8 <- (8,5) 76822528`|Block device|
|+21|`block_bio_queue` (Q)|Block layer|
|+25|`block_getrq` (G)|blk-mq|
|+26 đến +28|`block_plug`, `block_unplug`, `block_rq_insert` (I)|blk-mq|
|+32|`block_rq_issue` (D)|Driver|
|+35|`scsi_dispatch_cmd_start`: `WRITE_10 lba=232118272 txlen=8`, CDB `2a 00 0d d5 d8 00 00 00 08 00`|SCSI|
|+62|`irq_handler_entry`, `megasas0-msix15` (irq 166)|Ngắt|
|+63|`scsi_dispatch_cmd_done`, `status=SAM_STAT_GOOD`|SCSI|
|+65, +66|`softirq_entry [BLOCK]`, `block_rq_complete` (C)|Hoàn thành|
|+74|`sys_exit_pwrite64` = 0x1000|Về user|

## 2. Giải thích từng chặng

**Syscall và VFS** (`fg.txt`). Chuỗi hàm là `__x64_sys_pwrite64` → `ksys_pwrite64` → `vfs_write`. Trước khi ghi, kernel kiểm tra quyền qua `rw_verify_area` → `security_file_permission` → `apparmor_file_permission`. Đây là tầng LSM (Ubuntu bật AppArmor), chỉ tốn vài µs.

**ext4** (`fg.txt`). `ext4_file_write_iter` → `ext4_map_blocks` đổi offset trong file thành block trên đĩa (tra trong extent status tree, không phải đọc đĩa). Tiếp theo là `file_update_time` → `ext4_dirty_inode`, tức là **cập nhật mtime của file bằng một transaction journal** (khoảng 21 µs). Vì vậy ghi 4K dữ liệu vẫn kéo theo ghi metadata.

**Chuẩn bị cho direct I/O**. `generic_file_direct_write` gọi `filemap_write_and_wait_range` rồi `invalidate_inode_pages2_range`. Hai hàm này đảm bảo không còn page cache dirty hoặc cũ cho vùng sắp ghi (page cache và đĩa không được lệch nhau). Cả hai chạy nhanh vì cache rỗng.

**Dựng bio** (`fg.txt`). `ext4_direct_IO_write` → `__blockdev_direct_IO` → `blk_start_plug`, `do_direct_IO`, `bio_alloc_bioset`, `bio_add_page`, `submit_bio`. `bio_add_page` cho thấy bio **trỏ thẳng vào page của user buffer** (không có bước ghi vào page cache).

**Remap**. `block_bio_remap` đổi `(8,5) 76822528` thành `(8,0) 232118272`. Hiệu hai số là 155295744, đúng sector bắt đầu của `sda5`. Nghĩa là kernel chuyển địa chỉ từ "sector trong partition" sang "sector trên cả đĩa". Con số `9602816` trong `buf.txt` là block 4K của ext4, nhân 8 ra đúng `76822528`.

**Plug và dispatch**. Request được `blk_start_plug` giữ lại, rồi `blk_finish_plug` (khoảng 14 µs trong `fg.txt`) đẩy xuống driver. Chỗ này khớp với `block_plug/unplug` trong `ev.txt`, và với dòng `P`, `U` trong `bp.txt`.

**SCSI và driver**. `scsi_dispatch_cmd_start` hiện lệnh thật: `WRITE_10`, địa chỉ `host=0 channel=3 id=111 lun=0` (đúng địa chỉ `0:3:111:0` của ổ ảo RAID), 1 scatter-gather entry (`data_sgl=1`), độ dài 8 sector. Tôi đã kiểm tra: bytes `0d d5 d8 00` trong CDB là 0x0dd5d800 = 232118272, khớp `lba`.

**Chờ và hoàn thành**. Trong `fg.txt`, tiến trình gọi `io_schedule` khoảng 35 µs, tức là **ngủ chờ thiết bị**. Sau đó ngắt `megasas0-msix15` đến chính CPU 15 (CPU đã gửi lệnh), qua softirq `BLOCK` rồi `block_rq_complete`. Tiếp theo `dio_bio_complete`, `dio_complete` đánh thức tiến trình và `pwrite` trả về 4096.

## 3. Phân rã thời gian

Mình tính từ `ev.txt` (không bị nhiễu bởi function_graph):

|Chặng|Thời gian|Ghi chú|
|---|---|---|
|Syscall → bio queue (VFS, ext4, mtime, dựng bio)|khoảng 21 µs|phần mềm trên block layer|
|Q → D (block layer, plug, blk-mq)|khoảng 11 µs|khớp `btt`: Q2G + G2I + I2D vài µs|
|D → C (driver, RAID controller, SSD)|khoảng 31 µs|`scsi start→done` là 28 µs|
|C → syscall exit|khoảng 8 µs|đánh thức tiến trình|
|**Tổng `pwrite`**|**74 µs**||

Với `btt` (tính cho cả 19 request có cả jbd2): `D2C` chiếm 93,7% thời gian `Q2C`, trung bình 84 µs. Dù vậy với riêng request của `dw`, phần phần mềm (Q2D) chỉ khoảng 7 đến 8 µs trong `bp.txt`. Kết luận: **thời gian nằm ở sau driver, và phần phần mềm của block layer rất nhẹ**.

Hai lưu ý về con số. `fg.txt` hiện tổng 129 µs vì function_graph làm chậm kernel, đừng lấy làm độ trễ thật. Về 31 µs cho D→C: nhanh hơn nhiều so với một SSD SATA ghi thẳng ra NAND, nên mình **nghi** controller (hoặc cache của SSD) xác nhận lệnh trước khi dữ liệu xuống media. Đây mới là suy luận, cần `storcli /c0/vall show all` để xác nhận policy cache.

## 4. Sau khi `pwrite` xong: lệnh `fsync` sinh thêm I/O

Ở `ev.txt`, ngay sau `sys_exit_pwrite64` (+74 µs), tiến trình `jbd2/sda5-8` bắt đầu ghi. Đây là journal của ext4 bị `fsync` kích hoạt (vì mtime đã dirty ở bước trên). Có hai request:

- Một request gộp: nhiều bio 4K liền kề được `block_bio_backmerge` thành **một request 88 sector (45056 byte)**. Trong `bp.txt` đó là các dòng `M` (merge) rồi `I WS ... + 96`.
- Một block commit 4K.

Chi tiết quan trọng: block commit hiện ở `block_bio_remap` là **`FWFS`** (có `PREFLUSH` và `FUA`), nhưng đến `block_bio_queue` chỉ còn **`WS`**. Block layer đã **bỏ cờ flush/FUA** vì thiết bị khai báo `write through` (đúng với `/sys/block/sda/queue/write_cache` của bạn). Vì thế không có lệnh `SYNCHRONIZE CACHE` nào được gửi. Bằng chứng này khớp với phân tích trước đó của mình.

Tóm lại, **một lệnh `write` 4K cộng `fsync` tạo ra khoảng 3 request xuống controller**: 4K dữ liệu, journal gộp, và commit block.

## 5. So với buffered (`buf.txt`)

||O_DIRECT|Buffered|
|---|---|---|
|Thời gian `pwrite64`|74 µs|16 µs|
|`block_*` trong cửa sổ syscall|có (Q, G, I, D, C)|không|
|Sự kiện chính|`block_bio_queue`, `scsi_dispatch_cmd_start`|`block_dirty_buffer`, `writeback_dirty_page index=0`|
|Dữ liệu xuống đĩa|trong lúc `pwrite`|sau, khi `fsync`/`sync`|

Ở bản buffered, `pwrite` chỉ đánh dấu page `index=0` là dirty rồi về ngay. Request ghi 4K của `dw` (cùng sector 232118272, `block_rq_issue` ở `buf.txt` dòng khoảng +49 µs sau syscall) chỉ đến khi `fsync` chạy. Khoảng 1 giây sau còn có `writeback_start ... reason=sync` do lệnh `sync` của mình kích hoạt kworker ghi các block metadata (`WM`).

## 6. Trả lời các câu hỏi ban đầu của bạn

- **Có page cache không?** Không (O_DIRECT). Không có `writeback_dirty_page` của file dữ liệu trong cửa sổ syscall.
- **Có copy payload không?** Trong `fg.txt` bạn không thấy `copy_from_user` hay `copy_page_from_iter`. Nhưng `--max-graph-depth 12` cắt cụt nhánh dưới `do_direct_IO` (nơi kernel ghim page của user), nên **chưa chứng minh chắc**. Muốn chắc hơn, chạy lại với độ sâu lớn hơn, hoặc dùng bpftrace đếm byte `_copy_from_iter`.
- **Đến đâu thì software hết?** Đến `scsi_dispatch_cmd_start`. Sau đó là driver `megaraid_sas` và controller. Ngắt `megasas0-msix15` cho thấy đường về.
- **Ngắt về CPU nào?** Về đúng CPU 15 đã gửi lệnh. Ở lần ghi jbd2, CPU 3 nhận ngắt `msix59`, nên ánh xạ CPU với vector không đơn giản là cùng số. Để biết chắc, xem `cat /proc/irq/166/smp_affinity_list` và `/sys/block/sda/mq/*/cpu_list`.

## 7. Một chỗ mình cần sửa so với phần trước

Mình từng dự đoán chuỗi `ext4_dio_write` / `iomap_dio_rw`. Log của bạn cho thấy kernel này dùng **`ext4_direct_IO` → `__blockdev_direct_IO` → `do_direct_IO`**, tức đường direct I/O kiểu cũ (trước khi ext4 chuyển sang iomap, khoảng kernel 5.5). Nếu báo cáo cần ghi version kernel, dùng `uname -r` của com07. Mình đoán nó là bản 4.x hoặc 5.4, nhưng log không nói thẳng.

## 8. Nên làm tiếp

1. Chạy lại function_graph với `--max-graph-depth 20` để thấy bên trong `do_direct_IO` (pin page) và `blk_mq_submit_bio`, `scsi_queue_rq`, `megasas_queue_command`. Phần này vẫn nằm trong "đến driver", chưa đụng vào RAID.
2. Chạy bpftrace đếm `_copy_from_iter` khi chạy `dw` direct và buffered để chứng minh số copy.
3. Lưu thêm `strace -T` để có độ trễ nhìn từ user.

Bạn muốn mình viết sẵn lệnh cho bước 1 và 2 không, hay vẽ sơ đồ dòng thời gian này thành hình để dán vào báo cáo?
## Quy trình: bắt đúng một lệnh write O_DIRECT

**Chuẩn bị**: dùng chương trình `dw.c` ở trên nhưng **xoá dòng `getchar()`** (khi chạy qua `trace-cmd -F` không có nơi nhập). Chạy trên máy ít tải, ghi vào file trên filesystem (không ghi vào `/dev/sda`).

**Bước 1: thu tất cả sự kiện từ syscall đến thiết bị**

```bash
sudo trace-cmd record \
  -e syscalls:sys_enter_pwrite64 -e syscalls:sys_exit_pwrite64 \
  -e block -e scsi -e nvme -e writeback \
  -e irq:irq_handler_entry -e irq:softirq_entry \
  -F ./dw /storage/t.img
trace-cmd report | less
```

Nếu `ext4`/`iomap` có trên kernel của bạn thì thêm `-e ext4 -e iomap`. Đọc theo thứ tự thời gian: bạn sẽ thấy `sys_enter_pwrite64`, các sự kiện `block_*`, sự kiện driver, ngắt, `block_rq_complete`, `sys_exit_pwrite64`.

**Bước 2: xem chuỗi hàm bên trong kernel**

```bash
sudo trace-cmd record -p function_graph --max-graph-depth 12 \
  -g __x64_sys_pwrite64 -F ./dw /storage/t.img
trace-cmd report > fg.txt
```

Tìm `vfs_write` → hàm ghi của filesystem → nhánh direct I/O → `submit_bio`. Nếu thấy quá nhiều dòng, tăng/giảm `--max-graph-depth`. Hàm bị inline sẽ không hiện.

**Bước 3: góc nhìn block layer gọn hơn (blktrace)**

```bash
sudo blktrace -d /dev/<thiết bị chứa /storage> -o t &
./dw /storage/t.img
sudo kill -INT %1 ; sleep 1
blkparse -i t | less
```

Mỗi dòng có dạng `major,minor cpu seq time pid ACTION RWBS sector + len [comm]`. Các mã hành động chính:

- `Q`: bio vào queue, `X`: bị cắt, `M/F`: gộp
- `G`: cấp request, `I`: insert vào scheduler
- `D`: đẩy xuống driver, `C`: hoàn thành

RWBS có `W` (write), `S` (sync), `F`/`FUA` nếu là flush/FUA. Với một ghi 4K bạn sẽ thấy `... + 8` (8 sector × 512B). Dùng `btt -i t.bin` để có thời gian từng chặng: `Q2G`, `I2D`, `D2C`.

**Bước 4: so với đường buffered**  
Chạy lại với `./dw /storage/t.img buffered`. Lần này `pwrite64` trả về rất nhanh và **không có** `block_*` ngay sau đó. Sự kiện block chỉ xuất hiện về sau từ tiến trình writeback (hoặc khi `fsync`). Đây là cách nhìn trực tiếp sự khác biệt mà slide thể hiện bằng ô Page Cache.

## Đọc kết quả thế nào

Thứ tự bạn nên thấy với O_DIRECT (minh họa, không phải output thật):

```
sys_enter_pwrite64
 → (VFS/fs/iomap: ánh xạ offset → block)
 → block_bio_queue      Q
 → block_getrq          G
 → block_rq_issue       D   (+ scsi_dispatch_cmd_start hoặc nvme_setup_cmd)
 ... thiết bị xử lý ...
 → irq_handler_entry
 → scsi_dispatch_cmd_done / nvme_complete_rq
 → block_rq_complete    C
 → sys_exit_pwrite64
```

Chênh lệch thời gian giữa `D` và `C` chính là thời gian thiết bị (và đường truyền) xử lý. Chênh lệch giữa `sys_enter` và `Q` là phần mềm phía trên (VFS, filesystem, pin page).

## Phần "tận cùng ổ disk"

Từ driver trở xuống, phần mềm chỉ thấy lệnh gửi đi và lúc báo xong, không thấy bên trong ổ. Muốn biết thêm:

- **Lệnh cụ thể**: `scsi_dispatch_cmd_start` hiển thị CDB (opcode `2a` là WRITE(10), `8a` là WRITE(16), `35` là SYNCHRONIZE CACHE). Với NVMe, `nvme_setup_cmd` cho opcode, `slba`, `len`.
- **Cache ổ**: `hdparm -W`, `nvme get-feature -f 6`, `cat /sys/block/X/queue/write_cache`, `sg_modes`.
- **Trạng thái ổ**: `smartctl -a`, `nvme smart-log`.
- **Độ trễ thiết bị thuần**: chạy `fio --iodepth=1 --direct=1` rồi so `D2C` trong `btt`.
- **Bên trong thật sự** (firmware, FTL, media): chỉ xem được bằng thiết bị phân tích bus hoặc công cụ của hãng, không bằng trace của Linux.

## Với lab iSCSI của bạn

Đường đi trên host KVM không phải đĩa cục bộ: `dm-X` → `sdb` → `scsi` → `iscsi_tcp` → TCP. Thêm vào lệnh `-e` các nhóm `-e iscsi -e net:net_dev_xmit -e tcp` (nếu kernel có), và blktrace đồng thời trên cả `dm-0` lẫn `sdb` để thấy cùng request ở hai tầng. Phần đầu tiên nên làm trên đĩa cục bộ trước để có chuẩn so sánh.

Nếu bạn chạy rồi dán output (`blkparse` hoặc `trace-cmd report`) lên đây, mình sẽ đọc từng dòng cùng bạn và đánh dấu mỗi dòng thuộc tầng nào trong slide.



Mình chia lab thành 7 bước, mỗi bước có **mục tiêu, lệnh, thứ bạn phải thấy, và việc làm sau đó**. Mình chưa chạy được trên máy bạn nên phần "thứ bạn phải thấy" là theo kiến trúc, không phải output thật. Hãy so với output của bạn.

**Mình sửa một lỗi ở lệnh Bước 1 cũ.** Mình đã để `-F` trong lệnh `trace-cmd record -e ...`. Cờ `-F` lọc sự kiện theo PID của tiến trình, trong khi sự kiện hoàn thành (`block_rq_complete`, ngắt) xảy ra ở ngữ cảnh ngắt, không mang PID của `dw`, nên sẽ bị mất. Bản dưới đây bỏ `-F` ở bước thu sự kiện và lọc lúc đọc báo cáo.

## Bước 0: Cài công cụ và kiểm tra môi trường

Mục tiêu: có đủ tool, và biết ổ nào chứa `/storage`. Lệnh cho Ubuntu 22.04 (máy lab của bạn):

```bash
sudo apt update
sudo apt install -y build-essential trace-cmd blktrace strace \
     linux-tools-common linux-tools-$(uname -r) bpftrace fio \
     e2fsprogs sysstat sg3-utils smartmontools nvme-cli hdparm
```

Nếu `apt` lỗi mạng, kiểm tra proxy bằng `env | grep proxy` như bạn đã gặp ở lab trước. Kiểm tra môi trường:

```bash
uname -r                                  # ghi lại version kernel
mount | grep -E 'tracefs|debugfs'         # cần có tracefs
sudo ls /sys/kernel/tracing | head        # xem tracefs dùng được không
findmnt -T /storage                       # /storage nằm trên thiết bị nào, loại fs gì
lsblk -d -o NAME,ROTA,TRAN,MODEL,SIZE     # loại ổ: ROTA=0 là SSD, TRAN=sata/sas/nvme
cat /sys/block/sda/queue/{scheduler,nr_requests,max_sectors_kb,write_cache}
cat /sys/block/sda/device/queue_depth
```

Việc làm sau: ghi vào sổ **loại ổ, loại filesystem, scheduler, write_cache**, vì chúng quyết định bạn sẽ thấy gì ở các bước sau. Lưu ý: filesystem phải hỗ trợ `O_DIRECT` (ext4/xfs đều được, `tmpfs` thì không).

## Bước 1: Chuẩn bị chương trình và file test

Mục tiêu: có chương trình ghi **đúng một lệnh** và file test sạch.

Tạo `dw.c` (bản đã bỏ `getchar`):

```c
// gcc -O2 dw.c -o dw
#define _GNU_SOURCE
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main(int argc, char **argv) {
    int flags = O_WRONLY;
    if (!(argc > 2 && !strcmp(argv[2], "buffered"))) flags |= O_DIRECT;
    int fd = open(argv[1], flags);        // file phải tồn tại trước
    if (fd < 0) { perror("open"); return 1; }

    void *buf;
    if (posix_memalign(&buf, 4096, 4096)) return 1;
    memset(buf, 'A', 4096);

    ssize_t n = pwrite(fd, buf, 4096, 0); // MỘT lệnh ghi 4K
    printf("pwrite=%zd\n", n);
    fsync(fd);                            // tách riêng pha flush
    close(fd);
    return 0;
}
```

Tạo file test **đã được cấp phát và ghi sẵn**:

```bash
gcc -O2 dw.c -o dw
sudo dd if=/dev/zero of=/storage/t.img bs=1M count=16 oflag=direct conv=fsync
sync; sleep 2
filefrag -v /storage/t.img      # xem file nằm ở block nào trên đĩa
```

Vì sao phải ghi sẵn: ghi lần đầu vào một file mới khiến filesystem phải cấp phát block và ghi journal, làm trace lẫn thêm I/O không mong muốn. Ghi đè lên vùng đã có là trường hợp sạch nhất.

Kiểm tra chạy thử: `./dw /storage/t.img`. Phải in `pwrite=4096`. Nếu in lỗi `EINVAL` thì kiểm tra alignment hoặc filesystem có hỗ trợ `O_DIRECT` không.

Việc làm sau: ghi lại `physical_offset` đầu tiên từ `filefrag` (đơn vị là block 4K của filesystem). Bạn sẽ dùng nó để nhận ra đúng sector trong blkparse ở Bước 5.

## Bước 2: strace, ranh giới user ↔ kernel

Mục tiêu: xem syscall và đo độ trễ end-to-end của lệnh ghi.

```bash
strace -f -tt -T -e trace=openat,pwrite64,fsync ./dw /storage/t.img
```

Phải thấy `openat(... O_WRONLY|O_DIRECT)`, `pwrite64(3, ..., 4096, 0) = 4096 <0.00xxxx>`, `fsync(3) = 0 <...>`. Con số trong `<...>` của `pwrite64` là độ trễ của lệnh ghi O_DIRECT (khi `pwrite64` trả về nghĩa là thiết bị đã báo xong, không còn nằm trong page cache).

Việc làm sau: chạy 5 lần, ghi lại dải thời gian. Strace làm chậm chương trình, nên coi đây là số tham khảo.

## Bước 3: trace-cmd, toàn bộ sự kiện từ syscall đến thiết bị

Mục tiêu: thấy chuỗi sự kiện theo thứ tự thời gian, đúng một lệnh ghi.

Xem trước tracepoint nào có trên kernel của bạn:

```bash
trace-cmd list -e | grep -E '^(block|scsi|nvme|ext4|xfs|iomap|writeback|libata)' | head -80
```

Thu sự kiện (không dùng `-F`):

```bash
cd /tmp
sudo trace-cmd record \
  -e syscalls:sys_enter_pwrite64 -e syscalls:sys_exit_pwrite64 \
  -e block -e scsi -e writeback \
  -e irq:irq_handler_entry -e irq:irq_handler_exit -e irq:softirq_entry \
  ./dw /storage/t.img
```

Thêm `-e nvme` nếu ổ là NVMe, `-e libata` nếu SATA, `-e ext4` hoặc `-e xfs` tuỳ filesystem, và `-e iomap` nếu có. Nếu một tên sự kiện không tồn tại, trace-cmd báo lỗi, bạn bỏ nó đi.

Đọc kết quả:

```bash
trace-cmd report > ev.txt
grep -n 'sys_enter_pwrite64\|sys_exit_pwrite64' ev.txt   # tìm cửa sổ thời gian
```

Mở `ev.txt` và đọc các dòng **nằm giữa** `sys_enter_pwrite64` của `dw` và `sys_exit_pwrite64`. Vì ghi O_DIRECT chặn tới khi xong, toàn bộ chuỗi Q → G → D → ngắt → C nằm trong cửa sổ này. Bạn cần thấy theo thứ tự: `block_bio_queue`, `block_getrq`, `block_rq_issue` (kèm `scsi_dispatch_cmd_start`/`nvme_setup_cmd` nếu kernel có tracepoint đó), `irq_handler_entry`, `block_rq_complete`.

Việc làm sau: chép các dòng quan trọng vào bảng ghi chú (mẫu ở Bước 6) và tính chênh lệch thời gian các mốc.

Nếu cùng lúc có ghi từ tiến trình khác (journal, log hệ thống), bạn sẽ thấy các sự kiện xen lẫn. Dùng `ev.txt` để nhận ra dòng của `dw` qua tên tiến trình ở đầu dòng, và các sự kiện hoàn thành qua thời điểm.

## Bước 4: function_graph, chuỗi hàm bên trong kernel

Mục tiêu: thấy `write` đi qua những hàm nào trong VFS và filesystem trước khi tới `submit_bio`.

Tìm tên hàm syscall trên kernel của bạn:

```bash
sudo trace-cmd list -f | grep -E 'sys_pwrite64$'
```

Thu chuỗi hàm:

```bash
sudo trace-cmd record -p function_graph --max-graph-depth 12 \
     -g __x64_sys_pwrite64 -F ./dw /storage/t.img
trace-cmd report > fg.txt
```

Ở bước này dùng `-F` là đúng vì chỉ muốn chuỗi hàm trong ngữ cảnh của tiến trình `dw`. Nếu `--max-graph-depth` không được nhận, dùng bản `trace-cmd` mới hơn hoặc bỏ tuỳ chọn đó rồi lọc bằng `less`/`grep`.

Cần tìm trong `fg.txt` theo thứ tự: `ksys_pwrite64` → `vfs_write` hoặc `vfs_iter_write` → hàm ghi của filesystem (ext4: `ext4_file_write_iter`, nhánh `ext4_dio_write_*`; xfs: `xfs_file_write_iter`, nhánh `xfs_file_dio_write_*`; tên thay đổi theo kernel) → `iomap_dio_rw` / `__iomap_dio_rw` → `submit_bio` → `blk_mq_submit_bio` → `queue_rq` của driver. Sau đó bạn sẽ thấy `schedule()` (tiến trình ngủ chờ), rồi hàm tiếp tục khi được đánh thức. Hàm bị inline sẽ không hiện, và độ sâu nhỏ sẽ cắt mất nhánh dưới.

Việc làm sau: so sánh với lần chạy ở chế độ buffered (Bước 6). Ở buffered bạn sẽ thấy `generic_perform_write`, `copy_from_user`/`copy_page_from_iter` (copy vào page cache) và **không** thấy `submit_bio`.

## Bước 5: blktrace/blkparse/btt, góc nhìn block layer

Mục tiêu: thấy vòng đời bio/request và độ trễ từng chặng.

```bash
cd /tmp
sudo blktrace -d /dev/sda -w 6 -o bt &    # theo dõi cả đĩa 6 giây
sleep 2
./dw /storage/t.img
wait
blkparse -i bt -d bt.bin > bp.txt
btt -i bt.bin > btt.txt
```

Nếu `/storage` nằm trên thiết bị khác `sda` (LVM, dm, đĩa khác), thay bằng thiết bị tương ứng, kiểm tra bằng `findmnt` ở Bước 0. Với dm/LVM, trace cả `dm-X` lẫn đĩa thật bên dưới.

Cách nhận dòng của bạn: blktrace ghi cả đĩa nên có nhiễu. Tính sector kỳ vọng:

```bash
cat /sys/class/block/sda5/start       # sector bắt đầu của partition (thay sda5 bằng partition của /storage)
# sector_trên_đĩa ≈ start + physical_offset(filefrag) * 8
```

Rồi `grep` số sector đó trong `bp.txt`, hoặc `grep '\[dw\]' bp.txt` cho các dòng `Q`, `G`, `I`, `D` (hoàn thành `C` có thể không mang tên `dw`).

Đọc dòng blkparse: `maj,min cpu seq time pid ACTION RWBS sector + sectors [comm]`. Một ghi 4K cho `... + 8`. Chuỗi mong đợi: `Q` → `G` → (`I`) → `D` → `C`. RWBS chứa `W` (write); có `S` (sync) hoặc `FUA` tuỳ cấu hình. `btt.txt` cho `Q2G`, `G2I`, `I2D`, `D2C` và `Q2C`.

Việc làm sau: ghi `D2C` (thời gian thiết bị) và `Q2D` (thời gian phần mềm trước khi đẩy xuống) vào bảng. Với một ổ SSD cục bộ, `D2C` thường chiếm gần hết `Q2C`; nếu `Q2D` lớn thì hàng đợi hoặc scheduler đang chiếm thời gian.

## Bước 6: So sánh O_DIRECT với buffered

Mục tiêu: nhìn thấy sự khác biệt mà ô Page Cache trên slide thể hiện.

```bash
cd /tmp
sudo trace-cmd record -e syscalls:sys_enter_pwrite64 -e syscalls:sys_exit_pwrite64 \
  -e block -e writeback \
  sh -c './dw /storage/t.img buffered; sleep 1; sync'
trace-cmd report > buf.txt
```

Quan sát: `pwrite64` trả về rất nhanh và **không có `block_*` nằm trong cửa sổ** của nó. Sự kiện block xuất hiện **sau** đó, tên tiến trình khác (kworker flush) hoặc lúc `fsync`/`sync`, kèm sự kiện `writeback:*`. Đó là bằng chứng dữ liệu nằm trong page cache trước, xuống đĩa sau.

Bảng ghi chú nên điền (mỗi hàng lấy từ output thật của bạn):

|Mốc|O_DIRECT|Buffered|
|---|---|---|
|Thời gian `pwrite64` (strace)|||
|Có `block_bio_queue` trong cửa sổ syscall?|có|không|
|Có `writeback:*`?|không|có|
|Tiến trình phát sinh bio|`dw`|kworker/flush|
|Nhánh hàm (function_graph)|`iomap_dio_rw`…|`generic_perform_write`|
|`Q2D`, `D2C`|||

## Bước 7: Đi xuống tận cùng, xem lệnh gửi cho ổ

Mục tiêu: xác định lệnh thật mà driver gửi và trạng thái cache của ổ.

```bash
# lệnh SCSI/ATA/NVMe cụ thể: lọc từ ev.txt của Bước 3
grep -E 'scsi_dispatch_cmd_start|nvme_setup_cmd|ata_' ev.txt
# thông tin ổ
sudo smartctl -a /dev/sda
sudo hdparm -W /dev/sda                    # write cache của ổ (SATA)
sudo nvme get-feature /dev/nvme0 -f 6      # volatile write cache (nếu là NVMe)
cat /sys/block/sda/queue/write_cache
```

Dòng `scsi_dispatch_cmd_start` hiển thị CDB. Opcode `2a` là WRITE(10), `8a` là WRITE(16), `35` là SYNCHRONIZE CACHE (lệnh `fsync` sinh ra). Việc làm sau: viết một câu trả lời cho mỗi câu hỏi: dữ liệu dừng ở cache nào trước khi xuống media, và `fsync` có tạo thêm lệnh flush không (đối chiếu `FLUSH`/`FUA` trong RWBS và lệnh `35`).

## Lỗi hay gặp

|Triệu chứng|Nguyên nhân thường gặp|Cách xử lý|
|---|---|---|
|`open: Invalid argument`|Filesystem không hỗ trợ O_DIRECT (tmpfs) hoặc lỗi alignment|Dùng ext4/xfs, kiểm tra `posix_memalign(…,4096,…)`|
|`trace-cmd` báo không tìm thấy event|Kernel không có tracepoint đó|`trace-cmd list -e`, bỏ event|
|`function_graph` không có|Kernel chưa bật tracer|`cat /sys/kernel/tracing/available_tracers`|
|Không thấy `block_rq_complete`|Dùng `-F`|Bỏ `-F` ở bước thu sự kiện|
|blkparse quá nhiều dòng|Nhiễu từ tiến trình khác|Tắt dịch vụ nền, grep theo sector|
|Có thêm ghi journal|File chưa được ghi sẵn|Làm lại Bước 1|
|`Permission denied`|Cần root để dùng tracefs/blktrace|Dùng `sudo`|

## Sản phẩm nộp sau lab

1. `ev.txt`, `fg.txt`, `bp.txt`, `btt.txt` từ O_DIRECT và `buf.txt` từ buffered.
2. Bảng so sánh ở Bước 6.
3. Sơ đồ một dòng của đường đi O_DIRECT trên máy bạn, chú thích từng chặng bằng timestamp thật.
4. Trả lời: bao nhiêu copy payload (đối chiếu với `fg.txt`: có `copy_from_user` hay không), thời gian phần mềm so với thiết bị.

Bạn chạy xong bước nào cứ dán output (đoạn quanh `sys_enter_pwrite64` hoặc vài chục dòng `blkparse`), mình sẽ đọc từng dòng cùng bạn và chỉ ra dòng nào thuộc tầng nào trong slide. Nếu muốn, mình cũng có thể gom toàn bộ hướng dẫn này thành một file markdown để bạn mang theo khi làm lab.