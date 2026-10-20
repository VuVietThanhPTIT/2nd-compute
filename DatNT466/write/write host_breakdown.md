# TÀI LIỆU PHÂN TÍCH CHUYÊN SÂU: LUỒNG THI GIAO I/O TRONG LINUX KERNEL

---

## CHẶNG 1: KHỞI TẠO TẠI USER SPACE VÀ BÀN GIAO VÀO KERNEL (O_DIRECT RAW BLOCK I/O)

```text
+══════════════════════════════════════════════════════════════════════════════════════════════════════════════+
║           CHẶNG 1: KHỞI TẠO TẠI USER SPACE VÀ BÀN GIAO VÀO KERNEL (O_DIRECT RAW BLOCK I/O)                   ║
+══════════════════════════════════════════════════════════════════════════════════════════════════════════════+
 [ Ứng dụng User Space (fio) ]
   - posix_memalign(&buf, 4096, 4096) ──► Cấp phát Host VA căn chỉnh 4 KiB (0x7f9a1000)
   - Thiết lập System V AMD64 ABI: RAX=1, RDI=fd, RSI=0x7f9a1000, RDX=4096
        │
        ▼ Chỉ lệnh assembly SYSCALL (Opcode: 0F 05)
 [ CPU Core: Chuyển đổi đặc quyền phần cứng (Ring 3 -> Ring 0) ]
   - Lưu RIP -> RCX, RFLAGS -> R11; xóa CPL từ 3 -> 0
   - Tráo ngăn xếp: RSP = TSS.sp0 (Chuyển sang Kernel Stack của task)
   - Nạp RIP = MSR IA32_LSTAR (Nhảy vào entry_SYSCALL_64)
        │
        ▼ Bỏ qua Page Cache & Filesystem (Gọi thẳng def_blk_fops -> blkdev_write_iter)
 [ Software Page Table Walk & Ghim trang (pin_user_pages / GUP-Fast) ]
   - Kernel duyệt cây phân trang phần mềm: mm->pgd -> P4D -> PUD -> PMD -> PTE
   - Dịch Host VA (0x7f9a1000) -> Host PA (0x1a2b3000) & lấy struct page*
   - Khóa cứng vật lý: atomic_inc(&page->_pincount) (FOLL_PIN: Chống migrate/swap)
        │
        ▼ Cấp phát cấu trúc mô tả từ Mempool (fs_bio_set)
 [ Đóng gói Block I/O: struct bio & bio_vec ]
   - bio_vec[0] = { .bv_page = page, .bv_len = 4096, .bv_offset = 0 }
   - bio->bi_iter.bi_sector = file_pos >> 9
   - bio->bi_bdev = block_device(/dev/mapper/mpath0)
        │
        ▼ submit_bio(bio)
 ──► BÀN GIAO SANG CHẶNG 2: LINUX BLOCK LAYER (blk-mq)
```

---

### Bước 1.1: Khởi tạo và Căn chỉnh Bộ đệm tại User Space

#### 1.1.1. Mã nguồn chuẩn xác cấp phát bộ đệm căn chỉnh

Việc khai báo `char buffer[4096]` trên Stack hoặc gọi `malloc(4096)` trên Heap không cam kết căn chỉnh 4 KiB mà chỉ căn chỉnh theo 8 hoặc 16 bytes. Đối với `O_DIRECT`, đoạn mã bắt buộc phải dùng `posix_memalign()` hoặc `aligned_alloc()`:

```c
#define _GNU_SOURCE
#include <fcntl.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>

void *buffer = NULL;
size_t alignment = 4096; // Bội số kích thước sector logic
size_t size = 4096;

// Cấp phát 4096 bytes với địa chỉ bắt đầu chia hết cho 4096
int ret = posix_memalign(&buffer, alignment, size);
if (ret != 0) {
    /* Xử lý lỗi thiếu bộ nhớ ENOMEM hoặc sai alignment EINVAL */
}

memset(buffer, 'A', size); // Ghi dữ liệu 0x41 vào bộ đệm

// Mở trực tiếp khối đĩa thô với cờ O_DIRECT
int fd = open("/dev/mapper/mpath0", O_DIRECT | O_RDWR);
ssize_t bytes_written = write(fd, buffer, size);
```

#### 1.1.2. Bản chất toán học và Hiệu ứng "Tràn qua 2 trang vật lý" (Page Boundary Crossing)

Địa chỉ vùng nhớ $A$ được gọi là căn chỉnh theo kích thước trang $P = 4096 \text{ bytes}$ khi và chỉ khi:

$$A \pmod P = 0 \quad \Longleftrightarrow \quad A \ \& \ (P - 1) = 0$$

Điều này bảo đảm 12 bit nhị phân thấp nhất của địa chỉ ảo luôn bằng 0 (ví dụ: `0x7f9a1000`).

**Trường hợp 1: Căn chỉnh chuẩn 4 KiB (Aligned - 0x7f9a1000)**

```text
Địa chỉ ảo:     0x7f9a1000 ──────────────────────────────── 0x7f9a1FFF (4096 bytes)
                    │
                    ▼ (Bảng phân trang ánh xạ 1-1)
Khung trang RAM: [             PAGE FRAME VẬT LÝ DUY NHẤT: 0x1a2b3000              ]
                 (DMA Controller chỉ cần 1 descriptor duy nhất để đọc toàn bộ khối)
```

**Trường hợp 2: Lệch biên (Unaligned - ví dụ bắt đầu tại 0x7f9a1008)**

```text
Địa chỉ ảo:        0x7f9a1008 ───────────────────────────── 0x7f9a2007 (4096 bytes)
                       │                                        │
                       ▼                                        ▼
Khung trang RAM: [ Trang vật lý A: 4088 bytes ] ───(Phân mảnh)──► [ Trang vật lý B: 8 bytes ]
                 (Địa chỉ PA: 0x1a2b3008)                         (Địa chỉ PA: 0x9f8c0000)
```

**Tại sao lệch biên lại phá vỡ DMA?**  
Nếu bị lệch dù chỉ 1 byte, một khối ghi logic 4096 bytes sẽ bị xé thành 2 mảnh nằm trên 2 Page Frame vật lý hoàn toàn tách biệt trong DRAM. Phần cứng điều khiển lưu trữ (Controller/DMA Engine) giao tiếp qua các giao thức khối (SCSI/NVMe) dựa trên các sector logic nguyên vẹn; nó không thể thực hiện một lệnh ghi I/O đơn lẻ khi dữ liệu bị cắt vụn thành các đoạn kích thước lẻ (như 4088 bytes và 8 bytes) qua hai dải bus vật lý khác nhau mà không có cấu trúc phân tán phức tạp.

#### 1.1.3. Ba điều kiện biên kiểm tra của O_DIRECT tại Kernel

Khi cờ `O_DIRECT` được bật, VFS và Block Layer kiểm tra 3 bất biến toán học dựa trên `logical_block_size` (thường là 512 hoặc 4096 bytes) của thiết bị:

1. **Địa chỉ con trỏ (`buf`):**  
   $$(\text{uintptr\_t})\text{buf} \ \& \ (\text{align} - 1) = 0$$
2. **Độ dài truyền tải (`count`):**  
   $$\text{count} \ \& \ (\text{align} - 1) = 0$$
3. **Vị trí con trỏ tệp (`offset` / `file->f_pos`):**  
   $$\text{offset} \ \& \ (\text{align} - 1) = 0$$

Nếu bất kỳ điều kiện nào không thỏa mãn, hệ thống trả về ngay mã lỗi `-EINVAL`. Nếu kernel hỗ trợ Bounce Buffer (chế độ tương thích cũ), nó buộc phải cấp phát một trang đệm phụ và gọi CPU `memcpy` payload sang đó để căn chỉnh lại, làm mất hoàn toàn lợi thế Zero-Copy của `O_DIRECT`.

---

### Bước 1.2: Thiết lập ABI System V AMD64 và Thực thi Chỉ lệnh SYSCALL

#### 1.2.1. Nạp thanh ghi theo chuẩn System V AMD64 ABI

Trước khi phát lệnh chuyển quyền, wrapper `write()` trong thư viện C (`glibc`) nạp các thanh ghi đa năng của CPU:

| Thanh ghi | Giá trị nạp | Vai trò kỹ thuật |
| :--- | :--- | :--- |
| **RAX** | `1` | Số hiệu ngắt hệ thống (`__NR_write` trên kiến trúc Linux x86-64). |
| **RDI** | `fd` (ví dụ: `3`) | Đối số thứ nhất: File descriptor trỏ tới node `/dev/mapper/mpath0`. |
| **RSI** | `0x7f9a1000` | Đối số thứ hai: Con trỏ 64-bit trỏ tới địa chỉ ảo Host VA của buffer. |
| **RDX** | `4096` | Đối số thứ ba: Kích thước byte yêu cầu ghi (`count`). |

> **Điểm mấu chốt:** Thanh ghi `RSI` chỉ chứa con trỏ địa chỉ ảo 8-byte. Toàn bộ 4096 bytes ký tự `0x41` (payload) vẫn nằm bất động trên các chip DRAM vật lý và trong bộ nhớ đệm CPU L1/L2/L3 Cache.

#### 1.2.2. Vi phẫu chỉ lệnh phần cứng SYSCALL (Opcode: 0F 05)

Khi CPU x86-64 thực thi opcode `0F 05`, mạch logic giải mã lệnh trong CPU thực hiện chuỗi hành động nguyên tử ở cấp phần cứng:

1. **Lưu vết thực thi User Space:**
   * Thanh ghi `RIP` (con trỏ lệnh tiếp theo ở User Space) được phần cứng sao chép tự động vào thanh ghi `RCX`.
   * Thanh ghi `RFLAGS` (các cờ trạng thái CPU) được sao chép tự động vào thanh ghi `R11`.
2. **Hạ mức đặc quyền phần cứng (Ring 3 $\to$ Ring 0):**
   * CPU nạp các bộ chọn phân đoạn (Segment Selectors) từ Model-Specific Register `IA32_STAR` (MSR `0xC0000081`).
   * Cờ đặc quyền hiện tại CPL (Current Privilege Level) trong thanh ghi `CS` bị xóa từ 3 về 0.
3. **Mặt nạ hóa cờ trạng thái:**
   * CPU đọc giá trị từ `IA32_FMASK` (MSR `0xC0000084`) và xóa các bit tương ứng trong `RFLAGS` (vô hiệu hóa tạm thời ngắt ngoài IF, cờ bẫy đơn bước TF).
4. **Tráo đổi ngăn xếp (Stack Switch):**
   * Để ngăn chặn việc mã kernel thực thi trên ngăn xếp người dùng (vốn có thể bị tràn hoặc chứa mã độc), CPU không dùng tiếp `RSP` hiện tại.
   * Phần cứng trích xuất địa chỉ đỉnh ngăn xếp hạt nhân của luồng từ cấu trúc Task State Segment: $\text{RSP} = \text{TSS.sp0}$.
5. **Bẻ hướng luồng điều khiển vào Kernel Entry:**
   * CPU nạp giá trị từ thanh ghi chuyên biệt `IA32_LSTAR` (MSR `0xC0000082`) vào thanh ghi `RIP`.
   * Địa chỉ này trỏ thẳng tới điểm đón đầu: `entry_SYSCALL_64` trong mã nguồn Linux Kernel.

```text
Phân biệt bản chất:
Đây thuần túy là Privilege Switch (Đổi mức đặc quyền phần cứng trên cùng một CPU Core).
- KHÔNG PHẢI Context Switch: Không thay đổi tiến trình đang chạy.
- Thanh ghi CR3 không đổi (không tráo không gian địa chỉ ảo).
- Con trỏ current vẫn trỏ nguyên vẹn vào struct task_struct của tiến trình fio.
```

---

### Bước 1.3: Vượt qua VFS Block Device & Ghim Khung trang RAM (pin_user_pages)

#### 1.3.1. Sự vắng bóng của Filesystem (FS)

Do đường dẫn là một khối đĩa thô (`/dev/mapper/mpath0`), VFS nhận diện đây là file đặc biệt kiểu khối (`S_ISBLK`):

* VFS không gọi qua các driver `ext4`, `XFS` hay `Btrfs`.
* Bảng hàm thao tác được liên kết trực tiếp là `def_blk_fops`.
* Chuỗi gọi hàm hạt nhân:

$$\text{ksys\_write}() \longrightarrow \text{vfs\_write}() \longrightarrow \text{blkdev\_write\_iter}()$$

Tại đây, kernel nhận diện cờ `IOCB_DIRECT` (tương ứng với `O_DIRECT`), rẽ nhánh trực tiếp vào hàm thực thi I/O khối trực tiếp mà không chạm vào cấu trúc Page Cache (`struct address_space` không cấp phát trang đệm).

#### 1.3.2. Đóng gói Bộ lặp I/O (struct iov_iter)

Kernel đóng gói buffer người dùng thành cấu trúc:

```c
struct iov_iter iter;
// iter.iter_type = ITER_UBUF (hoặc ITER_IOVEC)
// iter.ubuf = 0x7f9a1000
// iter.count = 4096
```

#### 1.3.3. Duyệt cây phân trang bằng phần mềm (Software Page Table Walk qua GUP-Fast)

Kernel thực thi hàm `iov_iter_get_pages2()` $\to$ gọi cơ chế nội bộ GUP-Fast (`pin_user_pages_fast()`).

**Điểm cần hiệu chỉnh chính xác:** Ở bước này, không phải MMU phần cứng tự động walk bảng trang, mà là mã nguồn C của Kernel tự duyệt (Software Walk) qua cấu trúc bảng phân trang của tiến trình (`current->mm`) được lưu trong RAM:

```text
current->mm->pgd (Lấy từ địa chỉ gốc trong CR3)
       │
       ▼ pgd_offset()
     [ PGD Entry ] (Page Global Directory)
       │
       ▼ p4d_offset()
     [ P4D Entry ] (Page Level 4 Directory)
       │
       ▼ pud_offset()
     [ PUD Entry ] (Page Upper Directory)
       │
       ▼ pmd_offset()
     [ PMD Entry ] (Page Middle Directory)
       │
       ▼ pte_offset_map()
     [ PTE Entry ] (Page Table Entry)
       │
       ├─► Kiểm tra cờ bảo mật: (PTE_PRESENT == 1) && (PTE_WRITE == 1)
       └─► Trích xuất PFN (Page Frame Number): bit [12:51] của PTE
```

* **Duyệt con trỏ:** Kernel lần lượt tính toán các offset dựa trên 48 bit của địa chỉ ảo `0x7f9a1000`, đọc nội dung từng bảng phân trang để tìm tới mục bảng trang cấp cuối (PTE).
* **Kiểm tra tính hợp lệ của trang:**
  * Nếu bit `Present = 1` và `Writable = 1`: Kernel trích xuất giá trị PFN (Page Frame Number) từ PTE.
  * Nếu bit `Present = 0` (Demand Paging - trang ảo đã khai báo nhưng chưa có DRAM vật lý): GUP-Fast thất bại. Kernel chuyển sang Slow GUP và chủ động kích hoạt hàm `handle_mm_fault()`. Lệnh này ép hệ thống cấp phát một Page Frame 4 KiB trống từ Buddy Allocator, ghi số 0 vào trang, nạp PFN vào PTE và bật cờ `Present = 1`.
* **Tính toán Địa chỉ Vật lý Máy chủ (Host Physical Address - HPA):**

$$\text{Host PA} = (\text{PFN} \ll 12) \ \vert \ (\text{Host VA} \ \& \ 0\text{xFFF}) = 0\text{x1a2b3000}$$

* **Lấy đối tượng quản lý trang:**
  ```c
  struct page *page = pfn_to_page(PFN);
  ```

#### 1.3.4. Cơ chế Ghim cứng Khung trang (FOLL_PIN)

Để đảm bảo an toàn tuyệt đối cho phần cứng DMA ở các chặng sau:
* Kernel không chỉ gọi `get_page()` thông thường (chỉ tăng `_refcount`), mà kích hoạt cờ `FOLL_PIN`.
* Thao tác nguyên tử trên CPU:

$$\text{atomic\_add}(1, \&\text{page}\to\_pincount)$$

**Ý nghĩa sinh tử của việc Ghim trang (Page Pinning):**  
Trong điều kiện hệ thống chịu tải cao, các cơ chế ngầm của Kernel như Kswapd (Swap bộ nhớ), Memory Compaction (Gom cụm chống phân mảnh), hoặc NUMA Page Balancing liên tục di dời các trang RAM vật lý sang vị trí khác và sửa lại bảng trang PTE. Nếu một trang bị di dời trong khi card mạng hoặc controller SSD đang thực hiện DMA, phần cứng sẽ ghi/đọc nhầm vào dữ liệu của tiến trình khác. Cờ `_pincount > 0` cấm tuyệt đối kernel di dời hoặc thu hồi khung trang này cho đến khi I/O hoàn tất.

---

### Bước 1.4: Đóng gói Cấu trúc struct bio

`struct bio` là đơn vị nguyên tử của Block Layer trong Linux, chứa mọi thông tin cần thiết để điều phối một yêu cầu I/O mức khối.

```text
+──────────────────────────────────────────────────────────────────────────+
| struct bio                                                               |
|   bi_bdev        ──► Trỏ tới block_device của /dev/mapper/mpath0         |
|   bi_opf         ──► REQ_OP_WRITE | REQ_SYNC                             |
|   bi_iter                                                                |
|     .bi_sector   ──► 0x00000000 (Vị trí LBA bắt đầu: file_pos >> 9)      |
|     .bi_size     ──► 4096 (Tổng số byte của bio)                         |
|   bi_vcnt        ──► 1 (Số lượng phần tử phân tán bio_vec)               |
|   bi_io_vec      ──► [ Mảng bio_vec ]                                    |
+─────────────────────────────────────┬────────────────────────────────────+
                                      │
                                      ▼
                     +──────────────────────────────────+
                     | struct bio_vec [0]               |
                     |   bv_page   ──► struct page*     | ──► Trỏ tới Host PA
                     |   bv_len    ──► 4096             |     (0x1a2b3000)
                     |   bv_offset ──► 0                |
                     +──────────────────────────────────+
```

#### 1.4.1. Cấp phát an toàn từ Mempool (bio_set)

Để chống nguy cơ Deadlock khi hệ thống cạn kiệt bộ nhớ (OOM): Kernel không cấp phát `struct bio` bằng hàm `kmalloc()` thông thường.

Thay vào đó, nó trích xuất từ một vùng dự trữ cấu trúc dựng sẵn: `bio_set` (cụ thể là `fs_bio_set` qua hàm `bio_alloc_bioset()`). Mempool bảo đảm luôn luôn có sẵn cấu trúc trống dự phòng phục vụ I/O để xả bộ nhớ.

#### 1.4.2. Khởi tạo mảng phân tán struct bio_vec

Kernel gắn thông tin khung trang vừa ghim vào mảng phần tử bộ nhớ phân tán (`bio_vec`):
* `bv_page`: Con trỏ `struct page*` trỏ tới khung trang vật lý `0x1a2b3000`.
* `bv_len = 4096`: Toàn bộ kích thước 4 KiB của trang.
* `bv_offset = 0`: Bắt đầu từ byte thứ 0 của khung trang (nhờ bộ đệm đã căn chỉnh hoàn hảo ở Bước 1.1).

#### 1.4.3. Quy đổi Byte Offset sang LBA Sector

Block Layer của Linux chuẩn hóa mọi đơn vị lưu trữ theo Kernel Sector (luôn là 512 bytes):

$$\text{bi\_sector} = \text{file\_pos} \gg \text{SECTOR\_SHIFT} = 0 \gg 9 = 0$$

* `bio->bi_iter.bi_sector = 0`: Điểm bắt đầu ghi trên thiết bị logic.
* `bio->bi_iter.bi_size = 4096`: Dung lượng toàn bộ yêu cầu I/O.

#### 1.4.4. Gán Thiết bị và Phát lệnh

* `bio_set_dev(bio, bdev)`: Thiết lập con trỏ đích trỏ tới cấu trúc `struct block_device` đại diện cho `/dev/mapper/mpath0`.
* `bio->bi_opf = REQ_OP_WRITE | REQ_SYNC`: Thiết lập cờ thao tác là ghi đồng bộ.

Kết thúc quá trình đóng gói, kernel gọi hàm `submit_bio(bio)`. Gói mô tả bio chính thức rời khỏi tầng khởi tạo, tiến vào hàng đợi đa nhân `blk-mq` và bộ định tuyến đa đường `dm-multipath` của Chặng 2.

---

## CHẶNG 2: LINUX BLOCK LAYER (blk-mq) VÀ ĐỊNH TUYẾN MULTIPATH (dm-multipath)

```text
+══════════════════════════════════════════════════════════════════════════════════════════════════════════════+
║           CHẶNG 2: LINUX BLOCK LAYER (blk-mq) VÀ ĐỊNH TUYẾN MULTIPATH (dm-multipath)                         ║
+══════════════════════════════════════════════════════════════════════════════════════════════════════════════+
 [ Tiếp nhận từ Chặng 1: struct bio ]
   - bio->bi_bdev ──► /dev/mapper/mpath0 (Request-based DM Virtual Block Device)
   - Payload: Giữ nguyên con trỏ trỏ tới Host PA (0x1a2b3000), 0 lần memcpy
        │
        ▼ submit_bio(bio) ──► blk_mq_submit_bio()
 [ Kiến trúc Hàng đợi Đa nhân blk-mq trên DM ]
   - Cấp phát struct request gốc từ Mempool bằng Lockless SBitmap Tag
   - Ghép bio vào request (bio merging bị bỏ qua do I/O 4 KiB đơn lẻ)
   - Đưa vào Software Staging Queue (struct blk_mq_ctx) gắn với CPU Core cục bộ
        │
        ▼ Điều phối vào Hardware Dispatch Queue (struct blk_mq_hw_ctx)
 [ dm-multipath Target Driver: multipath_clone_and_map() ]
   - Tra cứu Priority Group (PG) đang Active
   - Thực thi thuật toán Path Selector (service-time / round-robin)
   - Quyết định định tuyến: Chọn Path 1 (/dev/sda) thay vì Path 2 (/dev/sdb)
        │
        ▼ Cơ chế Nhân bản Yêu cầu (Request Cloning)
 [ Khởi tạo Cloned Request (struct request *clone) ]
   - Cấp phát wrapper metadata: struct dm_rq_target_io (tio)
   - Clone mảng con trỏ bio_vec (bv_page ──► Host PA 0x1a2b3000)
   - Thiết lập Completion Callback: clone->end_io = dm_mq_end_io (Giữ chốt failover)
        │
        ▼ blk_insert_cloned_request()
 [ Đẩy vào Hàng đợi của Thiết bị Đích: /dev/sda ]
   - Nạp vào request_queue của driver đĩa SCSI /dev/sda
   - Sẵn sàng chuyển giao sang tầng SCSI Mid-layer
 ──► BÀN GIAO SANG CHẶNG 3: SCSI SUBSYSTEM & ĐÓNG GÓI iSCSI PDU
```

---

### Bước 2.1: Tiếp nhận tại Block Layer & Kiến trúc Đa hàng đợi blk-mq

Khi `submit_bio(bio)` được gọi với đích đến là `/dev/mapper/mpath0`, gói mô tả bước vào hệ thống hàng đợi đa nhân blk-mq (Multi-Queue Block Layer). Thiết bị DM Multipath hiện đại vận hành ở chế độ Request-based DM (`DM_TYPE_MQ_REQUEST_BASED_DM`), biến `/dev/mapper/mpath0` thành một block device ảo sở hữu cấu trúc `request_queue` dạng `blk-mq`.

```text
CPU Cores (Phần cứng)       Software Staging Queues           Hardware Dispatch Queues
┌─────────────┐             ┌─────────────────────┐          ┌─────────────────────┐
│  CPU Core 0 │ ──────────► │ blk_mq_ctx (Core 0) │ ───┐     │                     │
└─────────────┘             └─────────────────────┘    │     │    blk_mq_hw_ctx    │
┌─────────────┐             ┌─────────────────────┐    ├───► │       (hctx 0)      │
│  CPU Core 1 │ ──────────► │ blk_mq_ctx (Core 1) │ ───┘     │                     │
└─────────────┘             └─────────────────────┘          └──────────┬──────────┘
                                                                        │
                                                                        ▼
                                                             dm_mq_ops->queue_rq()
```

#### 2.1.1. Cấu trúc Hàng đợi Phần mềm hai tầng (ctx và hctx)

Để xóa bỏ hiện tượng tranh chấp khóa vòng (Global Spinlock Contention) từng làm tê liệt hiệu năng Linux ở thời kỳ đơn hàng đợi (single-queue), blk-mq tách rời hàng đợi làm hai lớp:

* **Software Staging Queue (`struct blk_mq_ctx` - viết tắt: `ctx`):**
  * Được cấp phát trên từng CPU core vật lý (Per-CPU structure).
  * Khi luồng fio đang chạy trên CPU Core 0 phát lệnh, kernel đẩy request vào chính `blk_mq_ctx` của Core 0 mà không cần tranh chấp khóa với các core khác (Lockless Execution).
* **Hardware Dispatch Queue (`struct blk_mq_hw_ctx` - viết tắt: `hctx`):**
  * Đại diện cho các kênh gửi lệnh thực tế của thiết bị phần cứng bên dưới.
  * Số lượng `hctx` phụ thuộc vào năng lực của driver/phần cứng. Kernel ánh xạ $N$ `ctx` phần mềm vào $M$ `hctx` phần cứng thông qua bảng định tuyến CPU Affinity (`q->tag_set->map`).

#### 2.1.2. Cấp phát Tag phần cứng bằng sbitmap (Shifted Bitmap)

Để tạo ra một `struct request` chứa bio:
* Mỗi request trong blk-mq được định danh bằng một số nguyên duy nhất gọi là Tag (nằm trong dải $[0, \text{queue\_depth} - 1]$).
* Kernel không duyệt mảng tìm tag trống mà sử dụng cấu trúc dữ liệu `struct sbitmap_queue`:
  * Mảng bit nhị phân được chia nhỏ thành nhiều từ 64-bit phân tán trên bộ nhớ cache của từng CPU.
  * CPU Core 0 dùng chỉ lệnh assembly nguyên tử `lock bts` (Bit Test and Set) để quét và xí chỗ một bit 0 thành 1 trong vài nano-giây.
  * Tag nhận được dùng làm chỉ mục trực tiếp vào mảng con trỏ `q->tag_set->tags[hctx_idx]->rqs[tag]` để lấy về `struct request` trống từ pre-allocated pool. Không có bất kỳ hàm `kmalloc()` nào bị gọi tại bước này.

#### 2.1.3. Bỏ qua Bio Merging và Per-task Plug List

* **Per-task Plug List (`struct blk_plug`):** Thông thường, nhân Linux sẽ giữ lại các bio trong danh sách cục bộ của tiến trình để gom cụm (merging) nếu ứng dụng ghi nhiều khối nhỏ liên tiếp.
* **Trường hợp O_DIRECT 4 KiB:** Do ứng dụng phát một khối ghi 4 KiB độc lập, thuật toán tra cứu nhanh nhận thấy không có bio nào liền kề sector 0 trong plug list $\to$ bio được đóng gói ngay vào `struct request` mới cấp phát:
  * `req->bi_opf = REQ_OP_WRITE | REQ_SYNC`
  * `req->bio = bio; req->biotail = bio`
  * `req->__sector = 0; req->__data_len = 4096`

> **Lưu ý:** Payload vẫn nằm bất động tại Khung trang vật lý `0x1a2b3000` (Host PA); `struct request` chỉ trỏ tới `struct bio`, và `struct bio` trỏ tới `struct page*`.

---

### Bước 2.2: Định tuyến tại Device Mapper (dm-multipath)

Sau khi request được đẩy vào `hctx`, hàm xử lý hàng đợi của Device Mapper ảo được kích hoạt: `dm_mq_ops->queue_rq()` $\to$ chuyển quyền cho module `dm_multipath.ko` thông qua hàm `multipath_clone_and_map()`.

```text
                 Cấu trúc Cây Đối tượng dm-multipath
                  ┌─────────────────────────────────┐
                  │    mapped_device: mpath0        │
                  └────────────────┬────────────────┘
                                   │
                                   ▼
                  ┌─────────────────────────────────┐
                  │ struct dm_target / multipath     │
                  └────────────────┬────────────────┘
                                   │
       ┌───────────────────────────┴───────────────────────────┐
       ▼                                                       ▼
┌───────────────────────────────┐               ┌───────────────────────────────┐
│ Priority Group 1 (Active)     │               │ Priority Group 2 (Standby)    │
│ Path Selector: [service-time] │               │ Path Selector: [service-time] │
└──────────────┬────────────────┘               └───────────────────────────────┘
               │
       ┌───────┴───────────────────────────────┐
       ▼                                       ▼
┌──────────────────────────────┐        ┌──────────────────────────────┐
│ struct pgpath: Path 1        │        │ struct pgpath: Path 2        │
│ bdev ──► /dev/sda            │        │ bdev ──► /dev/sdb            │
│ Target Portal: 192.168.10.50 │        │ Target Portal: 192.168.20.50 │
└──────────────────────────────┘        └──────────────────────────────┘
```

#### 2.2.1. Quản lý Priority Group (PG) và Trạng thái ALUA

Một thiết bị Multipath cấu hình kết nối tới hệ thống lưu trữ ngoài (SAN) thông qua giao thức SCSI ALUA (Asymmetric Logical Unit Access).

Cấu trúc `struct multipath` chia các đường dẫn thành các Nhóm Ưu Tiên (Priority Groups - PG):
* **PG 1 (Active/Optimized):** Các đường dẫn truy cập trực tiếp tới Node điều khiển đang sở hữu LUN bên phía Storage Target.
* **PG 2 (Active/Non-Optimized hoặc Standby):** Các đường dẫn vòng qua Controller thứ hai của Storage Target, chỉ kích hoạt khi PG 1 chết toàn bộ đường truyền.

Kernel xác định PG 1 đang giữ cờ `current_pg` để chuyển tiếp I/O.

#### 2.2.2. Vi phẫu Thuật toán Path Selector: service-time vs round-robin

Bộ chọn đường (Path Selector) là mô-đun quyết định xem trong nhóm PG 1, đường dẫn vật lý nào (`/dev/sda` hay `/dev/sdb`) sẽ nhận I/O.

**Trường hợp A: Thuật toán Vòng tròn Cổ điển (round-robin)**
* Duyệt tuần tự: $\text{Path 1} \to \text{Path 2} \to \text{Path 1} \to \text{Path 2}$.
* Tham số `rr_min_io = 1000` (hoặc `rr_min_io_rq = 1`): Gửi $N$ yêu cầu qua Path 1, sau đó chuyển sang Path 2.
* *Hạn chế:* Không phản ánh được tình trạng trễ mạng hay nghẽn I/O cục bộ trên từng card mạng.

**Trường hợp B: Thuật toán Tải Thực tế (service-time - Khuyên dùng cho iSCSI/SAN)**  
Mô-đun `dm-service-time.ko` duy trì trạng thái động của từng đường dẫn trong cấu trúc `struct path_info`:
* `in_flight_size`: Tổng số bytes I/O đang bay trên đường truyền chưa nhận được ACK hoàn tất.
* `relative_throughput`: Hệ số băng thông của đường dẫn (mặc định trọng số = 1).

Khi request 4 KiB đi vào, thuật toán tính toán giá trị dịch vụ ước tính ($ST$) cho mọi đường dẫn còn hoạt động:

$$ST_i = \frac{\text{in\_flight\_size}_i + \text{req\_size}}{\text{relative\_throughput}_i}$$

Giả định tại thời điểm thực thi:
* **Path 1 (`/dev/sda`):** Đang xử lý 2 I/O dở dang $\to \text{in\_flight\_size}_1 = 8192 \text{ bytes}$.
  $$ST_1 = \frac{8192 + 4096}{1} = 12288$$
* **Path 2 (`/dev/sdb`):** Đang bị nghẽn gói TCP, đọng 10 I/O dở dang $\to \text{in\_flight\_size}_2 = 40960 \text{ bytes}$.
  $$ST_2 = \frac{40960 + 4096}{1} = 45056$$

Thuật toán chọn đường dẫn có $ST$ nhỏ nhất: **Path 1 (`/dev/sda`) được chọn**. Kernel tăng biến đếm `in_flight_size` của Path 1 thêm 4096 bytes bằng chỉ lệnh nguyên tử:

$$\text{atomic\_add}(4096, \&\text{pi1}\to\text{in\_flight\_size})$$

---

### Bước 2.3: Vi phẫu Cơ chế Nhân bản Yêu cầu (Request Cloning)

Device Mapper tuyệt đối không được gửi trực tiếp `struct request` gốc xuống `/dev/sda`. Thay vào đó, nó bắt buộc phải kích hoạt cơ chế Request Cloning.

```text
      YÊU CẦU GỐC (ORIGINAL REQUEST)                       YÊU CẦU NHÂN BẢN (CLONED REQUEST)
┌──────────────────────────────────────────────┐     ┌──────────────────────────────────────────────┐
│ struct request (mpath0)                      │     │ struct request (sda)                         │
│ - q: request_queue của /dev/mapper/mpath0    │     │ - q: request_queue của /dev/sda (SCSI Disk)  │
│ - bio: struct bio gốc                        │     │ - bio: struct bio nhân bản                   │
└──────────────────────┬───────────────────────┘     └──────────────────────┬───────────────────────┘
                       │                                                    │
                       │             ┌────────────────────────┐             │
                       └───────────► │ struct dm_rq_target_io │ ◄───────────┘
                                     │ (tio wrapper metadata) │
                                     └────────────────────────┘
                                                  │
                                     Cài đặt Completion Hook:
                                     clone->end_io = dm_mq_end_io
                                                  │
                                                  ▼
                                 [ bio_vec TRỎ CHUNG KHUNG TRANG ]
                                   bv_page ──► Host PA: 0x1a2b3000
                                       (Hoàn toàn Zero-Copy)
```

#### 2.3.1. Tại sao bắt buộc phải Nhân bản (Clone)?

1. **Cách ly hàng đợi (Queue Isolation):** Hàng đợi của `/dev/mapper/mpath0` là hàng đợi ảo, còn `/dev/sda` là hàng đợi phần cứng với giới hạn phần cứng khác nhau (ví dụ: `max_segments`, `max_sectors_kb`, `dma_alignment`).
2. **Cơ chế Sống còn - Chống Mất Dữ liệu khi Đứt Cáp (Failover / Retry):** Nếu cắm thẳng request gốc xuống `/dev/sda`, khi dây mạng nối tới `/dev/sda` bị rút đột ngột, driver SCSI bên dưới sẽ trả về mã lỗi phần cứng `EIO` hoặc `TIMEOUT` và hủy lệnh. Request gốc sẽ bị hủy và báo lỗi lên ứng dụng fio. Bằng cách dùng Cloned Request:
   * Chỉ có Cloned Request bị hủy.
   * Hàm callback của DM chặn mã lỗi lại, nhận diện đường dẫn chết, đánh dấu Path 1 là `FAILED`.
   * DM lấy lại Request gốc (vẫn nguyên vẹn 100%), tạo một Cloned Request thứ hai và phát lại (re-issue) ngay lập tức sang Path 2 (`/dev/sdb`). Ứng dụng phía trên chỉ cảm nhận được độ trễ tăng nhẹ mà không hề bị vỡ tiến trình I/O.

#### 2.3.2. Cấp phát Vùng nhớ Wrapper: struct dm_rq_target_io

Kernel cấp phát đối tượng `dm_rq_target_io` (gọi tắt là `tio`) từ Mempool riêng biệt của DM Target: `md->tio_pool`.

Cấu trúc `tio` lưu giữ con trỏ trỏ ngược về:
* Con trỏ `orig`: Trỏ tới `struct request` gốc của `/dev/mapper/mpath0`.
* Con trỏ `md`: Trỏ tới `struct mapped_device`.
* Thông tin đường dẫn đã chọn: `struct pgpath *pgpath` (chỉ định `/dev/sda`).

#### 2.3.3. Cơ chế Khởi tạo Cloned Request (blk_mq_alloc_request)

* **Cấp phát vỏ lệnh từ Driver đích:** Kernel gọi `blk_mq_alloc_request(sda_queue, REQ_OP_WRITE, BLK_MQ_REQ_NOWAIT)`. Lệnh này lấy một Tag phần cứng trống từ `sbitmap` của hàng đợi `/dev/sda`.
* **Sao chép cấu trúc mô tả (Metadata Copy):**
  * `clone->cmd_flags = orig->cmd_flags`
  * `clone->__sector = orig->__sector = 0`
  * `clone->__data_len = orig->__data_len = 4096`
  * `clone->rq_disk = sda_disk`
* **Sao chép Danh sách Phân tán (Bio / Bio-vec Cloning):**
  * Kernel duyệt qua các bio của request gốc, nhân bản các cấu trúc vỏ `struct bio` cho clone.
  * **Điểm mấu chốt về Zero-Copy:** Mảng `bio_vec` của request nhân bản sao chép nguyên văn con trỏ trang từ request gốc:

$$\text{clone}\to\text{bio}\to\text{bi\_io\_vec}[0].\text{bv\_page} = \text{orig}\to\text{bio}\to\text{bi\_io\_vec}[0].\text{bv\_page}$$

Toàn bộ 4096 bytes dữ liệu ký tự `0x41` vẫn nằm cố định tại địa chỉ vật lý `0x1a2b3000` (Host PA) mà không hề bị di chuyển hay tạo bản sao trong RAM.

#### 2.3.4. Cài đặt Móc chặn Phục hồi (clone->end_io)

Kernel ghi đè con trỏ hàm hoàn tất của cloned request:

```c
clone->end_io = dm_mq_end_io;
clone->end_io_data = tio;
```

Khi tầng phần cứng bên dưới xử lý xong (hoặc lỗi), thay vì báo thẳng lên ứng dụng, CPU sẽ bị bẻ hướng nhảy vào hàm `dm_mq_end_io()` của Device Mapper trước để thẩm định kết quả.

---

### Bước 2.4: Bàn giao vào Hàng đợi của Thiết bị Đích (/dev/sda)

Khâu cuối cùng của Chặng 2 là đẩy Cloned Request vừa tạo vào đúng kênh xử lý của driver đĩa SCSI đại diện cho `/dev/sda`:

1. **Thực thi hàm bàn giao:**
   * Kernel gọi `blk_insert_cloned_request(clone)`.
   * Hàm kiểm tra lại các giới hạn phần cứng của `/dev/sda` (`queue_limits` của SCSI host).
   * Request được đưa thẳng vào Hardware Dispatch Queue (`hctx`) tương ứng của thiết bị `/dev/sda`.
2. **Đánh thức Tầng Driver Bên Dưới:**
   * Hàm dispatch kích hoạt phương thức hàng đợi của driver đĩa SCSI:
     ```c
     sda_queue->mq_ops->queue_rq(hctx, bd);
     ```
   * Tại đây, hàm xử lý của tầng SCSI Disk Driver (`sd_mod`) tiếp quản: `scsi_queue_rq()`.

---

### Bảng Đối chiếu Vi phân Chặng 2: Biến đổi Cấu trúc & Không gian Bộ nhớ

| Thuộc tính phân tích | Khi mới vào Block Layer | Tại Tầng dm-multipath | Khi rời Chặng 2 sang /dev/sda |
| :--- | :--- | :--- | :--- |
| **Đối tượng quản lý chính** | `struct bio` | `struct request` (Original) | `struct request` (Cloned) |
| **Vùng nhớ cấp phát** | Mempool `fs_bio_set` | `blk_mq_tags` (Tagset của `mpath0`) | `blk_mq_tags` (Tagset của `/dev/sda`) |
| **Cấu trúc bao bọc** | Nằm độc lập | Liên kết với `struct mapped_device` | Liên kết qua `struct dm_rq_target_io` |
| **Đích đến logic** | `/dev/mapper/mpath0` | Nhóm PG 1 (`dm-service-time`) | Node thiết bị khối thô `/dev/sda` |
| **Hàng đợi điều phối** | Chưa vào hàng đợi | `blk_mq_ctx` $\to$ `blk_mq_hw_ctx` | Dispatch List của `sd_mod` (SCSI) |
| **Trạng thái Payload (4 KiB)** | Host PA `0x1a2b3000` | Host PA `0x1a2b3000` | Host PA `0x1a2b3000` (Giữ nguyên) |
| **Số lần CPU Payload Memcpy** | 0 lần | 0 lần | 0 lần |
| **Chi phí tính toán chính** | Quét bit `sbitmap` tìm Tag | Thuật toán $ST = \frac{\text{Load}}{\text{Weight}}$ | Tạo Cloned `bio-vec` & gán con trỏ `end_io` |